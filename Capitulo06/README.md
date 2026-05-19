# Validación y reconciliación de datasets

## Metadatos

| Campo | Detalle |
|---|---|
| **Duración estimada** | 35 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (*Apply*) |
| **Módulo** | 6 — Validación y reconciliación de datos |
| **Plataforma** | Snowflake (Snowsight Worksheet) |
| **Schema de origen** | `LAB_SQL_INTERMEDIO.VENTAS_ORIGEN` |
| **Schema de destino** | `LAB_SQL_INTERMEDIO.VENTAS_DESTINO` |
| **Tabla principal** | `PEDIDOS` |

---

## Descripción General

En este laboratorio aplicarás técnicas de comparación de datasets para detectar discrepancias entre dos versiones de los mismos datos: el schema `VENTAS_ORIGEN` (datos crudos) y el schema `VENTAS_DESTINO` (datos procesados). Utilizarás operadores de conjunto (`EXCEPT`, `INTERSECT`), comparaciones de agregados y checksums de fila para construir un reporte de reconciliación completo. Al finalizar, habrás empaquetado todas las validaciones como CTEs reutilizables que simulan un conjunto de controles de calidad aplicables a cualquier pipeline de datos.

Este laboratorio representa una situación común en procesos de ingeniería de datos: un pipeline ETL/ELT mueve información desde una fuente hacia una capa destino, pero durante el proceso pueden ocurrir pérdidas de registros, inserciones no esperadas o modificaciones en valores críticos. Tu objetivo será identificar esas diferencias, cuantificarlas y producir un reporte claro para comunicar el estado del pipeline.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Comparar conteos totales y por categoría entre dos schemas para detectar discrepancias de volumen.
- [ ] Usar `EXCEPT` e `INTERSECT` para identificar registros faltantes, extra o coincidentes entre datasets de origen y destino.
- [ ] Implementar comparaciones de checksums (`MD5`) sobre columnas clave para detectar diferencias de valores.
- [ ] Construir un reporte de reconciliación consolidado usando `UNION ALL` y `CASE WHEN` con clasificación `PASS/FAIL`.
- [ ] Encapsular validaciones como CTEs reutilizables orientadas a monitoreo continuo de calidad de datos.
- [ ] Distinguir entre una diferencia por fila completa y una diferencia por llave de negocio (`PEDIDO_ID`).

---

## Prerrequisitos

### Conocimiento previo

| Requisito | Nivel esperado |
|---|---|
| CTEs (`WITH`) y subqueries | Sólido — Laboratorios 1 al 5 |
| `JOIN` (`INNER`, `LEFT`, `FULL OUTER`) | Sólido |
| Funciones de agregación (`COUNT`, `SUM`, `AVG`) | Sólido |
| Operadores de conjunto (`UNION`, `EXCEPT`, `INTERSECT`) | Básico a intermedio |
| `CASE WHEN` para clasificaciones | Intermedio |
| Concepto de pipeline ETL/ELT y calidad de datos | Conceptual |

### Acceso y configuración

| Requisito | Detalle |
|---|---|
| Cuenta Snowflake activa | Trial o corporativa con permisos para crear database, schemas y tablas |
| Script de setup ejecutado | No se asume script previo. Esta práctica incluye el setup completo de schemas, tablas y datos. |
| Database disponible | `LAB_SQL_INTERMEDIO` |
| Schemas requeridos | `VENTAS_ORIGEN` y `VENTAS_DESTINO`, creados en el Paso 0 |
| Tabla requerida | `PEDIDOS` en ambos schemas |
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

---

## Organización recomendada de Workspace en Snowsight

Para que la práctica sea ordenada y reutilizable, trabaja con un Workspace y 2 folders. En esta práctica se usa la palabra **workspace** como una separación lógica de trabajo dentro de Snowsight; técnicamente, en Snowflake trabajarás con **Worksheets** dentro de un Workspace.

| Workspace / Worksheet | Folder | Nombre sugerido | Uso |
|---|---|---|---|
| SNOWLABS-INT | SETUP-LABS | `06_SETUP_DATOS_RECONCILIACION` | Crear schemas, tablas y datos de origen/destino con discrepancias controladas. Se ejecuta una vez al inicio o cuando quieras reiniciar el laboratorio. |
| SNOWLABS-INT | SCRIPT-LABS | `06_LAB_RECONCILIACION_DATASETS` | Ejecutar los ejercicios del laboratorio sin mezclar el script de carga de datos. |

### Paso 0.0 — Crear el workspace de las prácticas

1. Entra a **Snowsight**.
2. Da clic en la opción **Projects**.
3. Da clic en **+**.
4. Selecciona la opción **Private workspace**.
5. Nómbralo: **`SNOWLABS-INT`**.
6. Da clic en **Create**.

### Paso 0.0.1 — Crear el folder y script que carga los datos

1. Dentro del workspace **SNOWLABS-INT**, da clic en **+ Add new**.
2. Selecciona **Folder**.
3. Nombra el folder: **`SETUP-LABS`**.
4. Dentro del folder **SETUP-LABS**, da clic en el símbolo **+**.
5. Crea un archivo de tipo **SQL**.
6. Nómbralo: **`06_SETUP_DATOS_RECONCILIACION`**.
7. Pega ahí el siguiente script completo.
8. Ejecuta el script completo antes de comenzar el laboratorio.

Este dataset está diseñado para activar todos los escenarios de la práctica:

- Registros idénticos entre origen y destino.
- Registros faltantes en destino.
- Registros extra en destino.
- Registros con `MONTO_TOTAL` modificado.
- Registros con `FECHA_PEDIDO` modificada.
- Registros con `CATEGORIA_PRODUCTO` modificada.
- Diferencias por categoría para analizar discrepancias agregadas.
- Comparación exacta con `EXCEPT` e `INTERSECT`.
- Auditoría por llave con `FULL OUTER JOIN`.
- Validación por checksum con `MD5`.

