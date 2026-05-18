# Reestructuración de consultas con CTE y subqueries

## Metadatos

| Campo | Detalle |
|---|---|
| **Duración estimada** | 60 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (*Apply*) |
| **Módulo** | 1 — Reestructuración y legibilidad de consultas SQL |
| **Plataforma** | Snowflake (Snowsight Worksheet) |
| **Schema de práctica** | `LAB_SQL_INTERMEDIO.VENTAS` |

---

## Descripción General

En este laboratorio partirás de consultas SQL monolíticas y difíciles de mantener, y las reestructurarás progresivamente usando dos herramientas fundamentales: **subconsultas** (en cláusulas `SELECT`, `WHERE` y `FROM`) y **Common Table Expressions (CTEs)**. A lo largo de cuatro ejercicios encadenados, aplicarás los conceptos de subconsultas correlacionadas y no correlacionadas vistos en la Lección 1.1, y luego transformarás esas mismas consultas en estructuras `WITH` para comparar legibilidad y mantenibilidad. Al finalizar, tendrás criterios concretos para decidir cuándo usar cada enfoque en un entorno real de negocio.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Identificar y escribir subconsultas no correlacionadas en cláusulas `WHERE` (con `IN`, `NOT IN`, `EXISTS`) y subconsultas escalares en `SELECT` para calcular métricas derivadas.
- [ ] Reescribir consultas anidadas complejas usando CTEs encadenadas (`WITH ... AS`) para mejorar la legibilidad y permitir la reutilización de bloques lógicos.
- [ ] Distinguir el comportamiento de una subconsulta correlacionada frente a una no correlacionada y explicar el impacto de cada una en la ejecución.
- [ ] Comparar lado a lado la versión con subquery y la versión con CTE de una misma consulta, y argumentar cuál es más apropiada según el contexto.

---

## Prerrequisitos

### Conocimientos previos

| Área | Nivel requerido |
|---|---|
| `SELECT` con `INNER JOIN` y `LEFT JOIN` | Dominio completo |
| `GROUP BY` con `SUM`, `COUNT`, `AVG` | Dominio completo |
| Orden lógico de evaluación de una consulta SQL | Comprensión básica |
| Concepto de subconsulta (definición y clasificación) | Lección 1.1 completada |

### Acceso y configuración

| Requisito | Detalle |
|---|---|
| Cuenta Snowflake activa | Trial o corporativa con rol `SYSADMIN` o `ACCOUNTADMIN` |
| Script de setup ejecutado | No se asume script previo. Esta práctica incluye el setup completo de base, schema, tablas y datos. |
| Database disponible | `LAB_SQL_INTERMEDIO` |
| Schema disponible | `LAB_SQL_INTERMEDIO.VENTAS` |
| Tablas requeridas | `CLIENTES`, `PEDIDOS`, `PRODUCTOS`, creadas en el Paso 0 |
| Warehouse activo | `COMPUTE_WH` (tamaño `X-SMALL`) |

---

## Entorno de Laboratorio

### Hardware recomendado

| Componente | Mínimo | Recomendado |
|---|---|---|
| Procesador | Intel Core i5 / AMD Ryzen 5 (64-bit) | Intel Core i7 / AMD Ryzen 7 |
| RAM | 8 GB | 16 GB |
| Almacenamiento libre | 500 MB | 2 GB |
| Conexión a Internet | 10 Mbps | 25 Mbps |
| Resolución de pantalla | 1280×768 | 1920×1080 |

### Software requerido

| Software | Versión mínima | Uso |
|---|---|---|
| Navegador web (Chrome / Firefox / Edge / Safari) | 110+ / 110+ / 110+ / 16+ | Acceso a Snowsight |
| Snowflake (Snowsight) | Versión web actual | Ejecución de consultas |
| Visual Studio Code *(opcional)* | 1.80+ | Edición de scripts localmente |
| SnowSQL *(opcional)* | 1.2.x+ | Ejecución desde terminal |


### Organización recomendada de Workspace en Snowsight

Para que la práctica sea ordenada y reutilizable, trabaja con un Workspace y 2 folders. En esta práctica se usa la palabra **workspace** como una separación lógica de trabajo dentro de Snowsight; técnicamente, en Snowflake trabajarás con **Workspace**.

| Workspace / Worksheet | Folder | Nombre sugerido | Uso |
|---|---|---|---|
| SNOWLABS-INT | SETUP-LABS | `01_SETUP_DATOS_CTE_SUBQUERIES` | Crear database, schema, tablas y datos de prueba. Se ejecuta una vez al inicio o cuando quieras reiniciar el laboratorio. |
| SNOWLABS-INT | SCRIPT-LABS | `02_LAB_CTE_SUBQUERIES` | Ejecutar los ejercicios del laboratorio sin mezclar el script de carga de datos. |

### Paso 0.0 — Crear el workspace de las prácticas

1. Entra a **Snowsight**.
2. Da clic en la opción **Projects**
3. Clic en **+**.
4. Luego selecciona la opción **Private workspace**
5. Nómbralo: **`SNOWLABS-INT`**
6. Clic en **Create**


#### Paso 0.0.1 — Crear el Folder y script que carga los datos

1. Ahora dentro del nuevo workspace da clic en **+ Add new**
2. Clic en **Folder** y nombralo: **`SETUP-LABS`**
3. Dentro del Folder **SETUP-LABS** da clic en el simbolo **+**
4. Crea un archivo de tipo **SQL**
5. Nómbralo: **`01_SETUP_DATOS_CTE_SUBQUERIES`**.
6. Pega ahí el siguiente script completo.
7. Ejecuta el script completo antes de comenzar el laboratorio.

Este dataset está diseñado para activar todos los escenarios de la práctica:

- Productos vendidos durante 2024 para practicar `IN` con subconsulta no correlacionada.
- Productos nunca vendidos para practicar `NOT EXISTS`.
- Clientes con dos pedidos para comparar pedidos individuales contra el promedio del mismo cliente.
- Clientes distribuidos por ciudad para calcular promedios por ciudad usando CTEs encadenadas.
- `PEDIDOS.PRODUCTO_ID` sin valores `NULL`, para que las validaciones con `NOT IN` no fallen por comportamiento de valores nulos.

