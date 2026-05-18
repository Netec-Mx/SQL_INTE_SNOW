# Clasificación de clientes y reglas de negocio con CASE WHEN

## Metadatos

| Campo | Detalle |
|---|---|
| **Duración estimada** | 60 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (*Apply*) |
| **Módulo** | 2 — Clasificación y segmentación con `CASE WHEN` |
| **Plataforma** | Snowflake (Snowsight Worksheet) |
| **Schema de práctica** | `LAB_SQL_INTERMEDIO.VENTAS` |
| **Laboratorio previo recomendado** | Práctica 1 — Reestructuración de consultas con CTE y subqueries |

---

## Descripción General

En este laboratorio aplicarás `CASE WHEN` para construir un sistema de segmentación de clientes basado en reglas de negocio. Partirás de un dataset controlado dentro del schema `LAB_SQL_INTERMEDIO.VENTAS` y generarás clasificaciones progresivas: primero una segmentación simple por monto de pedido, después una clasificación multinivel de clientes combinando monto, frecuencia y antigüedad, y finalmente un resumen ejecutivo por segmento con métricas agregadas.

Esta práctica incluye un **paso previo de setup de datos**, porque los ejercicios necesitan datos específicos para activar todos los escenarios: clientes `GOLD`, `SILVER`, `BRONZE`, `NEW`, pedidos cancelados, pedidos de bajo/medio/alto/premium valor y casos de riesgo de abandono.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Implementar `CASE WHEN` en forma buscada y simple para clasificar registros según reglas de negocio.
- [ ] Clasificar pedidos por rango de monto usando condiciones ordenadas.
- [ ] Construir una segmentación multinivel de clientes con condiciones compuestas usando `AND`, `OR` y `BETWEEN`.
- [ ] Integrar `CASE WHEN` dentro de agregaciones (`SUM`, `COUNT`, `AVG`) para generar reportes ejecutivos.
- [ ] Usar `IFF()` como alternativa simplificada de Snowflake para clasificaciones binarias.
- [ ] Preparar un dataset enriquecido con columnas derivadas de clasificación usando CTEs encadenadas.

---

## Prerrequisitos

### Conocimientos previos

| Área | Nivel requerido |
|---|---|
| `SELECT`, `FROM`, `WHERE`, `ORDER BY` | Sólido |
| `INNER JOIN` y `LEFT JOIN` | Intermedio |
| `GROUP BY` con `SUM`, `COUNT`, `AVG`, `MAX`, `MIN` | Intermedio |
| CTEs con `WITH ... AS` | Recomendado |
| Operadores lógicos `AND`, `OR`, `BETWEEN`, `IN` | Intermedio |
| Concepto de valor `NULL` y uso de `COALESCE` | Básico |

### Acceso y configuración

| Requisito | Detalle |
|---|---|
| Cuenta Snowflake activa | Trial o corporativa |
| Rol sugerido | `SYSADMIN` o rol equivalente asignado por el instructor |
| Script de setup previo | No se asume script externo. Esta práctica incluye el setup completo de tablas y datos |
| Database disponible | `LAB_SQL_INTERMEDIO` |
| Schema disponible | `LAB_SQL_INTERMEDIO.VENTAS` |
| Tablas requeridas | `CLIENTES`, `PEDIDOS`, `PRODUCTOS`, `VENTAS` |
| Warehouse activo | `COMPUTE_WH` tamaño `X-SMALL` |

---

## Entorno de Laboratorio

### Hardware recomendado

| Componente | Mínimo | Recomendado |
|---|---|---|
| Procesador | Intel Core i5 / AMD Ryzen 5 | Intel Core i7 / AMD Ryzen 7 |
| RAM | 8 GB | 16 GB |
| Almacenamiento libre | 500 MB | 2 GB |
| Conexión a Internet | 10 Mbps | 25 Mbps |
| Resolución de pantalla | 1280×768 | 1920×1080 |

### Software requerido

| Software | Versión mínima | Uso |
|---|---|---|
| Navegador web Chrome / Firefox / Edge / Safari | Versión reciente | Acceso a Snowsight |
| Snowflake Snowsight | Versión web actual | Ejecución de consultas SQL |
| Visual Studio Code *(opcional)* | 1.80+ | Edición local de scripts |
| SnowSQL *(opcional)* | 1.2.x+ | Ejecución desde terminal |

---

## Organización recomendada de Workspace en Snowsight

Para mantener ordenado el laboratorio, separa el script de carga de datos del script de ejercicios.

| Workspace | Folder | Nombre sugerido | Uso |
|---|---|---|---|
| `SNOWLABS-INT` | `SETUP-LABS` | `02_SETUP_DATOS_CASE_WHEN_SEGMENTACION` | Crear o recrear tablas y cargar datos del Lab 02 |
| `SNOWLABS-INT` | `SCRIPT-LABS` | `02_LAB_CASE_WHEN_SEGMENTACION` | Ejecutar los ejercicios del laboratorio |

> Nota: si ya creaste el workspace `SNOWLABS-INT` en la práctica 1, no lo vuelvas a crear. Solo agrega los folders o archivos faltantes.

---

## Paso 0 — Preparación del workspace y dataset

### Paso 0.0 — Crear o reutilizar el workspace de prácticas

1. Entra a **Snowsight**.
2. Da clic en **Projects**.
3. Si ya existe el workspace **`SNOWLABS-INT`**, ábrelo.
4. Si no existe:
   1. Da clic en **+**.
   2. Selecciona **Private workspace**.
   3. Nómbralo **`SNOWLABS-INT`**.
   4. Da clic en **Create**.

### Paso 0.1 — Crear el folder y script de setup

1. Dentro del workspace **`SNOWLABS-INT`**, da clic en **+ Add new**.
2. Crea un folder llamado **`SETUP-LABS`** si aún no existe.
3. Dentro de **`SETUP-LABS`**, crea un archivo de tipo **SQL**.
4. Nómbralo **`02_SETUP_DATOS_CASE_WHEN_SEGMENTACION`**.
5. Pega y ejecuta completo el siguiente script.

Este dataset está diseñado para activar todos los escenarios de la práctica:

- Pedidos en categorías `Bajo`, `Medio`, `Alto` y `Premium`.
- Estados `COMPLETADO`, `EN_PROCESO`, `ENVIADO` y `CANCELADO`.
- Clientes clasificados como `GOLD`, `SILVER`, `BRONZE` y `NEW`.
- Clientes sin pedidos válidos.
- Clientes registrados hace menos de 30 días.
- Pedidos recientes, de 90 a 180 días y de más de 180 días para analizar riesgo de abandono.
- Tabla `VENTAS` disponible para validaciones y laboratorios posteriores.

