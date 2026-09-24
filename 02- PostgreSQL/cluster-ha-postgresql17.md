
**Autor:** Pablo Mongar (Técnico ASIR)  
**Proyecto:** Cluster alta disponibilidad PostgreSQL17 
**Contacto:** pmongber@gmail.com  
**LinkedIn:** linkedin.com/in/pmongar

# Clúster de Alta Disponibilidad de PostgreSQL - Guía de Operaciones (Runbook)

Esta guía documenta cómo desplegar un clúster de Alta Disponibilidad (HA) de PostgreSQL listo para producción utilizando **etcd**, **Patroni**, **HAProxy**, **Keepalived** y **PostgreSQL 17** en tres nodos.

## Requisitos previos

- Sistema operativo Linux basado en Ubuntu/Debian en todos los nodos.
- Acceso root o sudo.
- Direcciones IP estáticas para cada servidor:
- Nodo 1: `<ip_nodo_1>`
- Nodo 2: `<ip_nodo_2>`
- Nodo 3: `<ip_nodo_3>`

 ## Configuración previa y personalización
 Antes de ejecutar los comandos de esta guía, **debes reemplazar las siguientes variables genéricas** por los valores reales de tu infraestructura:
 
 * **Identificadores de nodos:**
   * `<nombre_nodo_1>` ➔ Nombre de tu primer servidor (ej. `db-node-01`)
   * `<nombre_nodo_2>` ➔ Nombre de tu segundo servidor (ej. `db-node-02`)
   * `<nombre_nodo_3>` ➔ Nombre de tu tercer servidor (ej. `db-node-03`)
 
 * **Direcciones IP:**
   * `<ip_nodo_1>` ➔ IP estática del Nodo 1 (ej. `192.168.1.10`)
   * `<ip_nodo_2>` ➔ IP estática del Nodo 2 (ej. `192.168.1.11`)
   * `<ip_nodo_3>` ➔ IP estática del Nodo 3 (ej. `192.168.1.12`)
   * `<VIP_cluster>` ➔ IP Virtual (VIP) para Keepalived en la misma subred (ej. `192.168.1.50`)
 
 * **Credenciales y Contraseñas:**
   * `<contraseña_super_usuario>` ➔ Contraseña para el usuario `postgres`
   * `<contraseña_replicacion>` ➔ Contraseña para el usuario de replicación
   * `<keepalived_auth_pass>` ➔ Contraseña compartida para las instancias VRRP
 
 * **Redes:**
   * `eth0` *(en las secciones de Keepalived)* ➔ Cámbiala si tu interfaz de red principal tiene otro nombre (ej. `ens33`, `enp0s3`).


## 1. Instalación de PostgreSQL

### 1.1 Instalar repositorios de PostgreSQL

```bash
sudo apt update
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```

#### 1.1.1 Posibles errores de instalación

Tras ejecutar los comandos de instalación, es posible que aparezca este error:

```bash
E: gnupg, gnupg2 and gnupg1 do not seem to be installed, but one of them is required for this operation
```
Esto significa que el repositorio de PostgreSQL se ha añadido, pero no se ha importado su clave GPG para que `apt` pueda verificar las firmas.

Para solucionar este error, debemos ejecutar los siguientes comandos:

```bash
sudo sed -i '/apt\.postgresql\.org/d' /etc/apt/sources.list
sudo rm -f /etc/apt/sources.list.d/pgdg.list

```

```bash
sudo apt-get update && sudo apt-get install -y wget gnupg ca-certificates lsb-release
wget -qO - https://www.postgresql.org/media/keys/ACCC4CF8.asc | \
gpg --dearmor | sudo tee /usr/share/keyrings/postgresql.gpg >/dev/null
sudo chmod 0644 /usr/share/keyrings/postgresql.gpg

```
```bash
printf 'deb [signed-by=/usr/share/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt %s-pgdg main\n' "$(lsb_release -cs)" \
| sudo tee /etc/apt/sources.list.d/pgdg.list >/dev/null

```

Por último, ejecuta los comandos de instalación

```bash
sudo apt update
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```


### 1.2 Instalar PostgreSQL y módulos

```bash
sudo apt update
sudo apt install -y postgresql postgresql-contrib
```

### 1.3 Deshabilitar el servicio de PostgreSQL (gestionado por Patroni)

```bash
sudo systemctl stop postgresql
sudo systemctl disable postgresql
```

## 2. Instalación y configuración de Etcd

### 2.1 Descargar e instalar etcd

```bash
ETCD_VERSION=v3.5.0
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz -o etcd.tar.gz
tar xzvf etcd.tar.gz
cd etcd-${ETCD_VERSION}-linux-amd64
sudo cp etcd etcdctl /usr/bin
```

