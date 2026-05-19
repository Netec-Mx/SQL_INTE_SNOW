# Detección de duplicados y registros inconsistentes

## Metadatos

| Campo | Detalle |
|---|---|
| **Duración estimada** | 60 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar (*Apply*) |
| **Módulo** | 3 — Calidad de datos y detección de anomalías |
| **Plataforma** | Snowflake (Snowsight Worksheet) |
| **Schema de práctica** | `LAB_SQL_INTERMEDIO.VENTAS` |

---

## Descripción General

En este laboratorio trabajarás con versiones intencionalmente degradadas de las tablas del schema de práctica (`CLIENTES_DIRTY` y `PEDIDOS_DIRTY`) que contienen duplicados, valores nulos críticos y referencias inválidas. Aplicarás tres técnicas complementarias: detección de duplicados con `GROUP BY` + `HAVING`, marcado y aislamiento de duplicados con `ROW_NUMBER()`, y validación de integridad referencial con `LEFT JOIN` + `IS NULL`. Al finalizar, construirás un reporte consolidado de calidad de datos que integra todos los hallazgos.

Este laboratorio introduce formalmente las **window functions** como puente conceptual hacia el Laboratorio 4.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Detectar registros duplicados usando `GROUP BY` con `HAVING COUNT(*) > 1` sobre claves simples y compuestas.
- [ ] Aplicar `ROW_NUMBER() OVER(PARTITION BY ... ORDER BY ...)` para numerar duplicados e identificar el registro canónico a conservar.
- [ ] Utilizar la cláusula `QUALIFY` de Snowflake para filtrar resultados de window functions directamente en la consulta.
- [ ] Validar integridad referencial entre tablas mediante `LEFT JOIN` con filtro `WHERE ... IS NULL` para detectar registros huérfanos.
- [ ] Construir un reporte de calidad de datos que consolide duplicados, nulos críticos e inconsistencias de referencia.

---

## Prerrequisitos

### Conocimientos previos

| Área | Nivel requerido |
|---|---|
| `SELECT`, `FROM`, `WHERE`, `ORDER BY` | Sólido |
| `GROUP BY` con funciones de agregación (`COUNT`, `SUM`) | Sólido |
| `HAVING` para filtrado de grupos | Sólido |
| `JOIN`, especialmente `LEFT JOIN` | Intermedio |
| Concepto de `NULL` y operadores `IS NULL` / `IS NOT NULL` | Intermedio |
| Concepto introductorio de window functions | Básico |

### Acceso y configuración

| Requisito | Detalle |
|---|---|
| Cuenta Snowflake activa | Trial o corporativa con rol que permita crear objetos de laboratorio |
| Script de setup ejecutado | No se asume script previo. Esta práctica incluye el setup completo de base, schema, tablas limpias y tablas dirty. |
| Database disponible | `LAB_SQL_INTERMEDIO` |
| Schema disponible | `LAB_SQL_INTERMEDIO.VENTAS` |
| Tablas requeridas | `CLIENTES`, `PEDIDOS`, `CLIENTES_DIRTY`, `PEDIDOS_DIRTY`, creadas en el Paso 0 |
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
| Snowflake (Snowsight) | Versión web actual | Ejecución de consultas SQL |
| Visual Studio Code *(opcional)* | 1.80+ | Edición de scripts localmente |
| SnowSQL *(opcional)* | 1.2.x+ | Ejecución desde terminal |

---

## Organización recomendada de Workspace en Snowsight

Para que la práctica sea ordenada y reutilizable, trabaja con un Workspace y 2 folders. En esta práctica se usa la palabra **workspace** como una separación lógica de trabajo dentro de Snowsight; técnicamente, en Snowflake trabajarás con **Workspace**.

| Workspace / Worksheet | Folder | Nombre sugerido | Uso |
|---|---|---|---|
| `SNOWLABS-INT` | `SETUP-LABS` | `03_SETUP_DATOS_DUPLICADOS_DIRTY` | Crear database, schema, tablas limpias, tablas dirty y datos de prueba. Se ejecuta una vez al inicio o cuando quieras reiniciar el laboratorio. |
| `SNOWLABS-INT` | `SCRIPT-LABS` | `03_LAB_DUPLICADOS_CALIDAD_DATOS` | Ejecutar los ejercicios del laboratorio sin mezclar el script de carga de datos. |

---

## Paso 0 — Preparación del ambiente y carga de datos

### Paso 0.0 — Crear el workspace de las prácticas

1. Entra a **Snowsight**.
2. Da clic en la opción **Projects**.
3. Clic en **+**.
4. Selecciona la opción **Private workspace**.
5. Nómbralo: **`SNOWLABS-INT`**.
6. Clic en **Create**.

### Paso 0.0.1 — Crear el folder y script que carga los datos

1. Dentro del workspace **`SNOWLABS-INT`**, da clic en **+ Add new**.
2. Clic en **Folder** y nómbralo: **`SETUP-LABS`**.
3. Dentro del folder **`SETUP-LABS`**, da clic en el símbolo **+**.
4. Crea un archivo de tipo **SQL**.
5. Nómbralo: **`03_SETUP_DATOS_DUPLICADOS_DIRTY`**.
6. Pega ahí el siguiente script completo.
7. Ejecuta el script completo antes de comenzar el laboratorio.

Este dataset está diseñado para activar todos los escenarios de la práctica:

- Clientes duplicados por `ID_CLIENTE`.
- Clientes duplicados por `EMAIL`.
- Clientes con `EMAIL` nulo.
- Pedidos duplicados por `ID_PEDIDO`.
- Pedidos duplicados por combinación de negocio: `ID_CLIENTE + FECHA_PEDIDO + MONTO_TOTAL`.
- Pedidos con `ID_CLIENTE` nulo.
- Pedidos con referencia a clientes inexistentes.
- Clientes sin pedidos.
- Tablas limpias de referencia para comparación.

