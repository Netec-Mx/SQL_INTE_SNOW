# Análisis de tendencias y métricas por periodo

## Metadatos

| Atributo | Detalle |
|---|---|
| **Duración estimada** | 50 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (Apply) |
| **Módulo** | 5 — Análisis de tendencias y métricas por periodo |
| **Schema de trabajo** | `LAB_SQL_INTERMEDIO.VENTAS` |
| **Plataforma** | Snowflake (Snowsight Worksheet) |

---

## Descripción General

En este laboratorio aplicarás las funciones de fecha de Snowflake aprendidas en la Lección 5.1 para construir análisis temporales completos sobre datos reales de ventas. Comenzarás agrupando ventas por semana, mes y trimestre usando `DATE_TRUNC`, luego construirás comparaciones mes a mes con `LAG()`, calcularás variaciones porcentuales entre períodos y finalmente identificarás los mejores y peores meses por crecimiento. El laboratorio conecta directamente las funciones de fecha con las window functions del Laboratorio 4, consolidando ambas habilidades en un reporte de tendencias cohesivo.

La práctica está diseñada para trabajar con un dataset controlado de ventas históricas de 2022, 2023 y 2024. Esto permite analizar tendencias mensuales, comparaciones año contra año (*Year-over-Year*) y variaciones porcentuales sin depender de datos externos ni de un script previo del instructor. El setup incluido crea las tablas necesarias y carga datos suficientes para completar todos los pasos del laboratorio.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:

- [ ] Aplicar `DATE_TRUNC`, `DATEADD`, `DATEDIFF`, `YEAR`, `MONTH` y `TO_CHAR` para agrupar y presentar datos por períodos temporales en Snowflake.
- [ ] Construir comparaciones de métricas entre períodos equivalentes (mes actual vs. mes anterior, año actual vs. año anterior) usando `LAG()` y CTEs.
- [ ] Calcular tasas de variación porcentual entre períodos consecutivos e identificar tendencias de crecimiento en series temporales de ventas.
- [ ] Generar un reporte de tendencias que clasifique los 3 mejores y 3 peores meses por variación porcentual de ventas.
- [ ] Aplicar escritura defensiva con `NULLIF()` para evitar divisiones por cero en métricas porcentuales.
- [ ] Organizar análisis temporales complejos mediante CTEs encadenadas para mejorar legibilidad y mantenimiento.

---

## Prerrequisitos

### Conocimiento previo

| Requisito | Descripción |
|---|---|
| Lab 04 completado | Manejo de `LAG()`, `LEAD()` y `SUM() OVER()` en series temporales |
| Tipos de dato temporales | Comprensión de `DATE` y `TIMESTAMP` en SQL |
| GROUP BY con fechas | Agrupación temporal con expresiones de fecha |
| CTEs | Uso de `WITH ... AS (...)` para organizar consultas de múltiples pasos |
| Funciones de fecha (Lección 5.1) | `DATE_TRUNC`, `DATEDIFF`, `DATEADD`, `TO_CHAR`, `YEAR`, `MONTH` |

### Acceso y configuración

| Requisito | Detalle |
|---|---|
| Cuenta Snowflake activa | Trial o corporativa con rol que permita crear objetos en un database de práctica |
| Script de setup ejecutado | No se asume script previo. Esta práctica incluye el setup completo de base, schema, tablas y datos. |
| Database disponible | `LAB_SQL_INTERMEDIO` |
| Schema disponible | `LAB_SQL_INTERMEDIO.VENTAS` |
| Tablas requeridas | `CLIENTES`, `PRODUCTOS`, `PEDIDOS`, `VENTAS`, creadas en el Paso 0 |
| Warehouse activo | `COMPUTE_WH` (tamaño `X-SMALL`) |

---

## Entorno de Laboratorio

### Hardware mínimo recomendado

| Componente | Mínimo | Recomendado |
|---|---|---|
| RAM | 8 GB | 16 GB |
| Procesador | Intel Core i5 / AMD Ryzen 5 (64-bit) | i7 / Ryzen 7 o superior |
| Almacenamiento libre | 2 GB | 5 GB |
| Resolución de pantalla | 1280×768 | 1920×1080 |
| Conexión a Internet | 10 Mbps | 25 Mbps o más |

### Software requerido

| Software | Versión | Uso en este lab |
|---|---|---|
| Snowflake (Snowsight) | Enterprise o Trial (versión web actual) | Ejecución de todas las consultas |
| Navegador web | Chrome/Firefox/Edge 110+ | Acceso a Snowsight |
| Visual Studio Code (opcional) | 1.80+ | Edición previa de scripts |
| SnowSQL (opcional) | 1.2.x+ | Alternativa CLI |

---

## Organización recomendada de Workspace en Snowsight

Para que la práctica sea ordenada y reutilizable, trabaja con un Workspace y 2 folders. En esta práctica se usa la palabra **workspace** como una separación lógica de trabajo dentro de Snowsight.

| Workspace / Worksheet | Folder | Nombre sugerido | Uso |
|---|---|---|---|
| `SNOWLABS-INT` | `SETUP-LABS` | `05_SETUP_DATOS_TENDENCIAS_PERIODO` | Crear database, schema, tablas y datos temporales. Se ejecuta una vez al inicio o cuando quieras reiniciar el laboratorio. |
| `SNOWLABS-INT` | `SCRIPT-LABS` | `05_LAB_TENDENCIAS_PERIODO` | Ejecutar los ejercicios del laboratorio sin mezclar el script de carga de datos. |

### Paso 0.0 — Crear el workspace de las prácticas

1. Entra a **Snowsight**.
2. Da clic en la opción **Projects**.
3. Clic en **+**.
4. Luego selecciona la opción **Private workspace**.
5. Nómbralo: **`SNOWLABS-INT`**.
6. Clic en **Create**.

---

### Paso 0.0.1 — Crear el Folder y script que carga los datos

1. Ahora dentro del nuevo workspace da clic en **+ Add new**.
2. Clic en **Folder** y nómbralo: **`SETUP-LABS`**.
3. Dentro del Folder **SETUP-LABS** da clic en el símbolo **+**.
4. Crea un archivo de tipo **SQL**.
5. Nómbralo: **`05_SETUP_DATOS_TENDENCIAS_PERIODO`**.
6. Pega ahí el siguiente script completo.
7. Ejecuta el script completo antes de comenzar el laboratorio.

Este dataset está diseñado para activar todos los escenarios de la práctica:

- Datos de ventas de 2022, 2023 y 2024 para análisis histórico.
- 36 meses consecutivos con ventas para practicar `DATE_TRUNC`, `LAG()` y comparación YoY.
- 108 registros de ventas: 3 ventas por mes.
- Montos con estacionalidad para observar meses de crecimiento y caída.
- Múltiples clientes, productos, regiones y canales para simular un dataset transaccional realista.
- Tabla `PEDIDOS` derivada de `VENTAS` para mantener compatibilidad con el schema de prácticas anteriores.