```sql
-- 06_SETUP_DATOS_RECONCILIACION.sql
-- Práctica Snowflake Intermedio
-- Dataset controlado para completar el laboratorio:
-- Validación y reconciliación de datasets
--
-- Objetivo del dataset:
-- 1) Crear dos schemas: VENTAS_ORIGEN y VENTAS_DESTINO.
-- 2) Crear una tabla PEDIDOS con la misma estructura en ambos schemas.
-- 3) Cargar datos con discrepancias controladas:
--    - Registros faltantes en destino.
--    - Registros extra en destino.
--    - Registros con monto modificado.
--    - Registros con fecha modificada.
--    - Registros con categoría modificada.
--    - Registros idénticos.
-- 4) Permitir validar EXCEPT, INTERSECT, FULL OUTER JOIN, MD5 y reportes PASS/FAIL.

USE WAREHOUSE COMPUTE_WH;

CREATE DATABASE IF NOT EXISTS LAB_SQL_INTERMEDIO;
USE DATABASE LAB_SQL_INTERMEDIO;

CREATE SCHEMA IF NOT EXISTS VENTAS_ORIGEN;
CREATE SCHEMA IF NOT EXISTS VENTAS_DESTINO;

-- Reinicio controlado del laboratorio.
DROP TABLE IF EXISTS VENTAS_ORIGEN.PEDIDOS;
DROP TABLE IF EXISTS VENTAS_DESTINO.PEDIDOS;

CREATE OR REPLACE TABLE VENTAS_ORIGEN.PEDIDOS (
    PEDIDO_ID NUMBER(10,0) NOT NULL,
    CLIENTE_ID NUMBER(10,0) NOT NULL,
    FECHA_PEDIDO DATE NOT NULL,
    MONTO_TOTAL NUMBER(12,2) NOT NULL,
    CATEGORIA_PRODUCTO VARCHAR(80) NOT NULL,
    ESTADO_PEDIDO VARCHAR(30) NOT NULL,
    CANAL VARCHAR(40) NOT NULL,
    CONSTRAINT PK_PEDIDOS_ORIGEN PRIMARY KEY (PEDIDO_ID)
);

CREATE OR REPLACE TABLE VENTAS_DESTINO.PEDIDOS (
    PEDIDO_ID NUMBER(10,0) NOT NULL,
    CLIENTE_ID NUMBER(10,0) NOT NULL,
    FECHA_PEDIDO DATE NOT NULL,
    MONTO_TOTAL NUMBER(12,2) NOT NULL,
    CATEGORIA_PRODUCTO VARCHAR(80) NOT NULL,
    ESTADO_PEDIDO VARCHAR(30) NOT NULL,
    CANAL VARCHAR(40) NOT NULL,
    CONSTRAINT PK_PEDIDOS_DESTINO PRIMARY KEY (PEDIDO_ID)
);

-- ============================================================
-- DATOS DE ORIGEN
-- 30 registros base.
-- ============================================================

INSERT INTO VENTAS_ORIGEN.PEDIDOS
    (PEDIDO_ID, CLIENTE_ID, FECHA_PEDIDO, MONTO_TOTAL, CATEGORIA_PRODUCTO, ESTADO_PEDIDO, CANAL)
VALUES
    (6001, 101, '2024-01-05',  850.00, 'Electrónica', 'COMPLETADO', 'Web'),
    (6002, 102, '2024-01-08',  420.00, 'Hogar',       'COMPLETADO', 'Web'),
    (6003, 103, '2024-01-12', 1250.00, 'Software',    'COMPLETADO', 'Ejecutivo'),
    (6004, 104, '2024-01-18',  210.00, 'Ropa',        'COMPLETADO', 'Marketplace'),
    (6005, 105, '2024-01-21',  980.00, 'Electrónica', 'COMPLETADO', 'Web'),

    (6006, 106, '2024-02-03',  330.00, 'Ropa',        'COMPLETADO', 'Marketplace'),
    (6007, 107, '2024-02-07', 1420.00, 'Electrónica', 'COMPLETADO', 'Ejecutivo'),
    (6008, 108, '2024-02-11',  760.00, 'Hogar',       'COMPLETADO', 'Web'),
    (6009, 109, '2024-02-16',  540.00, 'Ropa',        'COMPLETADO', 'Web'),
    (6010, 110, '2024-02-20', 1890.00, 'Software',    'COMPLETADO', 'Ejecutivo'),

    (6011, 111, '2024-03-02',  450.00, 'Electrónica', 'COMPLETADO', 'Web'),
    (6012, 112, '2024-03-06',  670.00, 'Hogar',       'COMPLETADO', 'Marketplace'),
    (6013, 113, '2024-03-10', 2200.00, 'Software',    'COMPLETADO', 'Ejecutivo'),
    (6014, 114, '2024-03-15',  390.00, 'Ropa',        'COMPLETADO', 'Web'),
    (6015, 115, '2024-03-18', 1350.00, 'Electrónica', 'COMPLETADO', 'Ejecutivo'),

    (6016, 116, '2024-04-04',  880.00, 'Hogar',       'COMPLETADO', 'Web'),
    (6017, 117, '2024-04-08',  240.00, 'Ropa',        'COMPLETADO', 'Marketplace'),
    (6018, 118, '2024-04-12', 1720.00, 'Software',    'COMPLETADO', 'Ejecutivo'),
    (6019, 119, '2024-04-17',  990.00, 'Electrónica', 'COMPLETADO', 'Web'),
    (6020, 120, '2024-04-22',  610.00, 'Hogar',       'COMPLETADO', 'Web'),

    (6021, 121, '2024-05-03', 1510.00, 'Electrónica', 'COMPLETADO', 'Ejecutivo'),
    (6022, 122, '2024-05-07',  370.00, 'Ropa',        'COMPLETADO', 'Marketplace'),
    (6023, 123, '2024-05-11',  720.00, 'Hogar',       'COMPLETADO', 'Web'),
    (6024, 124, '2024-05-16', 2450.00, 'Software',    'COMPLETADO', 'Ejecutivo'),
    (6025, 125, '2024-05-20', 1180.00, 'Electrónica', 'COMPLETADO', 'Web'),

    (6026, 126, '2024-06-04',  530.00, 'Ropa',        'COMPLETADO', 'Web'),
    (6027, 127, '2024-06-08',  910.00, 'Hogar',       'COMPLETADO', 'Marketplace'),
    (6028, 128, '2024-06-12', 1990.00, 'Software',    'COMPLETADO', 'Ejecutivo'),
    (6029, 129, '2024-06-17', 1640.00, 'Electrónica', 'COMPLETADO', 'Ejecutivo'),
    (6030, 130, '2024-06-21',  460.00, 'Ropa',        'COMPLETADO', 'Web');

-- ============================================================
-- DATOS DE DESTINO
--
-- Reglas intencionales:
-- - Faltantes en destino por llave: 6007, 6015, 6029.
-- - Extras en destino por llave: 7001, 7002.
-- - Monto modificado: 6003, 6021.
-- - Fecha modificada: 6012.
-- - Categoría modificada: 6024.
-- - El resto se mantiene idéntico.
-- ============================================================

INSERT INTO VENTAS_DESTINO.PEDIDOS
    (PEDIDO_ID, CLIENTE_ID, FECHA_PEDIDO, MONTO_TOTAL, CATEGORIA_PRODUCTO, ESTADO_PEDIDO, CANAL)
VALUES
    (6001, 101, '2024-01-05',  850.00, 'Electrónica', 'COMPLETADO', 'Web'),
    (6002, 102, '2024-01-08',  420.00, 'Hogar',       'COMPLETADO', 'Web'),

    -- Modificado: monto diferente respecto al origen.
    (6003, 103, '2024-01-12', 1300.00, 'Software',    'COMPLETADO', 'Ejecutivo'),

    (6004, 104, '2024-01-18',  210.00, 'Ropa',        'COMPLETADO', 'Marketplace'),
    (6005, 105, '2024-01-21',  980.00, 'Electrónica', 'COMPLETADO', 'Web'),
    (6006, 106, '2024-02-03',  330.00, 'Ropa',        'COMPLETADO', 'Marketplace'),

    -- 6007 falta intencionalmente en destino.

    (6008, 108, '2024-02-11',  760.00, 'Hogar',       'COMPLETADO', 'Web'),
    (6009, 109, '2024-02-16',  540.00, 'Ropa',        'COMPLETADO', 'Web'),
    (6010, 110, '2024-02-20', 1890.00, 'Software',    'COMPLETADO', 'Ejecutivo'),
    (6011, 111, '2024-03-02',  450.00, 'Electrónica', 'COMPLETADO', 'Web'),

    -- Modificado: fecha diferente respecto al origen.
    (6012, 112, '2024-03-07',  670.00, 'Hogar',       'COMPLETADO', 'Marketplace'),

    (6013, 113, '2024-03-10', 2200.00, 'Software',    'COMPLETADO', 'Ejecutivo'),
    (6014, 114, '2024-03-15',  390.00, 'Ropa',        'COMPLETADO', 'Web'),

    -- 6015 falta intencionalmente en destino.

    (6016, 116, '2024-04-04',  880.00, 'Hogar',       'COMPLETADO', 'Web'),
    (6017, 117, '2024-04-08',  240.00, 'Ropa',        'COMPLETADO', 'Marketplace'),
    (6018, 118, '2024-04-12', 1720.00, 'Software',    'COMPLETADO', 'Ejecutivo'),
    (6019, 119, '2024-04-17',  990.00, 'Electrónica', 'COMPLETADO', 'Web'),
    (6020, 120, '2024-04-22',  610.00, 'Hogar',       'COMPLETADO', 'Web'),

    -- Modificado: monto diferente respecto al origen.
    (6021, 121, '2024-05-03', 1490.00, 'Electrónica', 'COMPLETADO', 'Ejecutivo'),

    (6022, 122, '2024-05-07',  370.00, 'Ropa',        'COMPLETADO', 'Marketplace'),
    (6023, 123, '2024-05-11',  720.00, 'Hogar',       'COMPLETADO', 'Web'),

    -- Modificado: categoría diferente respecto al origen.
    (6024, 124, '2024-05-16', 2450.00, 'Servicios',   'COMPLETADO', 'Ejecutivo'),

    (6025, 125, '2024-05-20', 1180.00, 'Electrónica', 'COMPLETADO', 'Web'),
    (6026, 126, '2024-06-04',  530.00, 'Ropa',        'COMPLETADO', 'Web'),
    (6027, 127, '2024-06-08',  910.00, 'Hogar',       'COMPLETADO', 'Marketplace'),
    (6028, 128, '2024-06-12', 1990.00, 'Software',    'COMPLETADO', 'Ejecutivo'),

    -- 6029 falta intencionalmente en destino.

    (6030, 130, '2024-06-21',  460.00, 'Ropa',        'COMPLETADO', 'Web'),

    -- Extras en destino: no existen en origen.
    (7001, 201, '2024-06-25',  999.00, 'Electrónica', 'COMPLETADO', 'Carga manual'),
    (7002, 202, '2024-06-26',  310.00, 'Hogar',       'COMPLETADO', 'Carga manual');

-- ============================================================
-- Validación rápida del dataset.
-- ============================================================

SELECT 'VENTAS_ORIGEN.PEDIDOS' AS TABLA, COUNT(*) AS FILAS
FROM VENTAS_ORIGEN.PEDIDOS
UNION ALL
SELECT 'VENTAS_DESTINO.PEDIDOS' AS TABLA, COUNT(*) AS FILAS
FROM VENTAS_DESTINO.PEDIDOS;

-- Resultado esperado:
-- VENTAS_ORIGEN.PEDIDOS  = 30
-- VENTAS_DESTINO.PEDIDOS = 29

-- Validación de faltantes por llave.
SELECT COUNT(*) AS FALTANTES_POR_LLAVE
FROM VENTAS_ORIGEN.PEDIDOS o
LEFT JOIN VENTAS_DESTINO.PEDIDOS d
    ON o.PEDIDO_ID = d.PEDIDO_ID
WHERE d.PEDIDO_ID IS NULL;

-- Resultado esperado: 3

-- Validación de extras por llave.
SELECT COUNT(*) AS EXTRAS_POR_LLAVE
FROM VENTAS_DESTINO.PEDIDOS d
LEFT JOIN VENTAS_ORIGEN.PEDIDOS o
    ON d.PEDIDO_ID = o.PEDIDO_ID
WHERE o.PEDIDO_ID IS NULL;

-- Resultado esperado: 2

-- Validación de modificaciones por tipo.
SELECT 'MONTO DIFERENTE' AS PROBLEMA, COUNT(*) AS CASOS
FROM VENTAS_ORIGEN.PEDIDOS o
INNER JOIN VENTAS_DESTINO.PEDIDOS d
    ON o.PEDIDO_ID = d.PEDIDO_ID
WHERE ROUND(o.MONTO_TOTAL, 2) <> ROUND(d.MONTO_TOTAL, 2)

UNION ALL

SELECT 'FECHA DIFERENTE' AS PROBLEMA, COUNT(*) AS CASOS
FROM VENTAS_ORIGEN.PEDIDOS o
INNER JOIN VENTAS_DESTINO.PEDIDOS d
    ON o.PEDIDO_ID = d.PEDIDO_ID
WHERE o.FECHA_PEDIDO <> d.FECHA_PEDIDO

UNION ALL

SELECT 'CATEGORÍA DIFERENTE' AS PROBLEMA, COUNT(*) AS CASOS
FROM VENTAS_ORIGEN.PEDIDOS o
INNER JOIN VENTAS_DESTINO.PEDIDOS d
    ON o.PEDIDO_ID = d.PEDIDO_ID
WHERE o.CATEGORIA_PRODUCTO <> d.CATEGORIA_PRODUCTO;

-- Resultado esperado:
-- MONTO DIFERENTE      = 2
-- FECHA DIFERENTE      = 1
-- CATEGORÍA DIFERENTE  = 1
```