```sql
-- 03_setup_datos_duplicados_dirty_snowflake.sql
-- Práctica Snowflake Intermedio
-- Dataset para completar el laboratorio:
-- Detección de duplicados y registros inconsistentes
--
-- Objetivo del dataset:
-- 1) Mantener tablas limpias de referencia: CLIENTES y PEDIDOS.
-- 2) Crear tablas degradadas: CLIENTES_DIRTY y PEDIDOS_DIRTY.
-- 3) Incluir duplicados técnicos por ID.
-- 4) Incluir duplicados de negocio por EMAIL o combinación de pedido.
-- 5) Incluir valores nulos críticos.
-- 6) Incluir referencias inválidas para practicar LEFT JOIN + IS NULL.
--
-- Importante:
-- Esta práctica usa los nombres de columnas estandarizados desde el Lab 02:
-- ID_CLIENTE, ID_PEDIDO, MONTO_TOTAL y ESTADO_PEDIDO.

USE WAREHOUSE COMPUTE_WH;

CREATE DATABASE IF NOT EXISTS LAB_SQL_INTERMEDIO;
USE DATABASE LAB_SQL_INTERMEDIO;

CREATE SCHEMA IF NOT EXISTS VENTAS;
USE SCHEMA VENTAS;

-- Opcional para repetir el laboratorio desde cero.
DROP TABLE IF EXISTS PEDIDOS_DIRTY;
DROP TABLE IF EXISTS CLIENTES_DIRTY;
DROP TABLE IF EXISTS PEDIDOS;
DROP TABLE IF EXISTS CLIENTES;

-- ============================================================
-- TABLAS LIMPIAS DE REFERENCIA
-- ============================================================

CREATE OR REPLACE TABLE CLIENTES (
    ID_CLIENTE NUMBER(10,0) NOT NULL,
    NOMBRE VARCHAR(120) NOT NULL,
    EMAIL VARCHAR(150),
    FECHA_REGISTRO DATE NOT NULL,
    CIUDAD VARCHAR(80) NOT NULL,
    PAIS VARCHAR(80) NOT NULL,
    CONSTRAINT PK_CLIENTES PRIMARY KEY (ID_CLIENTE)
);

CREATE OR REPLACE TABLE PEDIDOS (
    ID_PEDIDO NUMBER(10,0) NOT NULL,
    ID_CLIENTE NUMBER(10,0) NOT NULL,
    FECHA_PEDIDO DATE NOT NULL,
    MONTO_TOTAL NUMBER(12,2) NOT NULL,
    ESTADO_PEDIDO VARCHAR(30) NOT NULL,
    CANAL VARCHAR(40),
    CONSTRAINT PK_PEDIDOS PRIMARY KEY (ID_PEDIDO),
    CONSTRAINT FK_PEDIDOS_CLIENTES FOREIGN KEY (ID_CLIENTE) REFERENCES CLIENTES(ID_CLIENTE)
);

INSERT INTO CLIENTES (ID_CLIENTE, NOMBRE, EMAIL, FECHA_REGISTRO, CIUDAD, PAIS) VALUES
    (1,  'Ana Torres',       'ana.torres@demo.com',       '2023-01-15', 'CDMX',        'México'),
    (2,  'Luis Martínez',    'luis.martinez@demo.com',    '2023-03-02', 'CDMX',        'México'),
    (3,  'María López',      'maria.lopez@demo.com',      '2023-02-20', 'Guadalajara', 'México'),
    (4,  'Carlos Hernández', 'carlos.hernandez@demo.com', '2023-05-10', 'Guadalajara', 'México'),
    (5,  'Sofía Ramírez',    'sofia.ramirez@demo.com',    '2023-06-18', 'Monterrey',   'México'),
    (6,  'Jorge Castillo',   'jorge.castillo@demo.com',   '2023-07-22', 'Monterrey',   'México'),
    (7,  'Elena Flores',     'elena.flores@demo.com',     '2023-08-04', 'Puebla',      'México'),
    (8,  'Diego Sánchez',    'diego.sanchez@demo.com',    '2023-09-11', 'Puebla',      'México'),
    (9,  'Valeria Cruz',     'valeria.cruz@demo.com',     '2023-10-05', 'Mérida',      'México'),
    (10, 'Roberto Díaz',     'roberto.diaz@demo.com',     '2023-11-01', 'Mérida',      'México'),
    (11, 'Paola Medina',     'paola.medina@demo.com',     '2024-02-14', 'Querétaro',   'México'),
    (12, 'Andrés Navarro',   'andres.navarro@demo.com',   '2024-04-03', 'Querétaro',   'México');

INSERT INTO PEDIDOS (ID_PEDIDO, ID_CLIENTE, FECHA_PEDIDO, MONTO_TOTAL, ESTADO_PEDIDO, CANAL) VALUES
    (2001, 1,  '2024-01-12',  500.00, 'COMPLETADO', 'Web'),
    (2002, 1,  '2024-03-15', 1000.00, 'COMPLETADO', 'Ejecutivo'),
    (2003, 2,  '2024-02-03',  200.00, 'COMPLETADO', 'Web'),
    (2004, 2,  '2023-11-21',  500.00, 'ENVIADO',    'Web'),
    (2005, 3,  '2024-04-09',  300.00, 'COMPLETADO', 'Marketplace'),
    (2006, 3,  '2025-01-18',  600.00, 'COMPLETADO', 'Ejecutivo'),
    (2007, 4,  '2024-05-06',  700.00, 'COMPLETADO', 'Ejecutivo'),
    (2008, 4,  '2024-09-19',  900.00, 'COMPLETADO', 'Ejecutivo'),
    (2009, 5,  '2023-12-05',  500.00, 'COMPLETADO', 'Web'),
    (2010, 5,  '2024-06-14',  600.00, 'COMPLETADO', 'Marketplace'),
    (2011, 6,  '2024-07-07',  100.00, 'COMPLETADO', 'Web'),
    (2012, 6,  '2024-08-25',  200.00, 'COMPLETADO', 'Web'),
    (2013, 7,  '2024-10-02',  400.00, 'COMPLETADO', 'Ejecutivo'),
    (2014, 7,  '2025-02-12',  900.00, 'COMPLETADO', 'Ejecutivo'),
    (2015, 8,  '2024-11-17',  650.00, 'COMPLETADO', 'Marketplace'),
    (2016, 8,  '2024-12-09',  650.00, 'COMPLETADO', 'Marketplace'),
    (2017, 9,  '2024-01-28',  800.00, 'COMPLETADO', 'Ejecutivo'),
    (2018, 9,  '2025-03-03', 1200.00, 'COMPLETADO', 'Ejecutivo'),
    (2019, 10, '2023-10-30',  200.00, 'COMPLETADO', 'Web'),
    (2020, 10, '2024-04-22',  400.00, 'COMPLETADO', 'Web'),
    (2021, 11, '2025-04-10',  350.00, 'EN_PROCESO', 'Web'),
    (2022, 12, '2025-05-01',  450.00, 'COMPLETADO', 'Web');

-- ============================================================
-- TABLAS DIRTY
-- ============================================================

CREATE OR REPLACE TABLE CLIENTES_DIRTY (
    ID_CLIENTE NUMBER(10,0),
    NOMBRE VARCHAR(120),
    EMAIL VARCHAR(150),
    FECHA_REGISTRO DATE,
    CIUDAD VARCHAR(80),
    PAIS VARCHAR(80),
    FUENTE_CARGA VARCHAR(40),
    FECHA_CARGA TIMESTAMP_NTZ
);

CREATE OR REPLACE TABLE PEDIDOS_DIRTY (
    ID_PEDIDO NUMBER(10,0),
    ID_CLIENTE NUMBER(10,0),
    FECHA_PEDIDO DATE,
    MONTO_TOTAL NUMBER(12,2),
    ESTADO_PEDIDO VARCHAR(30),
    CANAL VARCHAR(40),
    FUENTE_CARGA VARCHAR(40),
    FECHA_CARGA TIMESTAMP_NTZ
);

-- Carga base limpia hacia CLIENTES_DIRTY.
INSERT INTO CLIENTES_DIRTY
SELECT
    ID_CLIENTE,
    NOMBRE,
    EMAIL,
    FECHA_REGISTRO,
    CIUDAD,
    PAIS,
    'CRM_BASE' AS FUENTE_CARGA,
    TO_TIMESTAMP_NTZ('2025-05-01 08:00:00') AS FECHA_CARGA
FROM CLIENTES;

-- Problemas intencionales en CLIENTES_DIRTY.
INSERT INTO CLIENTES_DIRTY
    (ID_CLIENTE, NOMBRE, EMAIL, FECHA_REGISTRO, CIUDAD, PAIS, FUENTE_CARGA, FECHA_CARGA)
VALUES
    (3,  'Maria López',      'maria.lopez@demo.com',      '2023-02-20', 'Guadalajara', 'México', 'CRM_REPROCESO', '2025-05-02 09:10:00'),
    (5,  'Sofia Ramírez',    'sofia.ramirez@demo.com',    '2023-06-18', 'Monterrey',   'México', 'CRM_REPROCESO', '2025-05-02 09:15:00'),
    (8,  'Diego Sánchez',    NULL,                        '2023-09-11', 'Puebla',      'México', 'CRM_REPROCESO', '2025-05-02 09:20:00'),
    (13, 'Ana T.',           'ana.torres@demo.com',       '2024-01-20', 'CDMX',        'México', 'IMPORT_EXCEL',  '2025-05-03 10:00:00'),
    (14, 'Roberto Diaz',     'roberto.diaz@demo.com',     '2024-03-10', 'Mérida',      'México', 'IMPORT_EXCEL',  '2025-05-03 10:05:00'),
    (15, 'Cliente Sin Email', NULL,                        '2025-04-25', 'Toluca',      'México', 'FORM_WEB',      '2025-05-03 11:00:00'),
    (16, 'Cliente Sin Pedido','cliente.sinpedido@demo.com','2025-04-27', 'León',        'México', 'FORM_WEB',      '2025-05-03 11:10:00');

-- Carga base limpia hacia PEDIDOS_DIRTY.
INSERT INTO PEDIDOS_DIRTY
SELECT
    ID_PEDIDO,
    ID_CLIENTE,
    FECHA_PEDIDO,
    MONTO_TOTAL,
    ESTADO_PEDIDO,
    CANAL,
    'ERP_BASE' AS FUENTE_CARGA,
    TO_TIMESTAMP_NTZ('2025-05-01 08:30:00') AS FECHA_CARGA
FROM PEDIDOS;

-- Problemas intencionales en PEDIDOS_DIRTY.
INSERT INTO PEDIDOS_DIRTY
    (ID_PEDIDO, ID_CLIENTE, FECHA_PEDIDO, MONTO_TOTAL, ESTADO_PEDIDO, CANAL, FUENTE_CARGA, FECHA_CARGA)
VALUES
    (2006, 3,   '2025-01-18',  600.00, 'COMPLETADO', 'Ejecutivo',   'ERP_REPROCESO', '2025-05-02 12:00:00'),
    (2012, 6,   '2024-08-25',  200.00, 'COMPLETADO', 'Web',         'ERP_REPROCESO', '2025-05-02 12:05:00'),
    (2015, 8,   '2024-11-17',  650.00, 'COMPLETADO', 'Marketplace', 'ERP_REPROCESO', '2025-05-02 12:10:00'),
    (3001, 1,   '2024-03-15', 1000.00, 'COMPLETADO', 'Ejecutivo',   'API_DUP',       '2025-05-03 13:00:00'),
    (3002, 5,   '2024-06-14',  600.00, 'COMPLETADO', 'Marketplace', 'API_DUP',       '2025-05-03 13:05:00'),
    (3003, NULL,'2025-04-20',  750.00, 'COMPLETADO', 'Web',         'FORM_ERROR',    '2025-05-03 14:00:00'),
    (3004, 999, '2025-04-21', 1200.00, 'COMPLETADO', 'Web',         'FORM_ERROR',    '2025-05-03 14:05:00'),
    (3005, 998, '2025-04-22',  300.00, 'ENVIADO',    'Ejecutivo',   'FORM_ERROR',    '2025-05-03 14:10:00');

-- Validación rápida del dataset.
SELECT 'CLIENTES' AS TABLA, COUNT(*) AS FILAS FROM CLIENTES
UNION ALL
SELECT 'PEDIDOS' AS TABLA, COUNT(*) AS FILAS FROM PEDIDOS
UNION ALL
SELECT 'CLIENTES_DIRTY' AS TABLA, COUNT(*) AS FILAS FROM CLIENTES_DIRTY
UNION ALL
SELECT 'PEDIDOS_DIRTY' AS TABLA, COUNT(*) AS FILAS FROM PEDIDOS_DIRTY
ORDER BY TABLA;

-- Resultado esperado:
-- CLIENTES        = 12
-- CLIENTES_DIRTY  = 19
-- PEDIDOS         = 22
-- PEDIDOS_DIRTY   = 30
```