```sql
-- ============================================================
-- 02_SETUP_DATOS_CASE_WHEN_SEGMENTACION
-- Práctica 2: Clasificación de clientes y reglas de negocio
-- Plataforma: Snowflake
-- Schema: LAB_SQL_INTERMEDIO.VENTAS
-- ============================================================

USE WAREHOUSE COMPUTE_WH;

CREATE DATABASE IF NOT EXISTS LAB_SQL_INTERMEDIO;
USE DATABASE LAB_SQL_INTERMEDIO;

CREATE SCHEMA IF NOT EXISTS VENTAS;
USE SCHEMA VENTAS;

-- Este setup recrea las tablas del schema de práctica.
-- Se incluyen columnas compatibles con la práctica 1 y con la práctica 2:
--   CLIENTE_ID / ID_CLIENTE
--   PEDIDO_ID  / ID_PEDIDO
--   MONTO      / MONTO_TOTAL
--
-- Esto permite que los laboratorios puedan ejecutarse sin depender de
-- nombres de columnas incompatibles entre prácticas.

DROP TABLE IF EXISTS VENTAS;
DROP TABLE IF EXISTS PEDIDOS;
DROP TABLE IF EXISTS PRODUCTOS;
DROP TABLE IF EXISTS CLIENTES;

CREATE OR REPLACE TABLE CLIENTES (
    ID_CLIENTE NUMBER(10,0) NOT NULL,
    CLIENTE_ID NUMBER(10,0) NOT NULL,
    NOMBRE VARCHAR(100) NOT NULL,
    EMAIL VARCHAR(150),
    CIUDAD VARCHAR(80) NOT NULL,
    PAIS VARCHAR(80) NOT NULL,
    SEGMENTO VARCHAR(40),
    FECHA_REGISTRO DATE NOT NULL,
    FECHA_ALTA DATE NOT NULL,
    CONSTRAINT PK_CLIENTES PRIMARY KEY (ID_CLIENTE)
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
    ID_PEDIDO NUMBER(10,0) NOT NULL,
    PEDIDO_ID NUMBER(10,0) NOT NULL,
    ID_CLIENTE NUMBER(10,0) NOT NULL,
    CLIENTE_ID NUMBER(10,0) NOT NULL,
    PRODUCTO_ID NUMBER(10,0) NOT NULL,
    FECHA_PEDIDO DATE NOT NULL,
    CANTIDAD NUMBER(10,0) NOT NULL,
    MONTO_TOTAL NUMBER(12,2) NOT NULL,
    MONTO NUMBER(12,2) NOT NULL,
    ESTADO_PEDIDO VARCHAR(30) NOT NULL,
    CANAL VARCHAR(40),
    CONSTRAINT PK_PEDIDOS PRIMARY KEY (ID_PEDIDO)
);

INSERT INTO CLIENTES (
    ID_CLIENTE, CLIENTE_ID, NOMBRE, EMAIL, CIUDAD, PAIS, SEGMENTO, FECHA_REGISTRO, FECHA_ALTA
) VALUES
    (1,  1,  'Ana Torres',        'ana.torres@techcommerce.com',        'CDMX',        'México', 'Retail',     DATEADD(day, -820, CURRENT_DATE()), DATEADD(day, -820, CURRENT_DATE())),
    (2,  2,  'Luis Martínez',     'luis.martinez@techcommerce.com',     'CDMX',        'México', 'Retail',     DATEADD(day, -760, CURRENT_DATE()), DATEADD(day, -760, CURRENT_DATE())),
    (3,  3,  'María López',       'maria.lopez@techcommerce.com',       'Guadalajara', 'México', 'PyME',       DATEADD(day, -430, CURRENT_DATE()), DATEADD(day, -430, CURRENT_DATE())),
    (4,  4,  'Carlos Hernández',  'carlos.hernandez@techcommerce.com',  'Guadalajara', 'México', 'Enterprise', DATEADD(day, -390, CURRENT_DATE()), DATEADD(day, -390, CURRENT_DATE())),
    (5,  5,  'Sofía Ramírez',     'sofia.ramirez@techcommerce.com',     'Monterrey',   'México', 'PyME',       DATEADD(day, -210, CURRENT_DATE()), DATEADD(day, -210, CURRENT_DATE())),
    (6,  6,  'Jorge Castillo',    'jorge.castillo@techcommerce.com',    'Monterrey',   'México', 'Retail',     DATEADD(day, -500, CURRENT_DATE()), DATEADD(day, -500, CURRENT_DATE())),
    (7,  7,  'Elena Flores',      'elena.flores@techcommerce.com',      'Puebla',      'México', 'Enterprise', DATEADD(day, -240, CURRENT_DATE()), DATEADD(day, -240, CURRENT_DATE())),
    (8,  8,  'Diego Sánchez',     'diego.sanchez@techcommerce.com',     'Puebla',      'México', 'PyME',       DATEADD(day, -10,  CURRENT_DATE()), DATEADD(day, -10,  CURRENT_DATE())),
    (9,  9,  'Valeria Cruz',      'valeria.cruz@techcommerce.com',      'Mérida',      'México', 'Enterprise', DATEADD(day, -650, CURRENT_DATE()), DATEADD(day, -650, CURRENT_DATE())),
    (10, 10, 'Roberto Díaz',      'roberto.diaz@techcommerce.com',      'Mérida',      'México', 'Retail',     DATEADD(day, -15,  CURRENT_DATE()), DATEADD(day, -15,  CURRENT_DATE())),
    (11, 11, 'Patricia Navarro',  'patricia.navarro@techcommerce.com',  'Querétaro',   'México', 'PyME',       DATEADD(day, -120, CURRENT_DATE()), DATEADD(day, -120, CURRENT_DATE())),
    (12, 12, 'Fernando Rivas',    'fernando.rivas@techcommerce.com',    'Querétaro',   'México', 'Retail',     DATEADD(day, -300, CURRENT_DATE()), DATEADD(day, -300, CURRENT_DATE()));

INSERT INTO PRODUCTOS (PRODUCTO_ID, NOMBRE, CATEGORIA, PRECIO_UNITARIO, ACTIVO) VALUES
    (1,  'Laptop Pro 14',              'Electrónica',     1200.00, TRUE),
    (2,  'Monitor 27 pulgadas',        'Electrónica',      350.00, TRUE),
    (3,  'Teclado mecánico',           'Accesorios',       120.00, TRUE),
    (4,  'Mouse inalámbrico',          'Accesorios',        80.00, TRUE),
    (5,  'Silla ergonómica',           'Oficina',          420.00, TRUE),
    (6,  'Escritorio ajustable',       'Oficina',          680.00, TRUE),
    (7,  'Licencia BI anual',          'Software',         900.00, TRUE),
    (8,  'Licencia CRM anual',         'Software',        1100.00, TRUE),
    (9,  'Servidor compacto',          'Infraestructura', 1500.00, TRUE),
    (10, 'NAS 8TB',                    'Infraestructura',  850.00, TRUE),
    (11, 'Tablet Ejecutiva',           'Electrónica',      550.00, TRUE),
    (12, 'Audífonos con cancelación',  'Accesorios',       220.00, TRUE);

INSERT INTO PEDIDOS (
    ID_PEDIDO, PEDIDO_ID, ID_CLIENTE, CLIENTE_ID, PRODUCTO_ID,
    FECHA_PEDIDO, CANTIDAD, MONTO_TOTAL, MONTO, ESTADO_PEDIDO, CANAL
) VALUES
    -- Cliente 1: GOLD activo. Total válido = 5200, pedidos válidos = 5
    (2001, 2001, 1, 1, 1,  DATEADD(day, -20, CURRENT_DATE()), 1, 1200.00, 1200.00, 'COMPLETADO', 'Web'),
    (2002, 2002, 1, 1, 8,  DATEADD(day, -35, CURRENT_DATE()), 1, 1300.00, 1300.00, 'ENVIADO',    'Ejecutivo'),
    (2003, 2003, 1, 1, 7,  DATEADD(day, -50, CURRENT_DATE()), 1,  800.00,  800.00, 'COMPLETADO', 'Web'),
    (2004, 2004, 1, 1, 2,  DATEADD(day, -65, CURRENT_DATE()), 2,  900.00,  900.00, 'COMPLETADO', 'Marketplace'),
    (2005, 2005, 1, 1, 6,  DATEADD(day, -75, CURRENT_DATE()), 1, 1000.00, 1000.00, 'EN_PROCESO', 'Ejecutivo'),

    -- Cliente 2: GOLD con riesgo alto. Total válido = 5800, pedidos válidos = 5
    (2006, 2006, 2, 2, 9,  DATEADD(day, -220, CURRENT_DATE()), 2, 3000.00, 3000.00, 'COMPLETADO', 'Ejecutivo'),
    (2007, 2007, 2, 2, 1,  DATEADD(day, -230, CURRENT_DATE()), 1, 1000.00, 1000.00, 'COMPLETADO', 'Web'),
    (2008, 2008, 2, 2, 10, DATEADD(day, -245, CURRENT_DATE()), 1,  700.00,  700.00, 'COMPLETADO', 'Marketplace'),
    (2009, 2009, 2, 2, 5,  DATEADD(day, -260, CURRENT_DATE()), 1,  600.00,  600.00, 'ENVIADO',    'Web'),
    (2010, 2010, 2, 2, 3,  DATEADD(day, -275, CURRENT_DATE()), 2,  500.00,  500.00, 'COMPLETADO', 'Web'),

    -- Cliente 3: SILVER con riesgo medio. Total válido = 1700, pedidos válidos = 2
    (2011, 2011, 3, 3, 7,  DATEADD(day, -120, CURRENT_DATE()), 1,  800.00,  800.00, 'COMPLETADO', 'Web'),
    (2012, 2012, 3, 3, 8,  DATEADD(day, -135, CURRENT_DATE()), 1,  900.00,  900.00, 'COMPLETADO', 'Ejecutivo'),

    -- Cliente 4: SILVER activo. Total válido = 2600, pedidos válidos = 3
    (2013, 2013, 4, 4, 1,  DATEADD(day, -25, CURRENT_DATE()), 1, 1200.00, 1200.00, 'COMPLETADO', 'Ejecutivo'),
    (2014, 2014, 4, 4, 7,  DATEADD(day, -60, CURRENT_DATE()), 1,  900.00,  900.00, 'ENVIADO',    'Web'),
    (2015, 2015, 4, 4, 5,  DATEADD(day, -80, CURRENT_DATE()), 1,  500.00,  500.00, 'COMPLETADO', 'Marketplace'),

    -- Cliente 5: BRONZE activo. Total válido = 150, pedidos válidos = 1
    (2016, 2016, 5, 5, 4,  DATEADD(day, -15, CURRENT_DATE()), 1,  150.00,  150.00, 'COMPLETADO', 'Web'),

    -- Cliente 6: BRONZE con riesgo alto. Total válido = 500, pedidos válidos = 1
    (2017, 2017, 6, 6, 3,  DATEADD(day, -240, CURRENT_DATE()), 1,  500.00,  500.00, 'COMPLETADO', 'Web'),

    -- Cliente 8: NEW por antigüedad menor a 30 días, aunque tenga compra válida.
    (2018, 2018, 8, 8, 9,  DATEADD(day, -5, CURRENT_DATE()),  1, 3000.00, 3000.00, 'COMPLETADO', 'Ejecutivo'),
    (2019, 2019, 8, 8, 4,  DATEADD(day, -3, CURRENT_DATE()),  1,  100.00,  100.00, 'EN_PROCESO', 'Web'),

    -- Cliente 9: SILVER con riesgo alto. Total válido = 1800, pedidos válidos = 2
    (2020, 2020, 9, 9, 10, DATEADD(day, -210, CURRENT_DATE()), 1,  850.00,  850.00, 'COMPLETADO', 'Ejecutivo'),
    (2021, 2021, 9, 9, 6,  DATEADD(day, -195, CURRENT_DATE()), 1,  950.00,  950.00, 'COMPLETADO', 'Marketplace'),

    -- Cliente 10: NEW. Solo pedido cancelado, no debe contar como compra válida.
    (2022, 2022, 10, 10, 1, DATEADD(day, -8, CURRENT_DATE()), 1, 4000.00, 4000.00, 'CANCELADO', 'Web'),

    -- Cliente 12: BRONZE con riesgo medio. Total válido = 750, pedidos válidos = 1
    (2023, 2023, 12, 12, 2, DATEADD(day, -100, CURRENT_DATE()), 1,  750.00,  750.00, 'COMPLETADO', 'Marketplace');

-- Tabla denormalizada de ventas para validaciones y laboratorios posteriores.
CREATE OR REPLACE TABLE VENTAS AS
SELECT
    p.ID_PEDIDO AS ID_VENTA,
    p.ID_PEDIDO,
    p.ID_CLIENTE,
    c.NOMBRE AS NOMBRE_CLIENTE,
    c.CIUDAD,
    c.PAIS,
    p.PRODUCTO_ID,
    pr.NOMBRE AS NOMBRE_PRODUCTO,
    pr.CATEGORIA,
    p.FECHA_PEDIDO AS FECHA_VENTA,
    p.CANTIDAD,
    p.MONTO_TOTAL,
    p.ESTADO_PEDIDO,
    p.CANAL
FROM PEDIDOS p
INNER JOIN CLIENTES c
    ON p.ID_CLIENTE = c.ID_CLIENTE
INNER JOIN PRODUCTOS pr
    ON p.PRODUCTO_ID = pr.PRODUCTO_ID;

-- Validación rápida del dataset.
SELECT 'CLIENTES' AS TABLA, COUNT(*) AS FILAS FROM CLIENTES
UNION ALL
SELECT 'PRODUCTOS' AS TABLA, COUNT(*) AS FILAS FROM PRODUCTOS
UNION ALL
SELECT 'PEDIDOS' AS TABLA, COUNT(*) AS FILAS FROM PEDIDOS
UNION ALL
SELECT 'VENTAS' AS TABLA, COUNT(*) AS FILAS FROM VENTAS
ORDER BY TABLA;

-- Resultado esperado:
-- CLIENTES  = 12
-- PEDIDOS   = 23
-- PRODUCTOS = 12
-- VENTAS    = 23

-- Validación de categorías de monto en pedidos válidos.
SELECT
    CASE
        WHEN MONTO_TOTAL < 200  THEN 'Bajo'
        WHEN MONTO_TOTAL < 1000 THEN 'Medio'
        WHEN MONTO_TOTAL < 3000 THEN 'Alto'
        ELSE 'Premium'
    END AS CATEGORIA_MONTO,
    COUNT(*) AS TOTAL_PEDIDOS
FROM PEDIDOS
WHERE ESTADO_PEDIDO != 'CANCELADO'
GROUP BY CATEGORIA_MONTO
ORDER BY
    CASE CATEGORIA_MONTO
        WHEN 'Bajo' THEN 1
        WHEN 'Medio' THEN 2
        WHEN 'Alto' THEN 3
        WHEN 'Premium' THEN 4
    END;

-- Validación de segmentos esperados.
WITH METRICAS_CLIENTE AS (
    SELECT
        c.ID_CLIENTE,
        DATEDIFF('day', c.FECHA_REGISTRO, CURRENT_DATE()) AS DIAS_COMO_CLIENTE,
        COUNT(p.ID_PEDIDO) AS TOTAL_PEDIDOS,
        COALESCE(SUM(p.MONTO_TOTAL), 0) AS MONTO_TOTAL_GASTADO
    FROM CLIENTES c
    LEFT JOIN PEDIDOS p
        ON c.ID_CLIENTE = p.ID_CLIENTE
       AND p.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY c.ID_CLIENTE, c.FECHA_REGISTRO
)
SELECT
    CASE
        WHEN TOTAL_PEDIDOS = 0 OR DIAS_COMO_CLIENTE < 30 THEN 'NEW'
        WHEN MONTO_TOTAL_GASTADO >= 5000 AND TOTAL_PEDIDOS >= 5 THEN 'GOLD'
        WHEN MONTO_TOTAL_GASTADO >= 1500 AND TOTAL_PEDIDOS >= 2 THEN 'SILVER'
        WHEN TOTAL_PEDIDOS >= 1 THEN 'BRONZE'
        ELSE 'NEW'
    END AS SEGMENTO_CLIENTE,
    COUNT(*) AS CANTIDAD_CLIENTES
FROM METRICAS_CLIENTE
GROUP BY SEGMENTO_CLIENTE
ORDER BY
    CASE SEGMENTO_CLIENTE
        WHEN 'GOLD' THEN 1
        WHEN 'SILVER' THEN 2
        WHEN 'BRONZE' THEN 3
        WHEN 'NEW' THEN 4
    END;
```