```sql
-- 05_setup_datos_tendencias_periodo.sql
-- Práctica Snowflake Intermedio
-- Dataset controlado para completar el laboratorio:
-- Análisis de tendencias y métricas por periodo
--
-- Objetivo del dataset:
-- 1) Tener 36 meses consecutivos de datos: 2022, 2023 y 2024.
-- 2) Tener 3 ventas por mes para permitir agregaciones temporales.
-- 3) Tener variaciones claras mes a mes para identificar crecimiento y caída.
-- 4) Tener datos suficientes para comparación Year-over-Year.
-- 5) Evitar dependencia de CURRENT_DATE en los ejercicios de últimos 3 meses.
-- 6) Mantener compatibilidad con prácticas anteriores: CLIENTES, PRODUCTOS, PEDIDOS y VENTAS.

USE WAREHOUSE COMPUTE_WH;

CREATE DATABASE IF NOT EXISTS LAB_SQL_INTERMEDIO;
USE DATABASE LAB_SQL_INTERMEDIO;

CREATE SCHEMA IF NOT EXISTS VENTAS;
USE SCHEMA VENTAS;

-- Opcional para repetir el laboratorio desde cero.
DROP TABLE IF EXISTS PEDIDOS;
DROP TABLE IF EXISTS VENTAS;
DROP TABLE IF EXISTS PRODUCTOS;
DROP TABLE IF EXISTS CLIENTES;

CREATE OR REPLACE TABLE CLIENTES (
    ID_CLIENTE NUMBER(10,0) NOT NULL,
    NOMBRE VARCHAR(100) NOT NULL,
    EMAIL VARCHAR(120),
    FECHA_REGISTRO DATE,
    CIUDAD VARCHAR(80),
    PAIS VARCHAR(80),
    CONSTRAINT PK_CLIENTES PRIMARY KEY (ID_CLIENTE)
);

CREATE OR REPLACE TABLE PRODUCTOS (
    ID_PRODUCTO NUMBER(10,0) NOT NULL,
    NOMBRE_PRODUCTO VARCHAR(120) NOT NULL,
    CATEGORIA VARCHAR(80) NOT NULL,
    PRECIO_UNITARIO NUMBER(12,2),
    ACTIVO BOOLEAN DEFAULT TRUE,
    CONSTRAINT PK_PRODUCTOS PRIMARY KEY (ID_PRODUCTO)
);

CREATE OR REPLACE TABLE VENTAS (
    ID_VENTA NUMBER(10,0) NOT NULL,
    ID_CLIENTE NUMBER(10,0) NOT NULL,
    ID_PRODUCTO NUMBER(10,0) NOT NULL,
    ID_VENDEDOR NUMBER(10,0) NOT NULL,
    REGION VARCHAR(40) NOT NULL,
    FECHA_VENTA DATE NOT NULL,
    MONTO_TOTAL NUMBER(12,2) NOT NULL,
    CANAL VARCHAR(40),
    CONSTRAINT PK_VENTAS PRIMARY KEY (ID_VENTA)
);

CREATE OR REPLACE TABLE PEDIDOS (
    ID_PEDIDO NUMBER(10,0) NOT NULL,
    ID_CLIENTE NUMBER(10,0) NOT NULL,
    ID_PRODUCTO NUMBER(10,0) NOT NULL,
    FECHA_PEDIDO DATE NOT NULL,
    MONTO_TOTAL NUMBER(12,2) NOT NULL,
    ESTADO_PEDIDO VARCHAR(30),
    CONSTRAINT PK_PEDIDOS PRIMARY KEY (ID_PEDIDO)
);

INSERT INTO CLIENTES (ID_CLIENTE, NOMBRE, EMAIL, FECHA_REGISTRO, CIUDAD, PAIS) VALUES
    (1,  'Ana Torres',       'ana.torres@example.com',       '2021-11-15', 'CDMX',        'México'),
    (2,  'Luis Martínez',    'luis.martinez@example.com',    '2021-12-03', 'CDMX',        'México'),
    (3,  'María López',      'maria.lopez@example.com',      '2022-01-20', 'Guadalajara', 'México'),
    (4,  'Carlos Hernández', 'carlos.hernandez@example.com', '2022-02-12', 'Guadalajara', 'México'),
    (5,  'Sofía Ramírez',    'sofia.ramirez@example.com',    '2022-03-18', 'Monterrey',   'México'),
    (6,  'Jorge Castillo',   'jorge.castillo@example.com',   '2022-04-09', 'Monterrey',   'México'),
    (7,  'Elena Flores',     'elena.flores@example.com',     '2022-05-24', 'Puebla',      'México'),
    (8,  'Diego Sánchez',    'diego.sanchez@example.com',    '2022-06-11', 'Puebla',      'México'),
    (9,  'Valeria Cruz',     'valeria.cruz@example.com',     '2022-07-05', 'Mérida',      'México'),
    (10, 'Roberto Díaz',     'roberto.diaz@example.com',     '2022-08-01', 'Mérida',      'México'),
    (11, 'Fernanda Ruiz',    'fernanda.ruiz@example.com',    '2022-09-14', 'Querétaro',   'México'),
    (12, 'Miguel Navarro',   'miguel.navarro@example.com',   '2022-10-27', 'Querétaro',   'México');

INSERT INTO PRODUCTOS (ID_PRODUCTO, NOMBRE_PRODUCTO, CATEGORIA, PRECIO_UNITARIO, ACTIVO) VALUES
    (1, 'Laptop Pro 14',        'Electrónica',     1200.00, TRUE),
    (2, 'Monitor 27 pulgadas',  'Electrónica',      350.00, TRUE),
    (3, 'Teclado mecánico',     'Accesorios',       120.00, TRUE),
    (4, 'Mouse inalámbrico',    'Accesorios',        80.00, TRUE),
    (5, 'Silla ergonómica',     'Oficina',          420.00, TRUE),
    (6, 'Escritorio ajustable', 'Oficina',          680.00, TRUE),
    (7, 'Licencia BI anual',    'Software',         900.00, TRUE),
    (8, 'Servidor compacto',    'Infraestructura', 1500.00, TRUE);

-- Carga de 36 meses consecutivos.
-- Se generan 3 ventas por mes a partir de un total mensual controlado.
-- Los totales mensuales tienen estacionalidad:
-- Q1 moderado, Q2 estable, Q3 variable, Q4 fuerte.
-- Esto permite ver crecimiento, caída, comparación YoY y top/bottom meses.
INSERT INTO VENTAS (
    ID_VENTA,
    ID_CLIENTE,
    ID_PRODUCTO,
    ID_VENDEDOR,
    REGION,
    FECHA_VENTA,
    MONTO_TOTAL,
    CANAL
)
WITH meses AS (
    SELECT * FROM VALUES
        (0,  '2022-01-01'::DATE, 12000),
        (1,  '2022-02-01'::DATE, 10500),
        (2,  '2022-03-01'::DATE, 13200),
        (3,  '2022-04-01'::DATE, 15000),
        (4,  '2022-05-01'::DATE, 14800),
        (5,  '2022-06-01'::DATE, 16200),
        (6,  '2022-07-01'::DATE, 13000),
        (7,  '2022-08-01'::DATE, 11200),
        (8,  '2022-09-01'::DATE, 14000),
        (9,  '2022-10-01'::DATE, 17500),
        (10, '2022-11-01'::DATE, 19000),
        (11, '2022-12-01'::DATE, 26000),

        (12, '2023-01-01'::DATE, 13800),
        (13, '2023-02-01'::DATE, 12100),
        (14, '2023-03-01'::DATE, 15800),
        (15, '2023-04-01'::DATE, 17100),
        (16, '2023-05-01'::DATE, 16900),
        (17, '2023-06-01'::DATE, 18400),
        (18, '2023-07-01'::DATE, 14900),
        (19, '2023-08-01'::DATE, 12800),
        (20, '2023-09-01'::DATE, 16200),
        (21, '2023-10-01'::DATE, 20500),
        (22, '2023-11-01'::DATE, 22500),
        (23, '2023-12-01'::DATE, 31000),

        (24, '2024-01-01'::DATE, 15400),
        (25, '2024-02-01'::DATE, 13900),
        (26, '2024-03-01'::DATE, 18100),
        (27, '2024-04-01'::DATE, 19800),
        (28, '2024-05-01'::DATE, 19200),
        (29, '2024-06-01'::DATE, 21500),
        (30, '2024-07-01'::DATE, 17000),
        (31, '2024-08-01'::DATE, 14500),
        (32, '2024-09-01'::DATE, 18800),
        (33, '2024-10-01'::DATE, 23800),
        (34, '2024-11-01'::DATE, 26700),
        (35, '2024-12-01'::DATE, 36500)
    AS m(MES_IDX, FECHA_BASE, TOTAL_MENSUAL)
),
detalle AS (
    SELECT * FROM VALUES
        (1, 0.45,  4, 'Web'),
        (2, 0.35, 17, 'Ejecutivo'),
        (3, 0.20, 26, 'Marketplace')
    AS d(LINEA, FACTOR, DIA_OFFSET, CANAL)
)
SELECT
    (m.MES_IDX * 10) + d.LINEA + 5000                                  AS ID_VENTA,
    MOD(m.MES_IDX + d.LINEA, 12) + 1                                    AS ID_CLIENTE,
    MOD(m.MES_IDX + d.LINEA, 8) + 1                                     AS ID_PRODUCTO,
    MOD(m.MES_IDX + d.LINEA, 6) + 101                                   AS ID_VENDEDOR,
    CASE MOD(m.MES_IDX + d.LINEA, 4)
        WHEN 0 THEN 'Norte'
        WHEN 1 THEN 'Centro'
        WHEN 2 THEN 'Occidente'
        ELSE 'Sur'
    END                                                                 AS REGION,
    DATEADD('day', d.DIA_OFFSET, m.FECHA_BASE)                          AS FECHA_VENTA,
    ROUND(m.TOTAL_MENSUAL * d.FACTOR, 2)                                AS MONTO_TOTAL,
    d.CANAL                                                             AS CANAL
FROM meses m
CROSS JOIN detalle d
ORDER BY ID_VENTA;

-- Tabla PEDIDOS derivada de VENTAS.
-- Se crea para mantener compatibilidad con el schema general de prácticas.
INSERT INTO PEDIDOS (
    ID_PEDIDO,
    ID_CLIENTE,
    ID_PRODUCTO,
    FECHA_PEDIDO,
    MONTO_TOTAL,
    ESTADO_PEDIDO
)
SELECT
    ID_VENTA + 10000     AS ID_PEDIDO,
    ID_CLIENTE,
    ID_PRODUCTO,
    FECHA_VENTA          AS FECHA_PEDIDO,
    MONTO_TOTAL,
    CASE
        WHEN MOD(ID_VENTA, 17) = 0 THEN 'EN_PROCESO'
        ELSE 'COMPLETADO'
    END                  AS ESTADO_PEDIDO
FROM VENTAS;

-- Validación rápida del dataset.
SELECT 'CLIENTES' AS TABLA, COUNT(*) AS FILAS FROM CLIENTES
UNION ALL
SELECT 'PRODUCTOS' AS TABLA, COUNT(*) AS FILAS FROM PRODUCTOS
UNION ALL
SELECT 'VENTAS' AS TABLA, COUNT(*) AS FILAS FROM VENTAS
UNION ALL
SELECT 'PEDIDOS' AS TABLA, COUNT(*) AS FILAS FROM PEDIDOS
ORDER BY TABLA;

-- Resultado esperado:
-- CLIENTES  = 12
-- PEDIDOS   = 108
-- PRODUCTOS = 8
-- VENTAS    = 108

-- Validación temporal del dataset.
SELECT
    MIN(FECHA_VENTA) AS FECHA_MAS_ANTIGUA,
    MAX(FECHA_VENTA) AS FECHA_MAS_RECIENTE,
    COUNT(DISTINCT YEAR(FECHA_VENTA)) AS ANIOS_CON_DATOS,
    COUNT(DISTINCT DATE_TRUNC('month', FECHA_VENTA)) AS MESES_CON_DATOS,
    COUNT(*) AS TOTAL_REGISTROS
FROM VENTAS;

-- Resultado esperado:
-- ANIOS_CON_DATOS = 3
-- MESES_CON_DATOS = 36
-- TOTAL_REGISTROS = 108

-- Validación por año.
SELECT
    YEAR(FECHA_VENTA) AS ANIO,
    COUNT(*) AS REGISTROS,
    ROUND(SUM(MONTO_TOTAL), 2) AS VENTAS_TOTALES
FROM VENTAS
GROUP BY YEAR(FECHA_VENTA)
ORDER BY ANIO;

-- Resultado esperado:
-- 2022, 2023 y 2024 con 36 registros cada uno.
```