---

### Paso 0.0.2 — Crear el folder y script de laboratorio

1. Da clic en el botón **+ Add new**.
2. Clic en **Folder** y nómbralo: **`SCRIPT-LABS`**.
3. Dentro de **`SCRIPT-LABS`**, crea un archivo de tipo **SQL**.
4. Nómbralo: **`03_LAB_DUPLICADOS_CALIDAD_DATOS`**.
5. Usa este archivo para ejecutar los ejercicios 1, 2, 3, 4 y 5.
6. **No pegues aquí el script de carga completo; solo usa las consultas de análisis del laboratorio.**

---

### Paso 0.1 — Confirmar el contexto de trabajo

Dentro del archivo **`03_LAB_DUPLICADOS_CALIDAD_DATOS`**, ejecuta:

```sql
USE WAREHOUSE COMPUTE_WH;
USE DATABASE LAB_SQL_INTERMEDIO;
USE SCHEMA VENTAS;

SELECT
    CURRENT_WAREHOUSE() AS WAREHOUSE_ACTUAL,
    CURRENT_DATABASE()  AS DATABASE_ACTUAL,
    CURRENT_SCHEMA()    AS SCHEMA_ACTUAL;
```

**Resultado esperado:**

| WAREHOUSE_ACTUAL | DATABASE_ACTUAL | SCHEMA_ACTUAL |
|---|---|---|
| COMPUTE_WH | LAB_SQL_INTERMEDIO | VENTAS |

---

### Paso 0.2 — Confirmar que las tablas quedaron disponibles

Ejecuta:

```sql
SHOW TABLES;
```

**Resultado esperado:** deben aparecer al menos estas tablas:

| Tabla | Uso en la práctica |
|---|---|
| `CLIENTES` | Tabla limpia de referencia para comparación. |
| `PEDIDOS` | Tabla limpia de referencia para comparación. |
| `CLIENTES_DIRTY` | Tabla degradada con duplicados, emails nulos e inconsistencias. |
| `PEDIDOS_DIRTY` | Tabla degradada con duplicados, cliente nulo y referencias inválidas. |

---

### Paso 0.3 — Validar volumen mínimo de datos

```sql
SELECT 'CLIENTES' AS TABLA, COUNT(*) AS FILAS FROM CLIENTES
UNION ALL
SELECT 'PEDIDOS' AS TABLA, COUNT(*) AS FILAS FROM PEDIDOS
UNION ALL
SELECT 'CLIENTES_DIRTY' AS TABLA, COUNT(*) AS FILAS FROM CLIENTES_DIRTY
UNION ALL
SELECT 'PEDIDOS_DIRTY' AS TABLA, COUNT(*) AS FILAS FROM PEDIDOS_DIRTY
ORDER BY TABLA;
```

**Resultado esperado:**

| TABLA | FILAS |
|---|---:|
| CLIENTES | 12 |
| CLIENTES_DIRTY | 19 |
| PEDIDOS | 22 |
| PEDIDOS_DIRTY | 30 |

---

### Paso 0.4 — Validar duplicados esperados

```sql
SELECT 'Duplicados por ID_CLIENTE' AS PROBLEMA, COUNT(*) AS CASOS
FROM (
    SELECT ID_CLIENTE
    FROM CLIENTES_DIRTY
    GROUP BY ID_CLIENTE
    HAVING COUNT(*) > 1
)

UNION ALL

SELECT 'Duplicados por EMAIL', COUNT(*)
FROM (
    SELECT EMAIL
    FROM CLIENTES_DIRTY
    WHERE EMAIL IS NOT NULL
    GROUP BY EMAIL
    HAVING COUNT(*) > 1
)

UNION ALL

SELECT 'Duplicados por ID_PEDIDO', COUNT(*)
FROM (
    SELECT ID_PEDIDO
    FROM PEDIDOS_DIRTY
    GROUP BY ID_PEDIDO
    HAVING COUNT(*) > 1
)

UNION ALL

SELECT 'Duplicados compuestos en pedidos', COUNT(*)
FROM (
    SELECT ID_CLIENTE, FECHA_PEDIDO, MONTO_TOTAL
    FROM PEDIDOS_DIRTY
    WHERE ID_CLIENTE IS NOT NULL
      AND FECHA_PEDIDO IS NOT NULL
      AND MONTO_TOTAL IS NOT NULL
    GROUP BY ID_CLIENTE, FECHA_PEDIDO, MONTO_TOTAL
    HAVING COUNT(*) > 1
);
```

**Resultado esperado:** todos los conteos deben ser mayores que `0`.

---

### Paso 0.5 — Validar nulos e inconsistencias esperadas

```sql
SELECT 'Emails nulos en CLIENTES_DIRTY' AS PROBLEMA, COUNT(*) AS CASOS
FROM CLIENTES_DIRTY
WHERE EMAIL IS NULL

UNION ALL

SELECT 'Pedidos con ID_CLIENTE nulo', COUNT(*)
FROM PEDIDOS_DIRTY
WHERE ID_CLIENTE IS NULL

UNION ALL

SELECT 'Pedidos con cliente inexistente', COUNT(*)
FROM PEDIDOS_DIRTY P
LEFT JOIN CLIENTES_DIRTY C
    ON P.ID_CLIENTE = C.ID_CLIENTE
WHERE P.ID_CLIENTE IS NOT NULL
  AND C.ID_CLIENTE IS NULL

UNION ALL

SELECT 'Clientes sin pedidos', COUNT(*)
FROM CLIENTES_DIRTY C
LEFT JOIN PEDIDOS_DIRTY P
    ON C.ID_CLIENTE = P.ID_CLIENTE
WHERE P.ID_PEDIDO IS NULL;
```

**Resultado esperado:** todos los conteos deben ser mayores que `0`.

---

## Ejercicios Paso a Paso

---

### Ejercicio 1 — Exploración inicial de las tablas con problemas

**Objetivo:** Familiarizarte con la estructura y el contenido de las tablas `CLIENTES_DIRTY` y `PEDIDOS_DIRTY` antes de comenzar el análisis de calidad.

#### Instrucciones

**Paso 1.1 — Examinar la estructura de las tablas dirty**

```sql
DESCRIBE TABLE CLIENTES_DIRTY;
DESCRIBE TABLE PEDIDOS_DIRTY;
```

**Paso 1.2 — Examina la estructura y primeras filas de `CLIENTES_DIRTY`**

```sql
-- Vista previa de CLIENTES_DIRTY
SELECT *
FROM CLIENTES_DIRTY
ORDER BY ID_CLIENTE, FECHA_CARGA
LIMIT 20;
```

**Paso 1.3 — Examina la estructura y primeras filas de `PEDIDOS_DIRTY`**

```sql
-- Vista previa de PEDIDOS_DIRTY
SELECT *
FROM PEDIDOS_DIRTY
ORDER BY ID_PEDIDO, FECHA_CARGA
LIMIT 30;
```

**Paso 1.4 — Obten un conteo general de tablas limpias y dirty**

```sql
SELECT 'CLIENTES_DIRTY' AS TABLA, COUNT(*) AS TOTAL_REGISTROS FROM CLIENTES_DIRTY
UNION ALL
SELECT 'PEDIDOS_DIRTY', COUNT(*) FROM PEDIDOS_DIRTY
UNION ALL
SELECT 'CLIENTES (limpia)', COUNT(*) FROM CLIENTES
UNION ALL
SELECT 'PEDIDOS (limpia)', COUNT(*) FROM PEDIDOS;
```

**Paso 1.5 — Identificar nulos en columnas críticas de `CLIENTES_DIRTY`**