### 2.2 Crear el usuario y el directorio para etcd

```bash
sudo useradd -r -s /sbin/nologin etcd
sudo mkdir -p /var/lib/etcd/mycluster
sudo chown -R etcd:etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd/mycluster
```




### 2.3 Crear el servicio systemd: `/etc/systemd/system/etcd.service`

```ini
[Unit]
Description=etcd key-value store
Documentation=https://github.com/coreos/etcd
After=network.target

[Service]
User=etcd
Group=etcd
Type=simple
EnvironmentFile=-/etc/default/etcd
ExecStart=/usr/bin/etcd
Restart=always
RestartSec=5s
LimitNOFILE=65536
TimeoutStartSec=0
WorkingDirectory=/var/lib/etcd
LimitFSIZE=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
```

Recargar el demonio y habilitar el servicio

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable etcd
```

Finalmente, iniciarlo y comprobar el estado

```
sudo systemctl start etcd
sudo systemctl status etcd
```
```bash
● etcd.service - etcd key-value store
     Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2025-08-06 07:15:38 UTC; 2min 17s ago
       Docs: https://github.com/coreos/etcd
   Main PID: 7725 (etcd)
      Tasks: 7 (limit: 1026)
     Memory: 7.3M
        CPU: 665ms
     CGroup: /system.slice/etcd.service
             └─7725 /usr/bin/etcd
```




### 2.4 Archivos de configuración predeterminados de etcd (`/etc/default/etcd`) por nodo

#### Nodo 1

```bash
ETCD_NAME="<nombre_nodo_1>"
ETCD_DATA_DIR="/var/lib/etcd/mycluster"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://<ip_nodo_1>:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://<ip_nodo_1>:2379"
ETCD_LISTEN_PEER_URLS="http://<ip_nodo_1>:2380"
ETCD_LISTEN_CLIENT_URLS="http://<ip_nodo_1>:2379,http://localhost:2379"
ETCD_INITIAL_CLUSTER="<nombre_nodo_1>=http://<ip_nodo_1>:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-1"
ETCD_LOG_OUTPUTS="/var/lib/etcd/mycluster.log"
```

#### Nodo 2

Para unir el nodo 2 al clúster:
```bash
etcdctl --endpoints=http://<ip_nodo_1>:2379 member add <nombre_nodo_2> --peer-urls="http://<ip_nodo_2>:2380"
```

```bash
ETCD_NAME="<nombre_nodo_2>"
ETCD_DATA_DIR="/var/lib/etcd/mycluster"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://<ip_nodo_2>:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://<ip_nodo_2>:2379"
ETCD_LISTEN_PEER_URLS="http://<ip_nodo_2>:2380"
ETCD_LISTEN_CLIENT_URLS="http://<ip_nodo_2>:2379,http://localhost:2379"
ETCD_INITIAL_CLUSTER="<nombre_nodo_1>=http://<ip_nodo_1>:2380,<nombre_nodo_2>=http://<ip_nodo_2>:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-1"
ETCD_LOG_OUTPUTS="/var/lib/etcd/mycluster.log"
```


#### Nodo 3
Para unir el nodo 3 al clúster
```bash
etcdctl --endpoints=http://<ip_nodo_1>:2379 member add <nombre_nodo_3> --peer-urls="http://<ip_nodo_3>:2380"
```

```bash
ETCD_NAME="<nombre_nodo_3>"
ETCD_DATA_DIR="/var/lib/etcd/mycluster"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://<ip_nodo_3>:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://<ip_nodo_3>:2379"
ETCD_LISTEN_PEER_URLS="http://<ip_nodo_3>:2380"
ETCD_LISTEN_CLIENT_URLS="http://<ip_nodo_3>:2379,http://localhost:2379"
ETCD_INITIAL_CLUSTER="<nombre_nodo_1>=http://<ip_nodo_1>:2380,<nombre_nodo_2>=http://<ip_nodo_2>:2380,<nombre_nodo_3>=http://<ip_nodo_3>:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-1"
ETCD_LOG_OUTPUTS="/var/lib/etcd/mycluster.log"
```

### 2.5 Reiniciar ETCD en todos los nodos

```bash
sudo systemctl restart etcd
```

```bash
sudo systemctl status etcd
● etcd.service - etcd key-value store
     Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2025-08-06 07:15:38 UTC; 2min 17s ago
       Docs: https://github.com/coreos/etcd
   Main PID: 7725 (etcd)
      Tasks: 7 (limit: 1026)
     Memory: 7.3M
        CPU: 665ms
     CGroup: /system.slice/etcd.service
             └─7725 /usr/bin/etcd
```

### 2.6 Verificar el clúster etcd

```bash
etcdctl member list