```sql
-- 01_setup_datos_cte_subqueries_snowflake.sql
-- Práctica Snowflake Intermedio
-- Dataset mínimo para completar el laboratorio:
-- Reestructuración de consultas con CTE y subqueries
--
-- Objetivo del dataset:
-- 1) Tener productos vendidos en 2024.
-- 2) Tener productos nunca vendidos.
-- 3) Tener varios pedidos por cliente para comparar pedidos contra promedio individual.
-- 4) Tener clientes por ciudad para calcular promedio por ciudad.
-- 5) Evitar NULL en PEDIDOS.PRODUCTO_ID para que la validación con NOT IN sea correcta.

USE WAREHOUSE COMPUTE_WH;

CREATE DATABASE IF NOT EXISTS LAB_SQL_INTERMEDIO;
USE DATABASE LAB_SQL_INTERMEDIO;

CREATE SCHEMA IF NOT EXISTS VENTAS;
USE SCHEMA VENTAS;

-- Opcional para repetir el laboratorio desde cero.
DROP TABLE IF EXISTS PEDIDOS;
DROP TABLE IF EXISTS PRODUCTOS;
DROP TABLE IF EXISTS CLIENTES;

CREATE OR REPLACE TABLE CLIENTES (
    CLIENTE_ID NUMBER(10,0) NOT NULL,
    NOMBRE VARCHAR(100) NOT NULL,
    CIUDAD VARCHAR(80) NOT NULL,
    SEGMENTO VARCHAR(40),
    FECHA_ALTA DATE,
    CONSTRAINT PK_CLIENTES PRIMARY KEY (CLIENTE_ID)
);

CREATE OR REPLACE TABLE PRODUCTOS (
    PRODUCTO_ID NUMBER(10,0) NOT NULL,
    NOMBRE VARCHAR(120) NOT NULL,
    CATEGORIA VARCHAR(80) NOT NULL,
    PRECIO_UNITARIO NUMBER(10,2) NOT NULL,
    ACTIVO BOOLEAN DEFAULT TRUE,
    CONSTRAINT PK_PRODUCTOS PRIMARY KEY (PRODUCTO_ID)
);

CREATE OR REPLACE TABLE PEDIDOS (
    PEDIDO_ID NUMBER(10,0) NOT NULL,
    CLIENTE_ID NUMBER(10,0) NOT NULL,
    PRODUCTO_ID NUMBER(10,0) NOT NULL,
    FECHA_PEDIDO DATE NOT NULL,
    CANTIDAD NUMBER(10,0) NOT NULL,
    MONTO NUMBER(12,2) NOT NULL,
    CANAL VARCHAR(40),
    CONSTRAINT PK_PEDIDOS PRIMARY KEY (PEDIDO_ID),
    CONSTRAINT FK_PEDIDOS_CLIENTES FOREIGN KEY (CLIENTE_ID) REFERENCES CLIENTES(CLIENTE_ID),
    CONSTRAINT FK_PEDIDOS_PRODUCTOS FOREIGN KEY (PRODUCTO_ID) REFERENCES PRODUCTOS(PRODUCTO_ID)
);

INSERT INTO CLIENTES (CLIENTE_ID, NOMBRE, CIUDAD, SEGMENTO, FECHA_ALTA) VALUES
    (1,  'Ana Torres',       'CDMX',        'Retail',     '2023-01-15'),
    (2,  'Luis Martínez',    'CDMX',        'Retail',     '2023-03-02'),
    (3,  'María López',      'Guadalajara', 'PyME',       '2023-02-20'),
    (4,  'Carlos Hernández', 'Guadalajara', 'Enterprise', '2023-05-10'),
    (5,  'Sofía Ramírez',    'Monterrey',   'PyME',       '2023-06-18'),
    (6,  'Jorge Castillo',   'Monterrey',   'Retail',     '2023-07-22'),
    (7,  'Elena Flores',     'Puebla',      'Enterprise', '2023-08-04'),
    (8,  'Diego Sánchez',    'Puebla',      'PyME',       '2023-09-11'),
    (9,  'Valeria Cruz',     'Mérida',      'Enterprise', '2023-10-05'),
    (10, 'Roberto Díaz',     'Mérida',      'Retail',     '2023-11-01');

INSERT INTO PRODUCTOS (PRODUCTO_ID, NOMBRE, CATEGORIA, PRECIO_UNITARIO, ACTIVO) VALUES
    (1,  'Laptop Pro 14',             'Electrónica', 1200.00, TRUE),
    (2,  'Monitor 27 pulgadas',       'Electrónica',  350.00, TRUE),
    (3,  'Teclado mecánico',          'Accesorios',   120.00, TRUE),
    (4,  'Mouse inalámbrico',         'Accesorios',    80.00, TRUE),
    (5,  'Silla ergonómica',          'Oficina',      420.00, TRUE),
    (6,  'Escritorio ajustable',      'Oficina',      680.00, TRUE),
    (7,  'Licencia BI anual',         'Software',     900.00, TRUE),
    (8,  'Licencia CRM anual',        'Software',    1100.00, TRUE),
    (9,  'Servidor compacto',         'Infraestructura', 1500.00, TRUE),
    (10, 'NAS 8TB',                   'Infraestructura', 850.00, TRUE),
    -- Estos productos quedan intencionalmente sin ventas para el ejercicio NOT EXISTS.
    (11, 'Tablet Ejecutiva',          'Electrónica',  550.00, TRUE),
    (12, 'Audífonos con cancelación', 'Accesorios',   220.00, TRUE);

INSERT INTO PEDIDOS (PEDIDO_ID, CLIENTE_ID, PRODUCTO_ID, FECHA_PEDIDO, CANTIDAD, MONTO, CANAL) VALUES
    (1001, 1,  1,  '2024-01-12', 1,  500.00,  'Web'),
    (1002, 1,  7,  '2024-03-15', 1, 1000.00,  'Ejecutivo'),

    (1003, 2,  3,  '2024-02-03', 2,  200.00,  'Web'),
    (1004, 2,  4,  '2023-11-21', 3,  500.00,  'Web'),

    (1005, 3,  2,  '2024-04-09', 1,  300.00,  'Marketplace'),
    (1006, 3,  5,  '2025-01-18', 1,  600.00,  'Ejecutivo'),

    (1007, 4,  8,  '2024-05-06', 1,  700.00,  'Ejecutivo'),
    (1008, 4,  9,  '2024-09-19', 1,  900.00,  'Ejecutivo'),

    (1009, 5,  6,  '2023-12-05', 1,  500.00,  'Web'),
    (1010, 5,  7,  '2024-06-14', 1,  600.00,  'Marketplace'),

    (1011, 6,  4,  '2024-07-07', 1,  100.00,  'Web'),
    (1012, 6,  3,  '2024-08-25', 1,  200.00,  'Web'),

    (1013, 7,  5,  '2024-10-02', 1,  400.00,  'Ejecutivo'),
    (1014, 7,  8,  '2025-02-12', 1,  900.00,  'Ejecutivo'),

    (1015, 8,  2,  '2024-11-17', 1,  650.00,  'Marketplace'),
    (1016, 8,  6,  '2024-12-09', 1,  650.00,  'Marketplace'),

    (1017, 9,  9,  '2024-01-28', 1,  800.00,  'Ejecutivo'),
    (1018, 9,  1,  '2025-03-03', 1, 1200.00,  'Ejecutivo'),

    (1019, 10, 10, '2023-10-30', 1,  200.00,  'Web'),
    (1020, 10, 5,  '2024-04-22', 1,  400.00,  'Web');

-- Validación rápida del dataset.
SELECT 'CLIENTES' AS TABLA, COUNT(*) AS FILAS FROM CLIENTES
UNION ALL
SELECT 'PRODUCTOS' AS TABLA, COUNT(*) AS FILAS FROM PRODUCTOS
UNION ALL
SELECT 'PEDIDOS' AS TABLA, COUNT(*) AS FILAS FROM PEDIDOS;

-- Resultado esperado:
-- CLIENTES  = 10
-- PRODUCTOS = 12
-- PEDIDOS   = 20

-- Validación de productos nunca vendidos.
SELECT
    (SELECT COUNT(*) FROM PRODUCTOS) AS TOTAL_PRODUCTOS,
    (SELECT COUNT(DISTINCT PRODUCTO_ID) FROM PEDIDOS) AS PRODUCTOS_CON_PEDIDOS,
    (SELECT COUNT(*) FROM PRODUCTOS PR
     WHERE NOT EXISTS (
        SELECT 1
        FROM PEDIDOS PE
        WHERE PE.PRODUCTO_ID = PR.PRODUCTO_ID
     )) AS PRODUCTOS_SIN_PEDIDOS;

-- Resultado esperado:
-- TOTAL_PRODUCTOS = 12
-- PRODUCTOS_CON_PEDIDOS = 10
-- PRODUCTOS_SIN_PEDIDOS = 2

-- Validación de distribución por ciudad para el Ejercicio 4.
WITH TOTALES_CLIENTE AS (
    SELECT CLIENTE_ID, SUM(MONTO) AS MONTO_TOTAL
    FROM PEDIDOS
    GROUP BY CLIENTE_ID
)
SELECT
    C.CIUDAD,
    COUNT(*) AS CLIENTES_CON_PEDIDOS,
    ROUND(AVG(TC.MONTO_TOTAL), 2) AS PROMEDIO_CIUDAD
FROM CLIENTES C
INNER JOIN TOTALES_CLIENTE TC
    ON C.CLIENTE_ID = TC.CLIENTE_ID
GROUP BY C.CIUDAD
ORDER BY C.CIUDAD;
```