### Paso 0.0.2 — Crear el folder y script de laboratorio

1. Da clic en el botón **+ Add new**.
2. Selecciona **Folder**.
3. Nómbralo: **`SCRIPT-LABS`**.
4. Dentro de **SCRIPT-LABS**, crea un archivo de tipo **SQL**.
5. Nómbralo: **`06_LAB_RECONCILIACION_DATASETS`**.
6. Usa este archivo para ejecutar los ejercicios 1, 2, 3, 4, 5 y 6.
7. **No pegues aquí el script de carga completo; solo usa las consultas de análisis del laboratorio.**

---

### Paso 0.1 — Confirmar contexto de trabajo

Dentro del archivo **06_LAB_RECONCILIACION_DATASETS**, ejecuta:

```sql
USE WAREHOUSE COMPUTE_WH;
USE DATABASE LAB_SQL_INTERMEDIO;

SELECT
    CURRENT_WAREHOUSE() AS WAREHOUSE_ACTUAL,
    CURRENT_DATABASE()  AS DATABASE_ACTUAL;
```

**Resultado esperado:**

| WAREHOUSE_ACTUAL | DATABASE_ACTUAL |
|---|---|
| COMPUTE_WH | LAB_SQL_INTERMEDIO |

---

### Paso 0.2 — Confirmar que los schemas quedaron disponibles

Ejecuta:

```sql
SHOW SCHEMAS IN DATABASE LAB_SQL_INTERMEDIO;
```

**Resultado esperado:** deben aparecer al menos estos schemas:

| Schema | Uso en la práctica |
|---|---|
| `VENTAS_ORIGEN` | Datos crudos o fuente original del pipeline. |
| `VENTAS_DESTINO` | Datos procesados o resultado del pipeline. |

---

### Paso 0.3 — Confirmar que las tablas quedaron disponibles

Ejecuta:

```sql
SHOW TABLES IN SCHEMA LAB_SQL_INTERMEDIO.VENTAS_ORIGEN;
SHOW TABLES IN SCHEMA LAB_SQL_INTERMEDIO.VENTAS_DESTINO;
```

**Resultado esperado:** ambos schemas deben contener la tabla `PEDIDOS`.

---

### Paso 0.4 — Validar volumen de origen y destino

Ejecuta:

```sql
SELECT 'VENTAS_ORIGEN' AS SCHEMA_FUENTE, COUNT(*) AS TOTAL_REGISTROS
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS

UNION ALL

SELECT 'VENTAS_DESTINO' AS SCHEMA_FUENTE, COUNT(*) AS TOTAL_REGISTROS
FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS

ORDER BY SCHEMA_FUENTE;
```

**Resultado esperado:**

| SCHEMA_FUENTE | TOTAL_REGISTROS |
|---|---:|
| VENTAS_DESTINO | 29 |
| VENTAS_ORIGEN | 30 |

---

### Paso 0.5 — Validar discrepancias esperadas del dataset

Ejecuta:

```sql
SELECT 'Faltantes por llave en destino' AS PROBLEMA, COUNT(*) AS CASOS
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
LEFT JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
    ON o.PEDIDO_ID = d.PEDIDO_ID
WHERE d.PEDIDO_ID IS NULL

UNION ALL

SELECT 'Extras por llave en destino' AS PROBLEMA, COUNT(*) AS CASOS
FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
LEFT JOIN LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
    ON d.PEDIDO_ID = o.PEDIDO_ID
WHERE o.PEDIDO_ID IS NULL

UNION ALL

SELECT 'Monto diferente' AS PROBLEMA, COUNT(*) AS CASOS
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
INNER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
    ON o.PEDIDO_ID = d.PEDIDO_ID
WHERE ROUND(o.MONTO_TOTAL, 2) <> ROUND(d.MONTO_TOTAL, 2)

UNION ALL

SELECT 'Fecha diferente' AS PROBLEMA, COUNT(*) AS CASOS
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
INNER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
    ON o.PEDIDO_ID = d.PEDIDO_ID
WHERE o.FECHA_PEDIDO <> d.FECHA_PEDIDO

UNION ALL

SELECT 'Categoría diferente' AS PROBLEMA, COUNT(*) AS CASOS
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
INNER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
    ON o.PEDIDO_ID = d.PEDIDO_ID
WHERE o.CATEGORIA_PRODUCTO <> d.CATEGORIA_PRODUCTO;
```

**Resultado esperado:**

| PROBLEMA | CASOS |
|---|---:|
| Faltantes por llave en destino | 3 |
| Extras por llave en destino | 2 |
| Monto diferente | 2 |
| Fecha diferente | 1 |
| Categoría diferente | 1 |

---

### Paso 0.6 — Validar categorías disponibles

Ejecuta:

```sql
SELECT
    'ORIGEN' AS FUENTE,
    CATEGORIA_PRODUCTO,
    COUNT(*) AS REGISTROS,
    ROUND(SUM(MONTO_TOTAL), 2) AS SUMA_MONTOS
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
GROUP BY CATEGORIA_PRODUCTO

UNION ALL

SELECT
    'DESTINO' AS FUENTE,
    CATEGORIA_PRODUCTO,
    COUNT(*) AS REGISTROS,
    ROUND(SUM(MONTO_TOTAL), 2) AS SUMA_MONTOS
FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
GROUP BY CATEGORIA_PRODUCTO

ORDER BY CATEGORIA_PRODUCTO, FUENTE;
```

**Resultado esperado:** deben aparecer varias categorías (`Electrónica`, `Hogar`, `Ropa`, `Software`) y en destino también `Servicios`, que fue introducida intencionalmente como categoría diferente para el pedido `6024`.

---

## Ejercicios Paso a Paso

---

### Ejercicio 1 — Reconocer los datasets: comparación de volumen total

**Objetivo:** Establecer una primera validación rápida comparando el número total de registros y la suma de montos entre `VENTAS_ORIGEN` y `VENTAS_DESTINO`. Esta es la prueba más básica de un pipeline ETL: **¿llegaron todos los registros al destino?**

#### Instrucciones

**Paso 1.1 — Comparar volumen total entre origen y destino**

Ejecuta la siguiente consulta para comparar el volumen total de registros y la suma de montos entre ambos schemas:

```sql
-- ============================================================
-- PASO 1: Comparación de volumen total entre origen y destino
-- ============================================================

SELECT
    'VENTAS_ORIGEN'          AS schema_fuente,
    COUNT(*)                 AS total_registros,
    ROUND(SUM(MONTO_TOTAL), 2) AS suma_montos,
    MIN(FECHA_PEDIDO)        AS fecha_minima,
    MAX(FECHA_PEDIDO)        AS fecha_maxima
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS

UNION ALL

SELECT
    'VENTAS_DESTINO'         AS schema_fuente,
    COUNT(*)                 AS total_registros,
    ROUND(SUM(MONTO_TOTAL), 2) AS suma_montos,
    MIN(FECHA_PEDIDO)        AS fecha_minima,
    MAX(FECHA_PEDIDO)        AS fecha_maxima
FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS

ORDER BY schema_fuente;
```

**Paso 1.2 — Observar los resultados**

Anota mentalmente si los valores de `total_registros` y `suma_montos` coinciden entre ambas filas.

#### Resultado esperado

Verás dos filas, una por schema. Con este dataset controlado, los valores **no deben coincidir**, porque el laboratorio está diseñado con discrepancias intencionales.

| SCHEMA_FUENTE | TOTAL_REGISTROS | FECHA_MINIMA | FECHA_MAXIMA |
|---|---:|---|---|
| VENTAS_DESTINO | 29 | 2024-01-05 | 2024-06-26 |
| VENTAS_ORIGEN | 30 | 2024-01-05 | 2024-06-21 |

> ⚠️ **Observación:** El destino tiene menos registros que el origen, pero también contiene registros extra que no existen en origen. Por eso no basta comparar el conteo total; necesitas revisar faltantes, extras y valores modificados.

#### Verificación

```sql
-- Verificación del Paso 1: diferencia de volumen
SELECT
    (SELECT COUNT(*) FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS) AS registros_origen,
    (SELECT COUNT(*) FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS) AS registros_destino,
    (SELECT COUNT(*) FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS)
    -
    (SELECT COUNT(*) FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS) AS diferencia_registros;
```

**Resultado esperado:** `diferencia_registros = 1`.

> 💡 **Punto clave:** Una diferencia de `1` en conteo total no significa que solo exista un problema. En este dataset hay 3 faltantes y 2 extras; el resultado neto es 1. Esta es la razón por la que el análisis por llave es indispensable.

---

### Ejercicio 2 — Comparación por categoría: desglose de discrepancias

**Objetivo:** Profundizar la comparación del Paso 1 desglosando por `CATEGORIA_PRODUCTO` para identificar si las discrepancias están concentradas en categorías específicas o distribuidas uniformemente.

