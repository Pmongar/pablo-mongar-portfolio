---
**Autor:** Pablo Mongar (Técnico ASIR)  
**Proyecto:** Stack de monitorización PLG  
**Contacto:** pmongber@gmail.com  
**LinkedIn:** linkedin.com/in/pmongar
---
# Instalación y configuración del cliente

# En el cliente Linux

Este documento detalla los pasos necesarios para instalar **Node Exporter** en una instancia (basada en Ubuntu), exponerlo externamente e integrarlo con un servidor Prometheus externo.

## 1. Instalación de Node Exporter

### 1.1 Descargar e instalar el binario de Node Exporter
```bash
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.8.1.linux-amd64.tar.gz
sudo cp node_exporter-1.8.1.linux-amd64/node_exporter /usr/local/bin/
```

### 1.2 Crear un usuario para Node Exporter
```bash
sudo useradd -rs /bin/false node_exporter
```

## 2. Crear un servicio de Systemd

### 2.1 Archivo de servicio

Crear el archivo de unidad de systemd:

Archivo: `/etc/systemd/system/node_exporter.service`

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter \
   --collector.systemd
[Install]
WantedBy=default.target
```

### 2.2 Recargar y habilitar el servicio
```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
```

## 3. Verificar que Node Exporter esté escuchando en el puerto 9100
```bash
ss -tuln | grep 9100
```

Resultado esperado:

```nginx
LISTEN 0 128 0.0.0.0:9100 ...
```


## 4. Probar la conectividad entre servidores

### 4.1 Comprobación con Telnet o netcat
```bash
telnet <ip_cliente> 9100
```
o
```bash
nc -vz <ip_cliente> 9100
```

Resultado esperado:
```css
Connection to <ip_cliente> 9100 port [tcp/*] succeeded!
```

### 4.2 Verificación con curl
```bash
curl http://<ip_cliente>:9100/metrics
```

Deberías recibir un **volcado de métricas** de Node Exporter.

**Tanto en 4.1 como 4.2, sustituye \<ip_cliente\> por la dirección del cliente en el que esta instalado node_Exporter**

Si recibes problemas de conectividad, recuerda abrir el puerto 9100 en el cliente.

## 5. Añadir el objetivo (target) del cliente en Prometheus (nodo_maestro)

### 5.1 Editar la configuración de Prometheus
Archivo: `/etc/prometheus/prometheus.yml`

```yaml
scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets:
          - '<ip_cliente>:9100'
          ...
```
Reemplaza **\<ip_cliente\>** por la **IP pública** real de la instancia **cliente**.
Puedes añadir todas las IP de cliente que necesites.

### 5.2 Reiniciar Prometheus
``` bash
sudo systemctl restart prometheus
```

## 6. Verificar en la interfaz web de Prometheus

Accede a la interfaz de Prometheus:

```arduino
http://<ip_servidor>:9090/targets
```
Sustituye **\<ip_servidor\>** por la dirección de la instancia donde se está ejecutando **Prometheus**.

Asegúrate de que el **objetivo recién añadido** (\<ip_cliente\>:9100) muestre el estado **UP**.

# Configuración de Promtail para enviar registros a Loki

Ahora vamos a configurar **Promtail** en nuestros nodos cliente para **enviar registros** a nuestra instancia de Loki en el nodo maestro.

## 1. Descargar e instalar Promtail
```bash
cd /tmp
wget https://github.com/grafana/loki/releases/download/v3.5.0/promtail-linux-amd64.zip
sudo apt install unzip
unzip promtail-linux-amd64.zip
chmod +x promtail-linux-amd64
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
```

## 2. Preparar los directorios necesarios

Antes de configurar Promtail, debes crear los directorios requeridos:

```bash
sudo mkdir -p /etc/promtail
sudo mkdir -p /var/log/...
```

`/etc/promtail/` contendrá el archivo de configuración de Promtail (`promtail.yaml`).

`/var/log/...` puede utilizarse para almacenar los registros del servicio Promtail (opcional); sigue la misma estructura que en el nodo maestro.

## 3. Crear el archivo de configuración /etc/promtail/promtail.yaml
```bash
sudo vim /etc/promtail/promtail.yaml
```

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


```

Reemplaza **\<ip_servidor\>** con la dirección IP pública o privada de tu **servidor Loki**.

Reemplaza **\<logs\>** con la ruta real a tus logs.

Reemplaza **\<appname\>**,**\<envname\>**,**\<srvname\>** con el nombre que desees asignar.

Puedes añadir **más directorios de registros**; explicamos cómo hacerlo en el **primer runbook**.

## 4. Crear el servicio systemd para Promtail
```bash
sudo vim /etc/systemd/system/promtail.service
```

```ini
[Unit]
Description=Promtail service
After=network.target

[Service]
User=root
Group=root
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/promtail.yaml
Restart=on-failure
StandardOutput=append:/var/log/monitoring/promtail.log
StandardError=append:/var/log/monitoring/promtail.log


[Install]
WantedBy=multi-user.target
```

El usuario **root** es importante para **leer /var/log/...**; sin embargo, si no estás seguro, puedes crear un **usuario** llamado **promtail**, **reemplazar** **root** y **asignar permisos** sobre el **directorio** del cual deseas **exportar los registros (logs)**.


## 5. Habilitar e iniciar Promtail

Recarga el gestor systemd e inicia el servicio:

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable promtail
sudo systemctl start promtail
```

Verifica el estado del servicio:

```bash
sudo systemctl status promtail
```

## 6. Verificar los registros en Loki a través de Grafana
Ve a Grafana → Explore

Selecciona la fuente de datos Loki

Ejecuta la siguiente consulta para ver los registros:

```logql
{job="appname"}
```

# En un cliente Windows

Ahora haremos lo mismo que hicimos en Linux, pero esta vez en un **cliente Windows**. En esta ocasión no utilizaremos `node_exporter`; en Windows debemos usar **`windows_exporter`**.

## 1. Exportación de métricas a Prometheus con `windows_exporter`

### 1.1 Descarga e instalación

Ve a la página de lanzamientos (releases) de **`windows_exporter`**.

Descarga la **última versión estable**; utilizaremos **`windows_exporter-0.30.7-amd64.exe`**.

Guarda el archivo en una carpeta como `C:\windows_exporter\`.

### 1.2 Crear un servicio de Windows (para que se ejecute en segundo plano)

Abre **PowerShell** como administrador.

Ejecuta el siguiente comando para registrarlo como un **servicio de Windows**:

```powershell
sc.exe create windows_exporter binPath= "\"C:\windows_exporter\windows_exporter-0.30.7-amd64.exe\"" start= auto
```

**Inicia** el servicio:

```powershell
Start-Service windows_exporter
```

### 1.3 Abrir el puerto 9182 en el firewall

```powershell
New-NetFirewallRule -DisplayName "Allow Windows Exporter" -Direction Inbound -LocalPort 9182 -Protocol TCP -Action Allow
```

### 1.4 Verificación
Accede desde el navegador en la máquina Windows a:

```bash
http://localhost:9182/metrics
```

---

# Configuración de Promtail para enviar registros a Loki

## 1. Descargar e instalar Promtail

1. Ve a la página oficial de lanzamientos de Loki en GitHub:
[https://github.com/grafana/loki/releases](https://github.com/grafana/loki/releases)

Como sabes, estamos utilizando la versión 3.5.0.

2. Descarga el archivo:

```
promtail-windows-amd64.exe.zip
```

3. Extrae el contenido en un directorio de trabajo:
Ejemplo:

```
C:\Promtail\
```

4. Ahora deberías tener:

```
C:\Promtail\promtail-windows-amd64.exe
```

## 2. Crear el archivo de configuración C:\Promtail\promtail-config.yaml

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
          __path__: C:\<logs>\<nombre>.log


```

Reemplaza **\<ip_servidor\>** con la dirección IP pública o privada de tu **servidor Loki**.

Reemplaza **\<logs\>** por la ruta real de tus logs y **\<nombre>** por el nombre del fichero o * si quieres que lea todos.

Reemplaza **\<appname\>**,**\<envname\>**,**\<srvname\>** con el nombre que desees asignar.

Puedes añadir **más directorios de registros**; explicamos cómo hacerlo en el **primer runbook**.


## 3. Probar Promtail manualmente

Abre PowerShell y ejecuta:

```powershell
cd C:\Promtail
.\promtail-windows-amd64.exe --config.file=promtail-config.yaml
```

Si no aparecen errores, Promtail está funcionando y enviando registros a Loki. 

## 4. Instalar Promtail como servicio de Windows con NSSM

### 4.1 Descargar y extraer NSSM

1. Descargar NSSM:
[https://nssm.cc/download](https://nssm.cc/download)

2. Extraer en:

```
C:\nssm\
```

### 4.2 Crear el servicio Promtail

Ejecuta los siguientes comandos en PowerShell:

```powershell
# Eliminar servicio antiguo (opcional)
& "C:\nssm\win64\nssm.exe" remove Promtail confirm

# Instalar servicio Promtail
& "C:\nssm\win64\nssm.exe" install Promtail `
  "C:\Promtail\promtail-windows-amd64.exe" `
  "--config.file=promtail-config.yaml"

# Establecer directorio de trabajo
& "C:\nssm\win64\nssm.exe" set Promtail AppDirectory "C:\Promtail"

# Redirigir registros de salida (opcional)
& "C:\nssm\win64\nssm.exe" set Promtail AppStdout "C:\Promtail\stdout.log"
& "C:\nssm\win64\nssm.exe" set Promtail AppStderr "C:\Promtail\stderr.log"

# Configurar el servicio para iniciar automáticamente
& "C:\nssm\win64\nssm.exe" set Promtail Start SERVICE_AUTO_START
```

### 4.3 (Opcional) Ejecutar Promtail como un usuario específico

Si la cuenta "Local System" (Sistema local) no tiene permisos para acceder a ciertas rutas de registro, configura un usuario específico:

```powershell
& "C:\nssm\win64\nssm.exe" set Promtail ObjectName ".\administrador" "contraseña"
```

Reemplaza `Administrator` y `YourPasswordHere` con tus propias credenciales.

## 5. Iniciar el servicio

Inicia el servicio Promtail:

```powershell
net start Promtail
```

O a través de `services.msc`. 

---

## 6. Aplicar cambios de configuración en el futuro

Cada vez que actualices `promtail-config.yaml`, reinicia el servicio:

```powershell
net stop Promtail
net start Promtail
```

---

## 7. Validar en Grafana

1. Abre Grafana
2. Ve a **Explore**
3. Realiza una consulta usando LogQL:

```logql
{job="appname"}
```

---

## 8. Notas

* Asegúrate de que todas las rutas utilicen barras inclinadas (`/`) o barras invertidas dobles (`\\`)
* YAML es sensible a los espacios en blanco; utiliza siempre espacios, no tabulaciones
* Promtail no recarga la configuración automáticamente; es necesario reiniciar el servicio
* Utiliza `stderr.log` para la resolución de problemas si el servicio no arranca