### Paso 0.2 — Crear el folder y script del laboratorio

1. En el workspace **`SNOWLABS-INT`**, da clic en **+ Add new**.
2. Crea o usa el folder **`SCRIPT-LABS`**.
3. Dentro de **`SCRIPT-LABS`**, crea un archivo de tipo **SQL**.
4. Nómbralo **`02_LAB_CASE_WHEN_SEGMENTACION`**.
5. Usa este archivo para ejecutar los ejercicios de la práctica.
6. No pegues aquí el script completo de carga; úsalo solo para las consultas del laboratorio.

### Paso 0.3 — Confirmar contexto de trabajo

Dentro del archivo **`02_LAB_CASE_WHEN_SEGMENTACION`**, ejecuta:

```sql
USE WAREHOUSE COMPUTE_WH;
USE DATABASE LAB_SQL_INTERMEDIO;
USE SCHEMA VENTAS;

SHOW TABLES;
```

**Resultado esperado:** deben aparecer al menos las tablas:

| Tabla | Uso en la práctica |
|---|---|
| `CLIENTES` | Maestro de clientes con ciudad, país, email y fecha de registro |
| `PEDIDOS` | Transacciones por cliente con monto, fecha y estado |
| `PRODUCTOS` | Catálogo de productos |
| `VENTAS` | Tabla denormalizada de ventas para validaciones y futuros laboratorios |

### Paso 0.4 — Validar volumen mínimo de datos

Ejecuta:

```sql
SELECT 'CLIENTES' AS TABLA, COUNT(*) AS FILAS FROM CLIENTES
UNION ALL
SELECT 'PRODUCTOS' AS TABLA, COUNT(*) AS FILAS FROM PRODUCTOS
UNION ALL
SELECT 'PEDIDOS' AS TABLA, COUNT(*) AS FILAS FROM PEDIDOS
UNION ALL
SELECT 'VENTAS' AS TABLA, COUNT(*) AS FILAS FROM VENTAS
ORDER BY TABLA;
```

**Resultado esperado:**

| TABLA | FILAS |
|---|---:|
| CLIENTES | 12 |
| PEDIDOS | 23 |
| PRODUCTOS | 12 |
| VENTAS | 23 |

### Paso 0.5 — Validar que existen estados de pedido

```sql
SELECT
    ESTADO_PEDIDO,
    COUNT(*) AS TOTAL_PEDIDOS
FROM PEDIDOS
GROUP BY ESTADO_PEDIDO
ORDER BY ESTADO_PEDIDO;
```

**Resultado esperado:** deben existir registros para `CANCELADO`, `COMPLETADO`, `EN_PROCESO` y `ENVIADO`.

### Paso 0.6 — Validar que existen pedidos en todas las categorías de monto

```sql
SELECT
    CASE
        WHEN MONTO_TOTAL < 200  THEN 'Bajo'
        WHEN MONTO_TOTAL < 1000 THEN 'Medio'
        WHEN MONTO_TOTAL < 3000 THEN 'Alto'
        ELSE 'Premium'
    END AS CATEGORIA_MONTO,
    COUNT(*) AS TOTAL_PEDIDOS
FROM PEDIDOS
WHERE ESTADO_PEDIDO != 'CANCELADO'
GROUP BY CATEGORIA_MONTO
ORDER BY
    CASE CATEGORIA_MONTO
        WHEN 'Bajo' THEN 1
        WHEN 'Medio' THEN 2
        WHEN 'Alto' THEN 3
        WHEN 'Premium' THEN 4
    END;
```

**Resultado esperado:** deben aparecer exactamente 4 categorías: `Bajo`, `Medio`, `Alto` y `Premium`.

---

## Ejercicios Paso a Paso

---

### Ejercicio 1 — Exploración de tablas base

**Objetivo:** Familiarizarte con la estructura de `CLIENTES`, `PEDIDOS`, `PRODUCTOS` y `VENTAS` antes de construir las clasificaciones.

#### Paso 1.1 — Explorar estructura de tablas

```sql
DESC TABLE CLIENTES;
DESC TABLE PEDIDOS;
DESC TABLE PRODUCTOS;
DESC TABLE VENTAS;
```

#### Paso 1.2 — Revisar una muestra de clientes

```sql
SELECT
    ID_CLIENTE,
    NOMBRE,
    EMAIL,
    FECHA_REGISTRO,
    CIUDAD,
    PAIS
FROM CLIENTES
ORDER BY ID_CLIENTE
LIMIT 10;
```

#### Paso 1.3 — Revisar una muestra de pedidos

```sql
SELECT
    ID_PEDIDO,
    ID_CLIENTE,
    FECHA_PEDIDO,
    MONTO_TOTAL,
    ESTADO_PEDIDO
FROM PEDIDOS
ORDER BY ID_PEDIDO
LIMIT 10;
```

#### Paso 1.4 — Calcula estadísticas de distribución del monto de pedidos para entender los rangos de datos antes de definir los umbrales de clasificación:

```sql
SELECT
    COUNT(*)                       AS TOTAL_PEDIDOS_VALIDOS,
    ROUND(MIN(MONTO_TOTAL), 2)     AS MONTO_MINIMO,
    ROUND(MAX(MONTO_TOTAL), 2)     AS MONTO_MAXIMO,
    ROUND(AVG(MONTO_TOTAL), 2)     AS MONTO_PROMEDIO,
    ROUND(MEDIAN(MONTO_TOTAL), 2)  AS MONTO_MEDIANA,
    COUNT(DISTINCT ID_CLIENTE)     AS CLIENTES_CON_PEDIDOS_VALIDOS
FROM PEDIDOS
WHERE ESTADO_PEDIDO != 'CANCELADO';
```

#### Paso 1.5 — Calcular frecuencia de compras por cliente

```sql
SELECT
    PEDIDOS_POR_CLIENTE,
    COUNT(*) AS CANTIDAD_CLIENTES
FROM (
    SELECT
        ID_CLIENTE,
        COUNT(*) AS PEDIDOS_POR_CLIENTE
    FROM PEDIDOS
    WHERE ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY ID_CLIENTE
) SUB
GROUP BY PEDIDOS_POR_CLIENTE
ORDER BY PEDIDOS_POR_CLIENTE;
```

**Resultado esperado:** observarás clientes con 1, 2, 3 y 5 pedidos válidos. También habrá clientes sin pedidos válidos, que se analizarán más adelante con `LEFT JOIN`.

#### Resultado esperado

Después de ejecutar las consultas de exploración deberías observar:
- La estructura de columnas de ambas tablas, incluyendo tipos de datos.
- El rango de montos de pedidos (valores mínimo, máximo y promedio).
- La distribución de frecuencia: cuántos clientes tienen 1, 2, 3 o más pedidos.

#### Verificación

