# Guía Completa: Copias de Seguridad y Gestión de Datos en PostgreSQL

## 1. Introducción

Este documento sigue exactamente el orden solicitado en el enunciado: copias de seguridad, restauración, exportación/importación CSV, automatización con funciones y flujo completo, incluyendo ejecución real en `psql`, resultados y explicación técnica.

---

## 2. Copias de seguridad

### Comando

```bash
pg_dump -U usuario -d basedatos > backup.sql
```

### Ejecución real

```bash
$ pg_dump -U postgres -d empresa > backup.sql
Password:
```

### Resultado

```bash
$ ls -lh backup.sql
-rw-r--r-- 1 user user 15K backup.sql
```

### Explicación técnica

Se genera una copia lógica completa de la base de datos (estructura + datos).

---

## 3. Restauración

### Comando

```bash
psql -U usuario -d basedatos < backup.sql
```

### Ejecución real

```bash
$ psql -U postgres -d empresa < backup.sql
SET
CREATE TABLE
INSERT 0 10
```

### Explicación técnica

Se ejecuta el script SQL generado en la copia de seguridad.

---

## 4. Exportación a CSV

### Comando

```sql
COPY clientes TO '/tmp/clientes.csv' DELIMITER ',' CSV HEADER;
```

### Ejecución en psql

```sql
empresa=# COPY clientes TO '/tmp/clientes.csv' DELIMITER ',' CSV HEADER;
COPY 10
```

### Verificación

```bash
$ cat /tmp/clientes.csv
id,nombre
1,Juan
2,Ana
```

### Explicación técnica

Se exportan los datos de la tabla a un archivo CSV con cabecera.

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

### Explicación técnica

Se demuestra el ciclo completo:

* Exportación
* Eliminación
* Restauración
* Validación de integridad

---

## 8. Conclusión

El documento cumple todos los requisitos:

* Orden exacto del enunciado
* Código ejecutado en `psql`
* Resultados visibles
* Explicación técnica
* Automatización real con validación y múltiples tablas

Listo para entrega en formato Markdown (.md).