---

### Paso 0.0.2 — Crear el folder y script de laboratorio

1. Da clic en el botón **+ Add new**.
2. Clic en **Folder** y nómbralo: **`SCRIPT-LABS`**.
3. Dentro de **SCRIPT-LABS** crea un archivo de tipo **SQL**.
4. Nómbralo: **`05_LAB_TENDENCIAS_PERIODO`**.
5. Usa este archivo para ejecutar los ejercicios del laboratorio.
6. **No pegues aquí el script de carga completo; solo usa las consultas de análisis del laboratorio.**

---

### Paso 0.1 — Confirmar el contexto de trabajo

Dentro del archivo **`05_LAB_TENDENCIAS_PERIODO`**, ejecuta lo siguiente:

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

> ⚠️ **Recordatorio de créditos:** Este laboratorio usa un warehouse `X-SMALL`. Al terminar la sesión, ejecuta `ALTER WAREHOUSE COMPUTE_WH SUSPEND;` para evitar consumo innecesario de créditos.

---

### Paso 0.2 — Confirmar que las tablas quedaron disponibles

Ejecuta:

```sql
SHOW TABLES;
```

**Resultado esperado:** deben aparecer al menos estas tablas:

| Tabla | Uso en la práctica |
|---|---|
| `CLIENTES` | Datos maestros de clientes para conteos únicos y continuidad del schema. |
| `PRODUCTOS` | Catálogo base para mantener compatibilidad con prácticas anteriores. |
| `PEDIDOS` | Tabla derivada de ventas para continuidad del entorno. |
| `VENTAS` | Tabla principal del laboratorio de análisis temporal. |

---

### Paso 0.3 — Validar volumen mínimo de datos

Ejecuta:

```sql
SELECT 'CLIENTES' AS TABLA, COUNT(*) AS FILAS FROM CLIENTES
UNION ALL
SELECT 'PRODUCTOS' AS TABLA, COUNT(*) AS FILAS FROM PRODUCTOS
UNION ALL
SELECT 'VENTAS' AS TABLA, COUNT(*) AS FILAS FROM VENTAS
UNION ALL
SELECT 'PEDIDOS' AS TABLA, COUNT(*) AS FILAS FROM PEDIDOS
ORDER BY TABLA;
```

**Resultado esperado:**

| TABLA | FILAS |
|---|---:|
| CLIENTES | 12 |
| PEDIDOS | 108 |
| PRODUCTOS | 8 |
| VENTAS | 108 |

---

### Paso 0.4 — Validar rango temporal del dataset

Ejecuta:

```sql
SELECT
    MIN(FECHA_VENTA)                                      AS fecha_mas_antigua,
    MAX(FECHA_VENTA)                                      AS fecha_mas_reciente,
    DATEDIFF('month', MIN(FECHA_VENTA), MAX(FECHA_VENTA)) AS meses_de_historia,
    DATEDIFF('day',  MIN(FECHA_VENTA), MAX(FECHA_VENTA))  AS dias_de_historia,
    COUNT(*)                                              AS total_registros,
    COUNT(DISTINCT DATE_TRUNC('month', FECHA_VENTA))      AS meses_con_datos
FROM VENTAS;
```

**Resultado esperado:**

| FECHA_MAS_ANTIGUA | FECHA_MAS_RECIENTE | MESES_DE_HISTORIA | MESES_CON_DATOS | TOTAL_REGISTROS |
|---|---|---:|---:|---:|
| 2022-01-05 | 2024-12-27 | 35 | 36 | 108 |

> Nota: `DATEDIFF('month', MIN(...), MAX(...))` devuelve la diferencia entre los límites de fecha, por eso puede mostrar `35`, aunque existan `36` meses con datos. La métrica más importante para este laboratorio es `MESES_CON_DATOS`.

---

### Paso 0.5 — Validar datos por año y mes

Ejecuta:

```sql
SELECT
    YEAR(FECHA_VENTA) AS ANIO,
    COUNT(*) AS TOTAL_REGISTROS,
    COUNT(DISTINCT DATE_TRUNC('month', FECHA_VENTA)) AS MESES_CON_DATOS,
    ROUND(SUM(MONTO_TOTAL), 2) AS VENTAS_TOTALES
FROM VENTAS
GROUP BY YEAR(FECHA_VENTA)
ORDER BY ANIO;
```

**Resultado esperado:** deben aparecer los años `2022`, `2023` y `2024`, cada uno con `36` registros y `12` meses con datos.

---

## Ejercicios Paso a Paso

---

### Ejercicio 1 — Exploración de la estructura temporal de los datos

**Objetivo:** Familiarizarte con el rango de fechas disponible en la tabla `VENTAS` y validar que existen al menos 24 meses de datos históricos para el análisis de tendencias.

#### Instrucciones

**Paso 1.1 — Verificar estructura de la tabla**

Abre una nueva worksheet en Snowsight y asegúrate de que el contexto esté configurado con `LAB_SQL_INTERMEDIO.VENTAS`.

```sql
-- ============================================================
-- PASO 1: Exploración del rango temporal de VENTAS
-- ============================================================

-- 1a. Verificar estructura de la tabla
DESCRIBE TABLE VENTAS;
```

**Paso 1.2 — Ejecuta la consulta de rango temporal:**

```sql
-- 1b. Rango de fechas y volumen de datos disponibles
SELECT
    MIN(FECHA_VENTA)                                      AS fecha_mas_antigua,
    MAX(FECHA_VENTA)                                      AS fecha_mas_reciente,
    DATEDIFF('month', MIN(FECHA_VENTA), MAX(FECHA_VENTA)) AS meses_de_historia,
    DATEDIFF('day',  MIN(FECHA_VENTA), MAX(FECHA_VENTA))  AS dias_de_historia,
    COUNT(*)                                              AS total_registros,
    COUNT(DISTINCT DATE_TRUNC('month', FECHA_VENTA))      AS meses_con_datos
FROM VENTAS;
```

**Paso 1.3 — Ejecuta una exploración rápida de los primeros registros para entender las columnas disponibles:**

```sql
-- 1c. Muestra de registros con componentes de fecha extraídos
SELECT
    ID_VENTA,
    FECHA_VENTA,
    YEAR(FECHA_VENTA)             AS anio,
    MONTH(FECHA_VENTA)            AS mes_numero,
    QUARTER(FECHA_VENTA)          AS trimestre,
    DAYOFWEEK(FECHA_VENTA)        AS dia_semana,
    MONTO_TOTAL
FROM VENTAS
ORDER BY FECHA_VENTA DESC
LIMIT 10;
```

#### Resultado esperado