#### Paso 0.0.2 — Crear el folder y script de laboratorio

1. Da clic en el botón **+ Add new**
2. Clic en **Folder** y nombralo: **`SCRIPT-LABS`**.
3. Dentro de **SCRIPT_LABS** crea un archivo de tipo **SQL**
4. Nómbralo: **`01_LAB_CTE_SUBQUERIES`**.
5. Usa este archivo para ejecutar los ejercicios 1, 2, 3 y 4.
6. **No pegues aquí el script de carga completo; solo usa las consultas de análisis del laboratorio.**

---

### Paso 0.2 — Confirmar que las tablas quedaron disponibles

1. Dento del archivo **01_LAB_CTE_SUBQUERIES** ejecuta lo siguiente:

```sql
USE WAREHOUSE COMPUTE_WH;
USE DATABASE LAB_SQL_INTERMEDIO;
USE SCHEMA VENTAS;

SHOW TABLES;
```

**Resultado esperado:** deben aparecer al menos estas tablas:

| Tabla | Uso en la práctica |
|---|---|
| `CLIENTES` | Datos maestros de clientes: nombre, ciudad, segmento y fecha de alta. |
| `PRODUCTOS` | Catálogo de productos, incluyendo productos vendidos y no vendidos. |
| `PEDIDOS` | Hechos transaccionales: cliente, producto, fecha, cantidad, monto y canal. |

### Paso 0.3 — Validar volumen mínimo de datos

Ejecuta:

```sql
SELECT 'CLIENTES' AS TABLA, COUNT(*) AS FILAS FROM CLIENTES
UNION ALL
SELECT 'PRODUCTOS' AS TABLA, COUNT(*) AS FILAS FROM PRODUCTOS
UNION ALL
SELECT 'PEDIDOS' AS TABLA, COUNT(*) AS FILAS FROM PEDIDOS;
```

**Resultado esperado:**

| TABLA | FILAS |
|---|---:|
| CLIENTES | 10 |
| PRODUCTOS | 12 |
| PEDIDOS | 20 |

### Paso 0.4 — Validar que hay productos sin ventas

Ejecuta:

```sql
SELECT
    (SELECT COUNT(*) FROM PRODUCTOS) AS TOTAL_PRODUCTOS,
    (SELECT COUNT(DISTINCT PRODUCTO_ID) FROM PEDIDOS) AS PRODUCTOS_CON_PEDIDOS,
    (SELECT COUNT(*)
     FROM PRODUCTOS PR
     WHERE NOT EXISTS (
        SELECT 1
        FROM PEDIDOS PE
        WHERE PE.PRODUCTO_ID = PR.PRODUCTO_ID
     )) AS PRODUCTOS_SIN_PEDIDOS;
```

**Resultado esperado:**

| TOTAL_PRODUCTOS | PRODUCTOS_CON_PEDIDOS | PRODUCTOS_SIN_PEDIDOS |
|---:|---:|---:|
| 12 | 10 | 2 |

### Paso 0.5 — Validar que existen pedidos de 2024

Ejecuta:

```sql
SELECT
    YEAR(FECHA_PEDIDO) AS ANIO,
    COUNT(*) AS TOTAL_PEDIDOS,
    SUM(MONTO) AS MONTO_TOTAL
FROM PEDIDOS
GROUP BY YEAR(FECHA_PEDIDO)
ORDER BY ANIO;
```

**Resultado esperado:** debe haber registros para `2023`, `2024` y `2025`. El año `2024` debe tener suficientes pedidos para que el Ejercicio 2.1 devuelva productos vendidos en ese año.

---

## Ejercicios Paso a Paso

---

### Ejercicio 1 — Exploración del esquema y consulta base monolítica

**Objetivo:** Familiarizarte con las tablas del laboratorio y analizar una consulta monolítica que será el punto de partida para los ejercicios siguientes.

#### Instrucciones

**Paso 1.1 — Explorar la estructura de las tablas**

Ejecuta las siguientes consultas de reconocimiento para entender el esquema con el que trabajarás durante todo el laboratorio:

```sql
-- Estructura de CLIENTES
DESCRIBE TABLE CLIENTES;

-- Estructura de PEDIDOS
DESCRIBE TABLE PEDIDOS;

-- Estructura de PRODUCTOS
DESCRIBE TABLE PRODUCTOS;
```

**Paso 1.2 — Revisar una muestra de datos de cada tabla**

```sql
-- Muestra de CLIENTES
SELECT * FROM CLIENTES LIMIT 5;

-- Muestra de PEDIDOS (incluye monto y cliente asociado)
SELECT * FROM PEDIDOS LIMIT 5;

-- Muestra de PRODUCTOS
SELECT * FROM PRODUCTOS LIMIT 5;
```

**Paso 1.3 — Analizar la consulta monolítica de partida**

La siguiente consulta responde a la pregunta de negocio: 

-**"¿Qué clientes han realizado pedidos por un monto total superior al promedio general de todos los pedidos, y cuántos pedidos tiene cada uno?"**

Ejecuta esta consulta **sin modificarla** y observa su estructura y resultado:

```sql
-- CONSULTA MONOLÍTICA DE PARTIDA (versión difícil de leer)
SELECT
    C.CLIENTE_ID,
    C.NOMBRE,
    C.CIUDAD,
    COUNT(P.PEDIDO_ID)    AS TOTAL_PEDIDOS,
    SUM(P.MONTO)          AS MONTO_TOTAL
FROM CLIENTES C
INNER JOIN PEDIDOS P
    ON C.CLIENTE_ID = P.CLIENTE_ID
WHERE C.CLIENTE_ID IN (
    SELECT CLIENTE_ID
    FROM PEDIDOS
    GROUP BY CLIENTE_ID
    HAVING SUM(MONTO) > (
        SELECT AVG(MONTO_CLIENTE)
        FROM (
            SELECT
                CLIENTE_ID,
                SUM(MONTO) AS MONTO_CLIENTE
            FROM PEDIDOS
            GROUP BY CLIENTE_ID
        ) AS TOTALES
    )
)
GROUP BY C.CLIENTE_ID, C.NOMBRE, C.CIUDAD
ORDER BY MONTO_TOTAL DESC;
```

**Salida esperada:** Una tabla con columnas `CLIENTE_ID`, `NOMBRE`, `CIUDAD`, `TOTAL_PEDIDOS` y `MONTO_TOTAL`, mostrando únicamente los clientes cuyo monto total supera el promedio. El número de filas variará según los datos cargados.

**Verificación:**

Anota mentalmente (o en un comentario SQL) las respuestas a estas preguntas antes de continuar:

```sql
-- Preguntas de análisis — responde en comentarios:
-- 1. ¿Cuántos niveles de anidamiento tiene esta consulta?
-- 2. ¿La subconsulta más interna es correlacionada o no correlacionada?
-- 3. ¿Cuántas veces se lee la tabla PEDIDOS en esta consulta?
-- 4. ¿Qué parte de la consulta sería más difícil de modificar si cambia el negocio?
```

> 💡 **Reflexión:** Esta consulta funciona correctamente, pero tiene tres niveles de anidamiento y lee la tabla `PEDIDOS` al menos dos veces. En los ejercicios siguientes la reestructurarás de forma progresiva.

---

### Ejercicio 2 — Subconsultas en `WHERE` y `SELECT`

**Objetivo:** Practicar el uso de subconsultas no correlacionadas y correlacionadas en las cláusulas `WHERE` y `SELECT`, resolviendo preguntas de negocio específicas.

#### Instrucciones

**Paso 2.1 — Subconsulta no correlacionada en `WHERE` con `IN`**

Pregunta de negocio: 

- **"¿Qué productos han sido incluidos en al menos un pedido durante el año 2024?"**

```sql
-- Subconsulta no correlacionada con IN
-- La subconsulta interna se ejecuta UNA SOLA VEZ y devuelve una lista de PRODUCTO_ID
SELECT
    PRODUCTO_ID,
    NOMBRE,
    CATEGORIA,
    PRECIO_UNITARIO
FROM PRODUCTOS
WHERE PRODUCTO_ID IN (
    SELECT DISTINCT PRODUCTO_ID
    FROM PEDIDOS
    WHERE YEAR(FECHA_PEDIDO) = 2024
)
ORDER BY CATEGORIA, NOMBRE;
```

**Salida esperada:** Lista de productos con al menos un pedido en 2024. Observa que la subconsulta interna es completamente independiente de la consulta externa (no referencia ninguna columna de `PRODUCTOS`).

---

**Paso 2.2 — Subconsulta correlacionada en `WHERE` con `NOT EXISTS`**

Pregunta de negocio: 

- **"¿Qué productos NUNCA han sido vendidos?"**

```sql
-- Subconsulta con NOT EXISTS (más segura que NOT IN cuando puede haber NULLs)
SELECT
    PR.PRODUCTO_ID,
    PR.NOMBRE,
    PR.CATEGORIA
FROM PRODUCTOS PR
WHERE NOT EXISTS (
    SELECT 1
    FROM PEDIDOS PE
    WHERE PE.PRODUCTO_ID = PR.PRODUCTO_ID  -- referencia a la consulta externa
)
ORDER BY PR.CATEGORIA, PR.NOMBRE;
```

> ⚠️ **Nota técnica:** Esta subconsulta **sí es correlacionada** porque referencia `PR.PRODUCTO_ID` de la consulta externa. Se ejecuta una vez por cada fila de `PRODUCTOS`. Compárala con el `IN` del paso anterior para notar la diferencia estructural.

**Salida esperada:** Lista de productos sin ningún registro en la tabla `PEDIDOS`. Si todos los productos tienen pedidos, el resultado será vacío (0 filas), lo cual es un resultado válido.

**Verificación:**

```sql
-- Verifica que la lógica es correcta: el total de productos con pedidos
-- más los productos sin pedidos debe sumar el total de productos
SELECT COUNT(*) AS TOTAL_PRODUCTOS FROM PRODUCTOS;

SELECT COUNT(DISTINCT PRODUCTO_ID) AS PRODUCTOS_CON_PEDIDOS FROM PEDIDOS;
```

---

**Paso 2.3 — Subconsulta escalar en `SELECT`**

Pregunta de negocio: 

- **"Para cada cliente, ¿cuánto representa su monto total de pedidos respecto al monto total global de todos los pedidos?"**

```sql
-- Subconsulta escalar en SELECT: se ejecuta una vez y su valor se reutiliza en cada fila
SELECT
    C.CLIENTE_ID,
    C.NOMBRE,
    C.CIUDAD,
    SUM(P.MONTO)                                          AS MONTO_CLIENTE,
    (SELECT SUM(MONTO) FROM PEDIDOS)                      AS MONTO_GLOBAL,
    ROUND(
        SUM(P.MONTO) / (SELECT SUM(MONTO) FROM PEDIDOS) * 100,
        2
    )                                                     AS PORCENTAJE_PARTICIPACION
FROM CLIENTES C
INNER JOIN PEDIDOS P
    ON C.CLIENTE_ID = P.CLIENTE_ID
GROUP BY C.CLIENTE_ID, C.NOMBRE, C.CIUDAD
ORDER BY PORCENTAJE_PARTICIPACION DESC;
```

**Salida esperada:** Tabla con el monto de cada cliente, el monto global total y el porcentaje de participación de cada cliente. La suma de la columna `PORCENTAJE_PARTICIPACION` debe ser aproximadamente `100.00`.

> 💡 **Observación:** La subconsulta `(SELECT SUM(MONTO) FROM PEDIDOS)` aparece **dos veces** en la consulta. Esto es una señal de que una CTE podría mejorar este código — lo verás en el Ejercicio 3.

---

**Paso 2.4 — Subconsulta correlacionada en `WHERE` para comparar contra promedio individual**

Pregunta de negocio: 

- **"¿Qué pedidos individuales superan el monto promedio de todos los pedidos realizados por ese mismo cliente?"**

```sql
-- Subconsulta correlacionada: el promedio se calcula POR CADA CLIENTE
-- La subconsulta se ejecuta una vez por cada fila de la consulta externa
SELECT
    P1.PEDIDO_ID,
    P1.CLIENTE_ID,
    C.NOMBRE                          AS NOMBRE_CLIENTE,
    P1.MONTO,
    P1.FECHA_PEDIDO
FROM PEDIDOS P1
INNER JOIN CLIENTES C
    ON P1.CLIENTE_ID = C.CLIENTE_ID
WHERE P1.MONTO > (
    SELECT AVG(P2.MONTO)
    FROM PEDIDOS P2
    WHERE P2.CLIENTE_ID = P1.CLIENTE_ID  -- correlación: referencia a P1 de la consulta externa
)
ORDER BY P1.CLIENTE_ID, P1.MONTO DESC;
```

**Salida esperada:** Lista de pedidos donde cada fila representa un pedido que supera el promedio de pedidos de ese cliente específico. Un cliente con tres pedidos de montos 100, 200 y 300 solo verá el pedido de 300 (que supera el promedio de 200).

**Verificación:**

```sql
-- Comprueba manualmente el resultado para un cliente específico
-- Reemplaza '1' con un CLIENTE_ID real de tu resultado anterior
SELECT
    CLIENTE_ID,
    PEDIDO_ID,
    MONTO,
    AVG(MONTO) OVER (PARTITION BY CLIENTE_ID) AS PROMEDIO_CLIENTE
FROM PEDIDOS
WHERE CLIENTE_ID = 1
ORDER BY MONTO DESC;
```

---

### Ejercicio 3 — Reescritura con CTEs simples

**Objetivo:** Transformar las subconsultas del Ejercicio 2 en estructuras CTE (`WITH ... AS`) para mejorar la legibilidad y eliminar la duplicación de lógica.

#### Instrucciones

**Paso 3.1 — Reescribir el Paso 2.3 con una CTE simple**

La consulta del Paso 2.3 calculaba `(SELECT SUM(MONTO) FROM PEDIDOS)` dos veces. Una CTE elimina esa duplicación:

```sql
-- Versión CTE: el monto global se calcula UNA SOLA VEZ y se referencia por nombre
WITH MONTO_GLOBAL AS (
    SELECT SUM(MONTO) AS TOTAL_GLOBAL
    FROM PEDIDOS
)
SELECT
    C.CLIENTE_ID,
    C.NOMBRE,
    C.CIUDAD,
    SUM(P.MONTO)                                              AS MONTO_CLIENTE,
    MG.TOTAL_GLOBAL                                           AS MONTO_GLOBAL,
    ROUND(SUM(P.MONTO) / MG.TOTAL_GLOBAL * 100, 2)           AS PORCENTAJE_PARTICIPACION
FROM CLIENTES C
INNER JOIN PEDIDOS P
    ON C.CLIENTE_ID = P.CLIENTE_ID
CROSS JOIN MONTO_GLOBAL MG   -- la CTE devuelve una sola fila, CROSS JOIN es correcto aquí
GROUP BY C.CLIENTE_ID, C.NOMBRE, C.CIUDAD, MG.TOTAL_GLOBAL
ORDER BY PORCENTAJE_PARTICIPACION DESC;
```

**Salida esperada:** Idéntica al resultado del Paso 2.3. Verifica que los valores de `PORCENTAJE_PARTICIPACION` coincidan exactamente.

> 💡 **Ventaja clave:** La lógica de `TOTAL_GLOBAL` está definida en un solo lugar. Si mañana el negocio necesita filtrar por año (`WHERE YEAR(FECHA_PEDIDO) = 2024`), solo modificas la CTE, no dos lugares distintos.

---

**Paso 3.2 — Reescribir la consulta monolítica del Ejercicio 1 con una CTE**

Vuelve a la consulta del Paso 1.3 y reescríbela usando una CTE que extraiga la lógica de "totales por cliente":

```sql
-- Versión CTE: separa la lógica de cálculo de la lógica de filtrado
WITH TOTALES_POR_CLIENTE AS (
    -- Bloque 1: calcula el monto total acumulado por cliente
    SELECT
        CLIENTE_ID,
        COUNT(PEDIDO_ID) AS TOTAL_PEDIDOS,
        SUM(MONTO)       AS MONTO_TOTAL
    FROM PEDIDOS
    GROUP BY CLIENTE_ID
),
PROMEDIO_GLOBAL AS (
    -- Bloque 2: calcula el promedio de los totales por cliente
    SELECT AVG(MONTO_TOTAL) AS PROMEDIO_MONTO
    FROM TOTALES_POR_CLIENTE
)
-- Consulta principal: une clientes con sus totales y filtra sobre el promedio
SELECT
    C.CLIENTE_ID,
    C.NOMBRE,
    C.CIUDAD,
    TPC.TOTAL_PEDIDOS,
    TPC.MONTO_TOTAL
FROM CLIENTES C
INNER JOIN TOTALES_POR_CLIENTE TPC
    ON C.CLIENTE_ID = TPC.CLIENTE_ID
CROSS JOIN PROMEDIO_GLOBAL PG
WHERE TPC.MONTO_TOTAL > PG.PROMEDIO_MONTO
ORDER BY TPC.MONTO_TOTAL DESC;
```

**Salida esperada:** Idéntica a la consulta monolítica del Paso 1.3. Los mismos clientes, los mismos montos, el mismo orden.

**Verificación:**

```sql
-- Ejecuta ambas versiones y compara el conteo de filas
-- Versión monolítica (del Ejercicio 1) — cuenta filas
SELECT COUNT(*) FROM (
    SELECT C.CLIENTE_ID
    FROM CLIENTES C
    INNER JOIN PEDIDOS P ON C.CLIENTE_ID = P.CLIENTE_ID
    WHERE C.CLIENTE_ID IN (
        SELECT CLIENTE_ID FROM PEDIDOS
        GROUP BY CLIENTE_ID
        HAVING SUM(MONTO) > (
            SELECT AVG(MONTO_CLIENTE)
            FROM (SELECT CLIENTE_ID, SUM(MONTO) AS MONTO_CLIENTE FROM PEDIDOS GROUP BY CLIENTE_ID) AS T
        )
    )
    GROUP BY C.CLIENTE_ID, C.NOMBRE, C.CIUDAD
) AS VERSION_MONOLITICA;

-- Versión CTE — cuenta filas
WITH TOTALES_POR_CLIENTE AS (
    SELECT CLIENTE_ID, COUNT(PEDIDO_ID) AS TOTAL_PEDIDOS, SUM(MONTO) AS MONTO_TOTAL
    FROM PEDIDOS GROUP BY CLIENTE_ID
),
PROMEDIO_GLOBAL AS (
    SELECT AVG(MONTO_TOTAL) AS PROMEDIO_MONTO FROM TOTALES_POR_CLIENTE
)
SELECT COUNT(*) AS FILAS_VERSION_CTE
FROM CLIENTES C
INNER JOIN TOTALES_POR_CLIENTE TPC ON C.CLIENTE_ID = TPC.CLIENTE_ID
CROSS JOIN PROMEDIO_GLOBAL PG
WHERE TPC.MONTO_TOTAL > PG.PROMEDIO_MONTO;
```

Ambos `COUNT(*)` deben devolver el mismo número. Si difieren, revisa la lógica de filtrado.

---

**Paso 3.3 — CTE para productos sin ventas (reescritura del Paso 2.2)**

```sql
-- Versión CTE: extrae los productos vendidos como bloque nombrado
WITH PRODUCTOS_VENDIDOS AS (
    SELECT DISTINCT PRODUCTO_ID
    FROM PEDIDOS
)
SELECT
    PR.PRODUCTO_ID,
    PR.NOMBRE,
    PR.CATEGORIA
FROM PRODUCTOS PR
WHERE PR.PRODUCTO_ID NOT IN (
    SELECT PRODUCTO_ID FROM PRODUCTOS_VENDIDOS
)
ORDER BY PR.CATEGORIA, PR.NOMBRE;
```

**Salida esperada:** Idéntica al resultado del Paso 2.2 (productos sin pedidos).

> ⚠️ **Advertencia sobre `NOT IN` con posibles NULLs:** Esta versión usa `NOT IN` contra la CTE. Si `PRODUCTO_ID` puede ser `NULL` en `PEDIDOS`, el resultado podría ser incorrecto (0 filas). La versión con `NOT EXISTS` del Paso 2.2 es más robusta. Aquí se usa `NOT IN` intencionalmente para ilustrar la diferencia.

---

### Ejercicio 4 — CTEs encadenadas y comparación final

**Objetivo:** Construir una consulta analítica compleja usando múltiples CTEs encadenadas, y comparar ambos enfoques (subquery vs. CTE) para desarrollar criterio de selección.

#### Instrucciones

**Paso 4.1 — Construir una consulta analítica con CTEs encadenadas**

Pregunta de negocio: *"Genera un reporte de clientes que muestre: su ciudad, su monto total de pedidos, el promedio de su ciudad, y si están por encima o por debajo del promedio de su ciudad."*

Construye la solución paso a paso usando CTEs encadenadas:

```sql
WITH
-- Bloque 1: total de pedidos por cliente
TOTALES_CLIENTE AS (
    SELECT
        CLIENTE_ID,
        SUM(MONTO)   AS MONTO_TOTAL,
        COUNT(*)     AS NUM_PEDIDOS
    FROM PEDIDOS
    GROUP BY CLIENTE_ID
),

-- Bloque 2: unir con datos del cliente para obtener la ciudad
CLIENTES_CON_TOTALES AS (
    SELECT
        C.CLIENTE_ID,
        C.NOMBRE,
        C.CIUDAD,
        TC.MONTO_TOTAL,
        TC.NUM_PEDIDOS
    FROM CLIENTES C
    INNER JOIN TOTALES_CLIENTE TC
        ON C.CLIENTE_ID = TC.CLIENTE_ID
),

-- Bloque 3: calcular el promedio por ciudad (usando el resultado del Bloque 2)
PROMEDIO_POR_CIUDAD AS (
    SELECT
        CIUDAD,
        AVG(MONTO_TOTAL)   AS PROMEDIO_CIUDAD,
        COUNT(CLIENTE_ID)  AS CLIENTES_EN_CIUDAD
    FROM CLIENTES_CON_TOTALES
    GROUP BY CIUDAD
)

-- Consulta final: combinar todo y clasificar cada cliente
SELECT
    CCT.CLIENTE_ID,
    CCT.NOMBRE,
    CCT.CIUDAD,
    CCT.MONTO_TOTAL,
    CCT.NUM_PEDIDOS,
    ROUND(PPC.PROMEDIO_CIUDAD, 2)                          AS PROMEDIO_CIUDAD,
    PPC.CLIENTES_EN_CIUDAD,
    CASE
        WHEN CCT.MONTO_TOTAL > PPC.PROMEDIO_CIUDAD THEN 'POR ENCIMA DEL PROMEDIO'
        WHEN CCT.MONTO_TOTAL < PPC.PROMEDIO_CIUDAD THEN 'POR DEBAJO DEL PROMEDIO'
        ELSE 'IGUAL AL PROMEDIO'
    END                                                    AS CLASIFICACION
FROM CLIENTES_CON_TOTALES CCT
INNER JOIN PROMEDIO_POR_CIUDAD PPC
    ON CCT.CIUDAD = PPC.CIUDAD
ORDER BY CCT.CIUDAD, CCT.MONTO_TOTAL DESC;
```

**Salida esperada:** Tabla con todos los clientes que tienen pedidos, mostrando su ciudad, monto total, el promedio de su ciudad y la clasificación. Dentro de cada ciudad, los clientes están ordenados de mayor a menor monto.

**Verificación:**

```sql
-- Verifica que el promedio de ciudad sea correcto para una ciudad específica
-- Reemplaza 'Madrid' con una ciudad real de tu dataset
SELECT
    CIUDAD,
    AVG(MONTO_TOTAL) AS VERIFICACION_PROMEDIO
FROM (
    SELECT C.CIUDAD, SUM(P.MONTO) AS MONTO_TOTAL
    FROM CLIENTES C
    INNER JOIN PEDIDOS P ON C.CLIENTE_ID = P.CLIENTE_ID
    GROUP BY C.CLIENTE_ID, C.CIUDAD
) AS T
GROUP BY CIUDAD
ORDER BY CIUDAD;
```

---

**Paso 4.2 — Escribir la misma consulta usando solo subqueries (sin CTEs)**

Ahora escribe la consulta equivalente del Paso 4.1 usando únicamente subconsultas anidadas, **sin** usar `WITH`:

```sql
-- Versión equivalente con subqueries puras (sin CTEs)
SELECT
    CCT.CLIENTE_ID,
    CCT.NOMBRE,
    CCT.CIUDAD,
    CCT.MONTO_TOTAL,
    CCT.NUM_PEDIDOS,
    ROUND(PPC.PROMEDIO_CIUDAD, 2)   AS PROMEDIO_CIUDAD,
    PPC.CLIENTES_EN_CIUDAD,
    CASE
        WHEN CCT.MONTO_TOTAL > PPC.PROMEDIO_CIUDAD THEN 'POR ENCIMA DEL PROMEDIO'
        WHEN CCT.MONTO_TOTAL < PPC.PROMEDIO_CIUDAD THEN 'POR DEBAJO DEL PROMEDIO'
        ELSE 'IGUAL AL PROMEDIO'
    END                             AS CLASIFICACION
FROM (
    -- Subquery nivel 1: clientes con sus totales
    SELECT
        C.CLIENTE_ID,
        C.NOMBRE,
        C.CIUDAD,
        TC.MONTO_TOTAL,
        TC.NUM_PEDIDOS
    FROM CLIENTES C
    INNER JOIN (
        -- Subquery nivel 2: totales por cliente
        SELECT CLIENTE_ID, SUM(MONTO) AS MONTO_TOTAL, COUNT(*) AS NUM_PEDIDOS
        FROM PEDIDOS
        GROUP BY CLIENTE_ID
    ) AS TC ON C.CLIENTE_ID = TC.CLIENTE_ID
) AS CCT
INNER JOIN (
    -- Subquery nivel 1b: promedio por ciudad
    SELECT
        CIUDAD,
        AVG(MONTO_TOTAL)  AS PROMEDIO_CIUDAD,
        COUNT(CLIENTE_ID) AS CLIENTES_EN_CIUDAD
    FROM (
        -- Subquery nivel 2b: necesitamos recalcular totales por cliente OTRA VEZ
        SELECT
            C.CLIENTE_ID,
            C.CIUDAD,
            SUM(P.MONTO) AS MONTO_TOTAL
        FROM CLIENTES C
        INNER JOIN PEDIDOS P ON C.CLIENTE_ID = P.CLIENTE_ID
        GROUP BY C.CLIENTE_ID, C.CIUDAD
    ) AS TOTALES_PARA_CIUDAD
    GROUP BY CIUDAD
) AS PPC ON CCT.CIUDAD = PPC.CIUDAD
ORDER BY CCT.CIUDAD, CCT.MONTO_TOTAL DESC;
```

**Salida esperada:** Idéntica al resultado del Paso 4.1.

> 💡 **Observación crítica:** Nota que en la versión con subqueries, la lógica de "totales por cliente" (unir `CLIENTES` con `PEDIDOS` y agrupar) aparece **dos veces**: una en `CCT` y otra en la subquery que alimenta `PPC`. En la versión CTE, esa lógica está definida una sola vez en `TOTALES_CLIENTE`.

---

**Paso 4.3 — Análisis comparativo formal**

Ejecuta este bloque de análisis y documenta tus observaciones en comentarios SQL:

```sql
-- ANÁLISIS COMPARATIVO: CTE vs. Subquery
-- Completa este bloque con tus observaciones después de ejecutar ambas versiones

/*
CRITERIO 1 — LEGIBILIDAD
  Versión CTE:     Cada bloque tiene un nombre descriptivo. Se lee de arriba hacia abajo.
  Versión Subquery: Requiere leer desde adentro hacia afuera. Más difícil de seguir.
  Ganador: CTE

CRITERIO 2 — REUTILIZACIÓN DE LÓGICA
  Versión CTE:     TOTALES_CLIENTE se define una vez y se usa en dos bloques posteriores.
  Versión Subquery: La misma lógica se repite en dos subqueries distintas.
  Ganador: CTE

CRITERIO 3 — MANTENIBILIDAD
  Versión CTE:     Cambiar el filtro de fecha requiere modificar solo la CTE TOTALES_CLIENTE.
  Versión Subquery: Requiere localizar y modificar todas las subqueries que repiten la lógica.
  Ganador: CTE

CRITERIO 4 — CASOS DONDE SUBQUERY ES PREFERIBLE
  - Subconsultas escalares simples en SELECT (una sola línea, no se reutiliza).
  - Filtros con IN/EXISTS donde la lógica es simple y no se repite.
  - Cuando el lector del código es familiarizado con subqueries y la consulta es corta.
  - Portabilidad: las subqueries son SQL estándar; las CTEs también, pero con variaciones.

CRITERIO 5 — PORTABILIDAD
  Versión CTE:     WITH es estándar SQL:1999 y soportado en Snowflake, PostgreSQL, MySQL 8+,
                   SQL Server, BigQuery. Muy portable.
  Versión Subquery: Completamente portable a cualquier motor SQL.
  Ganador: Empate (ambas son portables en motores modernos)
*/

-- Consulta de cierre: cuenta cuántos clientes quedaron clasificados en cada categoría
WITH
TOTALES_CLIENTE AS (
    SELECT CLIENTE_ID, SUM(MONTO) AS MONTO_TOTAL
    FROM PEDIDOS GROUP BY CLIENTE_ID
),
CLIENTES_CON_TOTALES AS (
    SELECT C.CLIENTE_ID, C.CIUDAD, TC.MONTO_TOTAL
    FROM CLIENTES C INNER JOIN TOTALES_CLIENTE TC ON C.CLIENTE_ID = TC.CLIENTE_ID
),
PROMEDIO_POR_CIUDAD AS (
    SELECT CIUDAD, AVG(MONTO_TOTAL) AS PROMEDIO_CIUDAD
    FROM CLIENTES_CON_TOTALES GROUP BY CIUDAD
)
SELECT
    CASE
        WHEN CCT.MONTO_TOTAL > PPC.PROMEDIO_CIUDAD THEN 'POR ENCIMA DEL PROMEDIO'
        WHEN CCT.MONTO_TOTAL < PPC.PROMEDIO_CIUDAD THEN 'POR DEBAJO DEL PROMEDIO'
        ELSE 'IGUAL AL PROMEDIO'
    END          AS CLASIFICACION,
    COUNT(*)     AS CANTIDAD_CLIENTES
FROM CLIENTES_CON_TOTALES CCT
INNER JOIN PROMEDIO_POR_CIUDAD PPC ON CCT.CIUDAD = PPC.CIUDAD
GROUP BY CLASIFICACION
ORDER BY CANTIDAD_CLIENTES DESC;
```

**Salida esperada:** Tabla resumen con las categorías de clasificación y la cantidad de clientes en cada una. La suma de `CANTIDAD_CLIENTES` debe igualar el número total de clientes que tienen al menos un pedido.

---

## Validación y Pruebas

Una vez completados todos los ejercicios, ejecuta las siguientes verificaciones para confirmar que tus consultas son correctas y equivalentes:

```sql
-- VALIDACIÓN 1: Confirmar equivalencia entre versión monolítica y versión CTE del Ejercicio 1
-- Ambas consultas deben devolver el mismo conjunto de CLIENTE_ID

-- Versión monolítica
SELECT C.CLIENTE_ID AS ID_MONOLITICA
FROM CLIENTES C
INNER JOIN PEDIDOS P ON C.CLIENTE_ID = P.CLIENTE_ID
WHERE C.CLIENTE_ID IN (
    SELECT CLIENTE_ID FROM PEDIDOS
    GROUP BY CLIENTE_ID
    HAVING SUM(MONTO) > (
        SELECT AVG(MONTO_CLIENTE)
        FROM (SELECT CLIENTE_ID, SUM(MONTO) AS MONTO_CLIENTE FROM PEDIDOS GROUP BY CLIENTE_ID) T
    )
)
GROUP BY C.CLIENTE_ID, C.NOMBRE, C.CIUDAD

EXCEPT  -- Snowflake soporta EXCEPT para comparar conjuntos

-- Versión CTE
SELECT * FROM (
    WITH TOTALES_POR_CLIENTE AS (
        SELECT CLIENTE_ID, SUM(MONTO) AS MONTO_TOTAL FROM PEDIDOS GROUP BY CLIENTE_ID
    ),
    PROMEDIO_GLOBAL AS (
        SELECT AVG(MONTO_TOTAL) AS PROMEDIO_MONTO FROM TOTALES_POR_CLIENTE
    )
    SELECT C.CLIENTE_ID AS ID_CTE
    FROM CLIENTES C
    INNER JOIN TOTALES_POR_CLIENTE TPC ON C.CLIENTE_ID = TPC.CLIENTE_ID
    CROSS JOIN PROMEDIO_GLOBAL PG
    WHERE TPC.MONTO_TOTAL > PG.PROMEDIO_MONTO
);

-- Si el resultado es vacío (0 filas), ambas versiones son equivalentes. ✅
```

```sql
-- VALIDACIÓN 2: Confirmar que productos sin ventas + productos con ventas = total de productos
-- Se usa NOT EXISTS para evitar problemas si en otro dataset PRODUCTO_ID pudiera contener NULL.
SELECT
    (SELECT COUNT(*) FROM PRODUCTOS) AS TOTAL_PRODUCTOS,
    (SELECT COUNT(DISTINCT PRODUCTO_ID) FROM PEDIDOS) AS CON_PEDIDOS,
    (SELECT COUNT(*)
     FROM PRODUCTOS PR
     WHERE NOT EXISTS (
        SELECT 1
        FROM PEDIDOS PE
        WHERE PE.PRODUCTO_ID = PR.PRODUCTO_ID
     )) AS SIN_PEDIDOS,
    (SELECT COUNT(DISTINCT PRODUCTO_ID) FROM PEDIDOS) +
    (SELECT COUNT(*)
     FROM PRODUCTOS PR
     WHERE NOT EXISTS (
        SELECT 1
        FROM PEDIDOS PE
        WHERE PE.PRODUCTO_ID = PR.PRODUCTO_ID
     )) AS SUMA_VERIFICACION;

-- Resultado esperado con este dataset:
-- TOTAL_PRODUCTOS = 12
-- CON_PEDIDOS = 10
-- SIN_PEDIDOS = 2
-- SUMA_VERIFICACION = 12 ✅
```

```sql
-- VALIDACIÓN 3: Confirmar que la suma de porcentajes de participación es ~100%
WITH MONTO_GLOBAL AS (
    SELECT SUM(MONTO) AS TOTAL_GLOBAL FROM PEDIDOS
),
PARTICIPACION_CLIENTE AS (
    SELECT
        P.CLIENTE_ID,
        SUM(P.MONTO) AS MONTO_CLIENTE,
        MAX(MG.TOTAL_GLOBAL) AS MONTO_GLOBAL,
        SUM(P.MONTO) / MAX(MG.TOTAL_GLOBAL) * 100 AS PORCENTAJE_PARTICIPACION
    FROM PEDIDOS P
    CROSS JOIN MONTO_GLOBAL MG
    GROUP BY P.CLIENTE_ID
)
SELECT
    ROUND(SUM(PORCENTAJE_PARTICIPACION), 2) AS SUMA_PORCENTAJES
FROM PARTICIPACION_CLIENTE;

-- Resultado esperado: 100.00 ✅
```

---

## Resultados esperados clave con el dataset cargado

Estos resultados ayudan al instructor y al alumno a validar rápidamente que el laboratorio se está ejecutando sobre el dataset correcto.

| Consulta / Ejercicio | Resultado esperado |
|---|---|
| Conteo de `CLIENTES` | 10 filas |
| Conteo de `PRODUCTOS` | 12 filas |
| Conteo de `PEDIDOS` | 20 filas |
| Productos vendidos al menos una vez | 10 productos |
| Productos nunca vendidos | 2 productos: `Tablet Ejecutiva` y `Audífonos con cancelación` |
| Monto global de pedidos | `11300.00` |
| Suma de participación por cliente | `100.00` |
| Clasificación final por ciudad | 4 clientes por encima, 4 por debajo y 2 igual al promedio |

