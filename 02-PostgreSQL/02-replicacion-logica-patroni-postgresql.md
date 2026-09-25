
**Autor:** Pablo Mongar (Técnico ASIR)  
**Proyecto:** Replicación lógica (Patroni Publisher + PostgreSQL)  
**Contacto:** pmongber@gmail.com  
**LinkedIn:** linkedin.com/in/pmongar

# Replicación lógica (Patroni Publisher + PostgreSQL)

Publicador (Patroni Leader): <ip_publicador>  
Suscriptor (PostgreSQL 17 independiente): <ip_suscriptor>

## Configuración previa y personalización
Antes de ejecutar los comandos de esta guía, **debes reemplazar las siguientes variables genéricas** por los valores reales de tu infraestructura:

* **Direcciones IP de los nodos:**
  * `<ip_publicador>` ➔ Dirección IP del nodo líder de Patroni (Publicador, ej. `192.168.1.10`)
  * `<ip_suscriptor>` ➔ Dirección IP del servidor PostgreSQL independiente (Suscriptor, ej. `192.168.1.11`)

* **Nombres de Clúster y Base de Datos:**
  * `<cluster-name>` ➔ Nombre de tu clúster administrado en Patroni (ej. `pg-cluster`)
  * `<nombre_bd>` ➔ Nombre de la base de datos que vas a utilizar y replicar (ej. `demo`)
  * `<nombre_esquema>` ➔ Nombre del esquema de trabajo en la base de datos (ej. `lr`)

* **Objetos de Replicación:**
  * `<nombre_publicacion>` ➔ Nombre que le darás a la publicación lógica (ej. `pub_customers`)
  * `<nombre_slot>` ➔ Nombre de la ranura (*slot*) de replicación manual (ej. `slot_customers`)
  * `<nombre_suscripcion>` ➔ Nombre que le darás a la suscripción en el destino (ej. `sub_customers`)

* **Credenciales y Accesos:**
  * `<usuario_replica>` ➔ Nombre del usuario o rol de replicación (ej. `repl`)
  * `<contraseña_replica>` ➔ Contraseña segura para el usuario de replicación
  * `<nueva_contraseña_replica>` ➔ *(Opcional)* Nueva contraseña en caso de tener que actualizarla durante la solución de incidencias

## 1. Habilitar la replicación lógica en Patroni (Publicador)

Edite la configuración dinámica de Patroni:

```bash
patronictl -c /etc/patroni/patroni.yml edit-config <cluster-name>
```

Agregue o actualice la sección `postgresql`:

```yaml
postgresql:
  parameters:
    wal_level: logical
    max_wal_senders: 20
    max_replication_slots: 20
    max_logical_replication_workers: 8
    max_sync_workers_per_subscription: 4

  pg_hba:
    - host replication repl <ip_suscriptor>/32 password
    - host all all <ip_suscriptor>/32 password

```

Reinicie el nodo líder de Patroni para aplicar los cambios:

```bash
patronictl -c /etc/patroni/patroni.yml reload <cluster-name>
patronictl -c /etc/patroni/patroni.yml restart <cluster-name> --force
```

Verifique los cambios:

```SQL
SHOW wal_level;
```
```SQL 
wal_level
-----------
logical
(1 row)
```
Debe mostrar **'logical'**. Si todavía aparece como **'replica'**, debe **reiniciar el clúster** nuevamente o asegurarse de haber realizado el **cambio en la configuración** dinámica de Patroni.

```SQL
SELECT pg_is_in_recovery();
```
```SQL 
pg_is_in_recovery
-------------------
f
(1 row)
```

Debe ser falso (`f`), ya que se trata del nodo líder.

## 2. Crear usuario, base de datos y tabla en el Publicador

```SQL
CREATE ROLE <usuario_replica> WITH LOGIN PASSWORD '<contraseña_replica>' REPLICATION;
CREATE DATABASE <nombre_bd>;
\c <nombre_bd>;

CREATE SCHEMA <nombre_esquema>;

CREATE TABLE <nombre_esquema>.customers (
  id BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT UNIQUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO <nombre_esquema>.customers(name, email)
VALUES ('Usuario 1','usuario1@example.com'),('Usuario 2','usuario2@example.com');

GRANT USAGE ON SCHEMA <nombre_esquema> TO <usuario_replica>;
GRANT SELECT ON <nombre_esquema>.customers TO <usuario_replica>;
```

## 3. Crear publicación en el publicador (Publisher)

```SQL
CREATE PUBLICATION <nombre_publicacion> FOR TABLE <nombre_esquema>.customers;
```

## 4. Crear ranura (slot) de replicación lógica manual

```SQL
SELECT * FROM pg_create_logical_replication_slot('<nombre_slot>','pgoutput');
```

Verificar