Anota los valores obtenidos en la siguiente tabla de referencia (los usarás para definir los umbrales de clasificación en los pasos siguientes):

| Métrica | Valor observado |
|---|---|
| Monto mínimo de pedido | __________ |
| Monto máximo de pedido | __________ |
| Monto promedio de pedido | __________ |
| Total de clientes con pedidos | __________ |

---

### Ejercicio 2 — Clasificación simple con `CASE WHEN` - Segmentación por monto

**Objetivo:** Implementar una clasificación de pedidos usando `CASE WHEN` en forma buscada y simple.

#### Paso 2.1 — Clasificar pedidos por monto

Pregunta de negocio:

- **"¿Qué categoría de valor tiene cada pedido válido?"**

```sql
-- Clasificación de pedidos por monto (CASE WHEN forma buscada)
-- Umbrales: Bajo < 200 | Medio 200-999 | Alto 1000-2999 | Premium >= 3000
SELECT
    ID_PEDIDO,
    ID_CLIENTE,
    FECHA_PEDIDO,
    MONTO_TOTAL,
    CASE
        WHEN MONTO_TOTAL < 200  THEN 'Bajo'
        WHEN MONTO_TOTAL < 1000 THEN 'Medio'
        WHEN MONTO_TOTAL < 3000 THEN 'Alto'
        ELSE 'Premium'
    END AS CATEGORIA_MONTO
FROM PEDIDOS
WHERE ESTADO_PEDIDO != 'CANCELADO'
ORDER BY MONTO_TOTAL DESC;
```

#### Paso 2.2 — Verifica la distribución de categorías para confirmar que los umbrales son razonables:

```sql
SELECT
    CASE
        WHEN MONTO_TOTAL < 200  THEN 'Bajo'
        WHEN MONTO_TOTAL < 1000 THEN 'Medio'
        WHEN MONTO_TOTAL < 3000 THEN 'Alto'
        ELSE 'Premium'
    END AS CATEGORIA_MONTO,
    COUNT(*) AS CANTIDAD_PEDIDOS,
    ROUND(SUM(MONTO_TOTAL), 2) AS MONTO_TOTAL_SEGMENTO,
    ROUND(AVG(MONTO_TOTAL), 2) AS MONTO_PROMEDIO
FROM PEDIDOS
WHERE ESTADO_PEDIDO != 'CANCELADO'
GROUP BY
    CASE
        WHEN MONTO_TOTAL < 200  THEN 'Bajo'
        WHEN MONTO_TOTAL < 1000 THEN 'Medio'
        WHEN MONTO_TOTAL < 3000 THEN 'Alto'
        ELSE 'Premium'
    END
ORDER BY MONTO_PROMEDIO DESC;
```

**Resultado esperado:** deben aparecer exactamente 4 categorías: `Premium`, `Alto`, `Medio` y `Bajo`.

#### Paso 2.3 — Traducir estados con `CASE WHEN` forma simple

Pregunta de negocio:

- **"¿Cómo mostrar los estados técnicos del pedido con una descripción amigable?"**

```sql
SELECT
    ID_PEDIDO,
    ESTADO_PEDIDO,
    CASE ESTADO_PEDIDO
        WHEN 'COMPLETADO' THEN 'Entregado al cliente'
        WHEN 'EN_PROCESO' THEN 'En preparación'
        WHEN 'ENVIADO'    THEN 'En camino'
        WHEN 'CANCELADO'  THEN 'Cancelado por cliente o sistema'
        ELSE 'Estado desconocido: ' || ESTADO_PEDIDO
    END AS DESCRIPCION_ESTADO,
    MONTO_TOTAL
FROM PEDIDOS
ORDER BY ID_PEDIDO
LIMIT 20;
```

> 📌 **Observa el patrón `ELSE 'Estado desconocido: ' || ESTADO_PEDIDO`**: concatena el valor original con un prefijo descriptivo. Esto es útil para detectar valores inesperados sin perder información.

#### Paso 2.4 — Comparar `IFF()` contra `CASE WHEN`

```sql
-- IFF() como alternativa a CASE WHEN de dos ramas (exclusivo de Snowflake)
-- Sintaxis: IFF(condición, valor_si_verdadero, valor_si_falso)

SELECT
    ID_PEDIDO,
    ID_CLIENTE,
    MONTO_TOTAL,
    IFF(MONTO_TOTAL >= 1000, 'Alto valor', 'Valor estándar') AS CLASIFICACION_IFF,
    CASE
        WHEN MONTO_TOTAL >= 1000 THEN 'Alto valor'
        ELSE 'Valor estándar'
    END AS CLASIFICACION_CASE
FROM PEDIDOS
WHERE ESTADO_PEDIDO != 'CANCELADO'
ORDER BY ID_PEDIDO
LIMIT 15;
```

> ⚠️ **Nota sobre portabilidad:** `IFF()` es una función exclusiva de Snowflake. No existe en PostgreSQL, MySQL ni SQL Server. Úsala cuando la simplicidad del código sea prioritaria, pero prefiere `CASE WHEN` cuando necesites portabilidad entre motores SQL.

**Verificación:**

```sql
-- Verificación: confirmar que no hay pedidos sin clasificar (NULL en categoria_monto)
SELECT
    COUNT(*) AS pedidos_sin_clasificar
FROM PEDIDOS
WHERE ESTADO_PEDIDO != 'CANCELADO'
  AND CASE
          WHEN MONTO_TOTAL < 200   THEN 'Bajo'
          WHEN MONTO_TOTAL < 1000  THEN 'Medio'
          WHEN MONTO_TOTAL < 3000  THEN 'Alto'
          ELSE                          'Premium'
      END IS NULL;
-- Resultado esperado: 0
```

**Resultado esperado:** `PEDIDOS_SIN_CLASIFICAR = 0`.

#### Resultado esperado

- La consulta 2.1 debe mostrar cada pedido con su columna `categoria_monto` asignada correctamente.
- La consulta 2.2 debe mostrar exactamente 4 filas (una por categoría), con sus conteos y montos.
- Las columnas `clasificacion_iff` y `clasificacion_case` en la consulta 2.4 deben mostrar valores idénticos.

---

### Ejercicio 3 — Construcción de métricas base por cliente

**Objetivo:** Calcular las métricas agregadas por cliente que servirán como base para la segmentación multinivel del Paso 4.

#### Paso 3.1 — Crea una CTE que calcule las métricas base de cada cliente. Este será el fundamento de toda la segmentación posterior:

```sql
-- Métricas base por cliente: monto total, frecuencia y antigüedad
WITH METRICAS_CLIENTE AS (
    SELECT
        C.ID_CLIENTE,
        C.NOMBRE,
        C.FECHA_REGISTRO,
        DATEDIFF('day', C.FECHA_REGISTRO, CURRENT_DATE()) AS DIAS_COMO_CLIENTE,
        COUNT(P.ID_PEDIDO) AS TOTAL_PEDIDOS,
        ROUND(SUM(P.MONTO_TOTAL), 2) AS MONTO_TOTAL_GASTADO,
        ROUND(AVG(P.MONTO_TOTAL), 2) AS MONTO_PROMEDIO_PEDIDO,
        MAX(P.FECHA_PEDIDO) AS FECHA_ULTIMO_PEDIDO
    FROM CLIENTES C
    LEFT JOIN PEDIDOS P
        ON C.ID_CLIENTE = P.ID_CLIENTE
       AND P.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY
        C.ID_CLIENTE,
        C.NOMBRE,
        C.FECHA_REGISTRO
)
SELECT *
FROM METRICAS_CLIENTE
ORDER BY MONTO_TOTAL_GASTADO DESC NULLS LAST;
```

> Nota: `LEFT JOIN` permite incluir clientes sin pedidos válidos. Para esos clientes, `MONTO_TOTAL_GASTADO` aparecerá como `NULL`. En los siguientes ejercicios se usará `COALESCE` para convertir esos valores a cero.

#### Paso 3.2 — Identificar clientes sin pedidos válidos

```sql
WITH METRICAS_CLIENTE AS (
    SELECT
        C.ID_CLIENTE,
        COUNT(P.ID_PEDIDO) AS TOTAL_PEDIDOS,
        SUM(P.MONTO_TOTAL) AS MONTO_TOTAL_GASTADO
    FROM CLIENTES C
    LEFT JOIN PEDIDOS P
        ON C.ID_CLIENTE = P.ID_CLIENTE
       AND P.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY C.ID_CLIENTE
)
SELECT
    COUNT(*) AS TOTAL_CLIENTES,
    COUNT(CASE WHEN TOTAL_PEDIDOS = 0 THEN 1 END) AS CLIENTES_SIN_PEDIDOS_VALIDOS,
    COUNT(CASE WHEN TOTAL_PEDIDOS > 0 THEN 1 END) AS CLIENTES_CON_PEDIDOS_VALIDOS
FROM METRICAS_CLIENTE;
```

**Resultado esperado:** El total debe ser `12` clientes. Debe haber clientes con pedidos válidos y clientes sin pedidos válidos.

#### Verificación

Confirma que el total de clientes en la CTE coincide con el total de la tabla `CLIENTES`:

```sql
-- La CTE debe retornar exactamente el mismo número de clientes que la tabla base
SELECT COUNT(*) AS total_en_tabla FROM CLIENTES;
-- Compara este número con total_clientes de la consulta 3.2
```

#### Resultado esperado

- La consulta 3.1 muestra una fila por cliente con sus métricas calculadas.
- Los clientes sin pedidos aparecen con `total_pedidos = 0` y `monto_total_gastado = NULL`.
- La consulta 3.2 muestra el desglose entre clientes con y sin historial de compras.

---

### Ejercicio 4 — Clasificación multinivel GOLD / SILVER / BRONZE / NEW

**Objetivo:** Implementar una segmentación de clientes combinando monto total, frecuencia y antigüedad.

#### Reglas de negocio

| Segmento | Regla |
|---|---|
| `GOLD` | Monto total válido >= 5000 y total de pedidos válidos >= 5 |
| `SILVER` | Monto total válido >= 1500 y total de pedidos válidos >= 2 |
| `BRONZE` | Tiene al menos 1 pedido válido, pero no cumple reglas superiores |
| `NEW` | Sin pedidos válidos o cliente registrado hace menos de 30 días |