```sql
-- Diagnóstico de nulos en CLIENTES_DIRTY
SELECT
    COUNT(*)                                         AS TOTAL_FILAS,
    COUNT(ID_CLIENTE)                                AS ID_CLIENTE_NO_NULOS,
    COUNT(*) - COUNT(ID_CLIENTE)                     AS ID_CLIENTE_NULOS,
    COUNT(EMAIL)                                     AS EMAIL_NO_NULOS,
    COUNT(*) - COUNT(EMAIL)                          AS EMAIL_NULOS,
    COUNT(NOMBRE)                                    AS NOMBRE_NO_NULOS,
    COUNT(*) - COUNT(NOMBRE)                         AS NOMBRE_NULOS
FROM CLIENTES_DIRTY;
```

**Paso 1.6 — Identificar nulos en columnas críticas de `PEDIDOS_DIRTY`**

```sql
-- Diagnóstico de nulos en PEDIDOS_DIRTY
SELECT
    COUNT(*)                                         AS TOTAL_FILAS,
    COUNT(ID_PEDIDO)                                 AS ID_PEDIDO_NO_NULOS,
    COUNT(*) - COUNT(ID_PEDIDO)                      AS ID_PEDIDO_NULOS,
    COUNT(ID_CLIENTE)                                AS ID_CLIENTE_NO_NULOS,
    COUNT(*) - COUNT(ID_CLIENTE)                     AS ID_CLIENTE_NULOS,
    COUNT(MONTO_TOTAL)                               AS MONTO_NO_NULOS,
    COUNT(*) - COUNT(MONTO_TOTAL)                    AS MONTO_NULOS
FROM PEDIDOS_DIRTY;
```

#### Resultado esperado

- `CLIENTES_DIRTY` debe tener más registros que `CLIENTES`, indicando duplicados introducidos.
- `PEDIDOS_DIRTY` debe tener más registros que `PEDIDOS`.
- Debes observar valores nulos en `EMAIL` dentro de `CLIENTES_DIRTY`.
- Debes observar valores nulos en `ID_CLIENTE` dentro de `PEDIDOS_DIRTY`.
- La diferencia entre `COUNT(*)` y `COUNT(columna)` revela exactamente cuántos nulos hay en cada campo.

#### Verificación

Anota los siguientes valores en tu hoja de trabajo. Los usarás en el Ejercicio 5.

| Métrica | Valor observado |
|---|---|
| Total filas `CLIENTES_DIRTY` | ______ |
| Total filas `CLIENTES` limpia | ______ |
| Nulos en `EMAIL` de `CLIENTES_DIRTY` | ______ |
| Total filas `PEDIDOS_DIRTY` | ______ |
| Nulos en `ID_CLIENTE` de `PEDIDOS_DIRTY` | ______ |

---

### Ejercicio 2 — Detección de duplicados con `GROUP BY` + `HAVING`

**Objetivo:** Aplicar la técnica de `GROUP BY` con `HAVING COUNT(*) > 1` para identificar claves repetidas en `CLIENTES_DIRTY` y `PEDIDOS_DIRTY`, tanto por clave simple como compuesta.

#### Instrucciones

**Paso 2.1 — Detectar duplicados de `ID_CLIENTE` en `CLIENTES_DIRTY`**

```sql
-- Detección de ID_CLIENTE duplicados
-- ID_CLIENTE representa una clave técnica que debería ser única.
SELECT
    ID_CLIENTE,
    COUNT(*) AS VECES_REGISTRADO
FROM CLIENTES_DIRTY
GROUP BY ID_CLIENTE
HAVING COUNT(*) > 1
ORDER BY VECES_REGISTRADO DESC, ID_CLIENTE;
```

**Paso 2.2 — Detectar duplicados por `EMAIL`**

```sql
-- Detección de EMAIL duplicados
-- EMAIL funciona como identificador de negocio.
-- Se excluyen NULL porque NULL no representa un email real.
SELECT
    EMAIL,
    COUNT(*) AS VECES_REGISTRADO
FROM CLIENTES_DIRTY
WHERE EMAIL IS NOT NULL
GROUP BY EMAIL
HAVING COUNT(*) > 1
ORDER BY VECES_REGISTRADO DESC, EMAIL;
```

> 💡 **Observa la diferencia:** Es posible encontrar más duplicados por `EMAIL` que por `ID_CLIENTE`, o viceversa. Esto refleja diferentes tipos de problemas de carga: en algunos casos se duplicó el registro completo con el mismo ID; en otros se crearon registros nuevos para el mismo cliente real usando el mismo email pero con ID diferente.

**Paso 2.3 — Calcular registros sobrantes en `CLIENTES_DIRTY`**

```sql
-- Total de registros duplicados sobrantes por ID_CLIENTE
-- Si un ID aparece 3 veces, sobran 2 registros.
SELECT
    SUM(VECES_REGISTRADO - 1) AS REGISTROS_SOBRANTES
FROM (
    SELECT
        ID_CLIENTE,
        COUNT(*) AS VECES_REGISTRADO
    FROM CLIENTES_DIRTY
    GROUP BY ID_CLIENTE
    HAVING COUNT(*) > 1
) AS DUPLICADOS_CLIENTE;
```

**Paso 2.4 — Detectar duplicados de `ID_PEDIDO` en `PEDIDOS_DIRTY`**

```sql
-- Detección de ID_PEDIDO duplicados
-- ID_PEDIDO representa una clave técnica que debería ser única.
SELECT
    ID_PEDIDO,
    COUNT(*) AS VECES_REGISTRADO
FROM PEDIDOS_DIRTY
GROUP BY ID_PEDIDO
HAVING COUNT(*) > 1
ORDER BY VECES_REGISTRADO DESC, ID_PEDIDO;
```

**Paso 2.5 — Detectar duplicados compuestos en `PEDIDOS_DIRTY`**

```sql
-- Duplicados compuestos:
-- mismo cliente, misma fecha y mismo monto.
-- Este patrón detecta posible doble procesamiento del mismo pedido,
-- incluso cuando el ID_PEDIDO es diferente.
SELECT
    ID_CLIENTE,
    FECHA_PEDIDO,
    MONTO_TOTAL,
    COUNT(*) AS VECES_REGISTRADO
FROM PEDIDOS_DIRTY
WHERE ID_CLIENTE   IS NOT NULL
  AND FECHA_PEDIDO IS NOT NULL
  AND MONTO_TOTAL  IS NOT NULL
GROUP BY ID_CLIENTE, FECHA_PEDIDO, MONTO_TOTAL
HAVING COUNT(*) > 1
ORDER BY VECES_REGISTRADO DESC, FECHA_PEDIDO DESC;
```

**Paso 2.6 — Visualizar detalle completo de clientes con `ID_CLIENTE` duplicado**

```sql
-- Ver filas completas de clientes con ID_CLIENTE duplicado.
-- El INNER JOIN permite recuperar el detalle completo después de detectar las claves duplicadas.
SELECT
    C.*
FROM CLIENTES_DIRTY C
INNER JOIN (
    SELECT ID_CLIENTE
    FROM CLIENTES_DIRTY
    GROUP BY ID_CLIENTE
    HAVING COUNT(*) > 1
) DUP
    ON C.ID_CLIENTE = DUP.ID_CLIENTE
ORDER BY C.ID_CLIENTE, C.FECHA_REGISTRO, C.FECHA_CARGA;
```

**Paso 2.7 — Visualiza el detalle completo de los registros duplicados para poder compararlos. Usa un INNER JOIN contra el resultado de la detección:**

```sql
-- Ver filas completas de pedidos con ID_PEDIDO duplicado.
SELECT
    P.*
FROM PEDIDOS_DIRTY P
INNER JOIN (
    SELECT ID_PEDIDO
    FROM PEDIDOS_DIRTY
    GROUP BY ID_PEDIDO
    HAVING COUNT(*) > 1
) DUP
    ON P.ID_PEDIDO = DUP.ID_PEDIDO
ORDER BY P.ID_PEDIDO, P.FECHA_CARGA;
```

#### Resultado esperado

- El paso 2.1 muestra una lista de `ID_CLIENTE` que aparecen más de una vez.
- El paso 2.2 muestra emails repetidos que indican duplicados de negocio.
- El paso 2.3 devuelve un número entero: la cantidad exacta de filas que sobran por duplicados de `ID_CLIENTE`.
- El paso 2.5 revela duplicados que no se detectarían solo por `ID_PEDIDO`.
- Los pasos 2.6 y 2.7 muestran pares o tríos de filas que representan el mismo cliente o pedido.

#### Verificación

```sql
-- Preguntas de análisis — responde en comentarios:
-- 1. ¿Cuántos ID_CLIENTE distintos tienen duplicados en CLIENTES_DIRTY?
-- 2. ¿El número de duplicados por EMAIL coincide con el de duplicados por ID_CLIENTE?
-- 3. ¿Qué indica si no coinciden?
-- 4. ¿Cuántos registros sobrantes calculó el Paso 2.3?
-- 5. ¿Qué duplicado compuesto no se detectaría si solo revisas ID_PEDIDO?
```