#### Instrucciones

**Paso 2.1 — Comparar volumen y montos por categoría usando `UNION ALL`**

```sql
-- ============================================================
-- PASO 2: Comparación de volumen y montos por categoría
-- ============================================================

SELECT
    'VENTAS_ORIGEN'           AS schema_fuente,
    CATEGORIA_PRODUCTO,
    COUNT(*)                  AS total_registros,
    ROUND(SUM(MONTO_TOTAL), 2) AS suma_montos
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
GROUP BY CATEGORIA_PRODUCTO

UNION ALL

SELECT
    'VENTAS_DESTINO'          AS schema_fuente,
    CATEGORIA_PRODUCTO,
    COUNT(*)                  AS total_registros,
    ROUND(SUM(MONTO_TOTAL), 2) AS suma_montos
FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
GROUP BY CATEGORIA_PRODUCTO

ORDER BY CATEGORIA_PRODUCTO, schema_fuente;
```

**Paso 2.2 — Construir una vista comparativa lado a lado**

Para una lectura más comparativa, ejecuta esta versión con `FULL OUTER JOIN` que pone las cifras lado a lado:

```sql
-- ============================================================
-- PASO 2b: Vista comparativa lado a lado por categoría
-- ============================================================

SELECT
    COALESCE(o.CATEGORIA_PRODUCTO,
             d.CATEGORIA_PRODUCTO)          AS categoria_producto,
    o.total_registros                       AS registros_origen,
    d.total_registros                       AS registros_destino,
    COALESCE(o.total_registros, 0)
      - COALESCE(d.total_registros, 0)      AS diferencia_registros,
    o.suma_montos                           AS montos_origen,
    d.suma_montos                           AS montos_destino,
    ROUND(
        COALESCE(o.suma_montos, 0)
        - COALESCE(d.suma_montos, 0),
        2
    )                                       AS diferencia_montos,
    CASE
        WHEN o.CATEGORIA_PRODUCTO IS NULL   THEN 'FAIL - Categoría extra en destino'
        WHEN d.CATEGORIA_PRODUCTO IS NULL   THEN 'FAIL - Categoría ausente en destino'
        WHEN o.total_registros
           = d.total_registros
         AND ROUND(o.suma_montos, 2)
           = ROUND(d.suma_montos, 2)        THEN 'PASS'
        ELSE                                     'FAIL - Discrepancia detectada'
    END                                     AS estado_validacion
FROM (
    SELECT
        CATEGORIA_PRODUCTO,
        COUNT(*)          AS total_registros,
        SUM(MONTO_TOTAL)  AS suma_montos
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
    GROUP BY CATEGORIA_PRODUCTO
) AS o
FULL OUTER JOIN (
    SELECT
        CATEGORIA_PRODUCTO,
        COUNT(*)          AS total_registros,
        SUM(MONTO_TOTAL)  AS suma_montos
    FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
    GROUP BY CATEGORIA_PRODUCTO
) AS d
    ON o.CATEGORIA_PRODUCTO = d.CATEGORIA_PRODUCTO
ORDER BY diferencia_registros DESC NULLS LAST;
```

#### Resultado esperado

La consulta del Paso 2.2 mostrará una fila por categoría con las diferencias calculadas y una columna `estado_validacion` que clasifica cada categoría como `PASS` o `FAIL`.

Con este dataset, varias categorías deben aparecer como `FAIL`, porque hay faltantes, extras y modificaciones intencionales.

#### Verificación

```sql
-- Verificación: contar categorías con discrepancia
WITH comparacion AS (
    SELECT
        COALESCE(o.CATEGORIA_PRODUCTO, d.CATEGORIA_PRODUCTO) AS categoria_producto,
        COALESCE(o.total_registros, 0) AS registros_origen,
        COALESCE(d.total_registros, 0) AS registros_destino,
        ROUND(COALESCE(o.suma_montos, 0), 2) AS montos_origen,
        ROUND(COALESCE(d.suma_montos, 0), 2) AS montos_destino
    FROM (
        SELECT CATEGORIA_PRODUCTO, COUNT(*) AS total_registros, SUM(MONTO_TOTAL) AS suma_montos
        FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
        GROUP BY CATEGORIA_PRODUCTO
    ) o
    FULL OUTER JOIN (
        SELECT CATEGORIA_PRODUCTO, COUNT(*) AS total_registros, SUM(MONTO_TOTAL) AS suma_montos
        FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
        GROUP BY CATEGORIA_PRODUCTO
    ) d
        ON o.CATEGORIA_PRODUCTO = d.CATEGORIA_PRODUCTO
)
SELECT
    COUNT(*) AS categorias_revisadas,
    COUNT(CASE
        WHEN registros_origen <> registros_destino
          OR montos_origen <> montos_destino
        THEN 1
    END) AS categorias_con_discrepancia
FROM comparacion;
```

> ✅ Identifica qué categorías tienen discrepancias. Estas serán el foco de la investigación en los pasos siguientes.

---

### Ejercicio 3 — Registros faltantes y extra: operadores `EXCEPT` e `INTERSECT`

**Objetivo:** Usar `EXCEPT` para identificar registros presentes en origen pero ausentes en destino, y `INTERSECT` para confirmar cuántos registros coinciden exactamente en ambos schemas.

#### Instrucciones

**Paso 3.1 — Ejecuta la siguiente consulta para encontrar registros en origen que no llegaron al destino:**

```sql
-- ============================================================
-- PASO 3a: Registros en ORIGEN que NO están en DESTINO
-- Comparación por fila completa.
-- ============================================================

SELECT
    PEDIDO_ID,
    CLIENTE_ID,
    FECHA_PEDIDO,
    MONTO_TOTAL,
    CATEGORIA_PRODUCTO
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS

EXCEPT

SELECT
    PEDIDO_ID,
    CLIENTE_ID,
    FECHA_PEDIDO,
    MONTO_TOTAL,
    CATEGORIA_PRODUCTO
FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS

ORDER BY PEDIDO_ID;
```

**Paso 3.2 — Ejecuta la consulta inversa para detectar registros que aparecen en destino pero no tienen correspondencia en origen (registros fantasma o generados incorrectamente):**

```sql
-- ============================================================
-- PASO 3b: Registros en DESTINO que NO están en ORIGEN
-- Comparación por fila completa.
-- ============================================================

SELECT
    PEDIDO_ID,
    CLIENTE_ID,
    FECHA_PEDIDO,
    MONTO_TOTAL,
    CATEGORIA_PRODUCTO
FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS

EXCEPT

SELECT
    PEDIDO_ID,
    CLIENTE_ID,
    FECHA_PEDIDO,
    MONTO_TOTAL,
    CATEGORIA_PRODUCTO
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS

ORDER BY PEDIDO_ID;
```

**Paso 3.3 — Finalmente, cuantifica cuántos registros coinciden exactamente en ambas tablas:**

```sql
-- ============================================================
-- PASO 3c: Registros que coinciden EXACTAMENTE en ambos schemas
-- ============================================================

SELECT COUNT(*) AS registros_coincidentes
FROM (
    SELECT
        PEDIDO_ID,
        CLIENTE_ID,
        FECHA_PEDIDO,
        MONTO_TOTAL,
        CATEGORIA_PRODUCTO
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS

    INTERSECT

    SELECT
        PEDIDO_ID,
        CLIENTE_ID,
        FECHA_PEDIDO,
        MONTO_TOTAL,
        CATEGORIA_PRODUCTO
    FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
) AS coincidentes;
```

#### Resultado esperado

- **Paso 3.1:** muestra registros de origen que no aparecen de forma idéntica en destino.
- **Paso 3.2:** muestra registros de destino que no aparecen de forma idéntica en origen.
- **Paso 3.3:** devuelve el número de registros que coinciden exactamente.

Con este dataset, `INTERSECT` debe devolver **23 registros coincidentes exactos**.

> 💡 **Nota conceptual:** `EXCEPT` compara filas completas. Si un registro tiene el mismo `PEDIDO_ID` pero un `MONTO_TOTAL`, `FECHA_PEDIDO` o `CATEGORIA_PRODUCTO` diferente, `EXCEPT` lo tratará como una fila distinta. Por eso el resultado de `EXCEPT` incluye tanto registros realmente faltantes como registros modificados.

---

**Paso 3.4 — Comparar faltantes por llave de negocio**

Ahora identifica únicamente registros que existen en origen pero no tienen ninguna llave correspondiente en destino:

```sql
-- ============================================================
-- PASO 3d: Faltantes por llave PEDIDO_ID
-- Esta consulta NO cuenta como faltantes los registros modificados.
-- ============================================================

SELECT
    o.PEDIDO_ID,
    o.CLIENTE_ID,
    o.FECHA_PEDIDO,
    o.MONTO_TOTAL,
    o.CATEGORIA_PRODUCTO
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
LEFT JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
    ON o.PEDIDO_ID = d.PEDIDO_ID
WHERE d.PEDIDO_ID IS NULL
ORDER BY o.PEDIDO_ID;
```

**Resultado esperado:** deben aparecer exactamente los pedidos `6007`, `6015` y `6029`.

---

**Paso 3.5 — Comparar extras por llave de negocio**

```sql
-- ============================================================
-- PASO 3e: Extras por llave PEDIDO_ID
-- Esta consulta identifica registros creados en destino sin fuente verificable.
-- ============================================================

SELECT
    d.PEDIDO_ID,
    d.CLIENTE_ID,
    d.FECHA_PEDIDO,
    d.MONTO_TOTAL,
    d.CATEGORIA_PRODUCTO
FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
LEFT JOIN LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
    ON d.PEDIDO_ID = o.PEDIDO_ID
WHERE o.PEDIDO_ID IS NULL
ORDER BY d.PEDIDO_ID;
```