> Importante: el orden del `CASE WHEN` es crítico. Snowflake evalúa de arriba hacia abajo y devuelve el resultado de la primera condición verdadera.

#### Paso 4.1 — Implementar clasificación principal

```sql
WITH METRICAS_CLIENTE AS (
    SELECT
        C.ID_CLIENTE,
        C.NOMBRE,
        C.FECHA_REGISTRO,
        DATEDIFF('day', C.FECHA_REGISTRO, CURRENT_DATE()) AS DIAS_COMO_CLIENTE,
        COUNT(P.ID_PEDIDO) AS TOTAL_PEDIDOS,
        COALESCE(SUM(P.MONTO_TOTAL), 0) AS MONTO_TOTAL_GASTADO,
        COALESCE(AVG(P.MONTO_TOTAL), 0) AS MONTO_PROMEDIO_PEDIDO,
        MAX(P.FECHA_PEDIDO) AS FECHA_ULTIMO_PEDIDO
    FROM CLIENTES C
    LEFT JOIN PEDIDOS P
        ON C.ID_CLIENTE = P.ID_CLIENTE
       AND P.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY
        C.ID_CLIENTE,
        C.NOMBRE,
        C.FECHA_REGISTRO
),
CLIENTES_SEGMENTADOS AS (
    SELECT
        ID_CLIENTE,
        NOMBRE,
        FECHA_REGISTRO,
        DIAS_COMO_CLIENTE,
        TOTAL_PEDIDOS,
        MONTO_TOTAL_GASTADO,
        MONTO_PROMEDIO_PEDIDO,
        FECHA_ULTIMO_PEDIDO,
        CASE
            WHEN TOTAL_PEDIDOS = 0 OR DIAS_COMO_CLIENTE < 30 THEN 'NEW'
            WHEN MONTO_TOTAL_GASTADO >= 5000 AND TOTAL_PEDIDOS >= 5 THEN 'GOLD'
            WHEN MONTO_TOTAL_GASTADO >= 1500 AND TOTAL_PEDIDOS >= 2 THEN 'SILVER'
            WHEN TOTAL_PEDIDOS >= 1 THEN 'BRONZE'
            ELSE 'NEW'
        END AS SEGMENTO_CLIENTE
    FROM METRICAS_CLIENTE
)
SELECT
    ID_CLIENTE,
    NOMBRE,
    SEGMENTO_CLIENTE,
    TOTAL_PEDIDOS,
    ROUND(MONTO_TOTAL_GASTADO, 2) AS MONTO_TOTAL_GASTADO,
    ROUND(MONTO_PROMEDIO_PEDIDO, 2) AS MONTO_PROMEDIO_PEDIDO,
    DIAS_COMO_CLIENTE,
    FECHA_ULTIMO_PEDIDO
FROM CLIENTES_SEGMENTADOS
ORDER BY
    CASE SEGMENTO_CLIENTE
        WHEN 'GOLD' THEN 1
        WHEN 'SILVER' THEN 2
        WHEN 'BRONZE' THEN 3
        WHEN 'NEW' THEN 4
    END,
    MONTO_TOTAL_GASTADO DESC;
```

> 📌 **Observa el `ORDER BY` con `CASE WHEN`:** Usamos `CASE WHEN` dentro del `ORDER BY` para controlar el orden de los segmentos (GOLD primero, luego SILVER, etc.). Esta es una técnica muy útil para presentar resultados en un orden lógico de negocio en lugar de orden alfabético.

#### Paso 4.2 — Agrega una columna adicional que clasifique también la antigüedad del cliente como una segunda dimensión de análisis:

```sql
WITH METRICAS_CLIENTE AS (
    SELECT
        C.ID_CLIENTE,
        C.NOMBRE,
        C.FECHA_REGISTRO,
        DATEDIFF('day', C.FECHA_REGISTRO, CURRENT_DATE()) AS DIAS_COMO_CLIENTE,
        COUNT(P.ID_PEDIDO) AS TOTAL_PEDIDOS,
        COALESCE(SUM(P.MONTO_TOTAL), 0) AS MONTO_TOTAL_GASTADO,
        COALESCE(AVG(P.MONTO_TOTAL), 0) AS MONTO_PROMEDIO_PEDIDO
    FROM CLIENTES C
    LEFT JOIN PEDIDOS P
        ON C.ID_CLIENTE = P.ID_CLIENTE
       AND P.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY
        C.ID_CLIENTE,
        C.NOMBRE,
        C.FECHA_REGISTRO
)
SELECT
    ID_CLIENTE,
    NOMBRE,
    TOTAL_PEDIDOS,
    ROUND(MONTO_TOTAL_GASTADO, 2) AS MONTO_TOTAL_GASTADO,
    DIAS_COMO_CLIENTE,
    CASE
        WHEN TOTAL_PEDIDOS = 0 OR DIAS_COMO_CLIENTE < 30 THEN 'NEW'
        WHEN MONTO_TOTAL_GASTADO >= 5000 AND TOTAL_PEDIDOS >= 5 THEN 'GOLD'
        WHEN MONTO_TOTAL_GASTADO >= 1500 AND TOTAL_PEDIDOS >= 2 THEN 'SILVER'
        WHEN TOTAL_PEDIDOS >= 1 THEN 'BRONZE'
        ELSE 'NEW'
    END AS SEGMENTO_VALOR,
    CASE
        WHEN DIAS_COMO_CLIENTE < 30 THEN 'Nuevo (< 1 mes)'
        WHEN DIAS_COMO_CLIENTE BETWEEN 30 AND 179 THEN 'Reciente (1-6 meses)'
        WHEN DIAS_COMO_CLIENTE BETWEEN 180 AND 364 THEN 'Establecido (6-12 meses)'
        WHEN DIAS_COMO_CLIENTE >= 365 THEN 'Veterano (> 1 año)'
        ELSE 'Sin clasificar'
    END AS SEGMENTO_ANTIGUEDAD,
    IFF(MONTO_TOTAL_GASTADO >= 3000, 'Sí', 'No') AS ES_ALTO_VALOR
FROM METRICAS_CLIENTE
ORDER BY MONTO_TOTAL_GASTADO DESC;
```

#### Paso 4.3 — Analiza la combinación de segmentos para identificar patrones

```sql
WITH METRICAS_CLIENTE AS (
    SELECT
        C.ID_CLIENTE,
        DATEDIFF('day', C.FECHA_REGISTRO, CURRENT_DATE()) AS DIAS_COMO_CLIENTE,
        COUNT(P.ID_PEDIDO) AS TOTAL_PEDIDOS,
        COALESCE(SUM(P.MONTO_TOTAL), 0) AS MONTO_TOTAL_GASTADO
    FROM CLIENTES C
    LEFT JOIN PEDIDOS P
        ON C.ID_CLIENTE = P.ID_CLIENTE
       AND P.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY C.ID_CLIENTE, C.FECHA_REGISTRO
),
CLIENTES_SEGMENTADOS AS (
    SELECT
        CASE
            WHEN TOTAL_PEDIDOS = 0 OR DIAS_COMO_CLIENTE < 30 THEN 'NEW'
            WHEN MONTO_TOTAL_GASTADO >= 5000 AND TOTAL_PEDIDOS >= 5 THEN 'GOLD'
            WHEN MONTO_TOTAL_GASTADO >= 1500 AND TOTAL_PEDIDOS >= 2 THEN 'SILVER'
            WHEN TOTAL_PEDIDOS >= 1 THEN 'BRONZE'
            ELSE 'NEW'
        END AS SEGMENTO_VALOR,
        CASE
            WHEN DIAS_COMO_CLIENTE < 30 THEN 'Nuevo'
            WHEN DIAS_COMO_CLIENTE BETWEEN 30 AND 179 THEN 'Reciente'
            WHEN DIAS_COMO_CLIENTE BETWEEN 180 AND 364 THEN 'Establecido'
            WHEN DIAS_COMO_CLIENTE >= 365 THEN 'Veterano'
            ELSE 'Sin clasificar'
        END AS SEGMENTO_ANTIGUEDAD
    FROM METRICAS_CLIENTE
)
SELECT
    SEGMENTO_VALOR,
    SEGMENTO_ANTIGUEDAD,
    COUNT(*) AS CANTIDAD_CLIENTES
FROM CLIENTES_SEGMENTADOS
GROUP BY SEGMENTO_VALOR, SEGMENTO_ANTIGUEDAD
ORDER BY
    CASE SEGMENTO_VALOR
        WHEN 'GOLD' THEN 1
        WHEN 'SILVER' THEN 2
        WHEN 'BRONZE' THEN 3
        WHEN 'NEW' THEN 4
    END,
    SEGMENTO_ANTIGUEDAD;
```

**Verificación:**

```sql
-- Verificar que TODOS los clientes tienen un segmento asignado (no NULL)
WITH METRICAS_CLIENTE AS (
    SELECT
        C.ID_CLIENTE,
        DATEDIFF('day', C.FECHA_REGISTRO, CURRENT_DATE()) AS DIAS_COMO_CLIENTE,
        COUNT(P.ID_PEDIDO) AS TOTAL_PEDIDOS,
        COALESCE(SUM(P.MONTO_TOTAL), 0) AS MONTO_TOTAL_GASTADO
    FROM CLIENTES C
    LEFT JOIN PEDIDOS P
        ON C.ID_CLIENTE = P.ID_CLIENTE
       AND P.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY C.ID_CLIENTE, C.FECHA_REGISTRO
)
SELECT
    COUNT(*) AS TOTAL_CLIENTES,
    COUNT(CASE
        WHEN TOTAL_PEDIDOS = 0 OR DIAS_COMO_CLIENTE < 30 THEN 'NEW'
        WHEN MONTO_TOTAL_GASTADO >= 5000 AND TOTAL_PEDIDOS >= 5 THEN 'GOLD'
        WHEN MONTO_TOTAL_GASTADO >= 1500 AND TOTAL_PEDIDOS >= 2 THEN 'SILVER'
        WHEN TOTAL_PEDIDOS >= 1 THEN 'BRONZE'
        ELSE 'NEW'
    END) AS CLIENTES_CON_SEGMENTO
FROM METRICAS_CLIENTE;
-- total_clientes debe ser igual a clientes_con_segmento
```