---

### Ejercicio 3 — Deduplicación con `ROW_NUMBER()`

**Objetivo:** Aplicar la función de ventana `ROW_NUMBER()` para numerar los duplicados dentro de cada grupo y aislar el registro canónico, es decir, el registro que se debe conservar según un criterio de negocio definido.

> 📘 **Concepto clave — Window Functions:** A diferencia de las funciones de agregación que colapsan múltiples filas en una sola, las window functions calculan un valor **para cada fila** considerando un conjunto de filas relacionadas, llamado "ventana". `ROW_NUMBER()` asigna un número secuencial único a cada fila dentro de una partición. La sintaxis es: `ROW_NUMBER() OVER(PARTITION BY columna ORDER BY columna)`. La cláusula `PARTITION BY` define los grupos, parecido a `GROUP BY`, y `ORDER BY` define el criterio de numeración.

#### Instrucciones

**Paso 3.1 — Aplica ROW_NUMBER() para numerar los duplicados de CLIENTE_ID, ordenando por FECHA_REGISTRO ascendente (conservar el registro más antiguo como canónico):**

```sql
-- Numerar registros por ID_CLIENTE con ROW_NUMBER()
-- El registro con RN = 1 es el que conservaríamos.
-- Criterio de negocio: conservar el registro más antiguo por FECHA_REGISTRO.
SELECT
    ID_CLIENTE,
    NOMBRE,
    EMAIL,
    FECHA_REGISTRO,
    FUENTE_CARGA,
    FECHA_CARGA,
    ROW_NUMBER() OVER (
        PARTITION BY ID_CLIENTE
        ORDER BY FECHA_REGISTRO ASC, FECHA_CARGA ASC
    ) AS RN
FROM CLIENTES_DIRTY
ORDER BY ID_CLIENTE, RN;
```

Observa cómo cada grupo de CLIENTE_ID recibe numeración independiente comenzando en 1.

**Paso 3.2 — Construye una consulta que aísle solo los duplicados (filas con rn > 1), usando una subconsulta:**

```sql
-- Aislar únicamente los registros duplicados.
-- Son las filas con RN > 1: registros que podrían eliminarse o revisarse.
SELECT *
FROM (
    SELECT
        ID_CLIENTE,
        NOMBRE,
        EMAIL,
        FECHA_REGISTRO,
        FUENTE_CARGA,
        FECHA_CARGA,
        ROW_NUMBER() OVER (
            PARTITION BY ID_CLIENTE
            ORDER BY FECHA_REGISTRO ASC, FECHA_CARGA ASC
        ) AS RN
    FROM CLIENTES_DIRTY
) AS NUMERADOS
WHERE RN > 1
ORDER BY ID_CLIENTE, RN;
```

**Paso 3.3 — Usa la cláusula QUALIFY de Snowflake para lograr el mismo resultado de forma más concisa. QUALIFY permite filtrar directamente sobre el resultado de una window function sin necesidad de subconsulta:**

```sql
-- Equivalente con QUALIFY.
-- QUALIFY permite filtrar directamente sobre el resultado de una window function
-- sin necesidad de envolver la consulta en una subconsulta.
SELECT
    ID_CLIENTE,
    NOMBRE,
    EMAIL,
    FECHA_REGISTRO,
    FUENTE_CARGA,
    FECHA_CARGA,
    ROW_NUMBER() OVER (
        PARTITION BY ID_CLIENTE
        ORDER BY FECHA_REGISTRO ASC, FECHA_CARGA ASC
    ) AS RN
FROM CLIENTES_DIRTY
QUALIFY RN > 1
ORDER BY ID_CLIENTE, RN;
```

> ⚠️ **Nota de portabilidad:** `QUALIFY` es una cláusula soportada por Snowflake y resulta muy útil para filtrar window functions de forma directa. Si necesitas portar este código a un motor que no soporte `QUALIFY`, usa la versión con subconsulta del Paso 3.2.

**Paso 3.4 — Construye la consulta de deduplicación: obtén solo el registro canónico de cada CLIENTE_ID (el que se conservaría tras una limpieza):**

```sql
-- Dataset deduplicado: un registro por ID_CLIENTE.
SELECT
    ID_CLIENTE,
    NOMBRE,
    EMAIL,
    FECHA_REGISTRO,
    CIUDAD,
    PAIS,
    FUENTE_CARGA,
    FECHA_CARGA
FROM CLIENTES_DIRTY
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY ID_CLIENTE
    ORDER BY FECHA_REGISTRO ASC, FECHA_CARGA ASC
) = 1
ORDER BY ID_CLIENTE;
```

**Paso 3.5 — Verifica que el resultado deduplicado tiene el número correcto de filas. Debería coincidir aproximadamente con la tabla limpia CLIENTES:**

```sql
-- Comparar conteos: deduplicado vs. tabla limpia.
-- En este dataset, el deduplicado puede tener más registros que CLIENTES
-- porque CLIENTES_DIRTY incluye clientes adicionales de carga externa.
SELECT
    'CLIENTES_DIRTY deduplicada' AS ORIGEN,
    COUNT(*)                     AS TOTAL_REGISTROS
FROM (
    SELECT ID_CLIENTE
    FROM CLIENTES_DIRTY
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY ID_CLIENTE
        ORDER BY FECHA_REGISTRO ASC, FECHA_CARGA ASC
    ) = 1
)

UNION ALL

SELECT
    'CLIENTES limpia' AS ORIGEN,
    COUNT(*)          AS TOTAL_REGISTROS
FROM CLIENTES;
```

**Paso 3.6 — Aplica el mismo enfoque a PEDIDOS_DIRTY, esta vez usando como criterio de conservación el registro con FECHA_PEDIDO más reciente en caso de duplicado por PEDIDO_ID:**

```sql
-- Deduplicación de PEDIDOS_DIRTY:
-- conservar el registro más reciente por FECHA_CARGA en caso de duplicado por ID_PEDIDO.
SELECT
    ID_PEDIDO,
    ID_CLIENTE,
    FECHA_PEDIDO,
    MONTO_TOTAL,
    ESTADO_PEDIDO,
    CANAL,
    FUENTE_CARGA,
    FECHA_CARGA,
    ROW_NUMBER() OVER (
        PARTITION BY ID_PEDIDO
        ORDER BY FECHA_CARGA DESC
    ) AS RN
FROM PEDIDOS_DIRTY
QUALIFY RN = 1
ORDER BY ID_PEDIDO;
```

**Paso 3.7 — Aislar pedidos sobrantes por `ID_PEDIDO`**

```sql
-- Pedidos duplicados que no conservaríamos según el criterio anterior.
SELECT
    ID_PEDIDO,
    ID_CLIENTE,
    FECHA_PEDIDO,
    MONTO_TOTAL,
    ESTADO_PEDIDO,
    CANAL,
    FUENTE_CARGA,
    FECHA_CARGA,
    ROW_NUMBER() OVER (
        PARTITION BY ID_PEDIDO
        ORDER BY FECHA_CARGA DESC
    ) AS RN
FROM PEDIDOS_DIRTY
QUALIFY RN > 1
ORDER BY ID_PEDIDO, RN;
```

#### Resultado esperado

- El paso 3.1 muestra todas las filas con su número dentro del grupo.
- Los clientes sin duplicados tienen solo `RN = 1`.
- Los clientes duplicados tienen `RN = 2`, `RN = 3`, etc.
- Los pasos 3.2 y 3.3 deben devolver exactamente el mismo número de filas.
- El paso 3.4 devuelve exactamente una fila por cada `ID_CLIENTE` único.
- El paso 3.6 produce un dataset de pedidos sin duplicados por `ID_PEDIDO`.

#### Verificación

```sql
-- Verificar equivalencia entre subconsulta y QUALIFY para duplicados de clientes.
SELECT 'Subconsulta' AS METODO, COUNT(*) AS FILAS
FROM (
    SELECT
        ID_CLIENTE,
        ROW_NUMBER() OVER (
            PARTITION BY ID_CLIENTE
            ORDER BY FECHA_REGISTRO ASC, FECHA_CARGA ASC
        ) AS RN
    FROM CLIENTES_DIRTY
) T
WHERE RN > 1

UNION ALL

SELECT 'QUALIFY' AS METODO, COUNT(*) AS FILAS
FROM (
    SELECT *
    FROM CLIENTES_DIRTY
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY ID_CLIENTE
        ORDER BY FECHA_REGISTRO ASC, FECHA_CARGA ASC
    ) > 1
);
```

Los dos conteos deben ser idénticos.

---

### Ejercicio 4 — Validación de integridad referencial

**Objetivo:** Detectar registros huérfanos usando `LEFT JOIN` con filtro `WHERE IS NULL`, identificando pedidos sin cliente válido y clientes sin ningún pedido asociado.

