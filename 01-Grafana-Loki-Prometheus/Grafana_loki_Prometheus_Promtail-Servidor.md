**Autor:** Pablo Mongar (Técnico ASIR)  
**Proyecto:** Stack de monitorización PLG  
**Contacto:** pmongber@gmail.com  
**LinkedIn:** linkedin.com/in/pmongar

# Instalación y configuración del servidor en Debian 12

Este documento describe los pasos completos para instalar y configurar **Prometheus**, **Grafana** y **Loki** (junto con **Promtail**) para obtener un stack de monitorización funcional.



## 1. Configuración inicial

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget unzip
```




## 2. Instalación de Prometheus

Prometheus es una base de datos de series temporales y un sistema de monitorización diseñado para registrar métricas de alta dimensionalidad. Recopila, almacena y consulta métricas utilizando su propio lenguaje de consulta (PromQL). Es ideal para monitorizar infraestructuras, servicios y aplicaciones. 

### 2.1 Descargar y extraer los binarios

```bash
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.52.0/prometheus-2.52.0.linux-amd64.tar.gz
mkdir -p /opt/prometheus
cd /opt/prometheus
sudo tar -xzf /tmp/prometheus-2.52.0.linux-amd64.tar.gz --strip-components=1
```

### 2.2 Crear usuario y directorios

```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin prometheus
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo cp -r /opt/prometheus/consoles console_libraries prometheus.yml /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus /opt/prometheus
```

### 2.3 Crear servicio de systemd

Archivo: `/etc/systemd/system/prometheus.service`

```ini
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
ExecStart=/opt/prometheus/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --storage.tsdb.retention.time=90d \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries
Restart=always

[Install]
WantedBy=multi-user.target
```

### 2.4 Iniciar el servicio

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

## 3. Instalación de Grafana

Grafana es una plataforma de análisis y visualización de código abierto utilizada para monitorear métricas y registros (logs) provenientes de múltiples fuentes de datos. Ofrece paneles potentes, sistemas de alertas y análisis en tiempo real. Grafana admite de forma nativa Prometheus, Loki y muchos otros sistemas.

```bash
sudo apt install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install -y grafana
```

### 3.1 Iniciar el servicio

```bash
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

## 4. Instalación de Loki

Loki es un sistema de agregación de registros (logs) desarrollado por Grafana Labs. A diferencia de otros sistemas, Loki indexa únicamente los metadatos (etiquetas), lo que lo hace eficiente y rentable. Se integra perfectamente con Grafana para realizar consultas de registros y correlacionarlos con métricas. 

### 4.1 Descargar el binario

```bash
cd /tmp
wget https://github.com/grafana/loki/releases/download/v3.5.0/loki-linux-amd64.zip
unzip loki-linux-amd64.zip
sudo mv loki-linux-amd64 /usr/local/bin/loki
sudo chmod +x /usr/local/bin/loki
```

### 4.2 Crear directorios

```bash
sudo mkdir -p /etc/loki
sudo mkdir -p /var/lib/loki
sudo chown -R nobody:nogroup /var/lib/loki
```

### 4.3 Configuración básica de Loki

Archivo: `/etc/loki/loki-config.yaml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9095
  log_level: info

common:
  path_prefix: /var/lib/loki
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /var/lib/loki/index
    cache_location: /var/lib/loki/boltdb-cache
  filesystem:
    directory: /var/lib/loki/chunks

limits_config:
  allow_structured_metadata: false

```

**auth_enabled: false**
Desactiva la autenticación. Loki aceptará solicitudes sin requerir credenciales ni tokens. Útil para entornos internos o de confianza.

**server:**
Define cómo Loki expone sus servicios:

- http_listen_port: 3100 – Puerto para la API HTTP (utilizado por Grafana, Promtail, etc.).

- grpc_listen_port: 9095 – Puerto para la comunicación interna gRPC (entre componentes).

- log_level: info – Establece el nivel de detalle de los registros (log) en `info`. **common:**
Configuraciones compartidas utilizadas por todos los componentes:

- path_prefix: /var/lib/loki – Directorio base para almacenar datos en el disco.

- replication_factor: 1 – Solo una réplica de los datos; típico para configuraciones de un solo nodo.

- ring: – Configura cómo los componentes de Loki se descubren y rastrean entre sí. Aquí, `inmemory` significa que no se utiliza ningún servicio externo (como Consul).

**schema_config:**
Define cómo Loki almacena e indexa los datos de registro (logs):

- from: 2024-01-01 – Aplicar esta configuración a partir de esta fecha.