**Resultado esperado:** `TOTAL_CLIENTES = CLIENTES_CON_SEGMENTO`.

#### Resultado esperado

- La consulta 4.1 muestra todos los clientes con su segmento asignado, ordenados de GOLD a NEW.
- La consulta 4.2 agrega dos dimensiones de clasificación por cliente: segmento de valor y segmento de antigüedad.
- La consulta 4.3 muestra una tabla cruzada con la distribución de clientes por combinación de segmentos.

---

### Ejercicio 5 — Resumen ejecutivo por segmento

**Objetivo:** Generar una tabla resumen con métricas clave por segmento usando CASE WHEN dentro de funciones de agregación, produciendo el reporte ejecutivo que el equipo de marketing necesita.

#### Paso 5.1 — Construye el reporte ejecutivo completo por segmento usando CASE WHEN dentro de SUM() y COUNT():

```sql
WITH METRICAS_CLIENTE AS (
    SELECT
        C.ID_CLIENTE,
        C.NOMBRE,
        C.FECHA_REGISTRO,
        DATEDIFF('day', C.FECHA_REGISTRO, CURRENT_DATE()) AS DIAS_COMO_CLIENTE,
        COUNT(P.ID_PEDIDO) AS TOTAL_PEDIDOS,
        COALESCE(SUM(P.MONTO_TOTAL), 0) AS MONTO_TOTAL_GASTADO,
        COALESCE(AVG(P.MONTO_TOTAL), 0) AS MONTO_PROMEDIO_PEDIDO
    FROM CLIENTES C
    LEFT JOIN PEDIDOS P
        ON C.ID_CLIENTE = P.ID_CLIENTE
       AND P.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY
        C.ID_CLIENTE,
        C.NOMBRE,
        C.FECHA_REGISTRO
),
CLIENTES_SEGMENTADOS AS (
    SELECT
        ID_CLIENTE,
        NOMBRE,
        TOTAL_PEDIDOS,
        MONTO_TOTAL_GASTADO,
        MONTO_PROMEDIO_PEDIDO,
        DIAS_COMO_CLIENTE,
        CASE
            WHEN TOTAL_PEDIDOS = 0 OR DIAS_COMO_CLIENTE < 30 THEN 'NEW'
            WHEN MONTO_TOTAL_GASTADO >= 5000 AND TOTAL_PEDIDOS >= 5 THEN 'GOLD'
            WHEN MONTO_TOTAL_GASTADO >= 1500 AND TOTAL_PEDIDOS >= 2 THEN 'SILVER'
            WHEN TOTAL_PEDIDOS >= 1 THEN 'BRONZE'
            ELSE 'NEW'
        END AS SEGMENTO_CLIENTE
    FROM METRICAS_CLIENTE
)
-- Reporte final: métricas agregadas por segmento
SELECT
    SEGMENTO_CLIENTE,
    COUNT(*) AS CANTIDAD_CLIENTES,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 1) AS PCT_CLIENTES,
    ROUND(SUM(MONTO_TOTAL_GASTADO), 2) AS REVENUE_TOTAL_SEGMENTO,
    ROUND(AVG(MONTO_TOTAL_GASTADO), 2) AS GASTO_PROMEDIO_CLIENTE,
    ROUND(MAX(MONTO_TOTAL_GASTADO), 2) AS GASTO_MAXIMO_CLIENTE,
    ROUND(AVG(TOTAL_PEDIDOS), 1) AS PEDIDOS_PROMEDIO,
    ROUND(AVG(DIAS_COMO_CLIENTE), 0) AS ANTIGUEDAD_PROMEDIO_DIAS,
    ROUND(
        SUM(MONTO_TOTAL_GASTADO) * 100.0 /
        NULLIF(SUM(SUM(MONTO_TOTAL_GASTADO)) OVER(), 0),
        1
    ) AS PCT_REVENUE_TOTAL
FROM CLIENTES_SEGMENTADOS
GROUP BY SEGMENTO_CLIENTE
ORDER BY
    CASE SEGMENTO_CLIENTE
        WHEN 'GOLD' THEN 1
        WHEN 'SILVER' THEN 2
        WHEN 'BRONZE' THEN 3
        WHEN 'NEW' THEN 4
    END;
```

> 📌 **Nota sobre `SUM(COUNT(*)) OVER()`:** Esta es una window function básica que calcula el total general para calcular porcentajes. Las window functions se estudiarán en profundidad en el Laboratorio 4; por ahora, úsala como está sin modificarla.

#### Paso 5.2 — Genera un reporte de métricas condicionales usando CASE WHEN dentro de SUM() — el patrón de "pivot condicional":

```sql
-- Pivot condicional: métricas de revenue por segmento en columnas separadas
-- Patrón: SUM(CASE WHEN condición THEN valor ELSE 0 END)
WITH METRICAS_CLIENTE AS (
    SELECT
        C.ID_CLIENTE,
        DATEDIFF('day', C.FECHA_REGISTRO, CURRENT_DATE()) AS DIAS_COMO_CLIENTE,
        COUNT(P.ID_PEDIDO) AS TOTAL_PEDIDOS,
        COALESCE(SUM(P.MONTO_TOTAL), 0) AS MONTO_TOTAL_GASTADO
    FROM CLIENTES C
    LEFT JOIN PEDIDOS P
        ON C.ID_CLIENTE = P.ID_CLIENTE
       AND P.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY C.ID_CLIENTE, C.FECHA_REGISTRO
),
CLIENTES_SEGMENTADOS AS (
    SELECT
        ID_CLIENTE,
        MONTO_TOTAL_GASTADO,
        TOTAL_PEDIDOS,
        CASE
            WHEN TOTAL_PEDIDOS = 0 OR DIAS_COMO_CLIENTE < 30 THEN 'NEW'
            WHEN MONTO_TOTAL_GASTADO >= 5000 AND TOTAL_PEDIDOS >= 5 THEN 'GOLD'
            WHEN MONTO_TOTAL_GASTADO >= 1500 AND TOTAL_PEDIDOS >= 2 THEN 'SILVER'
            WHEN TOTAL_PEDIDOS >= 1 THEN 'BRONZE'
            ELSE 'NEW'
        END AS SEGMENTO_CLIENTE
    FROM METRICAS_CLIENTE
)
-- Resumen global con columnas por segmento (pivot condicional)
SELECT
    COUNT(*) AS TOTAL_CLIENTES,
    COUNT(CASE WHEN SEGMENTO_CLIENTE = 'GOLD' THEN 1 END) AS CLIENTES_GOLD,
    COUNT(CASE WHEN SEGMENTO_CLIENTE = 'SILVER' THEN 1 END) AS CLIENTES_SILVER,
    COUNT(CASE WHEN SEGMENTO_CLIENTE = 'BRONZE' THEN 1 END) AS CLIENTES_BRONZE,
    COUNT(CASE WHEN SEGMENTO_CLIENTE = 'NEW' THEN 1 END) AS CLIENTES_NEW,
    ROUND(SUM(CASE WHEN SEGMENTO_CLIENTE = 'GOLD' THEN MONTO_TOTAL_GASTADO ELSE 0 END), 2) AS REVENUE_GOLD,
    ROUND(SUM(CASE WHEN SEGMENTO_CLIENTE = 'SILVER' THEN MONTO_TOTAL_GASTADO ELSE 0 END), 2) AS REVENUE_SILVER,
    ROUND(SUM(CASE WHEN SEGMENTO_CLIENTE = 'BRONZE' THEN MONTO_TOTAL_GASTADO ELSE 0 END), 2) AS REVENUE_BRONZE,
    ROUND(SUM(CASE WHEN SEGMENTO_CLIENTE = 'NEW' THEN MONTO_TOTAL_GASTADO ELSE 0 END), 2) AS REVENUE_NEW,
    ROUND(SUM(MONTO_TOTAL_GASTADO), 2) AS REVENUE_TOTAL
FROM CLIENTES_SEGMENTADOS;
```

**Resultado esperado:** una sola fila con conteos y revenue por segmento. La suma de clientes por segmento debe igualar `TOTAL_CLIENTES`.

#### Verificación

```sql
-- Verificación de consistencia: la suma de revenue por segmento debe igualar el total
WITH metricas_cliente AS (
    SELECT
        c.ID_CLIENTE,
        DATEDIFF('day', c.FECHA_REGISTRO, CURRENT_DATE())    AS dias_como_cliente,
        COUNT(p.ID_PEDIDO)                                    AS total_pedidos,
        COALESCE(SUM(p.MONTO_TOTAL), 0)                      AS monto_total_gastado
    FROM CLIENTES c
    LEFT JOIN PEDIDOS p
           ON c.ID_CLIENTE = p.ID_CLIENTE
          AND p.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY c.ID_CLIENTE, c.FECHA_REGISTRO
)
SELECT
    ROUND(SUM(monto_total_gastado), 2) AS revenue_directo_de_tabla
FROM metricas_cliente;
-- Este valor debe coincidir con revenue_total de la consulta 5.2
```

#### Resultado esperado

- La consulta 5.1 debe producir exactamente 4 filas (una por segmento: GOLD, SILVER, BRONZE, NEW) con todas las métricas calculadas.
- La consulta 5.2 debe producir exactamente **1 fila** con columnas separadas para cada segmento — el patrón pivot condicional.
- La suma de `clientes_gold + clientes_silver + clientes_bronze + clientes_new` debe ser igual a `total_clientes`.
- La suma de `revenue_gold + revenue_silver + revenue_bronze` debe ser igual a `revenue_total`.

---

### Ejercicio 6 — Dataset enriquecido para análisis posterior

**Objetivo:** Producir el dataset final enriquecido con todas las clasificaciones, listo para ser consumido por reportes o análisis posteriores, usando una estructura de CTEs encadenadas aprendida en el Laboratorio 1.