#### Instrucciones

**Paso 4.1 — Encuentra pedidos en PEDIDOS_DIRTY cuyo CLIENTE_ID no existe en CLIENTES_DIRTY (pedidos huérfanos — referencias rotas):**

```sql
-- Pedidos con ID_CLIENTE que no existe en CLIENTES_DIRTY.
-- Estos pedidos tienen una referencia rota.
SELECT
    P.ID_PEDIDO,
    P.ID_CLIENTE        AS ID_CLIENTE_EN_PEDIDO,
    P.FECHA_PEDIDO,
    P.MONTO_TOTAL,
    P.ESTADO_PEDIDO,
    C.ID_CLIENTE        AS ID_CLIENTE_EN_CLIENTES
FROM PEDIDOS_DIRTY P
LEFT JOIN CLIENTES_DIRTY C
    ON P.ID_CLIENTE = C.ID_CLIENTE
WHERE C.ID_CLIENTE IS NULL
  AND P.ID_CLIENTE IS NOT NULL
ORDER BY P.ID_PEDIDO;
```

> 💡 **¿Por qué funciona este patrón?** El `LEFT JOIN` mantiene **todas** las filas de `PEDIDOS_DIRTY`, independientemente de si tienen coincidencia en `CLIENTES_DIRTY`. Cuando no hay coincidencia, las columnas de `CLIENTES_DIRTY` aparecen como `NULL`. Filtrar por `WHERE C.ID_CLIENTE IS NULL` selecciona precisamente esos pedidos sin cliente válido.

**Paso 4.2 — Distingue entre pedidos huérfanos por referencia rota vs. pedidos con CLIENTE_ID nulo (son problemas diferentes):**

```sql
-- Categorizar los pedidos problemáticos.
-- Un ID_CLIENTE nulo y un ID_CLIENTE inexistente son problemas diferentes.
SELECT
    CASE
        WHEN P.ID_CLIENTE IS NULL          THEN 'ID_CLIENTE nulo en pedido'
        WHEN C.ID_CLIENTE IS NULL          THEN 'ID_CLIENTE no existe en clientes'
        ELSE                                    'Referencia válida'
    END                                    AS TIPO_PROBLEMA,
    COUNT(*)                               AS CANTIDAD
FROM PEDIDOS_DIRTY P
LEFT JOIN CLIENTES_DIRTY C
    ON P.ID_CLIENTE = C.ID_CLIENTE
GROUP BY TIPO_PROBLEMA
ORDER BY CANTIDAD DESC;
```

**Paso 4.3 — Encuentra clientes en CLIENTES_DIRTY que no tienen ningún pedido en PEDIDOS_DIRTY (clientes sin actividad — posibles registros fantasma o clientes nuevos sin compras):**

```sql
-- Clientes sin ningún pedido asociado.
-- Pueden ser clientes nuevos, registros fantasma o cargas incompletas.
SELECT
    C.ID_CLIENTE,
    C.NOMBRE,
    C.EMAIL,
    C.FECHA_REGISTRO,
    C.FUENTE_CARGA,
    P.ID_PEDIDO         AS ID_PEDIDO_ENCONTRADO
FROM CLIENTES_DIRTY C
LEFT JOIN PEDIDOS_DIRTY P
    ON C.ID_CLIENTE = P.ID_CLIENTE
WHERE P.ID_PEDIDO IS NULL
ORDER BY C.FECHA_REGISTRO DESC;
```

**Paso 4.4 — Valida contra las tablas limpias: ¿los pedidos huérfanos de PEDIDOS_DIRTY también son huérfanos en la tabla limpia PEDIDOS?**

```sql
-- ¿Los pedidos huérfanos en DIRTY también existen en la tabla limpia?
-- Esto ayuda a distinguir errores de carga externa vs. registros reales del sistema.
SELECT
    P_DIRTY.ID_PEDIDO,
    P_DIRTY.ID_CLIENTE,
    CASE
        WHEN P_LIMPIA.ID_PEDIDO IS NOT NULL THEN 'Existe en tabla limpia'
        ELSE                                      'No existe en tabla limpia'
    END AS ESTADO_EN_LIMPIA
FROM (
    SELECT P.ID_PEDIDO, P.ID_CLIENTE
    FROM PEDIDOS_DIRTY P
    LEFT JOIN CLIENTES_DIRTY C
        ON P.ID_CLIENTE = C.ID_CLIENTE
    WHERE C.ID_CLIENTE IS NULL
      AND P.ID_CLIENTE IS NOT NULL
) P_DIRTY
LEFT JOIN PEDIDOS P_LIMPIA
    ON P_DIRTY.ID_PEDIDO = P_LIMPIA.ID_PEDIDO
ORDER BY P_DIRTY.ID_PEDIDO;
```

**Paso 4.5 — Obtén un resumen cuantitativo de los problemas de integridad referencial encontrados:**

```sql
-- Resumen de integridad referencial.
SELECT
    'Pedidos con ID_CLIENTE nulo' AS PROBLEMA,
    COUNT(*) AS CANTIDAD
FROM PEDIDOS_DIRTY
WHERE ID_CLIENTE IS NULL

UNION ALL

SELECT
    'Pedidos con referencia a cliente inexistente' AS PROBLEMA,
    COUNT(*) AS CANTIDAD
FROM PEDIDOS_DIRTY P
LEFT JOIN CLIENTES_DIRTY C
    ON P.ID_CLIENTE = C.ID_CLIENTE
WHERE C.ID_CLIENTE IS NULL
  AND P.ID_CLIENTE IS NOT NULL

UNION ALL

SELECT
    'Clientes sin ningún pedido' AS PROBLEMA,
    COUNT(*) AS CANTIDAD
FROM CLIENTES_DIRTY C
LEFT JOIN PEDIDOS_DIRTY P
    ON C.ID_CLIENTE = P.ID_CLIENTE
WHERE P.ID_PEDIDO IS NULL;
```

#### Resultado esperado

- El paso 4.1 devuelve los pedidos cuyos `ID_CLIENTE` no existen en `CLIENTES_DIRTY`.
- El paso 4.2 categoriza claramente cuántos problemas son de tipo nulo y cuántos son referencias rotas.
- El paso 4.3 devuelve clientes sin pedidos asociados.
- El paso 4.5 produce una tabla resumen de 3 filas con los conteos de cada tipo de problema.

#### Verificación

```sql
-- Preguntas de análisis — responde en comentarios:
-- 1. ¿Cuántos pedidos tienen ID_CLIENTE nulo?
-- 2. ¿Cuántos pedidos referencian clientes inexistentes?
-- 3. ¿Por qué esos dos problemas no deben tratarse igual?
-- 4. ¿Cuántos clientes no tienen ningún pedido?
```

---

### Ejercicio 5 — Construcción del reporte de calidad de datos

**Objetivo:** Consolidar todos los hallazgos de los pasos anteriores en un único reporte de calidad de datos que pueda presentarse a stakeholders técnicos y de negocio.

#### Instrucciones

**Paso 5.1 — Construye el reporte consolidado usando UNION ALL para agregar todas las dimensiones de calidad analizadas:**

```sql
-- ============================================================
-- REPORTE DE CALIDAD DE DATOS - TABLAS DIRTY
-- Consolida:
-- 1) Duplicados por ID_CLIENTE.
-- 2) Duplicados por ID_PEDIDO.
-- 3) Nulos críticos en EMAIL.
-- 4) Nulos críticos en ID_CLIENTE de pedidos.
-- 5) Referencias a clientes inexistentes.
-- ============================================================

SELECT
    'CLIENTES_DIRTY'                                    AS TABLA,
    'Duplicados por ID_CLIENTE'                         AS DIMENSION_CALIDAD,
    COUNT(DISTINCT ID_CLIENTE)                          AS CLAVES_AFECTADAS,
    SUM(CNT - 1)                                        AS REGISTROS_SOBRANTES
FROM (
    SELECT ID_CLIENTE, COUNT(*) AS CNT
    FROM CLIENTES_DIRTY
    GROUP BY ID_CLIENTE
    HAVING COUNT(*) > 1
) DUP_CLIENTES

UNION ALL

SELECT
    'PEDIDOS_DIRTY'                                     AS TABLA,
    'Duplicados por ID_PEDIDO'                          AS DIMENSION_CALIDAD,
    COUNT(DISTINCT ID_PEDIDO)                           AS CLAVES_AFECTADAS,
    SUM(CNT - 1)                                        AS REGISTROS_SOBRANTES
FROM (
    SELECT ID_PEDIDO, COUNT(*) AS CNT
    FROM PEDIDOS_DIRTY
    GROUP BY ID_PEDIDO
    HAVING COUNT(*) > 1
) DUP_PEDIDOS

UNION ALL

SELECT
    'CLIENTES_DIRTY'                                    AS TABLA,
    'Nulos en columna EMAIL'                            AS DIMENSION_CALIDAD,
    COUNT(*) - COUNT(EMAIL)                             AS CLAVES_AFECTADAS,
    COUNT(*) - COUNT(EMAIL)                             AS REGISTROS_SOBRANTES
FROM CLIENTES_DIRTY

UNION ALL

SELECT
    'PEDIDOS_DIRTY'                                     AS TABLA,
    'Nulos en columna ID_CLIENTE'                       AS DIMENSION_CALIDAD,
    COUNT(*) - COUNT(ID_CLIENTE)                        AS CLAVES_AFECTADAS,
    COUNT(*) - COUNT(ID_CLIENTE)                        AS REGISTROS_SOBRANTES
FROM PEDIDOS_DIRTY

UNION ALL

SELECT
    'PEDIDOS_DIRTY'                                     AS TABLA,
    'Referencia a ID_CLIENTE inexistente'               AS DIMENSION_CALIDAD,
    COUNT(*)                                            AS CLAVES_AFECTADAS,
    COUNT(*)                                            AS REGISTROS_SOBRANTES
FROM PEDIDOS_DIRTY P
LEFT JOIN CLIENTES_DIRTY C
    ON P.ID_CLIENTE = C.ID_CLIENTE
WHERE C.ID_CLIENTE IS NULL
  AND P.ID_CLIENTE IS NOT NULL

ORDER BY TABLA, DIMENSION_CALIDAD;
```