**Resultado esperado:** deben aparecer exactamente los pedidos `7001` y `7002`.

#### Verificación

```sql
-- Verificación: comparación entre fila completa y llave
WITH except_origen_menos_destino AS (
    SELECT PEDIDO_ID, CLIENTE_ID, FECHA_PEDIDO, MONTO_TOTAL, CATEGORIA_PRODUCTO
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
    EXCEPT
    SELECT PEDIDO_ID, CLIENTE_ID, FECHA_PEDIDO, MONTO_TOTAL, CATEGORIA_PRODUCTO
    FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
),
faltantes_por_llave AS (
    SELECT o.PEDIDO_ID
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
    LEFT JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
        ON o.PEDIDO_ID = d.PEDIDO_ID
    WHERE d.PEDIDO_ID IS NULL
)
SELECT
    (SELECT COUNT(*) FROM except_origen_menos_destino) AS filas_no_identicas_en_destino,
    (SELECT COUNT(*) FROM faltantes_por_llave) AS faltantes_reales_por_llave;
```

**Resultado esperado:** `filas_no_identicas_en_destino` debe ser mayor que `faltantes_reales_por_llave`, porque también incluye registros modificados.

---

### Ejercicio 4 — Detección de diferencias de valores: `FULL OUTER JOIN` con auditoría

**Objetivo:** Construir una tabla de auditoría completa que clasifique cada registro según su estado de coincidencia, distinguiendo entre registros faltantes, extra y con valores modificados.

#### Instrucciones

**Paso 4.1 — Ejecuta la consulta de auditoría detallada con FULL OUTER JOIN**

```sql
-- ============================================================
-- PASO 4: Tabla de auditoría completa con FULL OUTER JOIN
-- Clasifica cada registro según su estado de coincidencia.
-- ============================================================

SELECT
    COALESCE(o.PEDIDO_ID,   d.PEDIDO_ID)    AS pedido_id,
    COALESCE(o.CLIENTE_ID,  d.CLIENTE_ID)   AS cliente_id,
    o.MONTO_TOTAL                           AS monto_origen,
    d.MONTO_TOTAL                           AS monto_destino,
    o.FECHA_PEDIDO                          AS fecha_origen,
    d.FECHA_PEDIDO                          AS fecha_destino,
    o.CATEGORIA_PRODUCTO                    AS categoria_origen,
    d.CATEGORIA_PRODUCTO                    AS categoria_destino,
    CASE
        WHEN o.PEDIDO_ID IS NULL                    THEN 'EXTRA EN DESTINO'
        WHEN d.PEDIDO_ID IS NULL                    THEN 'FALTANTE EN DESTINO'
        WHEN ROUND(o.MONTO_TOTAL, 2)
          <> ROUND(d.MONTO_TOTAL, 2)                THEN 'MONTO DIFERENTE'
        WHEN o.FECHA_PEDIDO <> d.FECHA_PEDIDO       THEN 'FECHA DIFERENTE'
        WHEN o.CATEGORIA_PRODUCTO
          <> d.CATEGORIA_PRODUCTO                   THEN 'CATEGORÍA DIFERENTE'
        ELSE                                             'OK'
    END                                     AS estado_auditoria
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS  AS o
FULL OUTER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS AS d
    ON o.PEDIDO_ID = d.PEDIDO_ID
ORDER BY
    CASE estado_auditoria
        WHEN 'FALTANTE EN DESTINO' THEN 1
        WHEN 'EXTRA EN DESTINO'    THEN 2
        WHEN 'MONTO DIFERENTE'     THEN 3
        WHEN 'FECHA DIFERENTE'     THEN 4
        WHEN 'CATEGORÍA DIFERENTE' THEN 5
        ELSE                            6
    END,
    pedido_id;
```

> 💡 **¿Por qué `FULL OUTER JOIN`?** Porque necesitas ver todos los casos: registros que existen en ambas tablas, registros que solo existen en origen y registros que solo existen en destino. Un `INNER JOIN` ocultaría faltantes y extras. Un `LEFT JOIN` desde origen ocultaría extras en destino.

---

**Paso 4.2 — Para obtener el resumen ejecutivo de la auditoría (cuántos registros caen en cada categoría), ejecuta:**

```sql
-- ============================================================
-- PASO 4b: Resumen ejecutivo de la auditoría
-- ============================================================

WITH auditoria AS (
    SELECT
        COALESCE(o.PEDIDO_ID, d.PEDIDO_ID) AS pedido_id,
        CASE
            WHEN o.PEDIDO_ID IS NULL                    THEN 'EXTRA EN DESTINO'
            WHEN d.PEDIDO_ID IS NULL                    THEN 'FALTANTE EN DESTINO'
            WHEN ROUND(o.MONTO_TOTAL, 2)
              <> ROUND(d.MONTO_TOTAL, 2)                THEN 'MONTO DIFERENTE'
            WHEN o.FECHA_PEDIDO <> d.FECHA_PEDIDO       THEN 'FECHA DIFERENTE'
            WHEN o.CATEGORIA_PRODUCTO
              <> d.CATEGORIA_PRODUCTO                   THEN 'CATEGORÍA DIFERENTE'
            ELSE                                             'OK'
        END AS estado_auditoria
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS  AS o
    FULL OUTER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS AS d
        ON o.PEDIDO_ID = d.PEDIDO_ID
)
SELECT
    estado_auditoria,
    COUNT(*)                                            AS cantidad,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS porcentaje
FROM auditoria
GROUP BY estado_auditoria
ORDER BY
    CASE estado_auditoria
        WHEN 'OK' THEN 1
        WHEN 'FALTANTE EN DESTINO' THEN 2
        WHEN 'EXTRA EN DESTINO' THEN 3
        WHEN 'MONTO DIFERENTE' THEN 4
        WHEN 'FECHA DIFERENTE' THEN 5
        WHEN 'CATEGORÍA DIFERENTE' THEN 6
        ELSE 7
    END;
```

#### Resultado esperado

Con este dataset, el resumen debe mostrar:

| ESTADO_AUDITORIA | CANTIDAD |
|---|---:|
| OK | 23 |
| FALTANTE EN DESTINO | 3 |
| EXTRA EN DESTINO | 2 |
| MONTO DIFERENTE | 2 |
| FECHA DIFERENTE | 1 |
| CATEGORÍA DIFERENTE | 1 |

La suma total de `cantidad` debe ser **32**, porque el `FULL OUTER JOIN` incluye:

- 28 llaves compartidas entre origen y destino.
- 3 llaves solo en origen.
- 2 llaves solo en destino.

Sin embargo, de las 28 llaves compartidas, 4 tienen diferencias de valores y 23 están completamente correctas en las columnas auditadas.

#### Verificación

```sql
-- Verificar que el total del resumen coincide con el total del FULL OUTER JOIN
WITH auditoria AS (
    SELECT COALESCE(o.PEDIDO_ID, d.PEDIDO_ID) AS pedido_id
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
    FULL OUTER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
        ON o.PEDIDO_ID = d.PEDIDO_ID
)
SELECT COUNT(*) AS total_filas_auditoria
FROM auditoria;
```

**Resultado esperado:** `32`.

---

### Ejercicio 5 — Checksums de fila: validación con `MD5`

**Objetivo:** Implementar una técnica de validación más robusta usando `MD5()` para generar un checksum de fila completa y detectar cualquier diferencia en cualquier campo, sin necesidad de comparar columna por columna.

#### Instrucciones

**Paso 5.1 — Primero, entiende cómo funciona MD5() en Snowflake para generar un hash de fila:**

```sql
-- ============================================================
-- PASO 5a: Demostración de MD5 para checksum de fila
-- ============================================================

-- MD5 convierte la concatenación de todos los campos en un hash único.
-- Si cualquier campo cambia, el hash cambia.
-- El separador "|" evita ambigüedades entre valores concatenados.
SELECT
    PEDIDO_ID,
    MD5(
        CONCAT(
            COALESCE(CAST(PEDIDO_ID AS VARCHAR), ''),
            '|',
            COALESCE(CAST(CLIENTE_ID AS VARCHAR), ''),
            '|',
            COALESCE(TO_CHAR(FECHA_PEDIDO, 'YYYY-MM-DD'), ''),
            '|',
            COALESCE(TO_CHAR(MONTO_TOTAL, '9999999990.00'), ''),
            '|',
            COALESCE(CATEGORIA_PRODUCTO, '')
        )
    ) AS checksum_fila
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
LIMIT 5;
```

> 💡 **Nota técnica:** El separador `|` en la concatenación para `MD5` es importante. Sin él, valores como `('12', '3')` y `('1', '23')` podrían producir la misma cadena final (`'123'`). Siempre usa un separador que no aparezca en los datos.

---

**Paso 5.2 — Ahora compara los checksums entre origen y destino para detectar cualquier modificación de datos:**