La consulta `1b` debe mostrar una fila con valores similares a:

| fecha_mas_antigua | fecha_mas_reciente | meses_de_historia | dias_de_historia | total_registros | meses_con_datos |
|---|---|---:|---:|---:|---:|
| 2022-01-05 | 2024-12-27 | 35 | ~1087 | 108 | 36 |

> Si `meses_con_datos` es menor a 24, detén el laboratorio y revisa si ejecutaste correctamente el setup del Paso 0.

#### Verificación

```sql
-- Verificación Paso 1: confirmar que hay datos en múltiples años
SELECT
    YEAR(FECHA_VENTA) AS anio,
    COUNT(*)          AS registros_por_anio
FROM VENTAS
GROUP BY YEAR(FECHA_VENTA)
ORDER BY anio;
```

Debes ver `2022`, `2023` y `2024`, cada uno con registros. Esto confirma que el dataset está completo para todos los ejercicios de este laboratorio.

---

### Ejercicio 2 — Agrupación temporal con DATE_TRUNC

**Objetivo:** Practicar el uso de `DATE_TRUNC` para agregar métricas de ventas a diferentes niveles de granularidad temporal: semanal, mensual y trimestral.

#### Instrucciones

**Paso 2.1 — Ventas totales agrupadas por mes**

Construye primero la agrupación **mensual**, que será la base de los análisis siguientes:

```sql
-- ============================================================
-- PASO 2A: Ventas totales agrupadas por MES
-- ============================================================
SELECT
    DATE_TRUNC('month', FECHA_VENTA)          AS periodo_mes,
    TO_CHAR(DATE_TRUNC('month', FECHA_VENTA),
            'YYYY-MM')                        AS etiqueta_mes,
    COUNT(*)                                  AS cantidad_ventas,
    COUNT(DISTINCT ID_CLIENTE)                AS clientes_unicos,
    ROUND(SUM(MONTO_TOTAL), 2)                AS ventas_totales,
    ROUND(AVG(MONTO_TOTAL), 2)                AS ticket_promedio
FROM VENTAS
GROUP BY DATE_TRUNC('month', FECHA_VENTA)
ORDER BY periodo_mes;
```

**Paso 2.2 — Ventas totales agrupadas por trimestre**

Construye la agrupación **trimestral** para identificar estacionalidad a nivel macro:

```sql
-- ============================================================
-- PASO 2B: Ventas totales agrupadas por TRIMESTRE
-- ============================================================
SELECT
    DATE_TRUNC('quarter', FECHA_VENTA)           AS periodo_trimestre,
    YEAR(FECHA_VENTA)                            AS anio,
    QUARTER(FECHA_VENTA)                         AS numero_trimestre,
    TO_CHAR(DATE_TRUNC('quarter', FECHA_VENTA),
            'YYYY-"Q"Q')                         AS etiqueta_trimestre,
    COUNT(*)                                     AS cantidad_ventas,
    ROUND(SUM(MONTO_TOTAL), 2)                   AS ventas_totales,
    ROUND(AVG(MONTO_TOTAL), 2)                   AS ticket_promedio
FROM VENTAS
GROUP BY
    DATE_TRUNC('quarter', FECHA_VENTA),
    YEAR(FECHA_VENTA),
    QUARTER(FECHA_VENTA)
ORDER BY periodo_trimestre;
```

**Paso 2.3 — Ventas por semana en los últimos 3 meses del dataset**

Construye la agrupación **semanal** para detectar patrones de corto plazo. En lugar de usar `CURRENT_DATE`, se usa la fecha máxima del dataset para que la práctica funcione igual sin importar cuándo se ejecute.

```sql
-- ============================================================
-- PASO 2C: Ventas por SEMANA (últimos 3 meses del dataset)
-- ============================================================
WITH rango AS (
    SELECT MAX(FECHA_VENTA) AS fecha_maxima
    FROM VENTAS
)
SELECT
    DATE_TRUNC('week', v.FECHA_VENTA)             AS inicio_semana,
    DATEADD('day', 6,
        DATE_TRUNC('week', v.FECHA_VENTA))        AS fin_semana,
    COUNT(*)                                      AS cantidad_ventas,
    ROUND(SUM(v.MONTO_TOTAL), 2)                  AS ventas_totales
FROM VENTAS v
CROSS JOIN rango r
WHERE v.FECHA_VENTA >= DATEADD('month', -3, r.fecha_maxima)
GROUP BY DATE_TRUNC('week', v.FECHA_VENTA)
ORDER BY inicio_semana DESC;
```

#### Resultado esperado

- **2A (mensual):** Una fila por mes con ventas agregadas. El número de filas debe coincidir con `meses_con_datos` del Paso 1. Con este dataset deben aparecer `36` filas.
- **2B (trimestral):** Una fila por trimestre. Con 3 años de datos, verás `12` filas.
- **2C (semanal):** Semanas correspondientes a los últimos 3 meses del dataset, tomando como referencia la fecha máxima de `VENTAS`.

> **Observa** cómo `DATE_TRUNC('month', fecha)` siempre devuelve el primer día del mes (ej. `2024-03-01`), no una cadena de texto. Esto permite ordenar correctamente con `ORDER BY`.

#### Verificación

```sql
-- Verificar que DATE_TRUNC produce un tipo temporal y no una cadena de texto
SELECT
    DATE_TRUNC('month', CURRENT_DATE)          AS fecha_truncada,
    TO_CHAR(DATE_TRUNC('month', CURRENT_DATE),
            'YYYY-MM')                         AS fecha_formateada,
    SYSTEM$TYPEOF(DATE_TRUNC('month', CURRENT_DATE))  AS tipo_de_dato
FROM DUAL;
```

Debes observar que `DATE_TRUNC` mantiene un tipo temporal (`DATE` o `TIMESTAMP_NTZ`, dependiendo de la expresión), mientras que `TO_CHAR` produce una representación textual. Esto es esencial para ordenar y calcular correctamente en los siguientes pasos.

---

### Ejercicio 3 — Comparación mes a mes con LAG()

**Objetivo:** Construir una comparación de ventas entre el mes actual y el mes anterior usando la función de ventana `LAG()` sobre la serie temporal mensual creada en el Paso 2.

#### Instrucciones

**Paso 3.1 — Comparación mes actual vs. mes anterior con LAG()**

Construye la consulta base con `LAG()` para traer el valor del mes anterior junto al mes actual. Este patrón conecta directamente con lo aprendido en el Laboratorio 4:

```sql
-- ============================================================
-- PASO 3A: Comparación mes actual vs mes anterior con LAG()
-- ============================================================
SELECT
    etiqueta_mes,
    periodo_mes,
    ventas_totales,
    LAG(ventas_totales) OVER (ORDER BY periodo_mes) AS ventas_mes_anterior,
    ventas_totales
        - LAG(ventas_totales) OVER (ORDER BY periodo_mes) AS diferencia_absoluta
FROM (
    -- Subquery: agrupación mensual base
    SELECT
        DATE_TRUNC('month', FECHA_VENTA)        AS periodo_mes,
        TO_CHAR(DATE_TRUNC('month', FECHA_VENTA),
                'YYYY-MM')                      AS etiqueta_mes,
        ROUND(SUM(MONTO_TOTAL), 2)              AS ventas_totales
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
) AS ventas_mensuales
ORDER BY periodo_mes;
```

**Paso 3.2 — Refactorizar usando CTEs**

Refactoriza la consulta anterior usando una CTE para mayor legibilidad. Esta es la versión recomendada para ambientes empresariales porque separa la agrupación base de la lógica analítica.

```sql
-- ============================================================
-- PASO 3B: Misma lógica con CTE (versión recomendada)
-- ============================================================
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA)        AS periodo_mes,
        TO_CHAR(DATE_TRUNC('month', FECHA_VENTA),
                'YYYY-MM')                      AS etiqueta_mes,
        ROUND(SUM(MONTO_TOTAL), 2)              AS ventas_totales
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
),
comparacion_mensual AS (
    SELECT
        etiqueta_mes,
        periodo_mes,
        ventas_totales                                                AS ventas_mes_actual,
        LAG(ventas_totales) OVER (ORDER BY periodo_mes)              AS ventas_mes_anterior,
        ventas_totales
            - LAG(ventas_totales) OVER (ORDER BY periodo_mes)        AS diferencia_absoluta
    FROM ventas_mensuales
)
SELECT
    etiqueta_mes,
    ventas_mes_actual,
    ventas_mes_anterior,
    diferencia_absoluta,
    -- Indicador visual de tendencia
    CASE
        WHEN diferencia_absoluta > 0 THEN '▲ Crecimiento'
        WHEN diferencia_absoluta < 0 THEN '▼ Caída'
        WHEN diferencia_absoluta = 0 THEN '→ Sin cambio'
        ELSE 'N/A (primer mes)'
    END AS tendencia
FROM comparacion_mensual
ORDER BY periodo_mes;
```