**Paso 5.2 — Agrega una columna de severidad y porcentaje de impacto al reporte:**

```sql
-- Reporte de calidad con severidad y porcentaje de impacto.
WITH TOTALES AS (
    SELECT
        (SELECT COUNT(*) FROM CLIENTES_DIRTY) AS TOTAL_CLIENTES,
        (SELECT COUNT(*) FROM PEDIDOS_DIRTY)  AS TOTAL_PEDIDOS
),

HALLAZGOS AS (
    SELECT
        'CLIENTES_DIRTY' AS TABLA,
        'Duplicados por ID_CLIENTE' AS PROBLEMA,
        COALESCE(SUM(CNT - 1), 0) AS REGISTROS_AFECTADOS
    FROM (
        SELECT ID_CLIENTE, COUNT(*) AS CNT
        FROM CLIENTES_DIRTY
        GROUP BY ID_CLIENTE
        HAVING COUNT(*) > 1
    ) D

    UNION ALL

    SELECT
        'PEDIDOS_DIRTY',
        'Duplicados por ID_PEDIDO',
        COALESCE(SUM(CNT - 1), 0)
    FROM (
        SELECT ID_PEDIDO, COUNT(*) AS CNT
        FROM PEDIDOS_DIRTY
        GROUP BY ID_PEDIDO
        HAVING COUNT(*) > 1
    ) D

    UNION ALL

    SELECT
        'CLIENTES_DIRTY',
        'Nulos en EMAIL',
        COUNT(*) - COUNT(EMAIL)
    FROM CLIENTES_DIRTY

    UNION ALL

    SELECT
        'PEDIDOS_DIRTY',
        'Nulos en ID_CLIENTE',
        COUNT(*) - COUNT(ID_CLIENTE)
    FROM PEDIDOS_DIRTY

    UNION ALL

    SELECT
        'PEDIDOS_DIRTY',
        'Referencia a ID_CLIENTE inexistente',
        COUNT(*)
    FROM PEDIDOS_DIRTY P
    LEFT JOIN CLIENTES_DIRTY C
        ON P.ID_CLIENTE = C.ID_CLIENTE
    WHERE C.ID_CLIENTE IS NULL
      AND P.ID_CLIENTE IS NOT NULL
)

SELECT
    H.TABLA,
    H.PROBLEMA,
    H.REGISTROS_AFECTADOS,
    CASE H.TABLA
        WHEN 'CLIENTES_DIRTY' THEN T.TOTAL_CLIENTES
        ELSE T.TOTAL_PEDIDOS
    END AS TOTAL_TABLA,
    ROUND(
        H.REGISTROS_AFECTADOS * 100.0 /
        CASE H.TABLA
            WHEN 'CLIENTES_DIRTY' THEN T.TOTAL_CLIENTES
            ELSE T.TOTAL_PEDIDOS
        END, 2
    ) AS PCT_IMPACTO,
    CASE
        WHEN H.REGISTROS_AFECTADOS * 100.0 /
             CASE H.TABLA
                 WHEN 'CLIENTES_DIRTY' THEN T.TOTAL_CLIENTES
                 ELSE T.TOTAL_PEDIDOS
             END > 10
            THEN 'CRÍTICO'
        WHEN H.REGISTROS_AFECTADOS * 100.0 /
             CASE H.TABLA
                 WHEN 'CLIENTES_DIRTY' THEN T.TOTAL_CLIENTES
                 ELSE T.TOTAL_PEDIDOS
             END > 3
            THEN 'MODERADO'
        ELSE 'BAJO'
    END AS SEVERIDAD
FROM HALLAZGOS H
CROSS JOIN TOTALES T
ORDER BY PCT_IMPACTO DESC;
```

#### Resultado esperado

El reporte final debe mostrar una tabla con las siguientes columnas:

| TABLA | PROBLEMA | REGISTROS_AFECTADOS | TOTAL_TABLA | PCT_IMPACTO | SEVERIDAD |
|---|---|---:|---:|---:|---|
| `PEDIDOS_DIRTY` | Duplicados por `ID_PEDIDO` | N | N | N% | CRÍTICO/MODERADO/BAJO |
| `CLIENTES_DIRTY` | Duplicados por `ID_CLIENTE` | N | N | N% | CRÍTICO/MODERADO/BAJO |
| `PEDIDOS_DIRTY` | Nulos en `ID_CLIENTE` | N | N | N% | CRÍTICO/MODERADO/BAJO |

Lo importante es que todos los problemas conocidos aparecen en el reporte y los porcentajes son coherentes con los conteos individuales de pasos anteriores.

#### Verificación

```sql
-- Verificar que el reporte muestra exactamente 5 dimensiones de calidad.
WITH HALLAZGOS AS (
    SELECT 'CLIENTES_DIRTY' AS TABLA, 'Duplicados por ID_CLIENTE' AS PROBLEMA
    UNION ALL SELECT 'PEDIDOS_DIRTY', 'Duplicados por ID_PEDIDO'
    UNION ALL SELECT 'CLIENTES_DIRTY', 'Nulos en EMAIL'
    UNION ALL SELECT 'PEDIDOS_DIRTY', 'Nulos en ID_CLIENTE'
    UNION ALL SELECT 'PEDIDOS_DIRTY', 'Referencia a ID_CLIENTE inexistente'
)
SELECT COUNT(*) AS TOTAL_DIMENSIONES
FROM HALLAZGOS;
-- Resultado esperado: 5
```

---

## Validación y Pruebas Finales

### Validación 1 — Equivalencia entre subconsulta y `QUALIFY`

```sql
-- Ambas consultas deben devolver el mismo número de filas.

SELECT COUNT(*) AS METODO_SUBCONSULTA
FROM (
    SELECT
        ID_CLIENTE,
        ROW_NUMBER() OVER (
            PARTITION BY ID_CLIENTE
            ORDER BY FECHA_REGISTRO ASC, FECHA_CARGA ASC
        ) AS RN
    FROM CLIENTES_DIRTY
) T
WHERE RN > 1;

SELECT COUNT(*) AS METODO_QUALIFY
FROM (
    SELECT *
    FROM CLIENTES_DIRTY
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY ID_CLIENTE
        ORDER BY FECHA_REGISTRO ASC, FECHA_CARGA ASC
    ) > 1
);
```

**Resultado esperado:** ambos conteos deben ser idénticos.

---

### Validación 2 — Verificar que la deduplicación no pierde clientes únicos

```sql
-- Clientes que no tienen duplicados por ID_CLIENTE.
SELECT COUNT(*) AS CLIENTES_SIN_DUPLICADOS_ORIGINAL
FROM CLIENTES_DIRTY
WHERE ID_CLIENTE NOT IN (
    SELECT ID_CLIENTE
    FROM CLIENTES_DIRTY
    GROUP BY ID_CLIENTE
    HAVING COUNT(*) > 1
);

-- Total de clientes después de deduplicar por ID_CLIENTE.
SELECT COUNT(*) AS CLIENTES_DEDUPLICADOS
FROM (
    SELECT ID_CLIENTE
    FROM CLIENTES_DIRTY
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY ID_CLIENTE
        ORDER BY FECHA_REGISTRO ASC, FECHA_CARGA ASC
    ) = 1
);
```

**Resultado esperado:** el segundo conteo debe ser mayor o igual que el primero, porque conserva todos los clientes únicos y uno por cada grupo duplicado.

---

### Validación 3 — Validar duplicados contra el reporte final