etcdctl --endpoints=http://<ip_nodo_1>:2379,http://<ip_nodo_2>:2379,http://<ip_nodo_3>:2379 endpoint health
```

## 3. Instalación y configuración de Patroni

### 3.1 Instalar Patroni

```bash
sudo apt update
sudo apt install -y patroni
```

### 3.2 Crear el servicio systemd de Patroni: `/etc/systemd/system/patroni.service`

```ini
[Unit]
Description=Patroni - PostgreSQL HA
Documentation=https://patroni.readthedocs.io/
After=network.target

[Service]
Type=simple
User=postgres
ExecStart=/usr/bin/patroni /etc/patroni/patroni.yml
Restart=always
TimeoutSec=300

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable patroni
```

### 3.3 Configurar el archivo YAML de Patroni: `/etc/patroni/patroni.yml`

Nodo 1: <nombre_nodo_1>
```yml
scope: mycluster
namespace: /service/
name: <nombre_nodo_1>

etcd3:
  hosts: <ip_nodo_1>:2379,<ip_nodo_2>:2379,<ip_nodo_3>:2379
  protocol: http

restapi:
  listen: 0.0.0.0:8008
  connect_address: <ip_nodo_1>:8008

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      pg_hba:
        - host replication replicator 127.0.0.1/32 password
        - host replication replicator <ip_nodo_1>/32 password
        - host replication replicator <ip_nodo_2>/32 password
        - host replication replicator <ip_nodo_3>/32 password
        - host all all 127.0.0.1/32 password
        - host all all 0.0.0.0/0 password
  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: <ip_nodo_1>:5432
  data_dir: /var/lib/postgresql/data #Puede cambiar, revisalo
  bin_dir: /usr/lib/postgresql/17/bin #Directorio del binario para PostgreSQL 17
  authentication:
    superuser:
      username: postgres
      password: <contraseña_super_usuario>
    replication:
      username: replicator
      password: <contraseña_replicacion>
  parameters:
    max_connections: 100
    shared_buffers: 256MB

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false

```


Nodo 2: <nombre_nodo_2>

```yml
scope: mycluster
namespace: /service/
name: <nombre_nodo_2>

etcd3:
  hosts: <ip_nodo_1>:2379,<ip_nodo_2>:2379,<ip_nodo_3>:2379
  protocol: http

restapi:
  listen: 0.0.0.0:8008
  connect_address: <ip_nodo_2>:8008

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      pg_hba:
        - host replication replicator 127.0.0.1/32 password
        - host replication replicator <ip_nodo_1>/32 password
        - host replication replicator <ip_nodo_2>/32 password
        - host replication replicator <ip_nodo_3>/32 password
        - host all all 127.0.0.1/32 password
        - host all all 0.0.0.0/0 password
  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: <ip_nodo_2>:5432
  data_dir: /var/lib/postgresql/data #Puede cambiar, revisalo
  bin_dir: /usr/lib/postgresql/17/bin #Directorio del binario para PostgreSQL 17
  authentication:
    superuser:
      username: postgres
      password: <contraseña_super_usuario>
    replication:
      username: replicator
      password: <contraseña_replicacion>
  parameters:
    max_connections: 100
    shared_buffers: 256MB

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
```

Nodo 3: <nombre_nodo_3>

```yml
scope: mycluster
namespace: /service/
name: <nombre_nodo_3>

etcd3:
  hosts: <ip_nodo_1>:2379,<ip_nodo_2>:2379,<ip_nodo_3>:2379
  protocol: http

restapi:
  listen: 0.0.0.0:8008
  connect_address: <ip_nodo_3>:8008

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      pg_hba:
        - host replication replicator 127.0.0.1/32 password
        - host replication replicator <ip_nodo_1>/32 password
        - host replication replicator <ip_nodo_2>/32 password
        - host replication replicator <ip_nodo_3>/32 password
        - host all all 127.0.0.1/32 password
        - host all all 0.0.0.0/0 password
  initdb:
    - encoding: UTF8
    - data-checksums

postgresql:
  listen: 0.0.0.0:5432
  connect_address: <ip_nodo_3>:5432
  data_dir: /var/lib/postgresql/data #Puede cambiar, revisalo
  bin_dir: /usr/lib/postgresql/17/bin #Directorio del binario para PostgreSQL 17
  authentication:
    superuser:
      username: postgres
      password: <contraseña_super_usuario>
    replication:
      username: replicator
      password: <contraseña_replicacion>
  parameters:
    max_connections: 100
    shared_buffers: 256MB

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false

logging:
  level: INFO
  file: /var/log/patroni/patroni.log