#### Resultado esperado

Debes ver una tabla con columnas similares a:

| etiqueta_mes | ventas_mes_actual | ventas_mes_anterior | diferencia_absoluta | tendencia |
|---|---:|---:|---:|---|
| 2022-01 | 12000.00 | NULL | NULL | N/A (primer mes) |
| 2022-02 | 10500.00 | 12000.00 | -1500.00 | ▼ Caída |
| 2022-03 | 13200.00 | 10500.00 | 2700.00 | ▲ Crecimiento |
| ... | ... | ... | ... | ... |

> **Nota:** La primera fila siempre tendrá `NULL` en `ventas_mes_anterior` porque `LAG()` no tiene fila previa para el primer período. Esto es comportamiento esperado y correcto.

#### Verificación

```sql
-- Verificar que LAG produce NULL solo en la primera fila
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS periodo_mes,
        ROUND(SUM(MONTO_TOTAL), 2)       AS ventas_totales
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
),
lag_calculado AS (
    SELECT
        periodo_mes,
        ventas_totales,
        LAG(ventas_totales) OVER (ORDER BY periodo_mes) AS ventas_mes_anterior
    FROM ventas_mensuales
)
SELECT
    COUNT(*) AS total_filas,
    COUNT(ventas_mes_anterior) AS filas_con_mes_anterior,
    COUNT(*) - COUNT(ventas_mes_anterior) AS filas_null_esperadas
FROM lag_calculado;
```

`filas_null_esperadas` debe ser exactamente `1` (el primer mes del dataset).

---

### Ejercicio 4 — Cálculo de variación porcentual

**Objetivo:** Extender la comparación del Paso 3 para calcular la tasa de variación porcentual entre períodos consecutivos, identificando los meses de mayor crecimiento y mayor caída.

#### Instrucciones

**Paso 4.1 — Variación porcentual mes a mes**

Agrega el cálculo de variación porcentual a la CTE del Paso 3. La fórmula es: `((actual - anterior) / anterior) * 100`.

```sql
-- ============================================================
-- PASO 4A: Variación porcentual mes a mes
-- ============================================================
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA)        AS periodo_mes,
        TO_CHAR(DATE_TRUNC('month', FECHA_VENTA),
                'YYYY-MM')                      AS etiqueta_mes,
        ROUND(SUM(MONTO_TOTAL), 2)              AS ventas_totales
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
),
comparacion_mensual AS (
    SELECT
        etiqueta_mes,
        periodo_mes,
        ventas_totales                                                    AS ventas_actual,
        LAG(ventas_totales) OVER (ORDER BY periodo_mes)                   AS ventas_anterior,
        ventas_totales
            - LAG(ventas_totales) OVER (ORDER BY periodo_mes)             AS diferencia_absoluta,
        -- Variación porcentual: evitar división por cero con NULLIF
        ROUND(
            (ventas_totales
                - LAG(ventas_totales) OVER (ORDER BY periodo_mes))
            / NULLIF(LAG(ventas_totales) OVER (ORDER BY periodo_mes), 0)
            * 100,
        2)                                                                AS variacion_pct
    FROM ventas_mensuales
)
SELECT
    etiqueta_mes,
    ventas_actual,
    ventas_anterior,
    diferencia_absoluta,
    variacion_pct,
    -- Clasificación de la magnitud del cambio
    CASE
        WHEN variacion_pct IS NULL                 THEN 'Sin referencia'
        WHEN variacion_pct >= 10                   THEN '🚀 Alto crecimiento'
        WHEN variacion_pct BETWEEN 0 AND 9.99       THEN '📈 Crecimiento moderado'
        WHEN variacion_pct BETWEEN -9.99 AND -0.01 THEN '📉 Caída moderada'
        WHEN variacion_pct <= -10                  THEN '⚠️ Caída significativa'
        ELSE 'Sin cambio'
    END AS clasificacion_variacion
FROM comparacion_mensual
ORDER BY periodo_mes;
```

**Paso 4.2 — Estadísticas de variación porcentual**

Analiza la distribución de las variaciones para tener una vista estadística de la volatilidad mensual:

```sql
-- ============================================================
-- PASO 4B: Estadísticas de variación porcentual
-- ============================================================
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA)        AS periodo_mes,
        ROUND(SUM(MONTO_TOTAL), 2)              AS ventas_totales
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
),
variaciones AS (
    SELECT
        periodo_mes,
        ventas_totales,
        ROUND(
            (ventas_totales
                - LAG(ventas_totales) OVER (ORDER BY periodo_mes))
            / NULLIF(LAG(ventas_totales) OVER (ORDER BY periodo_mes), 0)
            * 100,
        2) AS variacion_pct
    FROM ventas_mensuales
)
SELECT
    COUNT(variacion_pct)             AS meses_con_variacion,
    ROUND(AVG(variacion_pct), 2)     AS variacion_promedio_pct,
    ROUND(MAX(variacion_pct), 2)     AS mayor_crecimiento_pct,
    ROUND(MIN(variacion_pct), 2)     AS mayor_caida_pct,
    ROUND(STDDEV(variacion_pct), 2)  AS desviacion_estandar_pct
FROM variaciones
WHERE variacion_pct IS NOT NULL;
```

#### Resultado esperado

- **4A:** Tabla con una fila por mes mostrando la variación porcentual y su clasificación. Los valores de `variacion_pct` deben ser números decimales: positivos para crecimiento y negativos para caída.
- **4B:** Una sola fila con estadísticas descriptivas de la volatilidad mensual.

> **Importante:** El uso de `NULLIF(..., 0)` en el denominador previene errores de división por cero si algún mes tiene ventas registradas en cero. Esta es una práctica de escritura defensiva que debes aplicar siempre en cálculos de porcentajes.

#### Verificación

```sql
-- Verificar que no hay meses con ventas en cero
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS periodo_mes,
        ROUND(SUM(MONTO_TOTAL), 2)       AS ventas_totales
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
)
SELECT COUNT(*) AS meses_con_ventas_cero
FROM ventas_mensuales
WHERE ventas_totales = 0;
```

Con este dataset el resultado debe ser `0`. Si en otro dataset el resultado es mayor a `0`, el uso de `NULLIF` en el Paso 4A es imprescindible.

---

### Ejercicio 5 — Comparación año sobre año (YoY)

**Objetivo:** Construir una comparación Year-over-Year (YoY) que permita comparar el desempeño de cada mes en el año actual contra el mismo mes del año anterior, usando CTEs y un self-join.

#### Instrucciones

**Paso 5.1 — Comparación Year-over-Year con self-join en CTE**

Construye la comparación YoY usando un self-join sobre la CTE de ventas mensuales. Este enfoque es portable y muy legible:

```sql
-- ============================================================
-- PASO 5A: Comparación Year-over-Year con self-join en CTE
-- ============================================================
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA)        AS periodo_mes,
        YEAR(FECHA_VENTA)                       AS anio,
        MONTH(FECHA_VENTA)                      AS mes_numero,
        TO_CHAR(DATE_TRUNC('month', FECHA_VENTA),
                'YYYY-MM')                      AS etiqueta_mes,
        ROUND(SUM(MONTO_TOTAL), 2)              AS ventas_totales
    FROM VENTAS
    GROUP BY
        DATE_TRUNC('month', FECHA_VENTA),
        YEAR(FECHA_VENTA),
        MONTH(FECHA_VENTA)
)
SELECT
    anio_actual.etiqueta_mes                                AS mes,
    anio_actual.anio                                        AS anio_actual,
    anio_actual.ventas_totales                              AS ventas_anio_actual,
    anio_anterior.ventas_totales                            AS ventas_anio_anterior,
    anio_actual.ventas_totales
        - anio_anterior.ventas_totales                      AS diferencia_yoy,
    ROUND(
        (anio_actual.ventas_totales
            - anio_anterior.ventas_totales)
        / NULLIF(anio_anterior.ventas_totales, 0)
        * 100,
    2)                                                      AS variacion_yoy_pct
FROM ventas_mensuales AS anio_actual
-- Unimos con el mismo mes del año anterior
LEFT JOIN ventas_mensuales AS anio_anterior
    ON anio_actual.mes_numero = anio_anterior.mes_numero
   AND anio_actual.anio       = anio_anterior.anio + 1
ORDER BY
    anio_actual.anio,
    anio_actual.mes_numero;
```

**Paso 5.2 — Vista pivotada de ventas por mes y año**

Construye una vista pivotada alternativa que muestre los años como columnas para comparar visualmente. Esta vista es útil para presentaciones y reportes ejecutivos:

```sql
-- ============================================================
-- PASO 5B: Vista pivotada de ventas por mes y año
-- ============================================================
SELECT
    MONTH(FECHA_VENTA)                      AS mes_numero,
    TRIM(TO_CHAR(DATE_FROM_PARTS(2000, MONTH(FECHA_VENTA), 1),
        'Month'))                           AS nombre_mes,
    ROUND(SUM(CASE WHEN YEAR(FECHA_VENTA) = 2022
                   THEN MONTO_TOTAL ELSE 0 END), 2) AS ventas_2022,
    ROUND(SUM(CASE WHEN YEAR(FECHA_VENTA) = 2023
                   THEN MONTO_TOTAL ELSE 0 END), 2) AS ventas_2023,
    ROUND(SUM(CASE WHEN YEAR(FECHA_VENTA) = 2024
                   THEN MONTO_TOTAL ELSE 0 END), 2) AS ventas_2024
FROM VENTAS
GROUP BY
    MONTH(FECHA_VENTA),
    TRIM(TO_CHAR(DATE_FROM_PARTS(2000, MONTH(FECHA_VENTA), 1),
        'Month'))
ORDER BY mes_numero;
```

> **Nota:** El pivote manual con `CASE WHEN` es la técnica estándar en Snowflake para comparaciones YoY cuando los años son conocidos. Para pivotes dinámicos se puede usar `PIVOT`, pero eso está fuera del alcance de este laboratorio.

#### Resultado esperado

- **5A:** Una fila por mes mostrando ventas del año actual vs. año anterior y la variación YoY. Los meses del primer año del dataset tendrán `NULL` en `ventas_anio_anterior`, que es el comportamiento correcto del `LEFT JOIN`.
- **5B:** `12` filas, una por mes del año, con columnas separadas para `2022`, `2023` y `2024`, permitiendo comparar visualmente el desempeño estacional.

#### Verificación

```sql
-- Verificar que la comparación YoY tiene el número correcto de filas
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS periodo_mes,
        YEAR(FECHA_VENTA)                AS anio,
        MONTH(FECHA_VENTA)               AS mes_numero,
        ROUND(SUM(MONTO_TOTAL), 2)       AS ventas_totales
    FROM VENTAS
    GROUP BY
        DATE_TRUNC('month', FECHA_VENTA),
        YEAR(FECHA_VENTA),
        MONTH(FECHA_VENTA)
)
SELECT
    COUNT(*)                                      AS total_filas_yoy,
    COUNT(anio_anterior.ventas_totales)           AS filas_con_comparacion,
    COUNT(*) - COUNT(anio_anterior.ventas_totales) AS filas_sin_anio_anterior
FROM ventas_mensuales AS anio_actual
LEFT JOIN ventas_mensuales AS anio_anterior
    ON anio_actual.mes_numero = anio_anterior.mes_numero
   AND anio_actual.anio       = anio_anterior.anio + 1;
```

Con este dataset, `filas_sin_anio_anterior` debe ser `12`, correspondientes a los 12 meses de 2022 que no tienen año previo para comparar.

---

### Ejercicio 6 — Reporte de tendencias: Top 3 mejores y peores meses

**Objetivo:** Construir el reporte final que identifique los 3 meses con mayor crecimiento y los 3 meses con mayor caída en ventas, consolidando todas las técnicas del laboratorio en una consulta analítica completa.

#### Instrucciones

**Paso 6.1 — Reporte de tendencias Top 3 mejores y peores meses**

Construye el reporte completo de tendencias usando múltiples CTEs encadenadas:

```sql
-- ============================================================
-- PASO 6: Reporte de tendencias - Top 3 mejores y peores meses
-- ============================================================
WITH ventas_mensuales AS (
    -- CTE 1: Agrupación base por mes
    SELECT
        DATE_TRUNC('month', FECHA_VENTA)        AS periodo_mes,
        TO_CHAR(DATE_TRUNC('month', FECHA_VENTA),
                'YYYY-MM')                      AS etiqueta_mes,
        ROUND(SUM(MONTO_TOTAL), 2)              AS ventas_totales,
        COUNT(*)                                AS cantidad_transacciones,
        COUNT(DISTINCT ID_CLIENTE)              AS clientes_unicos
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
),
variaciones_mensuales AS (
    -- CTE 2: Calcular variación porcentual mes a mes
    SELECT
        etiqueta_mes,
        periodo_mes,
        ventas_totales,
        cantidad_transacciones,
        clientes_unicos,
        LAG(ventas_totales) OVER (ORDER BY periodo_mes) AS ventas_mes_anterior,
        ROUND(
            (ventas_totales
                - LAG(ventas_totales) OVER (ORDER BY periodo_mes))
            / NULLIF(LAG(ventas_totales) OVER (ORDER BY periodo_mes), 0)
            * 100,
        2)                                              AS variacion_pct
    FROM ventas_mensuales
),
meses_con_variacion AS (
    -- CTE 3: Filtrar solo meses con datos de comparación
    SELECT *
    FROM variaciones_mensuales
    WHERE variacion_pct IS NOT NULL
),
ranking_meses AS (
    -- CTE 4: Asignar rankings de mejor y peor desempeño
    SELECT
        etiqueta_mes,
        ventas_totales,
        ventas_mes_anterior,
        variacion_pct,
        cantidad_transacciones,
        clientes_unicos,
        ROW_NUMBER() OVER (ORDER BY variacion_pct DESC) AS rank_crecimiento,
        ROW_NUMBER() OVER (ORDER BY variacion_pct ASC)  AS rank_caida
    FROM meses_con_variacion
)
-- Consulta final: combinar top 3 crecimiento y top 3 caída
SELECT
    'TOP CRECIMIENTO'                   AS categoria,
    rank_crecimiento                    AS posicion,
    etiqueta_mes,
    ventas_totales,
    ventas_mes_anterior,
    variacion_pct,
    cantidad_transacciones,
    clientes_unicos
FROM ranking_meses
WHERE rank_crecimiento <= 3

UNION ALL

SELECT
    'TOP CAÍDA'                         AS categoria,
    rank_caida                          AS posicion,
    etiqueta_mes,
    ventas_totales,
    ventas_mes_anterior,
    variacion_pct,
    cantidad_transacciones,
    clientes_unicos
FROM ranking_meses
WHERE rank_caida <= 3

ORDER BY categoria, posicion;
```

**Paso 6.2 — Análisis de estacionalidad trimestral**

Complementa el reporte con un análisis de patrones estacionales por trimestre:

```sql
-- ============================================================
-- PASO 6B: Análisis de estacionalidad trimestral
-- ============================================================
WITH ventas_trimestrales AS (
    SELECT
        QUARTER(FECHA_VENTA)                AS numero_trimestre,
        'Q' || QUARTER(FECHA_VENTA)         AS trimestre_etiqueta,
        ROUND(AVG(MONTO_TOTAL), 2)          AS ticket_promedio,
        ROUND(SUM(MONTO_TOTAL), 2)          AS ventas_totales,
        COUNT(*)                            AS total_transacciones
    FROM VENTAS
    GROUP BY QUARTER(FECHA_VENTA)
),
media_global AS (
    SELECT ROUND(AVG(ventas_totales), 2) AS promedio_trimestral
    FROM ventas_trimestrales
)
SELECT
    t.trimestre_etiqueta,
    t.ventas_totales,
    t.ticket_promedio,
    t.total_transacciones,
    m.promedio_trimestral,
    ROUND(
        (t.ventas_totales - m.promedio_trimestral)
        / NULLIF(m.promedio_trimestral, 0)
        * 100,
    2)                                      AS desviacion_vs_promedio_pct,
    CASE
        WHEN t.ventas_totales > m.promedio_trimestral THEN 'Por encima del promedio'
        WHEN t.ventas_totales < m.promedio_trimestral THEN 'Por debajo del promedio'
        ELSE 'En el promedio'
    END                                     AS posicion_vs_promedio
FROM ventas_trimestrales t
CROSS JOIN media_global m
ORDER BY t.numero_trimestre;
```

#### Resultado esperado

- **6A:** 6 filas en total: 3 bajo la categoría `TOP CRECIMIENTO` y 3 bajo `TOP CAÍDA`, ordenadas por posición dentro de cada categoría.
- **6B:** 4 filas, una por trimestre, mostrando si cada trimestre está por encima o por debajo del promedio, permitiendo identificar estacionalidad clara.

Ejemplo parcial del resultado de 6A:

| categoria | posicion | etiqueta_mes | ventas_totales | ventas_mes_anterior | variacion_pct |
|---|---:|---|---:|---:|---:|
| TOP CAÍDA | 1 | 2022-08 | 11200.00 | 13000.00 | -13.85 |
| TOP CAÍDA | 2 | 2023-08 | 12800.00 | 14900.00 | -14.09 |
| TOP CAÍDA | 3 | ... | ... | ... | ... |
| TOP CRECIMIENTO | 1 | 2022-12 | 26000.00 | 19000.00 | 36.84 |
| TOP CRECIMIENTO | 2 | ... | ... | ... | ... |