```sql
-- ============================================================
-- PASO 5b: Comparación de checksums entre ORIGEN y DESTINO
-- ============================================================

WITH checksums_origen AS (
    SELECT
        PEDIDO_ID,
        MD5(
            CONCAT(
                COALESCE(CAST(PEDIDO_ID AS VARCHAR), ''),
                '|',
                COALESCE(CAST(CLIENTE_ID AS VARCHAR), ''),
                '|',
                COALESCE(TO_CHAR(FECHA_PEDIDO, 'YYYY-MM-DD'), ''),
                '|',
                COALESCE(TO_CHAR(MONTO_TOTAL, '9999999990.00'), ''),
                '|',
                COALESCE(CATEGORIA_PRODUCTO, '')
            )
        ) AS checksum_fila
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
),

checksums_destino AS (
    SELECT
        PEDIDO_ID,
        MD5(
            CONCAT(
                COALESCE(CAST(PEDIDO_ID AS VARCHAR), ''),
                '|',
                COALESCE(CAST(CLIENTE_ID AS VARCHAR), ''),
                '|',
                COALESCE(TO_CHAR(FECHA_PEDIDO, 'YYYY-MM-DD'), ''),
                '|',
                COALESCE(TO_CHAR(MONTO_TOTAL, '9999999990.00'), ''),
                '|',
                COALESCE(CATEGORIA_PRODUCTO, '')
            )
        ) AS checksum_fila
    FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
)

SELECT
    COALESCE(o.PEDIDO_ID, d.PEDIDO_ID)  AS pedido_id,
    o.checksum_fila                     AS checksum_origen,
    d.checksum_fila                     AS checksum_destino,
    CASE
        WHEN o.PEDIDO_ID IS NULL         THEN 'EXTRA EN DESTINO'
        WHEN d.PEDIDO_ID IS NULL         THEN 'FALTANTE EN DESTINO'
        WHEN o.checksum_fila
          = d.checksum_fila              THEN 'IDÉNTICO'
        ELSE                                  'MODIFICADO'
    END                                 AS estado_checksum
FROM checksums_origen  AS o
FULL OUTER JOIN checksums_destino AS d
    ON o.PEDIDO_ID = d.PEDIDO_ID
ORDER BY
    CASE estado_checksum
        WHEN 'FALTANTE EN DESTINO' THEN 1
        WHEN 'EXTRA EN DESTINO'    THEN 2
        WHEN 'MODIFICADO'          THEN 3
        ELSE                            4
    END,
    pedido_id;
```

---

**Paso 5.3 — Obtén el resumen de la comparación por checksum**

```sql
-- ============================================================
-- PASO 5c: Resumen de validación por checksum
-- ============================================================

WITH checksums_origen AS (
    SELECT
        PEDIDO_ID,
        MD5(CONCAT(
            COALESCE(CAST(PEDIDO_ID AS VARCHAR), ''), '|',
            COALESCE(CAST(CLIENTE_ID AS VARCHAR), ''), '|',
            COALESCE(TO_CHAR(FECHA_PEDIDO, 'YYYY-MM-DD'), ''), '|',
            COALESCE(TO_CHAR(MONTO_TOTAL, '9999999990.00'), ''), '|',
            COALESCE(CATEGORIA_PRODUCTO, '')
        )) AS checksum_fila
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
),
checksums_destino AS (
    SELECT
        PEDIDO_ID,
        MD5(CONCAT(
            COALESCE(CAST(PEDIDO_ID AS VARCHAR), ''), '|',
            COALESCE(CAST(CLIENTE_ID AS VARCHAR), ''), '|',
            COALESCE(TO_CHAR(FECHA_PEDIDO, 'YYYY-MM-DD'), ''), '|',
            COALESCE(TO_CHAR(MONTO_TOTAL, '9999999990.00'), ''), '|',
            COALESCE(CATEGORIA_PRODUCTO, '')
        )) AS checksum_fila
    FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
),
comparacion AS (
    SELECT
        CASE
            WHEN o.PEDIDO_ID IS NULL         THEN 'EXTRA EN DESTINO'
            WHEN d.PEDIDO_ID IS NULL         THEN 'FALTANTE EN DESTINO'
            WHEN o.checksum_fila
              = d.checksum_fila              THEN 'IDÉNTICO'
            ELSE                                  'MODIFICADO'
        END AS estado_checksum
    FROM checksums_origen  AS o
    FULL OUTER JOIN checksums_destino AS d
        ON o.PEDIDO_ID = d.PEDIDO_ID
)
SELECT
    estado_checksum,
    COUNT(*)                                            AS cantidad,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS porcentaje
FROM comparacion
GROUP BY estado_checksum
ORDER BY
    CASE estado_checksum
        WHEN 'IDÉNTICO' THEN 1
        WHEN 'FALTANTE EN DESTINO' THEN 2
        WHEN 'EXTRA EN DESTINO' THEN 3
        WHEN 'MODIFICADO' THEN 4
        ELSE 5
    END;
```

#### Resultado esperado

| ESTADO_CHECKSUM | CANTIDAD |
|---|---:|
| IDÉNTICO | 23 |
| FALTANTE EN DESTINO | 3 |
| EXTRA EN DESTINO | 2 |
| MODIFICADO | 4 |

#### Verificación

```sql
-- Verificación: MODIFICADO debe coincidir con monto + fecha + categoría diferentes
WITH diferencias AS (
    SELECT 'MONTO' AS TIPO, COUNT(*) AS CASOS
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
    INNER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
        ON o.PEDIDO_ID = d.PEDIDO_ID
    WHERE ROUND(o.MONTO_TOTAL, 2) <> ROUND(d.MONTO_TOTAL, 2)

    UNION ALL

    SELECT 'FECHA', COUNT(*)
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
    INNER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
        ON o.PEDIDO_ID = d.PEDIDO_ID
    WHERE o.FECHA_PEDIDO <> d.FECHA_PEDIDO

    UNION ALL

    SELECT 'CATEGORIA', COUNT(*)
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
    INNER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
        ON o.PEDIDO_ID = d.PEDIDO_ID
    WHERE o.CATEGORIA_PRODUCTO <> d.CATEGORIA_PRODUCTO
)
SELECT SUM(CASOS) AS modificaciones_por_columnas
FROM diferencias;
```

**Resultado esperado:** `4`.

---

### Ejercicio 6 — Reporte de reconciliación consolidado: CTEs reutilizables

**Objetivo:** Empaquetar todas las validaciones anteriores en un único script con CTEs reutilizables que genere un reporte de reconciliación completo con métricas cuantificables y clasificación `PASS/FAIL` para cada control.

#### Instrucciones

**Paso 6.1 — Ejecuta el reporte de reconciliación consolidado:**

```sql
-- ============================================================
-- PASO 6: REPORTE DE RECONCILIACIÓN CONSOLIDADO
-- Pipeline de calidad de datos: VENTAS_ORIGEN → VENTAS_DESTINO
-- Fecha de ejecución: CURRENT_TIMESTAMP()
-- ============================================================

WITH

-- ── Control 1: Conteo total de registros ────────────────────
ctrl_conteo AS (
    SELECT
        'CTRL-01'                               AS control_id,
        'Conteo total de registros'             AS descripcion_control,
        (SELECT COUNT(*) FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS)  AS valor_origen,
        (SELECT COUNT(*) FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS) AS valor_destino,
        ABS(
            (SELECT COUNT(*) FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS)
            -
            (SELECT COUNT(*) FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS)
        ) AS cantidad_hallazgos
),

-- ── Control 2: Suma total de montos ─────────────────────────
ctrl_suma AS (
    SELECT
        'CTRL-02'                               AS control_id,
        'Suma total de montos'                  AS descripcion_control,
        ROUND((SELECT SUM(MONTO_TOTAL) FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS), 2)  AS valor_origen,
        ROUND((SELECT SUM(MONTO_TOTAL) FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS), 2) AS valor_destino,
        ABS(
            ROUND((SELECT SUM(MONTO_TOTAL) FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS), 2)
            -
            ROUND((SELECT SUM(MONTO_TOTAL) FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS), 2)
        ) AS cantidad_hallazgos
),

-- ── Control 3: Registros faltantes por llave ────────────────
ctrl_faltantes AS (
    SELECT
        'CTRL-03'                               AS control_id,
        'Registros faltantes por PEDIDO_ID'     AS descripcion_control,
        COUNT(*)                                AS valor_origen,
        0                                       AS valor_destino,
        COUNT(*)                                AS cantidad_hallazgos
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
    LEFT JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
        ON o.PEDIDO_ID = d.PEDIDO_ID
    WHERE d.PEDIDO_ID IS NULL
),

-- ── Control 4: Registros extra por llave ────────────────────
ctrl_extra AS (
    SELECT
        'CTRL-04'                               AS control_id,
        'Registros extra por PEDIDO_ID'         AS descripcion_control,
        0                                       AS valor_origen,
        COUNT(*)                                AS valor_destino,
        COUNT(*)                                AS cantidad_hallazgos
    FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
    LEFT JOIN LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
        ON d.PEDIDO_ID = o.PEDIDO_ID
    WHERE o.PEDIDO_ID IS NULL
),

-- ── Control 5: Registros con monto modificado ───────────────
ctrl_montos AS (
    SELECT
        'CTRL-05'                               AS control_id,
        'Registros con monto modificado'        AS descripcion_control,
        COUNT(*)                                AS valor_origen,
        0                                       AS valor_destino,
        COUNT(*)                                AS cantidad_hallazgos
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS AS o
    INNER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS AS d
        ON o.PEDIDO_ID = d.PEDIDO_ID
    WHERE ROUND(o.MONTO_TOTAL, 2) <> ROUND(d.MONTO_TOTAL, 2)
),

-- ── Control 6: Registros con fecha modificada ───────────────
ctrl_fechas AS (
    SELECT
        'CTRL-06'                               AS control_id,
        'Registros con fecha modificada'        AS descripcion_control,
        COUNT(*)                                AS valor_origen,
        0                                       AS valor_destino,
        COUNT(*)                                AS cantidad_hallazgos
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS AS o
    INNER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS AS d
        ON o.PEDIDO_ID = d.PEDIDO_ID
    WHERE o.FECHA_PEDIDO <> d.FECHA_PEDIDO
),

-- ── Control 7: Registros con categoría modificada ───────────
ctrl_categorias AS (
    SELECT
        'CTRL-07'                               AS control_id,
        'Registros con categoría modificada'    AS descripcion_control,
        COUNT(*)                                AS valor_origen,
        0                                       AS valor_destino,
        COUNT(*)                                AS cantidad_hallazgos
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS AS o
    INNER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS AS d
        ON o.PEDIDO_ID = d.PEDIDO_ID
    WHERE o.CATEGORIA_PRODUCTO <> d.CATEGORIA_PRODUCTO
),

-- ── Unión de todos los controles ────────────────────────────
todos_los_controles AS (
    SELECT * FROM ctrl_conteo
    UNION ALL
    SELECT * FROM ctrl_suma
    UNION ALL
    SELECT * FROM ctrl_faltantes
    UNION ALL
    SELECT * FROM ctrl_extra
    UNION ALL
    SELECT * FROM ctrl_montos
    UNION ALL
    SELECT * FROM ctrl_fechas
    UNION ALL
    SELECT * FROM ctrl_categorias
)

-- ── Reporte final con clasificación PASS/FAIL ───────────────
SELECT
    control_id,
    descripcion_control,
    valor_origen,
    valor_destino,
    ROUND(ABS(valor_origen - valor_destino), 2) AS diferencia_absoluta,
    cantidad_hallazgos,
    CASE
        WHEN cantidad_hallazgos = 0 THEN 'PASS ✓'
        ELSE 'FAIL ✗'
    END AS resultado,
    CASE
        WHEN cantidad_hallazgos = 0 THEN 'Sin hallazgos'
        WHEN cantidad_hallazgos BETWEEN 1 AND 2 THEN 'Bajo'
        WHEN cantidad_hallazgos BETWEEN 3 AND 5 THEN 'Moderado'
        ELSE 'Crítico'
    END AS severidad,
    CURRENT_TIMESTAMP() AS timestamp_ejecucion
FROM todos_los_controles
ORDER BY control_id;
```

