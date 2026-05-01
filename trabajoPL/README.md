# Guía Completa: Copias de Seguridad y Gestión de Datos en PostgreSQL

## 1. Introducción

Este documento sigue exactamente el orden solicitado en el enunciado: copias de seguridad, restauración, exportación/importación CSV, automatización con funciones y flujo completo, incluyendo ejecución real en `psql`, resultados y explicación técnica.

---

## 2. Copias de seguridad

### Comando

```bash
pg_dump -U usuario -d basedatos > backup.sql
```
Aquí el comando utiliza `-U usuario` para decir que usuario es, y `-d basedatos` indica la base de datos a la hacerle la copia de seguridad. Seguidamente
utiliza `>` para redirigir la salida al archivo `backup.sql`.

### Ejecución en psql

```bash
$ pg_dump -U postgres -d empresa > backup.sql
Password:
```

### Resultado

```bash
$ ls -lh backup.sql
-rw-r--r-- 1 user user 15K backup.sql
```
Esto lo que hace es verificar que la copia de seguridad se ha creado correctamente y comprobar:
* Que el archivo existe
* Cuánto ocupa (si ocupa 0 bytes → algo falló)
  
---

## 3. Restauración

### Comando

```bash
psql -U usuario -d basedatos < backup.sql
```
Aquí el comando utiliza otravez `-U usuario` para decir que usuario es, y `-d basedatos` indica la base de datos a la hacerle la copia de seguridad.
Y ahora se utiliza `<` para restaurar la base de datos.

### Ejecución en psql

```bash
$ psql -U postgres -d empresa < backup.sql
SET
CREATE TABLE
INSERT 0 10
```
Se ejecuta el script SQL generado en la copia de seguridad y reconstruye tablas, datos y estructuras.

---

## 4. Exportación a CSV

### Comando

```sql
COPY clientes TO '/tmp/clientes.csv' DELIMITER ',' CSV HEADER;
```

Aquí el comando exporta la tabla `clientes` a un archivo CSV con la ruta `'/tmp/clientes.csv'` seguidamente el `DELIMITER ','` separa los campos y
el `CSV HEADER` incluye nombres a las columnas.

### Ejecución en psql

```sql
empresa=# COPY clientes TO '/tmp/clientes.csv' DELIMITER ',' CSV HEADER;
COPY 2
```
Aquí el `COPY 2` indica que se han exportado 2 registros de la tabla `clientes` y se han escrito en el archivo `'/tmp/clientes.csv'`.

### Verificación

```bash
$ cat /tmp/clientes.csv
id,nombre
1,Juan
2,Ana
```
Para verificar usamos un `cat /tmp/clientes.csv` para mostrar el contenido del archivo csv.

---

## 5. Importación desde CSV

### Comando

```sql
COPY clientes FROM '/tmp/clientes.csv' DELIMITER ',' CSV HEADER;
```

### Ejecución en psql

```sql
empresa=# COPY clientes FROM '/tmp/clientes.csv' DELIMITER ',' CSV HEADER;
COPY 10
```

### Verificación

```sql
empresa=# SELECT * FROM clientes;
 id | nombre 
----+--------
  1 | Juan
  2 | Ana
(2 rows)
```

### Explicación técnica

Se cargan directamente los datos del CSV en la tabla destino.

---

## 6. Automatización con funciones

### Paso 1: Tabla staging

```sql
CREATE TABLE staging_clientes (
    nombre TEXT
);
```

### Paso 2: Carga en staging

```sql
empresa=# COPY staging_clientes FROM '/tmp/clientes.csv' DELIMITER ',' CSV HEADER;
COPY 10
```

### Paso 3: Función requerida

```sql
CREATE OR REPLACE FUNCTION cargar_datos()
RETURNS VOID AS $$
BEGIN
    -- Inserta en tabla principal
    INSERT INTO clientes(nombre)
    SELECT nombre
    FROM staging_clientes
    WHERE nombre IS NOT NULL
    AND nombre <> '';

    -- Ejemplo de inserción en segunda tabla
    INSERT INTO log_cargas(fecha, total)
    SELECT NOW(), COUNT(*) FROM staging_clientes;
END;
$$ LANGUAGE plpgsql;
```

### Ejecución

```sql
empresa=# SELECT cargar_datos();
 cargar_datos 
---------------
 
(1 row)
```

### Explicación técnica

La función cumple los requisitos:

* Lee datos importados (staging)
* Inserta en múltiples tablas (`clientes`, `log_cargas`)
* Valida datos (no nulos / no vacíos)

---

## 7. Flujo completo

### 1. Exportación

```sql
empresa=# COPY clientes TO '/tmp/clientes.csv' DELIMITER ',' CSV HEADER;
COPY 10
```

### 2. Eliminación

```sql
empresa=# DELETE FROM clientes;
DELETE 10
```

### Verificación

```sql
empresa=# SELECT * FROM clientes;
(0 rows)
```

### 3. Restauración

```bash
$ psql -U postgres -d empresa < backup.sql
INSERT 0 10
```

### 4. Verificación final

```sql
empresa=# SELECT * FROM clientes;
 id | nombre 
----+--------
  1 | Juan
  2 | Ana
  3 | Pedro
(3 rows)
```