```SQL
SELECT slot_name, slot_type, database, active
FROM pg_replication_slots
WHERE slot_name = '<nombre_slot>';
```
```SQL
slot_name       | slot_type | database        | active
----------------+-----------+-----------------+--------
<nombre_slot>   | logical   | <nombre_bd>     | t
(1 fila)
```

## 5. Preparar el suscriptor (Subscriber)

```SQL
CREATE DATABASE <nombre_bd>;
\c <nombre_bd>;

CREATE SCHEMA <nombre_esquema>;

CREATE TABLE <nombre_esquema>.customers (
  id BIGINT PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT UNIQUE,
  created_at TIMESTAMPTZ NOT NULL
);
```
## 6. Crear suscripción utilizando la ranura manual (Suscriptor)

```SQL
CREATE SUBSCRIPTION <nombre_suscripcion>
​​ CONNECTION 'host=<ip_publicador> port=5432 user=<usuario_replica> password=<contraseña_replica> dbname=<nombre_bd>'
 PUBLICATION <nombre_publicacion>
​​ WITH (
   slot_name = '<nombre_slot>',
   create_slot = false,
   copy_data = true,
   enabled = true
);
```

Comprobar estado:

```SQL
SELECT subname, status, last_error FROM pg_stat_subscription;
```
```SQL
subname               | status | last_error 
----------------------+--------+------------
 <nombre_suscripcion> | r      | 
(1 fila)
```

El estado r (ready) indica que la suscripción está **activa** y **replicando correctamente**.

La columna **last_error** debe aparecer **vacía** (o en blanco), lo que confirma que no hay errores de conexión, credenciales o permisos con el publicador.


Verificar los datos replicados:
```SQL
SELECT * FROM <nombre_esquema>.customers;
```
```SQL
id  | name       | email                 | created_at
----+------------+-----------------------+-------------------------------
1   | Usuario 1  | usuario1@example.com  | 2026-09-25 11:38:09.718881+02
2   | Usuario 2  | usuario2@example.com  | 2026-09-25 11:38:09.718881+02
(2 filas)
```

### 6.1 Posibles errores

Aunque el estado sea ready, si last_error nos devuelve información, significa que ha habido un error relacionado con la conexion o las credenciales.

#### 6.1.1 Error de autenticación

```SQL
could not connect to publisher: FATAL: password authentication failed for user "<usuario_replica>"
```

- **Causa probable:** La contraseña del usuario de replicación configurada en el suscriptor no coincide con la creada en el publicador, o el usuario no tiene permisos de REPLICATION.

- **Solución:** Vuelve a establecer la contraseña del usuario en el publicador (ALTER ROLE <usuario_replica> WITH PASSWORD '<nueva_contraseña_replica>';) y actualiza la cadena de conexión en el suscriptor.

#### 6.1.2 Error de red o conectividad

```SQL
could not connect to publisher: connection to server at "<ip_publicador>", port 5432 failed: Connection refused
```

- **Causa probable:** El servidor publicador está apagado, el servicio PostgreSQL no acepta conexiones en esa IP, hay un firewall bloqueando el puerto, o las reglas del archivo pg_hba.conf en el publicador no permiten la conexión desde el suscriptor (<ip_publicador>).

- **Solución:** Comprueba la conectividad de red con ping o nc -zv <ip_publicador> 5432 y revisa que el pg_hba.conf del publicador incluya la IP correcta.

## 7. Probar la replicación

Insertar datos en el publicador:
```SQL
INSERT INTO <nombre_esquema>.customers (name, email)
VALUES ('Prueba', 'test@example.com');
```

Verificar en el suscriptor:
```SQL
SELECT * FROM <nombre_esquema>.customers;
```
```SQL
id  | name        | email                     | created_at
----+-------------+---------------------------+-------------------------------
1   | Usuario 1   | usuario1@example.com      | 2026-09-25 12:38:09.718881+02
2   | Usuario 2   | usuario2@example.com      | 2026-09-25 12:38:09.718881+02
3   | Prueba      | test@example.com          | 2026-09-25 12:40:28.628028+02
(3 filas)
```

## 8. Comandos útiles

### 8.1. Eliminar la suscripción de forma segura (PostgreSQL 17+)
```SQL
ALTER SUBSCRIPTION <nombre_suscripcion> DISABLE;
ALTER SUBSCRIPTION <nombre_suscripcion> SET (slot_name = NONE);
DROP SUBSCRIPTION <nombre_suscripcion>;
```

### 8.2. Eliminar ranura manual
```SQL
SELECT pg_drop_replication_slot('<nombre_slot>');
```

## 9. Notas

- La replicación lógica requiere que las tablas tengan una clave primaria (PRIMARY KEY).
-
- Las ranuras manuales deben eliminarse cuando no se utilicen para evitar el crecimiento excesivo de los registros WAL.
-
- La replicación es asíncrona por defecto.
-
- La ranura debe crearse en la misma base de datos que utiliza la suscripción (<nombre_bd>).