```sql
SELECT
    'Duplicados CLIENTES por ID_CLIENTE' AS ORIGEN,
    SUM(VECES_REGISTRADO - 1)            AS TOTAL
FROM (
    SELECT ID_CLIENTE, COUNT(*) AS VECES_REGISTRADO
    FROM CLIENTES_DIRTY
    GROUP BY ID_CLIENTE
    HAVING COUNT(*) > 1
)

UNION ALL

SELECT
    'Duplicados PEDIDOS por ID_PEDIDO' AS ORIGEN,
    SUM(VECES_REGISTRADO - 1)          AS TOTAL
FROM (
    SELECT ID_PEDIDO, COUNT(*) AS VECES_REGISTRADO
    FROM PEDIDOS_DIRTY
    GROUP BY ID_PEDIDO
    HAVING COUNT(*) > 1
);
```

**Resultado esperado:** los valores deben coincidir exactamente con los obtenidos en el Ejercicio 2 y con el reporte del Ejercicio 5.

---

## Resultados esperados clave con el dataset cargado

| Consulta / Validación | Resultado esperado |
|---|---:|
| Conteo de `CLIENTES` | 12 filas |
| Conteo de `PEDIDOS` | 22 filas |
| Conteo de `CLIENTES_DIRTY` | 19 filas |
| Conteo de `PEDIDOS_DIRTY` | 30 filas |
| IDs de cliente duplicados | 3 claves afectadas |
| Registros sobrantes por `ID_CLIENTE` | 3 registros |
| Emails duplicados | 2 emails afectados |
| Emails nulos | 2 registros |
| IDs de pedido duplicados | 3 claves afectadas |
| Registros sobrantes por `ID_PEDIDO` | 3 registros |
| Pedidos con `ID_CLIENTE` nulo | 1 registro |
| Pedidos con cliente inexistente | 2 registros |
| Clientes sin pedidos | Al menos 2 registros |
| Dimensiones del reporte final | 5 filas |

---

## Solución de Problemas

### Problema 1 — Error: "Object does not exist" al consultar `CLIENTES_DIRTY` o `PEDIDOS_DIRTY`

**Síntoma:** Al ejecutar cualquier consulta del laboratorio, Snowflake devuelve un error similar a:

```text
SQL compilation error: Object 'LAB_SQL_INTERMEDIO.VENTAS.CLIENTES_DIRTY' does not exist or not authorized.
```

**Causa:** El script de setup no fue ejecutado antes del laboratorio, fue ejecutado en un database/schema diferente al esperado, o el rol activo no tiene permisos sobre las tablas del schema `VENTAS`.

**Solución:**

1. Verifica el contexto activo:

```sql
SELECT CURRENT_DATABASE(), CURRENT_SCHEMA(), CURRENT_ROLE();
```

2. Si el database o schema no es correcto, ejecuta:

```sql
USE DATABASE LAB_SQL_INTERMEDIO;
USE SCHEMA VENTAS;
```

3. Verifica que las tablas existen:

```sql
SHOW TABLES IN SCHEMA LAB_SQL_INTERMEDIO.VENTAS;
```

4. Si `CLIENTES_DIRTY` o `PEDIDOS_DIRTY` no aparecen, regresa al archivo **`03_SETUP_DATOS_DUPLICADOS_DIRTY`** y ejecuta el script completo.

---

### Problema 2 — `QUALIFY` no es reconocida y genera error de sintaxis

**Síntoma:** Al ejecutar una consulta con `QUALIFY`, Snowflake devuelve un error como:

```text
SQL compilation error: syntax error line X at position Y unexpected 'QUALIFY'.
```

**Causa:** Puede ocurrir por un error de escritura, por ejecutar la consulta en otro motor SQL o por colocar `QUALIFY` en una posición incorrecta dentro de la consulta.

**Solución:**

1. Verifica que estás ejecutando en Snowflake:

```sql
SELECT CURRENT_ACCOUNT(), CURRENT_REGION();
```

2. Verifica la ortografía: `QUALIFY`.
3. Verifica que `QUALIFY` esté después de `FROM` / `WHERE` / `GROUP BY` si existen, y antes del `ORDER BY`.
4. Si necesitas una alternativa más portable, usa la versión con subconsulta:

```sql
SELECT *
FROM (
    SELECT
        columnas,
        ROW_NUMBER() OVER (...) AS RN
    FROM tabla
) T
WHERE RN = 1;
```

---

### Problema 3 — El conteo de duplicados por email incluye valores nulos

**Síntoma:** Al ejecutar la detección de duplicados por `EMAIL`, aparece un grupo `NULL` como si fuera un email duplicado.

**Causa:** `NULL` no representa un valor real. Si hay varios registros sin email, agrupar por `EMAIL` puede generar un grupo `NULL`, pero eso no significa que todos sean el mismo cliente.

**Solución:**

Filtra los nulos antes de agrupar:

```sql
SELECT
    EMAIL,
    COUNT(*) AS VECES_REGISTRADO
FROM CLIENTES_DIRTY
WHERE EMAIL IS NOT NULL
GROUP BY EMAIL
HAVING COUNT(*) > 1;
```

---

## Limpieza del entorno

Al finalizar el laboratorio, ejecuta los siguientes comandos para liberar recursos y evitar consumo innecesario de créditos Snowflake:

```sql
-- Suspender el warehouse para detener el consumo de créditos.
-- IMPORTANTE: ejecutar siempre al terminar la sesión.
ALTER WAREHOUSE COMPUTE_WH SUSPEND;
```

> ⚠️ **Recordatorio de créditos:** Las cuentas trial de Snowflake tienen 400 USD de créditos. Un warehouse `X-SMALL` consume aproximadamente 1 crédito por hora de actividad. Suspenderlo al terminar cada sesión es una práctica obligatoria en este curso.

No es necesario eliminar tablas ni datos, ya que el schema `LAB_SQL_INTERMEDIO.VENTAS` es compartido por todos los laboratorios del curso y sus datos deben persistir para las sesiones siguientes.

```sql
-- Confirmación: laboratorio completado.
SELECT 'Laboratorio completado. Warehouse suspendido al finalizar.' AS ESTADO;
```

---

## Resumen

En este laboratorio aplicaste tres técnicas fundamentales de auditoría y calidad de datos en Snowflake:

| Técnica | Cuándo usarla | Limitación |
|---|---|---|
| `GROUP BY` + `HAVING COUNT(*) > 1` | Detectar qué claves tienen duplicados y cuántos | No indica cuál registro conservar |
| `ROW_NUMBER() OVER(PARTITION BY ...)` | Numerar duplicados para seleccionar el registro canónico | Requiere definir un criterio de ordenación |
| `QUALIFY` | Filtrar resultados de window functions sin subconsulta | Puede no ser portable a todos los motores SQL |
| `LEFT JOIN` + `WHERE IS NULL` | Detectar registros huérfanos o referencias rotas | Requiere identificar correctamente la tabla padre |

### Conceptos clave para recordar

1. **`HAVING` vs. `WHERE`:** `WHERE` filtra filas individuales antes del agrupamiento; `HAVING` filtra grupos después de aplicar funciones de agregación. Para trabajar con `COUNT()`, necesitas `HAVING`.

2. **`COUNT(*)` vs. `COUNT(columna)`:** `COUNT(*)` cuenta todas las filas incluyendo nulos; `COUNT(columna)` solo cuenta filas donde esa columna no es nula. La diferencia entre ambos revela los nulos.

3. **`ROW_NUMBER()` como puente:** Esta función de ventana es la introducción formal a las window functions. En el Laboratorio 4 explorarás `RANK()`, `DENSE_RANK()`, `LAG()` y `LEAD()` con mayor profundidad.

4. **`QUALIFY` mejora la legibilidad:** Permite filtrar el resultado de una window function sin crear una subconsulta intermedia.

### Conexión con los próximos laboratorios

```text
Lab 01 — CTEs y subqueries
       ↓
Lab 02 — CASE WHEN y segmentación
       ↓
Lab 03 — Este laboratorio
  ├── GROUP BY + HAVING → técnica base de auditoría
  ├── ROW_NUMBER() → introducción a window functions
  └── LEFT JOIN + IS NULL → integridad referencial
       ↓
Lab 04 — Window functions avanzadas
       ↓
Lab 05 — Series temporales con window functions
```

### Recursos adicionales

| Recurso | URL |
|---|---|
| Documentación Snowflake: GROUP BY | https://docs.snowflake.com/en/sql-reference/constructs/group-by |
| Documentación Snowflake: HAVING | https://docs.snowflake.com/en/sql-reference/constructs/having |
| Documentación Snowflake: ROW_NUMBER | https://docs.snowflake.com/en/sql-reference/functions/row_number |
| Documentación Snowflake: QUALIFY | https://docs.snowflake.com/en/sql-reference/constructs/qualify |
| Documentación Snowflake: Window Functions | https://docs.snowflake.com/en/sql-reference/functions-analytic |

---