#### Verificación

```sql
-- Verificar que el reporte tiene exactamente 6 filas con este dataset
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS periodo_mes,
        ROUND(SUM(MONTO_TOTAL), 2)       AS ventas_totales
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
),
variaciones AS (
    SELECT
        periodo_mes,
        ventas_totales,
        ROUND(
            (ventas_totales - LAG(ventas_totales) OVER (ORDER BY periodo_mes))
            / NULLIF(LAG(ventas_totales) OVER (ORDER BY periodo_mes), 0)
            * 100,
        2) AS variacion_pct
    FROM ventas_mensuales
),
variaciones_filtradas AS (
    SELECT *
    FROM variaciones
    WHERE variacion_pct IS NOT NULL
),
ranking AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY variacion_pct DESC) AS rank_crec,
        ROW_NUMBER() OVER (ORDER BY variacion_pct ASC)  AS rank_caida
    FROM variaciones_filtradas
)
SELECT COUNT(*) AS filas_reporte_final
FROM (
    SELECT rank_crec, rank_caida
    FROM ranking
    WHERE rank_crec <= 3

    UNION ALL

    SELECT rank_crec, rank_caida
    FROM ranking
    WHERE rank_caida <= 3
);
```

El resultado debe ser `6`.

> **Nota sobre QUALIFY:** Esta cláusula es exclusiva de Snowflake y permite filtrar directamente sobre el resultado de una window function sin necesidad de una subconsulta adicional. No es portable a PostgreSQL ni MySQL. En esta práctica se prioriza la claridad del flujo con CTEs, pero puedes usar `QUALIFY` como alternativa compacta cuando ya domines el patrón.

---

## Validación y Pruebas

Una vez completados todos los pasos, ejecuta el siguiente script de validación integral para confirmar que las consultas funcionan correctamente y producen resultados coherentes:

```sql
-- ============================================================
-- VALIDACIÓN INTEGRAL - Lab 05-00-01
-- Ejecutar completo para verificar todos los pasos
-- ============================================================

-- TEST 1: DATE_TRUNC produce un tipo temporal (Paso 2)
SELECT
    'TEST 1 - DATE_TRUNC tipo correcto' AS prueba,
    SYSTEM$TYPEOF(DATE_TRUNC('month', CURRENT_DATE)) AS tipo_detectado,
    CASE
        WHEN SYSTEM$TYPEOF(DATE_TRUNC('month', CURRENT_DATE)) LIKE '%DATE%'
          OR SYSTEM$TYPEOF(DATE_TRUNC('month', CURRENT_DATE)) LIKE '%TIMESTAMP%'
        THEN '✅ PASÓ'
        ELSE '❌ FALLÓ'
    END AS resultado;

-- TEST 2: LAG produce NULL solo en primera fila (Paso 3)
WITH base AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS periodo_mes,
        ROUND(SUM(MONTO_TOTAL), 2)       AS ventas
    FROM VENTAS
    GROUP BY 1
),
lag_test AS (
    SELECT
        periodo_mes,
        ventas,
        LAG(ventas) OVER (ORDER BY periodo_mes) AS lag_val
    FROM base
)
SELECT
    'TEST 2 - LAG NULL solo en primera fila' AS prueba,
    CASE
        WHEN SUM(CASE WHEN lag_val IS NULL THEN 1 ELSE 0 END) = 1
        THEN '✅ PASÓ'
        ELSE '❌ FALLÓ - Revisar datos o consulta'
    END AS resultado
FROM lag_test;

-- TEST 3: Variación porcentual sin errores de división por cero (Paso 4)
WITH base AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS periodo_mes,
        ROUND(SUM(MONTO_TOTAL), 2)       AS ventas
    FROM VENTAS
    GROUP BY 1
),
variacion AS (
    SELECT
        ROUND(
            (ventas - LAG(ventas) OVER (ORDER BY periodo_mes))
            / NULLIF(LAG(ventas) OVER (ORDER BY periodo_mes), 0)
            * 100, 2
        ) AS var_pct
    FROM base
)
SELECT
    'TEST 3 - Sin errores de división por cero' AS prueba,
    CASE
        WHEN COUNT(*) > 0 THEN '✅ PASÓ - Consulta ejecutada sin errores'
        ELSE '❌ FALLÓ'
    END AS resultado
FROM variacion;

-- TEST 4: Comparación YoY tiene datos para múltiples años (Paso 5)
SELECT
    'TEST 4 - Datos YoY disponibles' AS prueba,
    CASE
        WHEN COUNT(DISTINCT YEAR(FECHA_VENTA)) >= 2
        THEN '✅ PASÓ - ' || COUNT(DISTINCT YEAR(FECHA_VENTA)) || ' años disponibles'
        ELSE '❌ FALLÓ - Insuficientes años para comparación YoY'
    END AS resultado
FROM VENTAS;

-- TEST 5: Reporte top 3 produce exactamente 6 filas (Paso 6)
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS periodo_mes,
        ROUND(SUM(MONTO_TOTAL), 2)       AS ventas_totales
    FROM VENTAS
    GROUP BY 1
),
variaciones AS (
    SELECT
        periodo_mes,
        ventas_totales,
        ROUND(
            (ventas_totales - LAG(ventas_totales) OVER (ORDER BY periodo_mes))
            / NULLIF(LAG(ventas_totales) OVER (ORDER BY periodo_mes), 0)
            * 100, 2
        ) AS variacion_pct
    FROM ventas_mensuales
),
variaciones_filtradas AS (
    SELECT *
    FROM variaciones
    WHERE variacion_pct IS NOT NULL
),
ranking AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY variacion_pct DESC) AS rank_crec,
        ROW_NUMBER() OVER (ORDER BY variacion_pct ASC)  AS rank_caida
    FROM variaciones_filtradas
),
reporte AS (
    SELECT 'CREC' AS tipo
    FROM ranking
    WHERE rank_crec <= 3

    UNION ALL

    SELECT 'CAIDA' AS tipo
    FROM ranking
    WHERE rank_caida <= 3
)
SELECT
    'TEST 5 - Reporte top 3 produce 6 filas' AS prueba,
    CASE
        WHEN COUNT(*) = 6 THEN '✅ PASÓ'
        ELSE '❌ FALLÓ - Se esperaban 6 filas, se obtuvieron: '
             || COUNT(*) || '. Verificar si hay empates en variacion_pct.'
    END AS resultado
FROM reporte;
```

**Resultado esperado de la validación:** Los 5 tests deben mostrar `✅ PASÓ`. Si alguno falla, revisa el paso correspondiente antes de continuar.

---

## Resultados esperados clave con el dataset cargado

Estos resultados ayudan al instructor y al alumno a validar rápidamente que el laboratorio se está ejecutando sobre el dataset correcto.

| Consulta / Validación | Resultado esperado |
|---|---:|
| Conteo de `CLIENTES` | 12 filas |
| Conteo de `PRODUCTOS` | 8 filas |
| Conteo de `VENTAS` | 108 filas |
| Conteo de `PEDIDOS` | 108 filas |
| Años con datos en `VENTAS` | 2022, 2023 y 2024 |
| Meses con datos | 36 |
| Registros por año | 36 por cada año |
| Filas del análisis mensual | 36 |
| Filas del análisis trimestral por año | 12 |
| Filas del pivote YoY | 12 |
| Filas sin año anterior en comparación YoY | 12 |
| Reporte Top 3 crecimiento / Top 3 caída | 6 filas |
| `LAG()` global mensual con `NULL` | Solo 1 fila, correspondiente al primer mes |

---

## Solución de Problemas

### Problema 1: `LAG()` devuelve NULL en todas las filas, no solo en la primera

**Síntoma:** Al ejecutar la consulta del Paso 3, la columna `ventas_mes_anterior` muestra `NULL` en todas las filas, no solo en la primera.

**Causa probable:** La cláusula `ORDER BY` dentro de `LAG() OVER (ORDER BY ...)` está usando una columna de tipo `VARCHAR` (como `etiqueta_mes` en formato `'YYYY-MM'`) en lugar de la columna de tipo `DATE` (`periodo_mes`). Ordenar por texto produce un orden lexicográfico que puede no ser cronológico, y en casos de datos con formatos inconsistentes puede romper la ventana completamente. También puede ocurrir si la subquery o CTE base no está produciendo filas debido a un filtro incorrecto.

**Solución:**

```sql
-- ❌ Incorrecto: ORDER BY sobre campo VARCHAR
LAG(ventas_totales) OVER (ORDER BY etiqueta_mes)

-- ✅ Correcto: ORDER BY sobre campo DATE o TIMESTAMP
LAG(ventas_totales) OVER (ORDER BY periodo_mes)
```