---

## Solución de Problemas

### Problema 1 — Error: "SQL compilation error: subquery returns more than one row"

**Síntoma:** Al ejecutar una subconsulta escalar en `SELECT` o al usar `=` con una subconsulta en `WHERE`, Snowflake devuelve el error `SQL compilation error: Scalar subquery produced more than one row`.

**Causa:** Una subconsulta escalar (usada con `=` o en la cláusula `SELECT`) devolvió más de una fila. Snowflake espera exactamente un valor y lanza error cuando recibe múltiples filas. Esto ocurre frecuentemente cuando se olvida agregar `GROUP BY`, `LIMIT 1`, o cuando la lógica de filtrado no es suficientemente restrictiva.

**Solución:**

```sql
-- ❌ Incorrecto: esta subconsulta puede devolver múltiples filas si hay varios productos
SELECT NOMBRE,
       (SELECT PRECIO_UNITARIO FROM PRODUCTOS WHERE CATEGORIA = 'Electrónica') AS PRECIO
FROM PRODUCTOS;

-- ✅ Correcto opción A: agregar agregación para garantizar un solo valor
SELECT NOMBRE,
       (SELECT AVG(PRECIO_UNITARIO) FROM PRODUCTOS WHERE CATEGORIA = 'Electrónica') AS PRECIO_PROMEDIO
FROM PRODUCTOS;

-- ✅ Correcto opción B: si necesitas una lista, usa IN en lugar de =
SELECT NOMBRE
FROM PRODUCTOS
WHERE CATEGORIA IN (
    SELECT DISTINCT CATEGORIA FROM PRODUCTOS WHERE PRECIO_UNITARIO > 100
);
```

---

### Problema 2 — La consulta con `NOT IN` devuelve 0 filas inesperadamente

**Síntoma:** Una consulta con `NOT IN` aplicada sobre una subconsulta devuelve 0 filas aunque visualmente hay registros que deberían cumplir la condición. El problema se manifiesta silenciosamente: no hay error, solo un resultado vacío incorrecto.

**Causa:** La subconsulta referenciada por `NOT IN` contiene al menos un valor `NULL`. En SQL, cualquier comparación con `NULL` usando `=` o `!=` devuelve `UNKNOWN` (no `TRUE` ni `FALSE`). Cuando `NOT IN` evalúa una lista que contiene `NULL`, la condición completa se evalúa como `UNKNOWN` para todas las filas, resultando en 0 filas devueltas.

**Solución:**

```sql
-- ❌ Problemático: si PEDIDOS.PRODUCTO_ID tiene algún NULL, esto devuelve 0 filas
SELECT NOMBRE FROM PRODUCTOS
WHERE PRODUCTO_ID NOT IN (
    SELECT PRODUCTO_ID FROM PEDIDOS  -- puede contener NULLs
);

-- ✅ Solución A: filtrar NULLs explícitamente en la subconsulta
SELECT NOMBRE FROM PRODUCTOS
WHERE PRODUCTO_ID NOT IN (
    SELECT PRODUCTO_ID FROM PEDIDOS
    WHERE PRODUCTO_ID IS NOT NULL   -- elimina NULLs de la lista
);

-- ✅ Solución B (recomendada): usar NOT EXISTS, que es inmune a NULLs
SELECT PR.NOMBRE
FROM PRODUCTOS PR
WHERE NOT EXISTS (
    SELECT 1
    FROM PEDIDOS PE
    WHERE PE.PRODUCTO_ID = PR.PRODUCTO_ID
);

-- Diagnóstico: verifica si hay NULLs en la columna referenciada
SELECT COUNT(*) AS NULOS_EN_PRODUCTO_ID
FROM PEDIDOS
WHERE PRODUCTO_ID IS NULL;
```

> 💡 **Buena práctica general:** En Snowflake y en cualquier motor SQL, prefiere `NOT EXISTS` sobre `NOT IN` cuando la subconsulta opera sobre columnas que pueden contener `NULL`. Es más seguro y frecuentemente más eficiente.

---

## Limpieza del entorno

Al finalizar el laboratorio, ejecuta los siguientes comandos para liberar recursos y evitar consumo innecesario de créditos Snowflake:

```sql
-- Suspender el warehouse para detener el consumo de créditos
-- IMPORTANTE: ejecutar siempre al terminar la sesión
ALTER WAREHOUSE COMPUTE_WH SUSPEND;
```

> ⚠️ **Recordatorio de créditos:** Las cuentas trial de Snowflake tienen 400 USD de créditos. Un warehouse `X-SMALL` consume aproximadamente 1 crédito por hora de actividad. Suspenderlo al terminar cada sesión es una práctica obligatoria en este curso.

No es necesario eliminar tablas ni datos, ya que el schema `LAB_SQL_INTERMEDIO.VENTAS` es compartido por todos los laboratorios del curso y sus datos deben persistir para las sesiones siguientes.

---

## Resumen

En este laboratorio aplicaste los conceptos de la Lección 1.1 en un contexto de negocio real, progresando desde la comprensión hasta la aplicación activa:

| Concepto practicado | Ejercicio | Resultado clave |
|---|---|---|
| Subconsulta no correlacionada con `IN` | 2.1 | Productos con pedidos en 2024 |
| Subconsulta correlacionada con `NOT EXISTS` | 2.2 | Productos sin ventas |
| Subconsulta escalar en `SELECT` | 2.3 | Porcentaje de participación por cliente |
| Subconsulta correlacionada en `WHERE` | 2.4 | Pedidos sobre promedio individual |
| CTE simple para eliminar duplicación | 3.1, 3.2, 3.3 | Reescritura de consultas del Ejercicio 2 |
| CTEs encadenadas (3 bloques) | 4.1 | Reporte de clientes vs. promedio de ciudad |
| Comparación CTE vs. subquery | 4.2, 4.3 | Criterios de selección por contexto |

### Conclusiones principales

1. **Las subconsultas son herramientas poderosas** para filtrado dinámico (`IN`, `EXISTS`) y cálculos derivados en `SELECT`, especialmente cuando la lógica es simple y no se repite.

2. **Las CTEs mejoran la legibilidad y el mantenimiento** cuando la misma lógica se necesita en múltiples partes de una consulta o cuando hay más de dos niveles de anidamiento.

3. **`NOT EXISTS` es más seguro que `NOT IN`** cuando la subconsulta puede contener valores `NULL`.

4. **La elección entre CTE y subquery depende del contexto:** para lógica compleja y reutilizable, usa CTEs; para filtros simples y puntuales, una subquery es suficiente y más directa.

### Próximos pasos

En el **Laboratorio 2**, profundizarás en el uso de subconsultas dentro de las cláusulas `SELECT` y `WHERE` para construir métricas derivadas más complejas, y comenzarás a explorar patrones de optimización de consultas en Snowflake.

### Recursos adicionales

| Recurso | URL |
|---|---|
| Documentación Snowflake: Subqueries | https://docs.snowflake.com/en/user-guide/querying-subqueries |
| Documentación Snowflake: WITH (CTEs) | https://docs.snowflake.com/en/sql-reference/constructs/with |
| Documentación Snowflake: SELECT Syntax | https://docs.snowflake.com/en/sql-reference/sql/select |
| Mode Analytics SQL Tutorial: Subqueries | https://mode.com/sql-tutorial/sql-sub-queries/ |

---