- store: boltdb-shipper – Utilizar boltdb-shipper para una indexación escalable.

- object_store: filesystem – Almacenar datos localmente en el sistema de archivos.

- schema: v13 – Versión del esquema de almacenamiento (se recomienda v13).

- index.period: 24h – Crear un nuevo archivo de índice cada 24 horas.

**storage_config:**
- Configura el almacenamiento local de Loki:

- boltdb_shipper: – Rutas para los archivos de índice y su caché.

- filesystem: – Directorio donde se almacenan los *chunks* (los registros propiamente dichos).

**limits_config:**
Establece límites operativos y activa/desactiva funciones:

**allow_structured_metadata:** false – Desactiva el soporte de metadatos estructurados para las entradas de registro con el fin de reducir la sobrecarga.

### 4.4 Servicio de Systemd

Archivo: `/etc/systemd/system/loki.service`

```ini
[Unit]
Description=Loki Log Aggregation
After=network.target

[Service]
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/loki-config.yaml
Restart=on-failure
User=nobody
Group=nogroup

[Install]
WantedBy=multi-user.target
```

### 4.5 Iniciar servicio

```bash
sudo systemctl daemon-reload
sudo systemctl enable loki
sudo systemctl start loki
```

## 5. Instalación de Promtail

Promtail es un agente de recolección de registros (*logs*) que lee archivos locales y envía los registros a Loki. Admite etiquetado, filtrado y etapas de procesamiento (*pipeline stages*). Promtail es ligero y adecuado para la ingesta básica de registros.

### 5.1 Descarga y preparación

```bash
cd /tmp
wget https://github.com/grafana/loki/releases/download/v3.5.0/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail
```

### 5.2 Crear directorios

```bash
sudo mkdir -p /etc/promtail
sudo mkdir -p /var/log/monitoring
```
`/etc/promtail/` contendrá el archivo de configuración de Promtail (`promtail.yaml`).

`/var/log/monitoring/` puede utilizarse para almacenar los registros del servicio Promtail (opcional); sigue la misma estructura que en el nodo maestro.

### 5.3 Configuración para leer archivos `.log`

Archivo: `/etc/promtail/promtail.yaml`

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://<ip_servidor>:3100/loki/api/v1/push

scrape_configs:
  - job_name: <appname>
    static_configs:
      - targets:
          - localhost
        labels:
          job: <appname>
          app: <appname>
          environment: <envname>
          service: <srvname>
          __path__: /var/log/<logs>
#    pipeline_stages:
#      - drop:
#          expression: '^\s*$'
#      - drop:
#          expression: '^(level=error ts=.*size=0.*)$'
#      - match:
#          selector: '{job="local-logs"}'
#          stages:
#            - drop:
#                expression: '^[^a-zA-Z0-9]*$'
```

**server:**
Define cómo Promtail expone sus propias métricas y puntos finales (*endpoints*) de estado:

- http_listen_port: 9080 – Puerto utilizado para el servidor HTTP interno de Promtail (métricas, comprobaciones de estado).

- grpc_listen_port: 0 – Desactiva gRPC (establecido en 0, lo que significa que no se utiliza). **positions:**
- Realiza un seguimiento de cuánto se ha leído de cada archivo de registro:

- filename: /tmp/positions.yaml – Promtail almacena aquí los desplazamientos (offsets) de los archivos para evitar volver a leer los registros tras un reinicio.

**clients:**
Especifica dónde enviar los registros:

- url: http://\<ip_servidor\>:3100/loki/api/v1/push – El endpoint de Loki donde Promtail envía los registros.

**Sustituye \<ip_servidor\> por la dirección del servidor de Loki (Aunque este corriendo en la misma maquina, es preferible usar la IP publica en lugar de 127.0.0.1)**

**scrape_configs:**
Define qué registros leer, cómo etiquetarlos y cómo procesarlos:

Define qué registros leer, cómo etiquetarlos y cómo procesarlos:
- job_name: \<appname\> – Nombre lógico para esta tarea de recolección (scrape job).
- static_configs: – Objetivos (targets) y etiquetas definidos manualmente. 
  - targets: [localhost] – Campo obligatorio, aunque no se utilice para la recolección de archivos locales. 
  - labels:
    - job: \<appname\> – Etiqueta añadida a cada línea de registro para identificar el trabajo.
    - app: \<appname\> – Etiqueta para identificar el nombre de la aplicación concreta.
    - environment: \<envname\> – Etiqueta que indica el entorno de ejecución (producción).
    - service: \<srvname\> – Etiqueta para agrupar los registros bajo el servicio de observabilidad.
    - __path__: /var/log/<logs> – Ruta al archivo de registro del sistema que se va a recopilar.

**(Comentado) pipeline_stages:**
Pipeline opcional de procesamiento de registros (actualmente deshabilitado):

- Permitiría filtrar o modificar los registros antes de enviarlos a Loki.

- Ejemplos mostrados:

- Descartar líneas vacías. 

- Descartar registros que coincidan con ciertos patrones (como size=0). 

- Descartar líneas que contengan solo símbolos. 

Reemplaza **\<appname\>**,**\<envname\>**,**\<srvname\>** con el nombre que desees asignar.

Puedes añadir **más directorios de registros**; explicamos cómo hacerlo en el **primer runbook**.

### 5.3 Servicio de systemd

Archivo: `/etc/systemd/system/promtail.service`

```ini
[Unit]
Description=Promtail service
After=network.target