#### Resultado esperado

El reporte debe mostrar **7 controles**:

| CONTROL_ID | DESCRIPCION_CONTROL | RESULTADO ESPERADO |
|---|---|---|
| CTRL-01 | Conteo total de registros | FAIL |
| CTRL-02 | Suma total de montos | FAIL |
| CTRL-03 | Registros faltantes por PEDIDO_ID | FAIL |
| CTRL-04 | Registros extra por PEDIDO_ID | FAIL |
| CTRL-05 | Registros con monto modificado | FAIL |
| CTRL-06 | Registros con fecha modificada | FAIL |
| CTRL-07 | Registros con categoría modificada | FAIL |

> 💡 **Buenas prácticas:** Este patrón de reporte con CTEs es directamente reutilizable. Para adaptarlo a otras tablas, solo necesitas cambiar los nombres de las tablas y los campos en cada CTE. La estructura de `UNION ALL` al final permite agregar nuevos controles sin modificar la lógica existente.

---

## Validación y Pruebas

Una vez completados todos los pasos, ejecuta este bloque de validación final para confirmar que el laboratorio fue completado correctamente:

```sql
-- ============================================================
-- VALIDACIÓN FINAL DEL LABORATORIO 6
-- Confirma que todas las consultas produjeron resultados
-- ============================================================

-- Prueba 1: Confirmar diferencia de volumen detectada
SELECT
    'Prueba 1: Diferencia de volumen' AS prueba,
    CASE
        WHEN ABS(
            (SELECT COUNT(*) FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS) -
            (SELECT COUNT(*) FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS)
        ) > 0 THEN 'PASS - Discrepancia detectada correctamente'
        ELSE 'REVISAR - No se detectó discrepancia de volumen'
    END AS resultado;
```

```sql
-- Prueba 2: Confirmar que EXCEPT identifica filas no idénticas
SELECT
    'Prueba 2: Filas no idénticas con EXCEPT' AS prueba,
    CASE
        WHEN COUNT(*) > 0 THEN 'PASS - Se identificaron ' || COUNT(*) || ' filas no idénticas'
        ELSE 'REVISAR - EXCEPT no devolvió resultados'
    END AS resultado
FROM (
    SELECT PEDIDO_ID, CLIENTE_ID, FECHA_PEDIDO, MONTO_TOTAL, CATEGORIA_PRODUCTO
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
    EXCEPT
    SELECT PEDIDO_ID, CLIENTE_ID, FECHA_PEDIDO, MONTO_TOTAL, CATEGORIA_PRODUCTO
    FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
) AS diferencias;
```

```sql
-- Prueba 3: Confirmar faltantes reales por llave
SELECT
    'Prueba 3: Faltantes reales por llave' AS prueba,
    CASE
        WHEN COUNT(*) = 3 THEN 'PASS - Se identificaron 3 faltantes reales por llave'
        ELSE 'REVISAR - Se esperaban 3 faltantes, se obtuvieron: ' || COUNT(*)
    END AS resultado
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
LEFT JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
    ON o.PEDIDO_ID = d.PEDIDO_ID
WHERE d.PEDIDO_ID IS NULL;
```

```sql
-- Prueba 4: Confirmar extras reales por llave
SELECT
    'Prueba 4: Extras reales por llave' AS prueba,
    CASE
        WHEN COUNT(*) = 2 THEN 'PASS - Se identificaron 2 extras reales por llave'
        ELSE 'REVISAR - Se esperaban 2 extras, se obtuvieron: ' || COUNT(*)
    END AS resultado
FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
LEFT JOIN LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
    ON d.PEDIDO_ID = o.PEDIDO_ID
WHERE o.PEDIDO_ID IS NULL;
```

```sql
-- Prueba 5: Confirmar resumen de auditoría
WITH auditoria AS (
    SELECT
        COALESCE(o.PEDIDO_ID, d.PEDIDO_ID) AS pedido_id,
        CASE
            WHEN o.PEDIDO_ID IS NULL THEN 'EXTRA EN DESTINO'
            WHEN d.PEDIDO_ID IS NULL THEN 'FALTANTE EN DESTINO'
            WHEN ROUND(o.MONTO_TOTAL, 2) <> ROUND(d.MONTO_TOTAL, 2) THEN 'MONTO DIFERENTE'
            WHEN o.FECHA_PEDIDO <> d.FECHA_PEDIDO THEN 'FECHA DIFERENTE'
            WHEN o.CATEGORIA_PRODUCTO <> d.CATEGORIA_PRODUCTO THEN 'CATEGORÍA DIFERENTE'
            ELSE 'OK'
        END AS estado_auditoria
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
    FULL OUTER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
        ON o.PEDIDO_ID = d.PEDIDO_ID
)
SELECT
    'Prueba 5: Auditoría completa' AS prueba,
    CASE
        WHEN COUNT(*) = 32 THEN 'PASS - Auditoría completa con 32 filas'
        ELSE 'REVISAR - Se esperaban 32 filas, se obtuvieron: ' || COUNT(*)
    END AS resultado
FROM auditoria;
```

```sql
-- Prueba 6: Confirmar que el reporte de reconciliación tiene 7 controles
WITH controles AS (
    SELECT 'CTRL-01' AS control_id
    UNION ALL SELECT 'CTRL-02'
    UNION ALL SELECT 'CTRL-03'
    UNION ALL SELECT 'CTRL-04'
    UNION ALL SELECT 'CTRL-05'
    UNION ALL SELECT 'CTRL-06'
    UNION ALL SELECT 'CTRL-07'
)
SELECT
    'Prueba 6: Estructura del reporte' AS prueba,
    CASE
        WHEN COUNT(*) = 7 THEN 'PASS - Reporte con 7 controles'
        ELSE 'REVISAR'
    END AS resultado
FROM controles;
```

**Criterios de éxito del laboratorio:**

| Prueba | Criterio de éxito |
|---|---|
| Prueba 1 | `PASS - Discrepancia detectada correctamente` |
| Prueba 2 | `PASS - Se identificaron N filas no idénticas` |
| Prueba 3 | `PASS - Se identificaron 3 faltantes reales por llave` |
| Prueba 4 | `PASS - Se identificaron 2 extras reales por llave` |
| Prueba 5 | `PASS - Auditoría completa con 32 filas` |
| Prueba 6 | `PASS - Reporte con 7 controles` |
| Paso 4.2 | Resumen ejecutivo con estados `OK`, `FALTANTE`, `EXTRA`, `MONTO`, `FECHA` y `CATEGORÍA` |
| Paso 5.3 | Porcentaje de `IDÉNTICO` calculado correctamente |
| Paso 6 | Reporte con clasificación `PASS/FAIL` |

---

## Resultados esperados clave con el dataset cargado

Estos resultados ayudan al instructor y al alumno a validar rápidamente que el laboratorio se está ejecutando sobre el dataset correcto.

| Consulta / Validación | Resultado esperado |
|---|---:|
| Registros en `VENTAS_ORIGEN.PEDIDOS` | 30 |
| Registros en `VENTAS_DESTINO.PEDIDOS` | 29 |
| Faltantes reales por `PEDIDO_ID` | 3 |
| Extras reales por `PEDIDO_ID` | 2 |
| Registros con monto diferente | 2 |
| Registros con fecha diferente | 1 |
| Registros con categoría diferente | 1 |
| Registros idénticos por checksum | 23 |
| Registros modificados por checksum | 4 |
| Filas del `FULL OUTER JOIN` de auditoría | 32 |
| Controles del reporte final | 7 |