```

### 3.4 Reiniciar Patroni en todos los nodos

```bash
sudo systemctl restart patroni
```
```bash
● patroni.service - Patroni - PostgreSQL HA
     Loaded: loaded (/etc/systemd/system/patroni.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2025-08-12 11:32:47 CEST; 14min ago
       Docs: https://patroni.readthedocs.io/
   Main PID: 11991 (patroni)
      Tasks: 14 (limit: 698)
     Memory: 70.6M
        CPU: 6min 44.828s
     CGroup: /system.slice/docker-d678815f57a6114bfa3de1028af7b2b842b19b9c50e85d3af45710f7ac0cec95.scope/system.slice/patroni.service
             ├─11991 /usr/bin/python3 /usr/bin/patroni /etc/patroni/patroni.yml
             ├─12008 /usr/lib/postgresql/17/bin/postgres -D /var/lib/postgresql/17/main/ --config-file=/var/lib/postgresql/17/main/postgresql.conf --listen_addresses=0.0.0.0 --port=5432 --cluster_name=my>
             ├─12010 "postgres: mycluster: checkpointer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ">
             ├─12011 "postgres: mycluster: background writer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             ├─12012 "postgres: mycluster: startup recovering 000000050000000000000005" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ">
             ├─12033 "postgres: mycluster: postgres postgres 127.0.0.1(40810) idle" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             ├─12035 "postgres: mycluster: postgres postgres 127.0.0.1(40826) idle" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             └─42337 "postgres: mycluster: walreceiver " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">

Aug 12 11:47:41 u3 patroni[42331]: 2025-08-12 11:47:41.697 CEST [42331] DETAIL:  End of WAL reached on timeline 5 at 0/5426968.
Aug 12 11:47:41 u3 patroni[42331]: 2025-08-12 11:47:41.698 CEST [42331] FATAL:  terminating walreceiver process due to administrator command
Aug 12 11:47:41 u3 patroni[12012]: 2025-08-12 11:47:41.698 CEST [12012] LOG:  new timeline 6 forked off current database system timeline 5 before current recovery point 0/54269E0
Aug 12 11:47:41 u3 patroni[12012]: 2025-08-12 11:47:41.698 CEST [12012] LOG:  waiting for WAL to become available at 0/5002000
Aug 12 11:47:41 u3 patroni[42332]: 2025-08-12 11:47:41.727 CEST [42332] LOG:  started streaming WAL from primary at 0/5000000 on timeline 5
Aug 12 11:47:41 u3 patroni[42332]: 2025-08-12 11:47:41.734 CEST [42332] LOG:  replication terminated by primary server
Aug 12 11:47:41 u3 patroni[42332]: 2025-08-12 11:47:41.734 CEST [42332] DETAIL:  End of WAL reached on timeline 5 at 0/5426968.
Aug 12 11:47:41 u3 patroni[42332]: 2025-08-12 11:47:41.736 CEST [42332] FATAL:  terminating walreceiver process due to administrator command
Aug 12 11:47:41 u3 patroni[12012]: 2025-08-12 11:47:41.737 CEST [12012] LOG:  new timeline 6 forked off current database system timeline 5 before current recovery point 0/54269E0
Aug 12 11:47:41 u3 patroni[12012]: 2025-08-12 11:47:41.737 CEST [12012] LOG:  waiting for WAL to become available at 0/5002000
```

Finalmente, deberíamos ver algo como:


```
Dec 03 22:16:05 int-p-postgres-01 patroni[770]: 2024-12-03 22:16:05,399 INFO: no action. I am (<nombre_nodo_1>), the leader with the lock
Dec 03 22:16:15 int-p-postgres-01 patroni[770]: 2024-12-03 22:16:15,399 INFO: no action. I am (<nombre_nodo_1>), the leader with the lock


Dec 03 22:16:21 int-p-postgres-02 patroni[768]: 2024-12-03 22:16:21,780 INFO: Lock owner: <nombre_nodo_1>; I am <nombre_nodo_2>
Dec 03 22:16:21 int-p-postgres-02 patroni[768]: 2024-12-03 22:16:21,823 INFO: bootstrap from leader '<nombre_nodo_1>' in progress
```

### 3.5 Posibles errores del clúster

Un error frecuente que puede surgir en el clúster es el siguiente:

```bash
patroni[737113]: 2025-08-13 13:15:59,122 CRITICAL: system ID mismatch, node <nombre_nodo_2> belongs to a different cluster: 7538026960717399015 != 7537600130036392307
```

Este error aparece al añadir un nuevo miembro a un clúster existente. Pero no hay de qué preocuparse; existe una solución. Solo es necesario reiniciar el clúster desde cero, eliminando los datos antiguos. Siga los pasos que se indican a continuación.

#### 3.5.1 Detener todos los servicios

En **todos los nodos**:

```bash
sudo systemctl stop patroni
sudo systemctl stop postgresql@17-main || true
sudo systemctl disable postgresql || true
sudo systemctl reset-failed patroni
```

##### 3.5.2 Limpiar el espacio de nombres en etcd
En cualquier nodo:

```bash
export ETCDCTL_API=3
etcdctl --endpoints=http://<ip_nodo_1>:2379,http://<ip_nodo_2>:2379,http://<ip_nodo_3>:2379 del /service/mycluster/ --prefix
```
Verificar que esté vacío:

```bash
etcdctl --endpoints=http://<ip_nodo_1>:2379,http://<ip_nodo_2>:2379,http://<ip_nodo_3>:2379 get /service/mycluster/ --prefix
```

#### 3.5.3 Limpiar los directorios de datos antiguos

En **todos los nodos**

```bash
sudo -u postgres rm -rf /var/lib/postgresql/17/main/
sudo mkdir /var/lib/postgresql/17/main
```

```bash
sudo mkdir -p /var/lib/postgresql/17/patroni
sudo -u postgres rm -rf /var/lib/postgresql/17/patroni/*
sudo chown -R postgres:postgres /var/lib/postgresql
sudo chmod 700 /var/lib/postgresql/17/main
```

##### 3.5.4 Iniciar el clúster
Iniciar el proceso de arranque (*bootstrap*) en el nodo líder (u1):

```bash
sudo systemctl start patroni
sudo journalctl -u patroni -f
```
Verificar que actúa como líder.

Réplicas (u2 y u3):

```bash
sudo systemctl start patroni
sudo journalctl -u patroni -f
```

Deberían realizar una copia de seguridad base (*basebackup*) desde el líder y mantenerse en estado de replicación continua (*streaming*).

Finalmente, verificar el estado:

```bash
patronictl -c /etc/patroni/patroni.yml list
+ Cluster: mycluster (7503512298789297066) -----+-----------+----+-----------+
| Member              | Host          | Role    | State     | TL | Lag in MB |
+---------------------+---------------+---------+-----------+----+-----------+
| <nombre_nodo_1>     | <ip_nodo_1>   | Replica | streaming |  1 |         0 |
| <nombre_nodo_2>     | <ip_nodo_2>   | Leader  | running   |  1 |           |
| <nombre_nodo_3>     | <ip_nodo_3>   | Replica | streaming |  1 |         0 |
+---------------------+---------------+---------+-----------+----+-----------+
```

#### 3.5.5. Reincorporar un nodo al clúster

Si alguno de los nodos deja de estar en estado "streaming" y pasa al estado "running", como se puede ver aquí:

```bash
patronictl -c /etc/patroni/patroni.yml list
+ Cluster: mycluster (7537600130036392307) ---+-----------+----+-----------+
| Member            | Host          | Role    | State     | TL | Lag in MB |
+-------------------+---------------+---------+-----------+----+-----------+
| <nombre_nodo_1>   | <ip_nodo_1>   | Replica | streaming | 10 |         0 |
| <nombre_nodo_2>   | <ip_nodo_2>   | Leader  | running   | 10 |           |
| <nombre_nodo_3>   | <ip_nodo_3>   | Replica | running   |  9 |        60 |
+-------------------+---------------+---------+-----------+----+-----------+
```

Solo tenemos que ejecutar este comando para reintegrar el nodo al modo de replicación:

```bash
patronictl -c /etc/patroni/patroni.yml reinit mycluster <nombre_nodo_3> --force
+ Cluster: mycluster (7537600130036392307) ---+-----------+----+-----------+
| Member            | Host          | Role    | State     | TL | Lag in MB |
+-------------------+---------------+---------+-----------+----+-----------+
| <nombre_nodo_1>   | <ip_nodo_1>   | Replica | streaming | 10 |         0 |
| <nombre_nodo_2>   | <ip_nodo_2>   | Leader  | running   | 10 |           |
| <nombre_nodo_3>   | <ip_nodo_3>   | Replica | running   |  9 |        60 |
+-------------------+---------------+---------+-----------+----+-----------+
Success: reinitialize for member <nombre_nodo_3>
```
```bash
patronictl -c /etc/patroni/patroni.yml list
+ Cluster: mycluster (7537600130036392307) ---+------------------+----+-----------+
| Member            | Host          | Role    | State            | TL | Lag in MB |
+-------------------+---------------+---------+------------------+----+-----------+
| <nombre_nodo_1>   | <ip_nodo_1>   | Replica | streaming        | 10 |         0 |
| <nombre_nodo_2>   | <ip_nodo_2>   | Leader  | running          | 10 |           |
| <nombre_nodo_3>   | <ip_nodo_3>   | Replica | creating replica |    |   unknown |
+-------------------+---------------+---------+------------------+----+-----------+
```
Y, finalmente, vuelve a estar en estado de streaming
```bash
patronictl -c /etc/patroni/patroni.yml list
+ Cluster: mycluster (7537600130036392307) ---+------------+----+----------+
| Member            | Host          | Role    | State     | TL | Lag in MB |
+-------------------+---------------+---------+-----------+----+-----------+
| <nombre_nodo_1>   | <ip_nodo_1>   | Replica | streaming | 10 |         0 |
| <nombre_nodo_2>   | <ip_nodo_2>   | Leader  | running   | 10 |           |
| <nombre_nodo_3>   | <ip_nodo_3>   | Replica | streaming | 10 |         0 |
+-------------------+---------------+---------+-----------+----+-----------+
```

### 3.6 Reconfiguración de nuestro clúster etcd (`/etc/etcd/etcd.env`)

Debemos cambiar esto en nuestro nodo líder:

```bash
ETCD_INITIAL_CLUSTER_STATE="new"
```

por esto:

```bash
ETCD_INITIAL_CLUSTER_STATE="existing"
```

Para ver el estado del clúster:

```bash
patronictl -c /etc/patroni/patroni.yml list
+ Cluster: mycluster (7503512298789297066) -----+-----------+----+-----------+
| Member              | Host          | Role    | State     | TL | Lag in MB |
+---------------------+---------------+---------+-----------+----+-----------+
| <nombre_nodo_1>     | <ip_nodo_1>   | Replica | streaming |  1 |         0 |
| <nombre_nodo_2>     | <ip_nodo_2>   | Leader  | running   |  1 |           |
| <nombre_nodo_3>     | <ip_nodo_3>   | Replica | streaming |  1 |         0 |
+---------------------+---------------+---------+-----------+----+-----------+
```

## 4. Verificación y monitoreo

### 4.1. Registros (Logs)

Líder:

```
INFO: no action. I am (<nombre_nodo_1>), the leader with the lock
```

Réplica:

```
INFO: Lock owner: <nombre_nodo_1>; I am <nombre_nodo_2>
```

### 4.2. Estado del clúster Patroni

```bash
patronictl -c /etc/patroni/patroni.yml list
+ Cluster: mycluster (7503512298789297066) -----+-----------+----+-----------+
| Member              | Host          | Role    | State     | TL | Lag in MB |
+---------------------+---------------+---------+-----------+----+-----------+
| <nombre_nodo_1>     | <ip_nodo_1>   | Replica | streaming |  1 |         0 |
| <nombre_nodo_2>     | <ip_nodo_2>   | Leader  | running   |  1 |           |
| <nombre_nodo_3>     | <ip_nodo_3>   | Replica | streaming |  1 |         0 |
+---------------------+---------------+---------+-----------+----+-----------+
```

### 4.3. Verificación de la API REST de Patroni

```bash
curl http://<node_ip>:8008/cluster | jq

{
  "members": [
    {
      "name": "<nombre_nodo_1>",
      "role": "replica",
      "state": "streaming",
      "api_url": "http://<ip_nodo_1>:8008/patroni",
      "host": "<ip_nodo_1>",
      "port": 5432,
      "timeline": 1,
      "lag": 0
    },
    {
      "name": "<nombre_nodo_2>",
      "role": "leader",
      "state": "running",
      "api_url": "http://<ip_nodo_2>:8008/patroni",
      "host": "<ip_nodo_2>",
      "port": 5432,
      "timeline": 1
    },
    {
      "name": "<nombre_nodo_3>",
      "role": "replica",
      "state": "streaming",
      "api_url": "http://<ip_nodo_3>:8008/patroni",
      "host": "<ip_nodo_3>",
      "port": 5432,
      "timeline": 1,
      "lag": 0
    }
  ],
  "scope": "mycluster"
}
```

### 4.4 Prueba de conmutación por error (Failover)

```bash
patronictl -c /etc/patroni/patroni.yml list
+ Cluster: mycluster (7503512298789297066) -----+-----------+----+-----------+
| Member              | Host          | Role    | State     | TL | Lag in MB |
+---------------------+---------------+---------+-----------+----+-----------+
| <nombre_nodo_1>     | <ip_nodo_1>   | Replica | streaming |  1 |         0 |
| <nombre_nodo_2>     | <ip_nodo_2>   | Leader  | running   |  1 |           |
| <nombre_nodo_3>     | <ip_nodo_3>   | Replica | streaming |  1 |         0 |
+---------------------+---------------+---------+-----------+----+-----------+
```
Con este comando podemos cambiar el nodo líder para realizar la prueba:

```bash
 patronictl -c /etc/patroni/patroni.yml switchover --candidate int-t-postgresql-02
```

También podemos detener Patroni en el nodo líder, y otro nodo asumirá el rol de líder.

## 5. Instalación y configuración de HAProxy

### 5.1 Instalación

```bash
sudo apt -y install haproxy
```

### 5.2 Configuración (`/etc/haproxy/haproxy.cfg`)

Esta configuración debe aplicarse en todos los nodos.

```bash
defaults
	log	global
	mode	tcp
	option  tcplog
	option	dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

frontend postgres_rw
    bind :5000
    mode tcp
    default_backend postgres_rw_backend

backend postgres_rw_backend
    balance roundrobin
    option httpchk GET /primary HTTP/1.1\r\nHost:\ localhost
    http-check expect status 200
    server <nombre_nodo_1> <ip_nodo_1>:5432 check port 8008
    server <nombre_nodo_2> <ip_nodo_2>:5432 check port 8008
    server <nombre_nodo_3> <ip_nodo_3>:5432 check port 8008

frontend postgres_ro
    bind :5001
    mode tcp
    default_backend postgres_ro_backend

backend postgres_ro_backend
    balance roundrobin
    option httpchk GET /replica HTTP/1.1\r\nHost:\ localhost
    http-check expect status 200
    server <nombre_nodo_1> <ip_nodo_1>:5432 check port 8008
    server <nombre_nodo_2> <ip_nodo_2>:5432 check port 8008
    server <nombre_nodo_3> <ip_nodo_3>:5432 check port 8008
```

### 5.3 Reiniciar HAProxy en todos los nodos

```bash
sudo systemctl restart haproxy
```
```bash
sudo systemctl status haproxy
● haproxy.service - HAProxy Load Balancer
     Loaded: loaded (/lib/systemd/system/haproxy.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2025-08-12 11:21:44 CEST; 28min ago
       Docs: man:haproxy(1)
             file:/usr/share/doc/haproxy/configuration.txt.gz
    Process: 9999 ExecStartPre=/usr/sbin/haproxy -Ws -f $CONFIG -c -q $EXTRAOPTS (code=exited, status=0/SUCCESS)
   Main PID: 10001 (haproxy)
      Tasks: 3 (limit: 698)
     Memory: 135.7M
        CPU: 2.569s
     CGroup: /system.slice/docker-d678815f57a6114bfa3de1028af7b2b842b19b9c50e85d3af45710f7ac0cec95.scope/system.slice/haproxy.service
             ├─10001 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock
             └─10003 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock

Aug 12 11:21:45 u3 haproxy[10003]: [WARNING]  (10003) : Server postgres_rw_backend/<nombre_nodo_3> is DOWN, reason: Layer7 wrong status, code: 503, info: "Service Unavailable", check duration: 2ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Aug 12 11:21:45 u3 haproxy[10003]: [WARNING]  (10003) : Server postgres_ro_backend/<nombre_nodo_2> is DOWN, reason: Layer7 wrong status, code: 503, info: "Service Unavailable", check duration: 3ms. 2 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Aug 12 11:31:02 u3 haproxy[10003]: [WARNING]  (10003) : Server postgres_rw_backend/<nombre_nodo_3> is UP, reason: Layer7 check passed, code: 200, check duration: 3ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
Aug 12 11:31:03 u3 haproxy[10003]: [WARNING]  (10003) : Server postgres_ro_backend/<nombre_nodo_3> is DOWN, reason: Layer7 wrong status, code: 503, info: "Service Unavailable", check duration: 3ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Aug 12 11:31:04 u3 haproxy[10003]: [WARNING]  (10003) : Server postgres_rw_backend/<nombre_nodo_2> is DOWN, reason: Layer7 wrong status, code: 503, info: "Service Unavailable", check duration: 3ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Aug 12 11:31:05 u3 haproxy[10003]: [WARNING]  (10003) : Server postgres_ro_backend/<nombre_nodo_2> is UP, reason: Layer7 check passed, code: 200, check duration: 2ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
Aug 12 11:32:28 u3 haproxy[10003]: [WARNING]  (10003) : Server postgres_rw_backend/<nombre_nodo_2> is UP, reason: Layer7 check passed, code: 200, check duration: 3ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
Aug 12 11:32:30 u3 haproxy[10003]: [WARNING]  (10003) : Server postgres_rw_backend/<nombre_nodo_3> is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 0ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Aug 12 11:32:31 u3 haproxy[10003]: [WARNING]  (10003) : Server postgres_ro_backend/<nombre_nodo_2> is DOWN, reason: Layer7 wrong status, code: 503, info: "Service Unavailable", check duration: 2ms. 1 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Aug 12 11:32:53 u3 haproxy[10003]: [WARNING]  (10003) : Server postgres_ro_backend/<nombre_nodo_3> is UP, reason: Layer7 check passed, code: 200, check duration: 20ms. 2 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
```

**No te preocupes por estos registros de advertencia (Warning logs)**

**Código: 200** significa que este es el **nodo líder**; acepta operaciones de **lectura y escritura**.
**Código: 503** significa que este es un **nodo réplica**; acepta **solo lectura**.

**Estos mensajes de advertencia NO significan que HAProxy no esté funcionando ni que haya un error de configuración.**

Para verificar que está funcionando, también puedes usar este comando:

En el **nodo líder**:

```bash
curl -sI http://<ip_lider>:8008/master		->	HTTP/1.0 200 OK
curl -sI http://<ip_replica>:8008/master	->	HTTP/1.0 503 Service Unavailable
```

En el **nodo réplica**:

```bash
curl -sI http://<ip_lider>:8008/replica		->	HTTP/1.0 503 Service Unavailable
curl -sI http://<ip_replica>:8008/replica	->	HTTP/1.0 200 OK
```

Sustituye **\<ip_lider\>** o **\<ip_replica\>** por la IP real de tus nodos.

## 6. Instalación y configuración de Keepalived

### 6.1 Instalación

Ahora debemos instalar Keepalived para crear una VIP. La VIP debe estar en el **mismo rango de IP que sus nodos**. 

```bash
sudo apt update
sudo apt install keepalived -y
```

### 6.2 Configuración (`/etc/keepalived/keepalived.conf`)

Nodo 1: <nombre_nodo_1>

```bash
global_defs {
    enable_script_security
    script_user keepalived_script
}

vrrp_script check_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
    fall 3
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0 # Modificala si es necesario
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass <keepalived_auth_pass>
    }
    virtual_ipaddress {
        <VIP_cluster>
    }
    track_script {
        check_haproxy
    }
}
```

Nodo 2: <nombre_nodo_2>

```bash
global_defs {
    enable_script_security
    script_user keepalived_script
}

vrrp_script check_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
    fall 3
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0 # Modificala si es necesario
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS 
        auth_pass <keepalived_auth_pass>
    }
    virtual_ipaddress {
        <VIP_cluster>
    }
    track_script {
        check_haproxy
    }
}
```

Nodo 3: <nombre_nodo_3>

```bash
global_defs {
    enable_script_security
    script_user keepalived_script
}

vrrp_script check_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
    fall 3
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0 # Modificala si es necesario
    virtual_router_id 51
    priority 80
    advert_int 1
    authentication {
        auth_type PASS 
        auth_pass <keepalived_auth_pass>
    }
    virtual_ipaddress {
        <VIP_cluster>
    }
    track_script {
        check_haproxy
    }
}
```

**Modifica <VIP_cluster> por la VIP real que vayas a usar, debe pertenecer a la misma subred que los nodos, y no ser utilizada por ninguno otro nodo.**

#### 6.2.1 Crear un script de comprobación en cada nodo (`/etc/keepalived/check_haproxy.sh`)

```bash
#!/bin/bash

#Define the port to check (HAProxy frontend port)
PORT1=5000
PORT2=5001


#Check if HAProxy is running
if ! pidof haproxy > /dev/null; then
    echo "HAProxy is not running"
    exit 1
fi

#Check if HAProxy is listening on the expected port
if ! ss -ltn | grep -q ":${PORT1}"; then
    echo "HAProxy is not listening on port ${PORT1}"
    exit 2
fi

if ! ss -ltn | grep -q ":${PORT2}"; then
    echo "HAProxy is not listening on port ${PORT2}"
    exit 2
fi

#All checks passed
exit 0
```

Necesitamos añadir un usuario para ejecutar estos scripts

```bash
sudo useradd -r -s /bin/false keepalived_script
```

```bash
sudo chmod +x /etc/keepalived/check_haproxy.sh
sudo chown keepalived_script:keepalived_script /etc/keepalived/check_haproxy.sh
sudo chmod 700 /etc/keepalived/check_haproxy.sh
```

### 6.3 Reiniciar keepalived en todos los nodos

```bash
sudo systemctl restart keepalived
```

Comprobar el estado o los registros (logs)

En el nodo maestro, deberíamos ver esto:

```bash
sudo journalctl -u keepalived -f
Aug 12 14:13:16 u1 Keepalived_vrrp[109478]: (VI_1) Entering MASTER STATE
```

y también debemos ver la VIP en nuestra lista de interfaces:

```bash
ip a
6: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet <VIP_cluster>/32 scope global eth0
       valid_lft forever preferred_lft forever
```

En los nodos de réplica, deberíamos ver esto:

```bash
Aug 12 14:13:41 u2 Keepalived_vrrp[235456]: (VI_1) Entering BACKUP STATE