[Service]
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/promtail.yaml
Restart=on-failure
User=nobody
Group=nogroup

[Install]
WantedBy=multi-user.target
```

### 5.4 Iniciar el servicio

```bash
sudo systemctl daemon-reload
sudo systemctl enable promtail
sudo systemctl start promtail
```

## 6. Configuración de Grafana

### 6.1 Acceder a Grafana

Accede a: `http://<IP>:3000`
Nombre de usuario: `admin`
Contraseña: `admin`

### 6.2 Añadir fuentes de datos

#### Prometheus:

* Tipo: **Prometheus**
* URL: `http://localhost:9090`

#### Loki:

* Tipo: **Loki**
* URL: `http://localhost:3100`

Si alguno de tus **servicios** no se encuentra en la **misma máquina** que el resto, **no uses localhost**; utiliza la **IP pública** de la **máquina** donde esté alojado.

## 7. Verificación en Grafana

En "Explore" (Explorar):

* Para métricas:

```promql
up
```

* Para registros (logs):

```logql
{job="job_name"}
```

Parece obvio, pero **cambia job_name** por el nombre que hayas asignado.

## 8. TTL en Prometheus y Loki

* Prometheus: `--storage.tsdb.retention.time=90d`
* Loki: no dispone de un TTL directo mediante argumentos de línea de comandos (CLI). Sin embargo, la retención de datos puede configurarse en el archivo `loki-config.yaml`.

En nuestro caso, dado que ejecutamos Loki como un servicio monolítico (un único binario), utilizamos el componente `table_manager` para habilitar la **eliminación automática de registros (logs) tras 90 días**. Esto se realiza añadiendo la siguiente configuración a nuestro archivo de configuración de Loki:

Archivo: `/etc/loki/loki-config.yaml`

```yaml
table_manager:
  retention_deletes_enabled: true
  retention_period: 2160h  # 90 days
```

## Qué se almacena en los directorios de datos de Prometheus y Loki

Comprender qué almacenan Prometheus y Loki en el disco ayuda en la planificación de copias de seguridad, la monitorización del uso del disco y las políticas de retención de datos.

### Prometheus – /var/lib/prometheus

Prometheus utiliza una base de datos de series temporales (TSDB) para almacenar métricas. Todos los datos se guardan en la ruta de almacenamiento configurada (por defecto: `/var/lib/prometheus`).

**Contenido típico:**

- chunks_head/
Almacena datos recientes que estaban en memoria y han sido serializados a disco antes de compactarse en bloques.

- wal/ (Write-Ahead Log / Registro de escritura previa)
Registros de transacciones escritos antes de que los datos se persistan. Se utiliza para la recuperación ante fallos.

- 01ABCDEF.../ (directorios de bloques)
Cada uno representa un bloque de métricas de 2 horas:

- chunks/: los valores reales de las métricas

- index: estructuras para búsquedas rápidas

- meta.json: metadatos sobre el bloque

### Loki – /var/lib/loki/

Al utilizar `boltdb-shipper` con el sistema de archivos (*filesystem*), Loki separa el almacenamiento de registros de la indexación:

**Desglose de directorios:**
- /var/lib/loki/chunks
Almacena los fragmentos (*chunks*) de datos de registro comprimidos. Cada archivo contiene registros agrupados por rango de tiempo y etiquetas (*labels*).

- /var/lib/loki/index
Contiene archivos de índice de `boltdb`, utilizados para localizar registros basándose en consultas de etiquetas (p. ej., `{job="nginx"}`). - /var/lib/loki/boltdb-cache
Caché temporal para acelerar el acceso a los archivos de índice.