---

## Solución de Problemas

### Problema 1 — Error "Object does not exist" al referenciar `VENTAS_ORIGEN` o `VENTAS_DESTINO`

**Síntoma:**

```text
SQL compilation error: Object 'LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS' does not exist or not authorized.
```

**Causa:**

El script de setup de esta práctica no fue ejecutado, fue ejecutado en otro database o el rol actual no tiene permisos sobre los schemas. También puede ocurrir si el alumno está parado en otra base de datos o si copió las consultas sin el prefijo completo del schema.

**Solución:**

```sql
-- Paso 1: Verificar el contexto actual
SELECT
    CURRENT_WAREHOUSE(),
    CURRENT_DATABASE(),
    CURRENT_SCHEMA(),
    CURRENT_ROLE();

-- Paso 2: Establecer el contexto correcto
USE WAREHOUSE COMPUTE_WH;
USE DATABASE LAB_SQL_INTERMEDIO;

-- Paso 3: Verificar que los schemas existen
SHOW SCHEMAS IN DATABASE LAB_SQL_INTERMEDIO;

-- Paso 4: Verificar que las tablas existen
SHOW TABLES IN SCHEMA LAB_SQL_INTERMEDIO.VENTAS_ORIGEN;
SHOW TABLES IN SCHEMA LAB_SQL_INTERMEDIO.VENTAS_DESTINO;
```

Si `VENTAS_ORIGEN` o `VENTAS_DESTINO` no aparecen, regresa al folder **SETUP-LABS** y ejecuta nuevamente el script **06_SETUP_DATOS_RECONCILIACION**.

---

### Problema 2 — `EXCEPT` devuelve más filas de las esperadas

**Síntoma:**

El Paso 3.1 devuelve más filas que los faltantes reales por llave, aunque el Paso 3.4 muestra solo 3 faltantes.

**Causa:**

`EXCEPT` compara filas completas. Si el número de columnas seleccionadas en ambas partes es idéntico, pero un valor cambia, Snowflake considera que la fila de origen no existe en destino. Por ejemplo, si `PEDIDO_ID = 6003` existe en ambos schemas pero `MONTO_TOTAL` cambió de `1250.00` a `1300.00`, `EXCEPT` lo reportará como una diferencia.

**Solución:**

Usa `EXCEPT` cuando quieras detectar filas no idénticas. Usa `LEFT JOIN` por `PEDIDO_ID` cuando quieras detectar registros realmente faltantes por llave.

```sql
-- Faltantes reales por llave
SELECT o.*
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS o
LEFT JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS d
    ON o.PEDIDO_ID = d.PEDIDO_ID
WHERE d.PEDIDO_ID IS NULL;
```

> 💡 **Buena práctica:** En reconciliación de datos, siempre separa dos preguntas:
>
> 1. ¿La llave existe en ambos datasets?
> 2. Si existe, ¿los valores son iguales?

---

### Problema 3 — El checksum marca modificados aunque visualmente los datos parecen iguales

**Síntoma:**

El Paso 5 muestra registros como `MODIFICADO`, pero al revisar los valores parecen iguales.

**Causa:**

El checksum es sensible al formato de texto usado para construir la cadena. Si una fecha o un número se convierte a texto con diferente formato, el hash cambia aunque el valor de negocio parezca igual.

**Solución:**

Estandariza siempre los formatos antes de aplicar `MD5`:

```sql
TO_CHAR(FECHA_PEDIDO, 'YYYY-MM-DD')
TO_CHAR(MONTO_TOTAL, '9999999990.00')
```

Este laboratorio ya usa formatos explícitos para reducir falsos positivos.

---

### Problema 4 — El reporte final marca `PASS` cuando sí hay extras en destino

**Síntoma:**

Un control de extras aparece como `PASS` aunque existen registros extra en destino.

**Causa:**

Esto ocurre cuando la lógica de `PASS/FAIL` evalúa solo una columna que siempre es cero. Por ejemplo, si `valor_origen = 0` y `valor_destino = cantidad_extras`, una regla como `valor_origen = 0 THEN PASS` estaría mal diseñada.

**Solución:**

Usa una columna normalizada como `cantidad_hallazgos` y evalúa:

```sql
CASE
    WHEN cantidad_hallazgos = 0 THEN 'PASS ✓'
    ELSE 'FAIL ✗'
END
```

El reporte final de esta práctica ya aplica este patrón para evitar falsos `PASS`.

---

## Limpieza del entorno

Al finalizar el laboratorio, ejecuta los siguientes comandos para liberar recursos y evitar consumo innecesario de créditos Snowflake:

```sql
-- ============================================================
-- LIMPIEZA POST-LABORATORIO 6
-- ============================================================

-- Suspender el warehouse para detener el consumo de créditos.
-- IMPORTANTE: ejecutar siempre al terminar la sesión.
ALTER WAREHOUSE COMPUTE_WH SUSPEND;
```

```sql
-- Verificar que el warehouse fue suspendido correctamente.
SHOW WAREHOUSES LIKE 'COMPUTE_WH';
-- El campo STATE debe mostrar SUSPENDED.
```

> ⚠️ **Recordatorio de créditos:** Las cuentas trial de Snowflake tienen 400 USD de créditos. Un warehouse `X-SMALL` consume aproximadamente 1 crédito por hora de actividad. Suspenderlo al terminar cada sesión es una práctica obligatoria en este curso.

No es necesario eliminar schemas ni tablas, ya que esta práctica puede reutilizarse para repasar reconciliación de datasets. Si necesitas reiniciar el laboratorio, vuelve a ejecutar el script **06_SETUP_DATOS_RECONCILIACION**.

---

## Resumen

En este laboratorio aplicaste un flujo completo de validación y reconciliación de datasets usando técnicas SQL avanzadas en Snowflake:

| Técnica | Uso en el laboratorio | Ejercicio |
|---|---|---|
| `UNION ALL` | Comparación de métricas agregadas entre origen y destino | 1, 2 |
| `FULL OUTER JOIN` + `CASE WHEN` | Tabla de auditoría detallada con clasificación de estado | 2, 4 |
| `EXCEPT` | Identificación de filas presentes en origen pero no idénticas en destino | 3 |
| `INTERSECT` | Cuantificación de registros exactamente coincidentes | 3 |
| `LEFT JOIN` por llave | Detección de faltantes y extras reales por `PEDIDO_ID` | 3 |
| `MD5()` + `CONCAT()` | Checksum de fila para detectar cualquier modificación de datos | 5 |
| CTEs encadenadas | Encapsulación de cada control como bloque reutilizable | 6 |
| `PASS/FAIL` con `CASE WHEN` | Reporte ejecutivo de calidad con clasificación binaria | 6 |

### Hallazgos clave del laboratorio

Al completar este laboratorio, identificaste que el pipeline `VENTAS_ORIGEN → VENTAS_DESTINO` presenta los siguientes problemas intencionales:

1. **Pérdida de registros:** 3 pedidos existen en origen pero no fueron migrados al destino.
2. **Registros fantasma:** 2 pedidos aparecen en destino sin correspondencia en origen.
3. **Modificación de valores:** 2 registros presentan diferencias en `MONTO_TOTAL`.
4. **Modificación de fechas:** 1 registro presenta diferencia en `FECHA_PEDIDO`.
5. **Modificación de categoría:** 1 registro presenta diferencia en `CATEGORIA_PRODUCTO`.
6. **Registros correctos:** 23 registros coinciden exactamente entre origen y destino.

### Patrón de reconciliación reutilizable

El flujo aplicado puede usarse como patrón estándar de auditoría de pipelines:

```text
1. VOLUMEN     → ¿Cuántos registros llegaron?
2. DESGLOSE    → ¿En qué categorías hay discrepancias?
3. FALTANTES   → ¿Qué registros específicos se perdieron?
4. EXTRAS      → ¿Qué registros aparecieron sin fuente?
5. VALORES     → ¿Qué registros llegaron pero con datos modificados?
6. CHECKSUM    → ¿Qué filas completas son idénticas o diferentes?
7. REPORTE     → ¿Cómo se comunica el estado general del pipeline?
```

Este patrón, empaquetado como CTEs reutilizables, puede adaptarse a cualquier tabla y cualquier pipeline con mínimas modificaciones.

### Conexión con los próximos laboratorios

Las técnicas de este laboratorio son la base para prácticas de calidad de datos más avanzadas:

- Validación de cargas incrementales.
- Comparación entre capas bronze, silver y gold.
- Auditoría de migraciones.
- Control de calidad de pipelines ETL/ELT.
- Generación de reportes automáticos de reconciliación.

### Recursos adicionales

| Recurso | URL |
|---|---|
| Documentación Snowflake: Operadores de conjunto (`EXCEPT`, `INTERSECT`, `UNION`) | https://docs.snowflake.com/en/sql-reference/operators-query |
| Documentación Snowflake: Función `MD5` | https://docs.snowflake.com/en/sql-reference/functions/md5 |
| Documentación Snowflake: `JOIN` | https://docs.snowflake.com/en/sql-reference/constructs/join |
| Documentación Snowflake: `CASE` | https://docs.snowflake.com/en/sql-reference/functions/case |
| dbt Labs: Data tests | https://docs.getdbt.com/docs/build/data-tests |
| Great Expectations: Data validation | https://docs.greatexpectations.io/docs/ |

---