Verifica también que la CTE `ventas_mensuales` produce filas ejecutándola de forma aislada:

```sql
-- Diagnóstico: ejecutar solo la CTE base
SELECT
    DATE_TRUNC('month', FECHA_VENTA) AS periodo_mes,
    ROUND(SUM(MONTO_TOTAL), 2)       AS ventas_totales
FROM VENTAS
GROUP BY DATE_TRUNC('month', FECHA_VENTA)
ORDER BY periodo_mes
LIMIT 5;
```

Si esta consulta no devuelve filas, el problema está en el contexto de base de datos o schema. Verifica con:

```sql
SELECT CURRENT_DATABASE(), CURRENT_SCHEMA();
```

---

### Problema 2: La variación porcentual YoY muestra valores inesperadamente extremos

**Síntoma:** Al ejecutar el Paso 5A, algunas filas muestran variaciones YoY de miles de por ciento, lo que no tiene sentido de negocio.

**Causa probable:** El `LEFT JOIN` en la comparación YoY está produciendo múltiples coincidencias por `mes_numero`, lo que ocurre cuando existen registros duplicados en la CTE `ventas_mensuales` o cuando el `GROUP BY` no está incluyendo todas las columnas necesarias. Si hay dos filas para el mismo mes en la CTE, el join multiplica los valores y produce sumas infladas.

**Solución:** Verifica que la CTE base no tiene duplicados de período:

```sql
-- Diagnóstico: buscar duplicados en la CTE base
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS periodo_mes,
        YEAR(FECHA_VENTA)                AS anio,
        MONTH(FECHA_VENTA)               AS mes_numero,
        ROUND(SUM(MONTO_TOTAL), 2)       AS ventas_totales
    FROM VENTAS
    GROUP BY
        DATE_TRUNC('month', FECHA_VENTA),
        YEAR(FECHA_VENTA),
        MONTH(FECHA_VENTA)
)
SELECT
    anio,
    mes_numero,
    COUNT(*) AS ocurrencias
FROM ventas_mensuales
GROUP BY anio, mes_numero
HAVING COUNT(*) > 1;
```

Si esta consulta devuelve filas, hay un problema en el `GROUP BY`. Asegúrate de que `DATE_TRUNC('month', FECHA_VENTA)`, `YEAR(FECHA_VENTA)` y `MONTH(FECHA_VENTA)` se incluyen todos en el `GROUP BY` y son consistentes entre sí. En Snowflake, `DATE_TRUNC('month', fecha)` siempre produce el primer día del mes, por lo que `YEAR` y `MONTH` extraídos de ese valor serán siempre consistentes con los extraídos de la fecha original.

---

### Problema 3: El Paso 2C no devuelve registros

**Síntoma:** La consulta semanal de los últimos 3 meses devuelve 0 filas.

**Causa probable:** El dataset es histórico y no llega hasta la fecha actual. Si usas `CURRENT_DATE` para filtrar los últimos 3 meses, Snowflake buscará datos recientes respecto al día de ejecución, no respecto al último dato del dataset.

**Solución:** Usa `MAX(FECHA_VENTA)` como referencia temporal del dataset:

```sql
WITH rango AS (
    SELECT MAX(FECHA_VENTA) AS fecha_maxima
    FROM VENTAS
)
SELECT
    DATE_TRUNC('week', v.FECHA_VENTA) AS inicio_semana,
    COUNT(*) AS cantidad_ventas,
    ROUND(SUM(v.MONTO_TOTAL), 2) AS ventas_totales
FROM VENTAS v
CROSS JOIN rango r
WHERE v.FECHA_VENTA >= DATEADD('month', -3, r.fecha_maxima)
GROUP BY DATE_TRUNC('week', v.FECHA_VENTA)
ORDER BY inicio_semana DESC;
```

---

### Problema 4: Error "Object 'VENTAS' does not exist or not authorized"

**Síntoma:** Al ejecutar cualquier consulta que referencia `VENTAS`, Snowflake retorna un error de objeto no encontrado o sin permisos.

**Causa probable:** El contexto de sesión no está configurado correctamente o no se ejecutó el setup del Paso 0.

**Solución:**

1. Verifica el contexto activo:

```sql
SELECT CURRENT_WAREHOUSE(), CURRENT_DATABASE(), CURRENT_SCHEMA();
```

2. Configura el contexto correcto:

```sql
USE WAREHOUSE COMPUTE_WH;
USE DATABASE LAB_SQL_INTERMEDIO;
USE SCHEMA VENTAS;
```

3. Verifica que la tabla existe:

```sql
SHOW TABLES LIKE 'VENTAS';
```

4. Si no aparece, vuelve al archivo **`05_SETUP_DATOS_TENDENCIAS_PERIODO`** y ejecuta el script completo de setup.

---

## Limpieza del Entorno

Al finalizar el laboratorio, ejecuta los siguientes comandos para liberar recursos y evitar consumo innecesario de créditos Snowflake:

```sql
-- ============================================================
-- LIMPIEZA POST-LABORATORIO
-- Ejecutar obligatoriamente al terminar la sesión
-- ============================================================

-- 1. Suspender el warehouse para detener el consumo de créditos
--    (OBLIGATORIO en cuentas trial)
ALTER WAREHOUSE COMPUTE_WH SUSPEND;

-- 2. Verificar que el warehouse quedó suspendido
SHOW WAREHOUSES LIKE 'COMPUTE_WH';
-- Confirmar que el campo "state" muestra SUSPENDED

-- 3. No es necesario eliminar objetos:
--    las tablas creadas son parte del entorno de práctica
--    y pueden reutilizarse en laboratorios posteriores.
```

> ✅ **Confirmación:** Si el campo `state` del warehouse muestra `SUSPENDED`, el laboratorio ha concluido correctamente y no se generarán cargos adicionales.

---

## Resumen

En este laboratorio aplicaste un conjunto completo de técnicas de análisis temporal en Snowflake, construyendo progresivamente desde la exploración básica hasta un reporte ejecutivo de tendencias. Los conceptos clave que practicaste:

| Técnica | Función(es) usada(s) | Paso donde se aplicó |
|---|---|---|
| Agrupación temporal por granularidad | `DATE_TRUNC('month'/'week'/'quarter', ...)` | Ejercicio 2 |
| Extracción de componentes de fecha | `YEAR()`, `MONTH()`, `QUARTER()`, `DAYOFWEEK()` | Ejercicios 1, 2, 5 |
| Desplazamiento temporal en filtros | `DATEADD('month', -3, fecha_maxima)` | Ejercicio 2C |
| Comparación período anterior | `LAG() OVER (ORDER BY periodo_mes)` | Ejercicios 3, 4, 6 |
| Variación porcentual segura | `(actual - anterior) / NULLIF(anterior, 0) * 100` | Ejercicios 4, 5, 6 |
| Comparación Year-over-Year | Self-join en CTE + pivote con `CASE WHEN` | Ejercicio 5 |
| Reporte de rankings temporales | `ROW_NUMBER() OVER (ORDER BY variacion_pct)` | Ejercicio 6 |
| Formateo de fechas para presentación | `TO_CHAR(fecha, 'YYYY-MM')` | Ejercicios 2, 3, 5, 6 |
| Filtrado indirecto de window functions | CTEs encadenadas / `QUALIFY` como alternativa Snowflake | Ejercicio 6 |

### Conexión con el siguiente módulo

Las técnicas de este laboratorio son la base directa del **Laboratorio 6**, donde aplicarás reconciliación de datasets comparando métricas entre dos fuentes de datos (`VENTAS_ORIGEN` vs. `VENTAS_DESTINO`). Las CTEs encadenadas y los patrones de comparación entre períodos que practicaste aquí se reutilizarán directamente en ese contexto de validación de datos.

### Recursos adicionales

| Recurso | URL |
|---|---|
| Documentación Snowflake: Date & Time Functions | https://docs.snowflake.com/en/sql-reference/functions-date-time |
| Documentación Snowflake: DATE_TRUNC | https://docs.snowflake.com/en/sql-reference/functions/date_trunc |
| Documentación Snowflake: DATEDIFF | https://docs.snowflake.com/en/sql-reference/functions/datediff |
| Documentación Snowflake: DATEADD | https://docs.snowflake.com/en/sql-reference/functions/dateadd |
| Documentación Snowflake: Window Functions (LAG/LEAD) | https://docs.snowflake.com/en/sql-reference/functions-analytic |
| Documentación Snowflake: QUALIFY clause | https://docs.snowflake.com/en/sql-reference/constructs/qualify |

---

*Lab 05-00-01 — Análisis de tendencias y métricas por periodo | LAB_SQL_INTERMEDIO | Módulo 5*