#### Paso 6.1 — Construye la consulta final que integra todo lo aprendido en este laboratorio: métricas base, clasificación multinivel, segmentación de antigüedad y resumen ejecutivo en una estructura de CTEs encadenadas:

```sql
-- ============================================================
-- CONSULTA FINAL DEL LABORATORIO
-- Dataset enriquecido de clientes con segmentación completa
-- Estructura: 3 CTEs encadenadas + SELECT final
-- ============================================================

-- CTE 1: Métricas base por cliente
WITH metricas_base AS (
    SELECT
        c.ID_CLIENTE,
        c.NOMBRE,
        c.EMAIL,
        c.CIUDAD,
        c.PAIS,
        c.FECHA_REGISTRO,
        DATEDIFF('day', c.FECHA_REGISTRO, CURRENT_DATE())    AS dias_como_cliente,
        COUNT(p.ID_PEDIDO)                                    AS total_pedidos,
        COALESCE(COUNT(p.ID_PEDIDO), 0)                      AS pedidos_completados,
        COALESCE(SUM(p.MONTO_TOTAL), 0)                      AS monto_total_gastado,
        COALESCE(AVG(p.MONTO_TOTAL), 0)                      AS monto_promedio_pedido,
        COALESCE(MAX(p.MONTO_TOTAL), 0)                      AS pedido_maximo,
        MAX(p.FECHA_PEDIDO)                                   AS fecha_ultimo_pedido,
        DATEDIFF('day',
                 MAX(p.FECHA_PEDIDO),
                 CURRENT_DATE())                              AS dias_desde_ultimo_pedido
    FROM CLIENTES c
    LEFT JOIN PEDIDOS p
           ON c.ID_CLIENTE = p.ID_CLIENTE
          AND p.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY
        c.ID_CLIENTE, c.NOMBRE, c.EMAIL,
        c.CIUDAD, c.PAIS, c.FECHA_REGISTRO
),

-- CTE 2: Aplicar clasificaciones con CASE WHEN
clientes_clasificados AS (
    SELECT
        ID_CLIENTE,
        NOMBRE,
        EMAIL,
        CIUDAD,
        PAIS,
        FECHA_REGISTRO,
        dias_como_cliente,
        total_pedidos,
        monto_total_gastado,
        monto_promedio_pedido,
        pedido_maximo,
        fecha_ultimo_pedido,
        dias_desde_ultimo_pedido,
        -- Clasificación principal: segmento de valor
        CASE
            WHEN total_pedidos = 0
              OR dias_como_cliente < 30                       THEN 'NEW'
            WHEN monto_total_gastado >= 5000
             AND total_pedidos >= 5                           THEN 'GOLD'
            WHEN monto_total_gastado >= 1500
             AND total_pedidos >= 2                           THEN 'SILVER'
            WHEN total_pedidos >= 1                           THEN 'BRONZE'
            ELSE                                                   'NEW'
        END AS segmento_valor,
        -- Clasificación secundaria: segmento de antigüedad
        CASE
            WHEN dias_como_cliente < 30                       THEN 'Nuevo'
            WHEN dias_como_cliente BETWEEN 30 AND 179         THEN 'Reciente'
            WHEN dias_como_cliente BETWEEN 180 AND 364        THEN 'Establecido'
            WHEN dias_como_cliente >= 365                     THEN 'Veterano'
            ELSE                                                   'Indefinido'
        END AS segmento_antiguedad,
        -- Clasificación de riesgo de abandono (churn risk)
        CASE
            WHEN fecha_ultimo_pedido IS NULL                  THEN 'Sin compras'
            WHEN dias_desde_ultimo_pedido > 180               THEN 'Riesgo alto'
            WHEN dias_desde_ultimo_pedido BETWEEN 90 AND 180  THEN 'Riesgo medio'
            WHEN dias_desde_ultimo_pedido < 90                THEN 'Activo'
            ELSE                                                   'Indefinido'
        END AS riesgo_abandono,
        -- Indicador de cliente premium con IFF
        IFF(monto_total_gastado >= 3000
            AND total_pedidos >= 3,
            'Premium', 'Estándar')                            AS tier_cliente
    FROM metricas_base
),

-- CTE 3: Agregar prioridad de acción de marketing
clientes_con_prioridad AS (
    SELECT
        *,
        -- Prioridad de acción combinando segmento y riesgo
        CASE
            WHEN segmento_valor = 'GOLD'
             AND riesgo_abandono = 'Riesgo alto'              THEN '1 - Retención urgente GOLD'
            WHEN segmento_valor = 'GOLD'                      THEN '2 - Fidelización GOLD'
            WHEN segmento_valor = 'SILVER'
             AND riesgo_abandono IN ('Riesgo alto',
                                     'Riesgo medio')          THEN '3 - Retención SILVER'
            WHEN segmento_valor = 'SILVER'                    THEN '4 - Upgrade a GOLD'
            WHEN segmento_valor = 'BRONZE'                    THEN '5 - Desarrollo BRONZE'
            WHEN segmento_valor = 'NEW'                       THEN '6 - Activación NEW'
            ELSE                                                   '7 - Revisar manualmente'
        END AS prioridad_accion_marketing
    FROM clientes_clasificados
)

-- SELECT FINAL: Dataset enriquecido completo
SELECT
    ID_CLIENTE,
    NOMBRE,
    EMAIL,
    CIUDAD,
    PAIS,
    segmento_valor,
    segmento_antiguedad,
    riesgo_abandono,
    tier_cliente,
    prioridad_accion_marketing,
    total_pedidos,
    ROUND(monto_total_gastado, 2)           AS monto_total_gastado,
    ROUND(monto_promedio_pedido, 2)         AS monto_promedio_pedido,
    dias_como_cliente,
    dias_desde_ultimo_pedido,
    fecha_ultimo_pedido,
    FECHA_REGISTRO
FROM clientes_con_prioridad
ORDER BY prioridad_accion_marketing, monto_total_gastado DESC;
```

#### Paso 6.2 — Resumen ejecutivo del dataset enriquecido

```sql
-- Resumen ejecutivo: distribución de clientes por prioridad de acción
WITH metricas_base AS (
    SELECT
        c.ID_CLIENTE,
        c.FECHA_REGISTRO,
        DATEDIFF('day', c.FECHA_REGISTRO, CURRENT_DATE())    AS dias_como_cliente,
        COUNT(p.ID_PEDIDO)                                    AS total_pedidos,
        COALESCE(SUM(p.MONTO_TOTAL), 0)                      AS monto_total_gastado,
        MAX(p.FECHA_PEDIDO)                                   AS fecha_ultimo_pedido,
        DATEDIFF('day', MAX(p.FECHA_PEDIDO), CURRENT_DATE()) AS dias_desde_ultimo_pedido
    FROM CLIENTES c
    LEFT JOIN PEDIDOS p
           ON c.ID_CLIENTE = p.ID_CLIENTE
          AND p.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY c.ID_CLIENTE, c.FECHA_REGISTRO
),
clientes_clasificados AS (
    SELECT
        CASE
            WHEN total_pedidos = 0 OR dias_como_cliente < 30  THEN 'NEW'
            WHEN monto_total_gastado >= 5000 AND total_pedidos >= 5 THEN 'GOLD'
            WHEN monto_total_gastado >= 1500 AND total_pedidos >= 2 THEN 'SILVER'
            WHEN total_pedidos >= 1 THEN 'BRONZE'
            ELSE 'NEW'
        END AS segmento_valor,
        CASE
            WHEN fecha_ultimo_pedido IS NULL               THEN 'Sin compras'
            WHEN dias_desde_ultimo_pedido > 180            THEN 'Riesgo alto'
            WHEN dias_desde_ultimo_pedido BETWEEN 90 AND 180 THEN 'Riesgo medio'
            ELSE 'Activo'
        END AS riesgo_abandono,
        monto_total_gastado
    FROM metricas_base
)
SELECT
    segmento_valor,
    COUNT(*)                                                   AS total_clientes,
    COUNT(CASE WHEN riesgo_abandono = 'Activo'      THEN 1 END) AS activos,
    COUNT(CASE WHEN riesgo_abandono = 'Riesgo medio' THEN 1 END) AS riesgo_medio,
    COUNT(CASE WHEN riesgo_abandono = 'Riesgo alto'  THEN 1 END) AS riesgo_alto,
    COUNT(CASE WHEN riesgo_abandono = 'Sin compras'  THEN 1 END) AS sin_compras,
    ROUND(SUM(monto_total_gastado), 2)                          AS revenue_segmento
FROM clientes_clasificados
GROUP BY segmento_valor
ORDER BY
    CASE segmento_valor
        WHEN 'GOLD' THEN 1 WHEN 'SILVER' THEN 2
        WHEN 'BRONZE' THEN 3 WHEN 'NEW' THEN 4
    END;
```

---

## Validación y Pruebas

Ejecuta las siguientes consultas para confirmar que el laboratorio quedó correcto.

### Validación 1 — Conteo total de clientes

```sql
SELECT
    'Clientes en tabla base' AS VERIFICACION,
    COUNT(*) AS VALOR
FROM CLIENTES
UNION ALL
SELECT
    'Clientes en métricas CTE' AS VERIFICACION,
    COUNT(DISTINCT C.ID_CLIENTE) AS VALOR
FROM CLIENTES C
LEFT JOIN PEDIDOS P
    ON C.ID_CLIENTE = P.ID_CLIENTE;
```

**Resultado esperado:** ambos valores deben ser `12`.

### Validación 2 — Ningún cliente queda sin segmento

```sql
WITH METRICAS AS (
    SELECT
        C.ID_CLIENTE,
        DATEDIFF('day', C.FECHA_REGISTRO, CURRENT_DATE()) AS DIAS_CLIENTE,
        COUNT(P.ID_PEDIDO) AS N_PEDIDOS,
        COALESCE(SUM(P.MONTO_TOTAL), 0) AS MONTO
    FROM CLIENTES C
    LEFT JOIN PEDIDOS P
        ON C.ID_CLIENTE = P.ID_CLIENTE
       AND P.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY C.ID_CLIENTE, C.FECHA_REGISTRO
)
SELECT
    COUNT(*) AS TOTAL_CLIENTES,
    COUNT(CASE
        WHEN N_PEDIDOS = 0 OR DIAS_CLIENTE < 30 THEN 'NEW'
        WHEN MONTO >= 5000 AND N_PEDIDOS >= 5 THEN 'GOLD'
        WHEN MONTO >= 1500 AND N_PEDIDOS >= 2 THEN 'SILVER'
        WHEN N_PEDIDOS >= 1 THEN 'BRONZE'
        ELSE 'NEW'
    END) AS CLIENTES_CON_SEGMENTO
FROM METRICAS;
```

**Resultado esperado:** `TOTAL_CLIENTES = CLIENTES_CON_SEGMENTO`.

### Validación 3 — Suma de clientes por segmento

```sql
WITH METRICAS AS (
    SELECT
        C.ID_CLIENTE,
        DATEDIFF('day', C.FECHA_REGISTRO, CURRENT_DATE()) AS DIAS_CLIENTE,
        COUNT(P.ID_PEDIDO) AS N_PEDIDOS,
        COALESCE(SUM(P.MONTO_TOTAL), 0) AS MONTO
    FROM CLIENTES C
    LEFT JOIN PEDIDOS P
        ON C.ID_CLIENTE = P.ID_CLIENTE
       AND P.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY C.ID_CLIENTE, C.FECHA_REGISTRO
),
SEGMENTADOS AS (
    SELECT
        CASE
            WHEN N_PEDIDOS = 0 OR DIAS_CLIENTE < 30 THEN 'NEW'
            WHEN MONTO >= 5000 AND N_PEDIDOS >= 5 THEN 'GOLD'
            WHEN MONTO >= 1500 AND N_PEDIDOS >= 2 THEN 'SILVER'
            WHEN N_PEDIDOS >= 1 THEN 'BRONZE'
            ELSE 'NEW'
        END AS SEGMENTO
    FROM METRICAS
)
SELECT
    COUNT(*) AS TOTAL_GENERAL,
    COUNT(CASE WHEN SEGMENTO = 'GOLD' THEN 1 END) AS GOLD,
    COUNT(CASE WHEN SEGMENTO = 'SILVER' THEN 1 END) AS SILVER,
    COUNT(CASE WHEN SEGMENTO = 'BRONZE' THEN 1 END) AS BRONZE,
    COUNT(CASE WHEN SEGMENTO = 'NEW' THEN 1 END) AS NEW_CLIENTES,
    COUNT(CASE WHEN SEGMENTO IS NULL THEN 1 END) AS SIN_SEGMENTO
FROM SEGMENTADOS;
```

**Resultado esperado:** `SIN_SEGMENTO = 0` y la suma de `GOLD + SILVER + BRONZE + NEW_CLIENTES` debe ser igual a `TOTAL_GENERAL`.

### Validación 4 — Revenue por segmento contra revenue total

```sql
WITH METRICAS AS (
    SELECT
        C.ID_CLIENTE,
        DATEDIFF('day', C.FECHA_REGISTRO, CURRENT_DATE()) AS DIAS_CLIENTE,
        COUNT(P.ID_PEDIDO) AS N_PEDIDOS,
        COALESCE(SUM(P.MONTO_TOTAL), 0) AS MONTO
    FROM CLIENTES C
    LEFT JOIN PEDIDOS P
        ON C.ID_CLIENTE = P.ID_CLIENTE
       AND P.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY C.ID_CLIENTE, C.FECHA_REGISTRO
)
SELECT
    ROUND(SUM(MONTO), 2) AS REVENUE_DIRECTO_DE_METRICAS
FROM METRICAS;
```

Este valor debe coincidir con `REVENUE_TOTAL` del Paso 5.2.

---

## Resultados esperados clave con el dataset cargado

| Consulta / Ejercicio | Resultado esperado |
|---|---|
| Conteo de `CLIENTES` | 12 filas |
| Conteo de `PRODUCTOS` | 12 filas |
| Conteo de `PEDIDOS` | 23 filas |
| Conteo de `VENTAS` | 23 filas |
| Categorías de monto | 4 categorías: `Bajo`, `Medio`, `Alto`, `Premium` |
| Estados de pedido | `CANCELADO`, `COMPLETADO`, `EN_PROCESO`, `ENVIADO` |
| Segmentos de cliente | `GOLD`, `SILVER`, `BRONZE`, `NEW` |
| Clientes `GOLD` | 2 |
| Clientes `SILVER` | 3 |
| Clientes `BRONZE` | 3 |
| Clientes `NEW` | 4 |
| Clientes sin pedidos válidos | Al menos 2 |
| Clientes con riesgo de abandono | Deben existir casos `Activo`, `Riesgo medio`, `Riesgo alto` y `Sin compras` |

---

## Solución de Problemas

### Problema 1 — Error: `Object 'CLIENTES' does not exist or not authorized`

**Síntoma:** Snowflake devuelve error indicando que la tabla no existe o no tienes permisos.

**Causa probable:** no se ejecutó el script de setup, no estás en el database/schema correcto o el rol activo no tiene permisos.

**Solución:**

```sql
USE WAREHOUSE COMPUTE_WH;
USE DATABASE LAB_SQL_INTERMEDIO;
USE SCHEMA VENTAS;

SHOW TABLES;
```

Si las tablas no aparecen, vuelve a ejecutar el script **`03_SETUP_DATOS_CASE_WHEN_SEGMENTACION`**.

---

### Problema 2 — La clasificación `CASE WHEN` asigna segmentos incorrectos

**Síntoma:** clientes con alto monto aparecen como `BRONZE` o clientes nuevos aparecen como `GOLD`.

**Causa probable:** el orden de condiciones del `CASE WHEN` fue modificado.

**Solución:** respeta el orden lógico:

```sql
CASE
    WHEN TOTAL_PEDIDOS = 0 OR DIAS_COMO_CLIENTE < 30 THEN 'NEW'
    WHEN MONTO_TOTAL_GASTADO >= 5000 AND TOTAL_PEDIDOS >= 5 THEN 'GOLD'
    WHEN MONTO_TOTAL_GASTADO >= 1500 AND TOTAL_PEDIDOS >= 2 THEN 'SILVER'
    WHEN TOTAL_PEDIDOS >= 1 THEN 'BRONZE'
    ELSE 'NEW'
END
```

---

### Problema 3 — Los clientes sin compras aparecen con montos `NULL`

**Síntoma:** algunos clientes no se clasifican correctamente porque su monto total aparece como `NULL`.

**Causa probable:** se usó `SUM(P.MONTO_TOTAL)` sin `COALESCE`.

**Solución:**

```sql
COALESCE(SUM(P.MONTO_TOTAL), 0) AS MONTO_TOTAL_GASTADO
```

Esto convierte los `NULL` en `0` para poder comparar correctamente.

---

### Problema 4 — El reporte de riesgo de abandono no muestra todas las categorías

**Síntoma:** no aparecen `Activo`, `Riesgo medio`, `Riesgo alto` o `Sin compras`.

**Causa probable:** el dataset fue modificado o las fechas fueron reemplazadas por fechas fijas.

**Solución:** usa el dataset incluido en esta práctica. Las fechas se generan con `DATEADD` relativo a `CURRENT_DATE()` para que los rangos de riesgo funcionen sin importar cuándo se ejecute la práctica.

---

## Limpieza del entorno

Al finalizar el laboratorio, ejecuta:

```sql
ALTER WAREHOUSE COMPUTE_WH SUSPEND;
```

> Importante: suspender el warehouse evita consumo innecesario de créditos.

No es necesario eliminar tablas ni datos. El schema `LAB_SQL_INTERMEDIO.VENTAS` puede reutilizarse en prácticas posteriores.

---

## Resumen

En este laboratorio implementaste un sistema completo de clasificación de clientes usando reglas de negocio con `CASE WHEN`.

| Concepto practicado | Ejercicio | Resultado clave |
|---|---|---|
| `CASE WHEN` forma buscada | Ejercicio 2 | Clasificación de pedidos por monto |
| `CASE WHEN` forma simple | Ejercicio 2 | Traducción de estados de pedido |
| `IFF()` | Ejercicio 2 y 6 | Clasificación binaria simplificada |
| `LEFT JOIN` + `COALESCE` | Ejercicio 3 | Inclusión de clientes sin pedidos |
| Clasificación multinivel | Ejercicio 4 | Segmentos `GOLD`, `SILVER`, `BRONZE`, `NEW` |
| `CASE WHEN` en agregaciones | Ejercicio 5 | Resumen ejecutivo y pivot condicional |
| CTEs encadenadas | Ejercicio 6 | Dataset enriquecido para análisis posterior |

### Conclusiones principales

1. **`CASE WHEN` permite transformar reglas de negocio en columnas analíticas reutilizables.**
2. **El orden de las condiciones importa:** la primera condición verdadera es la que define el resultado.
3. **`COALESCE` es esencial cuando se trabaja con `LEFT JOIN` y clientes sin transacciones.**
4. **`IFF()` simplifica clasificaciones binarias, pero `CASE WHEN` es más flexible y portable.**
5. **Las CTEs ayudan a separar métricas base, reglas de clasificación y presentación final.**

### Próximos pasos

En la siguiente práctica se puede reutilizar este dataset enriquecido para trabajar detección de duplicados, reglas de calidad de datos o primeras window functions como `ROW_NUMBER()`.

### Recursos adicionales

| Recurso | URL |
|---|---|
| Documentación Snowflake: CASE | https://docs.snowflake.com/en/sql-reference/functions/case |
| Documentación Snowflake: IFF | https://docs.snowflake.com/en/sql-reference/functions/iff |
| Documentación Snowflake: COALESCE | https://docs.snowflake.com/en/sql-reference/functions/coalesce |
| Documentación Snowflake: WITH / CTE | https://docs.snowflake.com/en/sql-reference/constructs/with |

---
