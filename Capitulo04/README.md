# Análisis de rankings y secuencias con window functions

## Metadatos

| Campo | Detalle |
|---|---|
| **Duración estimada** | 60 minutos |
| **Complejidad** | Alta (*Hard*) |
| **Nivel Bloom** | Aplicar (*Apply*) |
| **Módulo** | 4 — Funciones Analíticas en Snowflake |
| **Plataforma** | Snowflake (Snowsight Worksheet) |
| **Schema de práctica** | `LAB_SQL_INTERMEDIO.VENTAS` |

---

## Descripción General

En este laboratorio aplicarás las funciones analíticas (*window functions*) más importantes de Snowflake sobre datos de ventas y pedidos. Trabajarás con `RANK`, `DENSE_RANK` y `ROW_NUMBER` para construir rankings de productos y vendedores por región, observando cómo cada función trata los empates de manera diferente. Luego usarás `LAG` y `LEAD` para comparar ventas entre períodos consecutivos, y finalizarás construyendo acumulados y promedios móviles con `SUM() OVER()` y `AVG() OVER()` usando marcos de ventana (`ROWS BETWEEN`). Cada ejercicio incluye una variante con `PARTITION BY` para segmentar el análisis por región o categoría.

---

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Aplicar `ROW_NUMBER`, `RANK` y `DENSE_RANK` para generar rankings de productos, vendedores y clientes según métricas de negocio, identificando las diferencias en el manejo de empates.
- [ ] Usar `LAG` y `LEAD` para calcular la variación de ventas respecto al período anterior y siguiente en una serie temporal.
- [ ] Dominar la sintaxis `OVER(PARTITION BY ... ORDER BY ...)` para controlar el alcance y orden de las funciones analíticas dentro de grupos de datos.
- [ ] Calcular totales acumulados y promedios móviles usando `SUM() OVER()` y `AVG() OVER()` con marcos de ventana (`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`).
- [ ] Filtrar resultados de window functions directamente con la cláusula `QUALIFY`, reconociéndola como una extensión exclusiva de Snowflake.

---

## Prerrequisitos

### Conocimientos previos

| Área | Nivel requerido |
|---|---|
| `SELECT`, `FROM`, `WHERE`, `ORDER BY` | Sólido |
| `GROUP BY` con `SUM`, `AVG`, `COUNT` | Sólido |
| CTEs (`WITH ... AS`) | Intermedio |
| `CASE WHEN` | Intermedio |
| `ROW_NUMBER()` básico | Completado en la Práctica 3 o equivalente |
| Concepto de partición lógica de datos | Básico |

### Acceso y configuración

| Requisito | Detalle |
|---|---|
| Cuenta Snowflake activa | Trial o corporativa con acceso a Snowsight |
| Script de setup ejecutado | No se asume script previo. Esta práctica incluye el setup completo de base, schema, tablas y datos. |
| Database disponible | `LAB_SQL_INTERMEDIO` |
| Schema disponible | `LAB_SQL_INTERMEDIO.VENTAS` |
| Tablas requeridas | `CLIENTES`, `PEDIDOS`, `PRODUCTOS`, `VENTAS`, creadas en el Paso 0 |
| Warehouse activo | `COMPUTE_WH` (tamaño `X-SMALL`) |

---

## Entorno de Laboratorio

### Hardware recomendado

| Componente | Mínimo | Recomendado |
|---|---|---|
| Procesador | Intel Core i5 / AMD Ryzen 5 (64-bit) | Intel Core i7 / AMD Ryzen 7 |
| RAM | 8 GB | 16 GB |
| Almacenamiento libre | 2 GB | 5 GB |
| Conexión a Internet | 10 Mbps | 25 Mbps o superior |
| Resolución de pantalla | 1280×768 | 1920×1080 |

### Software requerido

| Software | Versión mínima | Uso |
|---|---|---|
| Navegador web (Chrome / Firefox / Edge / Safari) | 110+ / 110+ / 110+ / 16+ | Acceso a Snowsight |
| Snowflake (Snowsight) | Versión web actual | Ejecución de consultas SQL |
| Visual Studio Code *(opcional)* | 1.80+ | Edición previa de scripts |
| SnowSQL CLI *(opcional)* | 1.2.x+ | Ejecución alternativa por terminal |

---

## Organización recomendada de Workspace en Snowsight

Para que la práctica sea ordenada y reutilizable, trabaja con un Workspace y dos folders. En esta práctica se usa la palabra **workspace** como separación lógica de trabajo dentro de Snowsight; técnicamente trabajarás dentro de **Projects / Private Workspace**.

| Workspace / Worksheet | Folder | Nombre sugerido | Uso |
|---|---|---|---|
| `SNOWLABS-INT` | `SETUP-LABS` | `04_SETUP_DATOS_WINDOW_FUNCTIONS` | Crear database, schema, tablas y datos de ventas mensuales. Se ejecuta una vez al inicio o cuando quieras reiniciar el laboratorio. |
| `SNOWLABS-INT` | `SCRIPT-LABS` | `04_LAB_WINDOW_FUNCTIONS` | Ejecutar los ejercicios del laboratorio sin mezclar el script de carga de datos. |

### Paso 0.0 — Crear el workspace de las prácticas

1. Entra a **Snowsight**.
2. Da clic en **Projects**.
3. Da clic en **+**.
4. Selecciona **Private workspace**.
5. Nómbralo: **`SNOWLABS-INT`**.
6. Da clic en **Create**.

### Paso 0.0.1 — Crear el folder y script que carga los datos

1. Dentro del workspace **`SNOWLABS-INT`**, da clic en **+ Add new**.
2. Clic en **Folder** y nómbralo: **`SETUP-LABS`**.
3. Dentro del folder **`SETUP-LABS`**, da clic en el símbolo **+**.
4. Crea un archivo de tipo **SQL**.
5. Nómbralo: **`04_SETUP_DATOS_WINDOW_FUNCTIONS`**.
6. Pega ahí el siguiente script completo.
7. Ejecuta el script completo antes de comenzar el laboratorio.

Este dataset está diseñado para activar todos los escenarios de la práctica:

- Ventas distribuidas en 12 meses para practicar `LAG`, `LEAD`, acumulados y promedios móviles.
- 4 regiones para practicar `PARTITION BY region`.
- 8 productos para construir rankings Top 5 por región.
- 8 vendedores para practicar ranking global, ranking por región y `NTILE(4)`.
- Tablas `CLIENTES`, `PEDIDOS`, `PRODUCTOS` y `VENTAS` para mantener continuidad con las prácticas anteriores.
- Montos con variación mensual y regional para que las tendencias no sean planas.

```sql
-- 04_setup_datos_window_functions.sql
-- Práctica Snowflake Intermedio
-- Dataset controlado para completar el laboratorio:
-- Análisis de rankings y secuencias con window functions
--
-- Objetivo del dataset:
-- 1) Crear ventas mensuales para 12 meses.
-- 2) Tener 4 regiones para practicar PARTITION BY.
-- 3) Tener 8 productos para rankings Top 5.
-- 4) Tener 8 vendedores para ranking global, ranking regional y cuartiles.
-- 5) Mantener CLIENTES, PEDIDOS, PRODUCTOS y VENTAS disponibles en el mismo schema.

USE WAREHOUSE COMPUTE_WH;

CREATE DATABASE IF NOT EXISTS LAB_SQL_INTERMEDIO;
USE DATABASE LAB_SQL_INTERMEDIO;

CREATE SCHEMA IF NOT EXISTS VENTAS;
USE SCHEMA VENTAS;

-- Opcional para repetir el laboratorio desde cero.
DROP TABLE IF EXISTS VENTAS;
DROP TABLE IF EXISTS PEDIDOS;
DROP TABLE IF EXISTS PRODUCTOS;
DROP TABLE IF EXISTS CLIENTES;

CREATE OR REPLACE TABLE CLIENTES (
    ID_CLIENTE NUMBER(10,0) NOT NULL,
    NOMBRE VARCHAR(100) NOT NULL,
    EMAIL VARCHAR(150),
    FECHA_REGISTRO DATE,
    CIUDAD VARCHAR(80),
    PAIS VARCHAR(80),
    CONSTRAINT PK_CLIENTES PRIMARY KEY (ID_CLIENTE)
);

CREATE OR REPLACE TABLE PRODUCTOS (
    PRODUCTO_ID NUMBER(10,0) NOT NULL,
    NOMBRE_PRODUCTO VARCHAR(120) NOT NULL,
    CATEGORIA VARCHAR(80) NOT NULL,
    PRECIO_UNITARIO NUMBER(10,2) NOT NULL,
    ACTIVO BOOLEAN DEFAULT TRUE,
    CONSTRAINT PK_PRODUCTOS PRIMARY KEY (PRODUCTO_ID)
);

CREATE OR REPLACE TABLE VENTAS (
    VENTA_ID NUMBER(10,0) NOT NULL,
    PRODUCTO_ID NUMBER(10,0) NOT NULL,
    ID_CLIENTE NUMBER(10,0) NOT NULL,
    VENDEDOR_ID NUMBER(10,0) NOT NULL,
    REGION VARCHAR(40) NOT NULL,
    FECHA_VENTA DATE NOT NULL,
    MONTO_VENTA NUMBER(12,2) NOT NULL,
    CANAL VARCHAR(40),
    CONSTRAINT PK_VENTAS PRIMARY KEY (VENTA_ID)
);

CREATE OR REPLACE TABLE PEDIDOS (
    ID_PEDIDO NUMBER(10,0) NOT NULL,
    ID_CLIENTE NUMBER(10,0) NOT NULL,
    FECHA_PEDIDO DATE NOT NULL,
    MONTO_TOTAL NUMBER(12,2) NOT NULL,
    ESTADO_PEDIDO VARCHAR(30) NOT NULL,
    CONSTRAINT PK_PEDIDOS PRIMARY KEY (ID_PEDIDO)
);

INSERT INTO CLIENTES (ID_CLIENTE, NOMBRE, EMAIL, FECHA_REGISTRO, CIUDAD, PAIS) VALUES
    (1,  'Ana Torres',       'ana.torres@techcommerce.com',       '2023-01-15', 'CDMX',        'México'),
    (2,  'Luis Martínez',    'luis.martinez@techcommerce.com',    '2023-03-02', 'CDMX',        'México'),
    (3,  'María López',      'maria.lopez@techcommerce.com',      '2023-02-20', 'Guadalajara', 'México'),
    (4,  'Carlos Hernández', 'carlos.hernandez@techcommerce.com', '2023-05-10', 'Guadalajara', 'México'),
    (5,  'Sofía Ramírez',    'sofia.ramirez@techcommerce.com',    '2023-06-18', 'Monterrey',   'México'),
    (6,  'Jorge Castillo',   'jorge.castillo@techcommerce.com',   '2023-07-22', 'Monterrey',   'México'),
    (7,  'Elena Flores',     'elena.flores@techcommerce.com',     '2023-08-04', 'Puebla',      'México'),
    (8,  'Diego Sánchez',    'diego.sanchez@techcommerce.com',    '2023-09-11', 'Puebla',      'México'),
    (9,  'Valeria Cruz',     'valeria.cruz@techcommerce.com',     '2023-10-05', 'Mérida',      'México'),
    (10, 'Roberto Díaz',     'roberto.diaz@techcommerce.com',     '2023-11-01', 'Mérida',      'México'),
    (11, 'Fernanda Ruiz',    'fernanda.ruiz@techcommerce.com',    '2024-01-12', 'Querétaro',   'México'),
    (12, 'Andrés Moreno',    'andres.moreno@techcommerce.com',    '2024-02-19', 'Querétaro',   'México'),
    (13, 'Paola Vega',       'paola.vega@techcommerce.com',       '2024-03-08', 'Toluca',      'México'),
    (14, 'Ricardo Silva',    'ricardo.silva@techcommerce.com',    '2024-04-14', 'Toluca',      'México'),
    (15, 'Gabriela Núñez',   'gabriela.nunez@techcommerce.com',   '2024-05-21', 'León',        'México'),
    (16, 'Héctor Molina',    'hector.molina@techcommerce.com',    '2024-06-10', 'León',        'México');

INSERT INTO PRODUCTOS (PRODUCTO_ID, NOMBRE_PRODUCTO, CATEGORIA, PRECIO_UNITARIO, ACTIVO) VALUES
    (1, 'Laptop Pro 14',             'Electrónica',      1200.00, TRUE),
    (2, 'Monitor 27 pulgadas',       'Electrónica',       350.00, TRUE),
    (3, 'Teclado mecánico',          'Accesorios',        120.00, TRUE),
    (4, 'Mouse inalámbrico',         'Accesorios',         80.00, TRUE),
    (5, 'Silla ergonómica',          'Oficina',           420.00, TRUE),
    (6, 'Escritorio ajustable',      'Oficina',           680.00, TRUE),
    (7, 'Licencia BI anual',         'Software',          900.00, TRUE),
    (8, 'Servidor compacto',         'Infraestructura',  1500.00, TRUE);

-- Generación controlada de 384 ventas:
-- 12 meses x 4 regiones x 8 productos = 384 filas.
-- La fórmula genera variación mensual, regional, por producto y por vendedor.
INSERT INTO VENTAS (VENTA_ID, PRODUCTO_ID, ID_CLIENTE, VENDEDOR_ID, REGION, FECHA_VENTA, MONTO_VENTA, CANAL)
WITH
meses AS (
    SELECT COLUMN1::NUMBER AS MES
    FROM VALUES (1),(2),(3),(4),(5),(6),(7),(8),(9),(10),(11),(12)
),
regiones AS (
    SELECT COLUMN1::VARCHAR AS REGION, COLUMN2::NUMBER AS FACTOR_REGION, COLUMN3::NUMBER AS REGION_ORDEN
    FROM VALUES
        ('Norte',     120, 1),
        ('Sur',        95, 2),
        ('Centro',    110, 3),
        ('Occidente', 105, 4)
),
productos_base AS (
    SELECT COLUMN1::NUMBER AS PRODUCTO_ID, COLUMN2::NUMBER AS BASE_PRODUCTO
    FROM VALUES
        (1, 1200),
        (2,  700),
        (3,  250),
        (4,  180),
        (5,  620),
        (6,  850),
        (7,  950),
        (8, 1500)
),
base AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY m.MES, r.REGION_ORDEN, p.PRODUCTO_ID) AS VENTA_ID,
        p.PRODUCTO_ID,
        1 + MOD(m.MES + r.REGION_ORDEN + p.PRODUCTO_ID, 16) AS ID_CLIENTE,
        1 + MOD((r.REGION_ORDEN * 2) + p.PRODUCTO_ID + m.MES, 8) AS VENDEDOR_ID,
        r.REGION,
        DATE_FROM_PARTS(2024, m.MES, 1 + MOD(p.PRODUCTO_ID * 3 + r.REGION_ORDEN, 24)) AS FECHA_VENTA,
        (
            p.BASE_PRODUCTO
            + (r.FACTOR_REGION * 8)
            + (m.MES * 65)
            + (MOD(m.MES + p.PRODUCTO_ID + r.REGION_ORDEN, 5) * 90)
        )::NUMBER(12,2) AS MONTO_VENTA,
        CASE MOD(p.PRODUCTO_ID + m.MES + r.REGION_ORDEN, 3)
            WHEN 0 THEN 'Web'
            WHEN 1 THEN 'Ejecutivo'
            ELSE 'Marketplace'
        END AS CANAL
    FROM meses m
    CROSS JOIN regiones r
    CROSS JOIN productos_base p
)
SELECT *
FROM base;

-- PEDIDOS se deriva de VENTAS para mantener continuidad con prácticas anteriores.
INSERT INTO PEDIDOS (ID_PEDIDO, ID_CLIENTE, FECHA_PEDIDO, MONTO_TOTAL, ESTADO_PEDIDO)
SELECT
    VENTA_ID AS ID_PEDIDO,
    ID_CLIENTE,
    FECHA_VENTA AS FECHA_PEDIDO,
    MONTO_VENTA AS MONTO_TOTAL,
    CASE
        WHEN MOD(VENTA_ID, 29) = 0 THEN 'CANCELADO'
        WHEN MOD(VENTA_ID, 11) = 0 THEN 'ENVIADO'
        WHEN MOD(VENTA_ID, 7)  = 0 THEN 'EN_PROCESO'
        ELSE 'COMPLETADO'
    END AS ESTADO_PEDIDO
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
-- CLIENTES  = 16
-- PEDIDOS   = 384
-- PRODUCTOS = 8
-- VENTAS    = 384

-- Validación de distribución temporal y regional.
SELECT
    COUNT(DISTINCT DATE_TRUNC('month', FECHA_VENTA)) AS MESES_CON_DATOS,
    COUNT(DISTINCT REGION) AS REGIONES_CON_DATOS,
    COUNT(DISTINCT VENDEDOR_ID) AS VENDEDORES_CON_DATOS,
    COUNT(DISTINCT PRODUCTO_ID) AS PRODUCTOS_CON_DATOS
FROM VENTAS;

-- Resultado esperado:
-- MESES_CON_DATOS = 12
-- REGIONES_CON_DATOS = 4
-- VENDEDORES_CON_DATOS = 8
-- PRODUCTOS_CON_DATOS = 8
```

### Paso 0.0.2 — Crear el folder y script de laboratorio

1. Da clic en **+ Add new**.
2. Clic en **Folder** y nómbralo: **`SCRIPT-LABS`**.
3. Dentro de **`SCRIPT-LABS`**, crea un archivo de tipo **SQL**.
4. Nómbralo: **`04_LAB_WINDOW_FUNCTIONS`**.
5. Usa este archivo para ejecutar los ejercicios 1, 2, 3, 4 y 5.
6. **No pegues aquí el script de carga completo; solo usa las consultas de análisis del laboratorio.**

---

### Paso 0.1 — Confirmar contexto de trabajo

Dentro del archivo **`04_LAB_WINDOW_FUNCTIONS`**, ejecuta:

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

### Paso 0.2 — Confirmar que las tablas quedaron disponibles

```sql
SHOW TABLES;
```

**Resultado esperado:** deben aparecer al menos estas tablas:

| Tabla | Uso en la práctica |
|---|---|
| `CLIENTES` | Referencia de clientes para mantener continuidad con prácticas anteriores. |
| `PEDIDOS` | Pedidos derivados de ventas para mantener el modelo transaccional. |
| `PRODUCTOS` | Catálogo de productos usado para rankings por producto y categoría. |
| `VENTAS` | Tabla principal para window functions, rankings, secuencias y series temporales. |

### Paso 0.3 — Validar volumen mínimo de datos

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
| CLIENTES | 16 |
| PEDIDOS | 384 |
| PRODUCTOS | 8 |
| VENTAS | 384 |

### Paso 0.4 — Validar distribución temporal y regional

```sql
SELECT
    COUNT(DISTINCT DATE_TRUNC('month', FECHA_VENTA)) AS MESES_CON_DATOS,
    COUNT(DISTINCT REGION)                           AS REGIONES_CON_DATOS,
    COUNT(DISTINCT VENDEDOR_ID)                      AS VENDEDORES_CON_DATOS,
    COUNT(DISTINCT PRODUCTO_ID)                      AS PRODUCTOS_CON_DATOS
FROM VENTAS;
```

**Resultado esperado:**

| MESES_CON_DATOS | REGIONES_CON_DATOS | VENDEDORES_CON_DATOS | PRODUCTOS_CON_DATOS |
|---:|---:|---:|---:|
| 12 | 4 | 8 | 8 |

### Paso 0.5 — Validar que hay datos suficientes para series temporales

```sql
SELECT
    DATE_TRUNC('month', FECHA_VENTA) AS MES,
    COUNT(*)                         AS TOTAL_VENTAS,
    ROUND(SUM(MONTO_VENTA), 2)       AS MONTO_MENSUAL
FROM VENTAS
GROUP BY DATE_TRUNC('month', FECHA_VENTA)
ORDER BY MES;
```

**Resultado esperado:** deben aparecer 12 meses, de enero a diciembre de 2024. Cada mes debe tener 32 ventas: 4 regiones × 8 productos.

---

## Ejercicios Paso a Paso

---

### Ejercicio 1 — Exploración inicial: entendiendo la ventana de datos

**Objetivo:** Familiarizarte con la estructura de los datos de ventas mensuales y observar la diferencia fundamental entre una agregación con `GROUP BY` y una función analítica con `OVER()`.

#### Instrucciones

**Paso 1.1 — Explorar la estructura de la tabla `VENTAS`**

```sql
DESCRIBE TABLE VENTAS;
```

**Paso 1.2 — Revisar una muestra de ventas con producto y mes**

```sql
-- Exploración de la tabla VENTAS
SELECT
    v.VENTA_ID,
    v.PRODUCTO_ID,
    p.NOMBRE_PRODUCTO,
    p.CATEGORIA,
    v.REGION,
    v.VENDEDOR_ID,
    v.FECHA_VENTA,
    DATE_TRUNC('month', v.FECHA_VENTA) AS MES_VENTA,
    v.MONTO_VENTA
FROM VENTAS v
JOIN PRODUCTOS p
    ON v.PRODUCTO_ID = p.PRODUCTO_ID
ORDER BY v.FECHA_VENTA
LIMIT 20;
```

**Paso 1.3 — Ahora compara el comportamiento de GROUP BY versus una window function ejecutando ambas consultas en paralelo:**

```sql
-- CONSULTA A: Agregación tradicional con GROUP BY
-- Resultado: UNA fila por región (detalle de ventas individuales PERDIDO)
SELECT
    REGION,
    SUM(MONTO_VENTA) AS TOTAL_VENTAS_REGION
FROM VENTAS
GROUP BY REGION
ORDER BY TOTAL_VENTAS_REGION DESC;
```

```sql
-- CONSULTA B: Función analítica con OVER()
-- Resultado: TODAS las filas originales + columna con el total de la región
SELECT
    VENTA_ID,
    REGION,
    VENDEDOR_ID,
    MONTO_VENTA,
    SUM(MONTO_VENTA) OVER (PARTITION BY REGION) AS TOTAL_VENTAS_REGION,
    ROUND(
        MONTO_VENTA / SUM(MONTO_VENTA) OVER (PARTITION BY REGION) * 100,
        2
    ) AS PCT_CONTRIBUCION
FROM VENTAS
ORDER BY REGION, MONTO_VENTA DESC
LIMIT 30;
```

#### Resultado esperado

- La **Consulta A** devuelve una fila por región, sin detalle individual.
- La **Consulta B** devuelve todas las filas de `VENTAS`, pero cada una tiene la columna `TOTAL_VENTAS_REGION` y el porcentaje de contribución de esa venta al total de su región.

#### Verificación

La validación original de porcentajes se ajusta usando una CTE, porque primero se debe calcular el porcentaje por fila y después sumar por región.

```sql
WITH ventas_con_pct AS (
    SELECT
        REGION,
        ROUND(
            MONTO_VENTA / SUM(MONTO_VENTA) OVER (PARTITION BY REGION) * 100,
            2
        ) AS PCT_CONTRIBUCION
    FROM VENTAS
)
SELECT
    REGION,
    ROUND(SUM(PCT_CONTRIBUCION), 2) AS SUMA_PORCENTAJES
FROM ventas_con_pct
GROUP BY REGION
ORDER BY REGION;
```

**Resultado esperado:** cada región debe sumar aproximadamente `100.00`. Puede haber una diferencia mínima por redondeo.

> 💡 **Punto clave:** `PARTITION BY REGION` en la cláusula `OVER()` actúa como un "GROUP BY interno" que no colapsa las filas. Cada fila conserva su identidad y además recibe el valor calculado sobre su partición.

---

### Ejercicio 2 — Rankings con RANK, DENSE_RANK y ROW_NUMBER

**Objetivo:** Construir un ranking de los Top 5 productos por región usando las tres funciones de numeración, observando cómo cada una maneja los empates de forma diferente.

#### Instrucciones

**Paso 2.1 — Primero construye la base del ranking: ventas totales por producto y región:**

```sql
-- CTE base: total de ventas por producto y región
WITH ventas_por_producto_region AS (
    SELECT
        p.NOMBRE_PRODUCTO,
        p.CATEGORIA,
        v.REGION,
        SUM(v.MONTO_VENTA) AS TOTAL_VENTAS
    FROM VENTAS v
    JOIN PRODUCTOS p
        ON v.PRODUCTO_ID = p.PRODUCTO_ID
    GROUP BY p.NOMBRE_PRODUCTO, p.CATEGORIA, v.REGION
)
SELECT *
FROM ventas_por_producto_region
ORDER BY REGION, TOTAL_VENTAS DESC;
```

**Paso 2.2 — Ahora aplica las tres funciones de ranking sobre esa base y observa las diferencias:**

```sql
-- Comparación de ROW_NUMBER, RANK y DENSE_RANK
WITH ventas_por_producto_region AS (
    SELECT
        p.NOMBRE_PRODUCTO,
        p.CATEGORIA,
        v.REGION,
        SUM(v.MONTO_VENTA) AS TOTAL_VENTAS
    FROM VENTAS v
    JOIN PRODUCTOS p
        ON v.PRODUCTO_ID = p.PRODUCTO_ID
    GROUP BY p.NOMBRE_PRODUCTO, p.CATEGORIA, v.REGION
),
rankings AS (
    SELECT
        NOMBRE_PRODUCTO,
        CATEGORIA,
        REGION,
        TOTAL_VENTAS,
        ROW_NUMBER() OVER (PARTITION BY REGION ORDER BY TOTAL_VENTAS DESC) AS ROW_NUM,
        RANK()       OVER (PARTITION BY REGION ORDER BY TOTAL_VENTAS DESC) AS RANK_POS,
        DENSE_RANK() OVER (PARTITION BY REGION ORDER BY TOTAL_VENTAS DESC) AS DENSE_RANK_POS
    FROM ventas_por_producto_region
)
SELECT *
FROM rankings
ORDER BY REGION, TOTAL_VENTAS DESC;
```

**Paso 2.3 — Filtra para ver solo el Top 5 por región usando QUALIFY (extensión de Snowflake):**

```sql
-- Top 5 productos por región usando QUALIFY con DENSE_RANK
-- QUALIFY es exclusivo de Snowflake; no existe en PostgreSQL ni MySQL
WITH ventas_por_producto_region AS (
    SELECT
        p.NOMBRE_PRODUCTO,
        p.CATEGORIA,
        v.REGION,
        SUM(v.MONTO_VENTA) AS TOTAL_VENTAS
    FROM VENTAS v
    JOIN PRODUCTOS p
        ON v.PRODUCTO_ID = p.PRODUCTO_ID
    GROUP BY p.NOMBRE_PRODUCTO, p.CATEGORIA, v.REGION
)
SELECT
    NOMBRE_PRODUCTO,
    CATEGORIA,
    REGION,
    TOTAL_VENTAS,
    DENSE_RANK() OVER (PARTITION BY REGION ORDER BY TOTAL_VENTAS DESC) AS POSICION
FROM ventas_por_producto_region
QUALIFY DENSE_RANK() OVER (PARTITION BY REGION ORDER BY TOTAL_VENTAS DESC) <= 5
ORDER BY REGION, POSICION;
```

**Paso 2.4 — Para entender la diferencia entre las tres funciones, busca deliberadamente un caso de empate. Ejecuta esta consulta que fuerza un escenario de empate artificial para visualizar el comportamiento:**

```sql
-- Demostración de empates: mismo total_ventas para dos productos
-- Observa cómo cada función asigna números distintos
WITH datos_empate AS (
    SELECT 'Producto A' AS PRODUCTO, 'Norte' AS REGION, 5000 AS TOTAL_VENTAS
    UNION ALL
    SELECT 'Producto B', 'Norte', 5000
    UNION ALL
    SELECT 'Producto C', 'Norte', 4800
    UNION ALL
    SELECT 'Producto D', 'Norte', 4200
    UNION ALL
    SELECT 'Producto E', 'Norte', 4200
    UNION ALL
    SELECT 'Producto F', 'Norte', 3900
)
SELECT
    PRODUCTO,
    REGION,
    TOTAL_VENTAS,
    ROW_NUMBER() OVER (PARTITION BY REGION ORDER BY TOTAL_VENTAS DESC) AS ROW_NUM,
    RANK()       OVER (PARTITION BY REGION ORDER BY TOTAL_VENTAS DESC) AS RANK_POS,
    DENSE_RANK() OVER (PARTITION BY REGION ORDER BY TOTAL_VENTAS DESC) AS DENSE_RANK_POS
FROM datos_empate;
```

#### Resultado esperado del Paso 2.4

| PRODUCTO | TOTAL_VENTAS | ROW_NUM | RANK_POS | DENSE_RANK_POS |
|---|---:|---:|---:|---:|
| Producto A | 5000 | 1 | 1 | 1 |
| Producto B | 5000 | 2 | 1 | 1 |
| Producto C | 4800 | 3 | 3 | 2 |
| Producto D | 4200 | 4 | 4 | 3 |
| Producto E | 4200 | 5 | 4 | 3 |
| Producto F | 3900 | 6 | 6 | 4 |

> 🔍 **Diferencias clave:**
> - `ROW_NUMBER`: siempre asigna números únicos consecutivos, incluso en empates. El orden entre empatados puede depender del criterio de ordenamiento.
> - `RANK`: en empates asigna el mismo número, pero **salta** posiciones.
> - `DENSE_RANK`: en empates asigna el mismo número y **no salta** posiciones.

> ⚠️ **Nota de portabilidad:** La cláusula `QUALIFY` es exclusiva de Snowflake. En otros motores como PostgreSQL o MySQL deberías envolver la consulta en una subconsulta o CTE adicional para filtrar por el ranking.

#### Verificación

```sql
-- Verificar cuántos productos aparecen en el Top 5 por región
WITH ventas_por_producto_region AS (
    SELECT
        p.NOMBRE_PRODUCTO,
        p.CATEGORIA,
        v.REGION,
        SUM(v.MONTO_VENTA) AS TOTAL_VENTAS
    FROM VENTAS v
    JOIN PRODUCTOS p ON v.PRODUCTO_ID = p.PRODUCTO_ID
    GROUP BY p.NOMBRE_PRODUCTO, p.CATEGORIA, v.REGION
),
top_5 AS (
    SELECT
        REGION,
        NOMBRE_PRODUCTO,
        DENSE_RANK() OVER (PARTITION BY REGION ORDER BY TOTAL_VENTAS DESC) AS POSICION
    FROM ventas_por_producto_region
    QUALIFY DENSE_RANK() OVER (PARTITION BY REGION ORDER BY TOTAL_VENTAS DESC) <= 5
)
SELECT
    REGION,
    COUNT(*) AS PRODUCTOS_EN_TOP
FROM top_5
GROUP BY REGION
ORDER BY REGION;
```

**Resultado esperado:** cada región debe tener al menos 5 productos en el Top. Si aparece más de 5, significa que hubo empates dentro del ranking.

---

### Ejercicio 3 — Análisis temporal con LAG y LEAD

**Objetivo:** Usar `LAG` y `LEAD` para calcular la variación de ventas respecto al mes anterior y siguiente, identificando tendencias de crecimiento o caída.

#### Instrucciones

**Paso 3.1 — Construye primero la serie temporal de ventas mensuales totales**

```sql
-- Serie temporal de ventas mensuales agregadas
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA)  AS MES,
        SUM(MONTO_VENTA)                  AS TOTAL_VENTAS,
        COUNT(DISTINCT VENTA_ID)          AS NUM_TRANSACCIONES,
        COUNT(DISTINCT VENDEDOR_ID)       AS NUM_VENDEDORES_ACTIVOS
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
)
SELECT
    MES,
    TOTAL_VENTAS,
    NUM_TRANSACCIONES,
    NUM_VENDEDORES_ACTIVOS
FROM ventas_mensuales
ORDER BY MES;
```

**Paso 3.2 — Aplica LAG para obtener el valor del mes anterior y calcular la variación:**

```sql
-- LAG: comparar ventas con el mes anterior
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS MES,
        SUM(MONTO_VENTA)                 AS TOTAL_VENTAS
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
)
SELECT
    MES,
    TOTAL_VENTAS,
    -- LAG(columna, offset, valor_default) OVER(ORDER BY ...)
    LAG(TOTAL_VENTAS, 1, 0) OVER (ORDER BY MES) AS VENTAS_MES_ANTERIOR,
    TOTAL_VENTAS - LAG(TOTAL_VENTAS, 1, 0) OVER (ORDER BY MES) AS VARIACION_ABSOLUTA,
    ROUND(
        (TOTAL_VENTAS - LAG(TOTAL_VENTAS, 1, 0) OVER (ORDER BY MES))
        / NULLIF(LAG(TOTAL_VENTAS, 1, 0) OVER (ORDER BY MES), 0) * 100,
        2
    ) AS VARIACION_PCT
FROM ventas_mensuales
ORDER BY MES;
```

**Paso 3.3 — Agrega LEAD para ver también el mes siguiente y construir una vista completa de contexto temporal:**

```sql
-- LAG y LEAD combinados: contexto temporal completo
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS MES,
        SUM(MONTO_VENTA)                 AS TOTAL_VENTAS
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
),
serie AS (
    SELECT
        MES,
        TOTAL_VENTAS,
        LAG(TOTAL_VENTAS,  1, NULL) OVER (ORDER BY MES) AS VENTAS_MES_ANTERIOR,
        LEAD(TOTAL_VENTAS, 1, NULL) OVER (ORDER BY MES) AS VENTAS_MES_SIGUIENTE
    FROM ventas_mensuales
)
SELECT
    MES,
    TOTAL_VENTAS,
    VENTAS_MES_ANTERIOR,
    VENTAS_MES_SIGUIENTE,
    ROUND(
        (TOTAL_VENTAS - VENTAS_MES_ANTERIOR)
        / NULLIF(VENTAS_MES_ANTERIOR, 0) * 100,
        2
    ) AS VAR_PCT_VS_ANTERIOR,
    CASE
        WHEN TOTAL_VENTAS > VENTAS_MES_ANTERIOR
         AND TOTAL_VENTAS > VENTAS_MES_SIGUIENTE THEN 'Pico local'
        WHEN TOTAL_VENTAS < VENTAS_MES_ANTERIOR
         AND TOTAL_VENTAS < VENTAS_MES_SIGUIENTE THEN 'Valle local'
        WHEN TOTAL_VENTAS > VENTAS_MES_ANTERIOR THEN 'Crecimiento'
        WHEN TOTAL_VENTAS < VENTAS_MES_ANTERIOR THEN 'Caída'
        ELSE 'Sin cambio / Primer mes'
    END AS TENDENCIA
FROM serie
ORDER BY MES;
```

**Paso 3.4 — Aplica el mismo análisis segmentado por región usando PARTITION BY**

```sql
-- LAG particionado por región: cada región tiene su propia serie temporal
WITH ventas_mensuales_region AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS MES,
        REGION,
        SUM(MONTO_VENTA)                 AS TOTAL_VENTAS
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA), REGION
)
SELECT
    MES,
    REGION,
    TOTAL_VENTAS,
    -- PARTITION BY REGION garantiza que LAG no "cruce" entre regiones
    LAG(TOTAL_VENTAS, 1, NULL) OVER (PARTITION BY REGION ORDER BY MES) AS VENTAS_MES_ANTERIOR,
    ROUND(
        (TOTAL_VENTAS - LAG(TOTAL_VENTAS, 1, NULL) OVER (PARTITION BY REGION ORDER BY MES))
        / NULLIF(LAG(TOTAL_VENTAS, 1, NULL) OVER (PARTITION BY REGION ORDER BY MES), 0) * 100,
        2
    ) AS VAR_PCT_VS_ANTERIOR
FROM ventas_mensuales_region
ORDER BY REGION, MES;
```

#### Resultado esperado

- En el Paso 3.2, el primer mes tendrá `VENTAS_MES_ANTERIOR = 0` por el valor default especificado y `VARIACION_PCT = NULL`.
- En el Paso 3.3, el último mes tendrá `VENTAS_MES_SIGUIENTE = NULL` porque no hay mes siguiente.
- En el Paso 3.4, el primer mes de **cada región** tendrá `VENTAS_MES_ANTERIOR = NULL`, ya que `PARTITION BY REGION` reinicia la secuencia para cada región.

#### Verificación

```sql
-- Verificar que LAG no cruza entre regiones
WITH ventas_mensuales_region AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS MES,
        REGION,
        SUM(MONTO_VENTA) AS TOTAL_VENTAS
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA), REGION
),
con_lag AS (
    SELECT
        REGION,
        MES,
        LAG(TOTAL_VENTAS, 1, NULL) OVER (PARTITION BY REGION ORDER BY MES) AS VENTAS_MES_ANTERIOR,
        ROW_NUMBER() OVER (PARTITION BY REGION ORDER BY MES) AS RN
    FROM ventas_mensuales_region
)
SELECT
    COUNT(DISTINCT REGION) AS TOTAL_REGIONES,
    SUM(CASE WHEN RN = 1 AND VENTAS_MES_ANTERIOR IS NULL THEN 1 ELSE 0 END) AS PRIMEROS_MESES_SIN_ANTERIOR
FROM con_lag;
```

**Resultado esperado:** ambos valores deben ser iguales.

> 💡 **Concepto clave:** El tercer argumento de `LAG(columna, offset, default)` define qué valor devolver cuando no existe una fila anterior. Usar `NULL` es más preciso que `0` para cálculos de porcentajes, ya que `0` puede generar divisiones por cero o variaciones artificiales.

---

### Ejercicio 4 — Acumulados y promedios móviles con marcos de ventana

**Objetivo:** Calcular totales acumulados y promedios móviles usando `SUM() OVER()` y `AVG() OVER()` con la cláusula `ROWS BETWEEN`, dominando el concepto de marco de ventana (*window frame*).

#### Instrucciones

**Paso 4.1 — Construye un acumulado de ventas por mes (total acumulado desde el inicio del período):**

```sql
-- Acumulado de ventas mensuales (running total)
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS MES,
        SUM(MONTO_VENTA)                 AS TOTAL_VENTAS
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
)
SELECT
    MES,
    TOTAL_VENTAS,
    -- Marco: desde el inicio del dataset hasta la fila actual
    SUM(TOTAL_VENTAS) OVER (
        ORDER BY MES
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS VENTAS_ACUMULADAS,
    -- Porcentaje del acumulado sobre el gran total
    ROUND(
        SUM(TOTAL_VENTAS) OVER (
            ORDER BY MES
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        )
        / SUM(TOTAL_VENTAS) OVER () * 100,
        2
    ) AS PCT_ACUMULADO
FROM ventas_mensuales
ORDER BY MES;
```

**Paso 4.2 — Implementa un promedio móvil de 3 meses para suavizar las fluctuaciones estacionales:**

```sql
-- Promedio móvil de 3 meses y 5 meses
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS MES,
        SUM(MONTO_VENTA)                 AS TOTAL_VENTAS
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
)
SELECT
    MES,
    TOTAL_VENTAS,
    -- Promedio móvil: ventana de 3 meses (2 anteriores + actual)
    ROUND(
        AVG(TOTAL_VENTAS) OVER (
            ORDER BY MES
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ),
        2
    ) AS PROMEDIO_MOVIL_3M,
    -- Promedio móvil: ventana de 5 meses para comparar
    ROUND(
        AVG(TOTAL_VENTAS) OVER (
            ORDER BY MES
            ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
        ),
        2
    ) AS PROMEDIO_MOVIL_5M
FROM ventas_mensuales
ORDER BY MES;
```

**Paso 4.3 — Combina acumulados con PARTITION BY para calcular el acumulado por región:**

```sql
-- Acumulado mensual particionado por región
WITH ventas_mensuales_region AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS MES,
        REGION,
        SUM(MONTO_VENTA)                 AS TOTAL_VENTAS
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA), REGION
)
SELECT
    MES,
    REGION,
    TOTAL_VENTAS,
    -- Acumulado reinicia para cada región gracias a PARTITION BY
    SUM(TOTAL_VENTAS) OVER (
        PARTITION BY REGION
        ORDER BY MES
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS VENTAS_ACUMULADAS_REGION,
    -- Promedio móvil de 3 meses por región
    ROUND(
        AVG(TOTAL_VENTAS) OVER (
            PARTITION BY REGION
            ORDER BY MES
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ),
        2
    ) AS PROMEDIO_MOVIL_3M_REGION
FROM ventas_mensuales_region
ORDER BY REGION, MES;
```

**Paso 4.4 — Construye la consulta completa de análisis integrado que combina todas las técnicas del ejercicio:**

```sql
-- CONSULTA INTEGRADORA: acumulado + promedio móvil + variación vs mes anterior
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS MES,
        SUM(MONTO_VENTA)                 AS TOTAL_VENTAS,
        COUNT(DISTINCT VENTA_ID)         AS NUM_TRANSACCIONES
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
),
analisis AS (
    SELECT
        MES,
        TOTAL_VENTAS,
        NUM_TRANSACCIONES,
        SUM(TOTAL_VENTAS) OVER (
            ORDER BY MES
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS VENTAS_ACUMULADAS,
        ROUND(
            AVG(TOTAL_VENTAS) OVER (
                ORDER BY MES
                ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
            ),
            2
        ) AS PROMEDIO_MOVIL_3M,
        LAG(TOTAL_VENTAS, 1, NULL) OVER (ORDER BY MES) AS VENTAS_MES_ANTERIOR,
        RANK() OVER (ORDER BY TOTAL_VENTAS DESC) AS RANKING_MES
    FROM ventas_mensuales
)
SELECT
    MES,
    TOTAL_VENTAS,
    NUM_TRANSACCIONES,
    VENTAS_ACUMULADAS,
    PROMEDIO_MOVIL_3M,
    VENTAS_MES_ANTERIOR,
    ROUND(
        (TOTAL_VENTAS - VENTAS_MES_ANTERIOR)
        / NULLIF(VENTAS_MES_ANTERIOR, 0) * 100,
        2
    ) AS VARIACION_PCT_MOM,
    RANKING_MES
FROM analisis
ORDER BY MES;
```

#### Resultado esperado

- En el Paso 4.1, `VENTAS_ACUMULADAS` debe aumentar mes a mes y el último mes debe mostrar `PCT_ACUMULADO = 100.00`.
- En el Paso 4.2, los primeros dos meses tendrán `PROMEDIO_MOVIL_3M` calculado sobre menos de tres meses, lo cual es el comportamiento correcto de `ROWS BETWEEN`.
- En el Paso 4.3, el acumulado de cada región reinicia en su primer mes, demostrando que `PARTITION BY` funciona correctamente.

#### Verificación

```sql
-- Verificación: el acumulado del último mes debe igualar el gran total
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS MES,
        SUM(MONTO_VENTA)                 AS TOTAL_VENTAS
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
),
acumulados AS (
    SELECT
        MES,
        TOTAL_VENTAS,
        SUM(TOTAL_VENTAS) OVER (
            ORDER BY MES
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS ACUMULADO
    FROM ventas_mensuales
)
SELECT
    MAX(ACUMULADO) AS ACUMULADO_ULTIMO_MES,
    SUM(TOTAL_VENTAS) AS GRAN_TOTAL_DIRECTO,
    MAX(ACUMULADO) = SUM(TOTAL_VENTAS) AS ES_IGUAL
FROM acumulados;
```

**Resultado esperado:** `ES_IGUAL` debe ser `TRUE`.

> 💡 **Anatomía del marco de ventana:**
> - `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` → desde la primera fila hasta la actual.
> - `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` → las dos filas anteriores más la actual.
> - `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` → todas las filas de la ventana.

---

### Ejercicio 5 — Desafío integrador: Reporte ejecutivo de desempeño

**Objetivo:** Combinar todas las técnicas del laboratorio en una consulta analítica compleja que genere un reporte ejecutivo de desempeño de vendedores con ranking, tendencia y clasificación por cuartil.

#### Instrucciones

**Paso 5.1 — Construye el reporte ejecutivo completo en una sola consulta con CTEs:**

```sql
-- REPORTE EJECUTIVO DE DESEMPEÑO DE VENDEDORES
-- Combina: RANK, DENSE_RANK, LAG, SUM acumulado y clasificación por cuartil

WITH
-- CTE 1: Ventas totales por vendedor y mes
ventas_vendedor_mes AS (
    SELECT
        VENDEDOR_ID,
        DATE_TRUNC('month', FECHA_VENTA) AS MES,
        REGION,
        SUM(MONTO_VENTA)                 AS TOTAL_VENTAS,
        COUNT(DISTINCT VENTA_ID)         AS NUM_TRANSACCIONES
    FROM VENTAS
    GROUP BY VENDEDOR_ID, DATE_TRUNC('month', FECHA_VENTA), REGION
),

-- CTE 2: Ventas totales anuales por vendedor y región
ventas_vendedor_anual AS (
    SELECT
        VENDEDOR_ID,
        REGION,
        SUM(TOTAL_VENTAS)       AS VENTAS_ANUALES,
        SUM(NUM_TRANSACCIONES)  AS TRANSACCIONES_ANUALES,
        COUNT(DISTINCT MES)     AS MESES_ACTIVO
    FROM ventas_vendedor_mes
    GROUP BY VENDEDOR_ID, REGION
),

-- CTE 3: Métricas de análisis temporal por vendedor
metricas_temporales AS (
    SELECT
        VENDEDOR_ID,
        MES,
        REGION,
        TOTAL_VENTAS,
        NUM_TRANSACCIONES,
        -- Ventas del mes anterior para este vendedor en esta región
        LAG(TOTAL_VENTAS, 1, NULL) OVER (
            PARTITION BY VENDEDOR_ID, REGION
            ORDER BY MES
        ) AS VENTAS_MES_ANTERIOR,
        -- Acumulado del vendedor en la región
        SUM(TOTAL_VENTAS) OVER (
            PARTITION BY VENDEDOR_ID, REGION
            ORDER BY MES
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS VENTAS_ACUMULADAS,
        -- Promedio móvil 3 meses del vendedor
        ROUND(
            AVG(TOTAL_VENTAS) OVER (
                PARTITION BY VENDEDOR_ID, REGION
                ORDER BY MES
                ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
            ),
            2
        ) AS PROMEDIO_MOVIL_3M
    FROM ventas_vendedor_mes
),

-- CTE 4: Ranking y clasificación por cuartil sobre ventas anuales
ranking_vendedores AS (
    SELECT
        VENDEDOR_ID,
        REGION,
        VENTAS_ANUALES,
        TRANSACCIONES_ANUALES,
        MESES_ACTIVO,
        -- Ranking global: vendedor-región contra todos los demás
        RANK()       OVER (ORDER BY VENTAS_ANUALES DESC) AS RANKING_GLOBAL,
        DENSE_RANK() OVER (ORDER BY VENTAS_ANUALES DESC) AS DENSE_RANKING_GLOBAL,
        -- Ranking dentro de la región
        RANK() OVER (
            PARTITION BY REGION
            ORDER BY VENTAS_ANUALES DESC
        ) AS RANKING_REGION,
        -- Clasificación por cuartil usando NTILE
        NTILE(4) OVER (ORDER BY VENTAS_ANUALES DESC) AS CUARTIL,
        CASE NTILE(4) OVER (ORDER BY VENTAS_ANUALES DESC)
            WHEN 1 THEN 'Top 25% — Élite'
            WHEN 2 THEN 'Top 50% — Alto'
            WHEN 3 THEN 'Top 75% — Medio'
            WHEN 4 THEN 'Bottom 25% — En desarrollo'
        END AS CATEGORIA_DESEMPENO
    FROM ventas_vendedor_anual
),

-- CTE 5: Último mes por vendedor-región para no usar MAX() de forma ambigua
ultimo_mes AS (
    SELECT
        VENDEDOR_ID,
        REGION,
        MES AS ULTIMO_MES_CON_DATOS,
        TOTAL_VENTAS AS VENTAS_ULTIMO_MES,
        PROMEDIO_MOVIL_3M AS PROMEDIO_MOVIL_ULTIMO_MES,
        VENTAS_ACUMULADAS AS VENTAS_ACUMULADAS_ULTIMO_MES
    FROM metricas_temporales
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY VENDEDOR_ID, REGION
        ORDER BY MES DESC
    ) = 1
)

-- CONSULTA FINAL: unir todas las métricas
SELECT
    r.VENDEDOR_ID,
    r.REGION,
    r.VENTAS_ANUALES,
    r.TRANSACCIONES_ANUALES,
    r.MESES_ACTIVO,
    r.RANKING_GLOBAL,
    r.DENSE_RANKING_GLOBAL,
    r.RANKING_REGION,
    r.CUARTIL,
    r.CATEGORIA_DESEMPENO,
    u.ULTIMO_MES_CON_DATOS,
    u.VENTAS_ULTIMO_MES,
    u.PROMEDIO_MOVIL_ULTIMO_MES,
    u.VENTAS_ACUMULADAS_ULTIMO_MES
FROM ranking_vendedores r
JOIN ultimo_mes u
    ON r.VENDEDOR_ID = u.VENDEDOR_ID
   AND r.REGION = u.REGION
ORDER BY r.RANKING_GLOBAL;
```

**Paso 5.2 — Filtra el resultado para ver solo los vendedores del Top 25% (Élite) usando QUALIFY:**

```sql
-- Versión con QUALIFY para filtrar directamente por cuartil
-- QUALIFY es exclusivo de Snowflake
WITH ventas_vendedor_anual AS (
    SELECT
        VENDEDOR_ID,
        REGION,
        SUM(MONTO_VENTA)         AS VENTAS_ANUALES,
        COUNT(DISTINCT VENTA_ID) AS TRANSACCIONES_ANUALES
    FROM VENTAS
    GROUP BY VENDEDOR_ID, REGION
)
SELECT
    VENDEDOR_ID,
    REGION,
    VENTAS_ANUALES,
    TRANSACCIONES_ANUALES,
    RANK()   OVER (ORDER BY VENTAS_ANUALES DESC) AS RANKING_GLOBAL,
    RANK()   OVER (PARTITION BY REGION ORDER BY VENTAS_ANUALES DESC) AS RANKING_REGION,
    NTILE(4) OVER (ORDER BY VENTAS_ANUALES DESC) AS CUARTIL
FROM ventas_vendedor_anual
QUALIFY NTILE(4) OVER (ORDER BY VENTAS_ANUALES DESC) = 1
ORDER BY RANKING_GLOBAL;
```

#### Resultado esperado

- El reporte del Paso 5.1 debe mostrar una fila por combinación `VENDEDOR_ID` + `REGION` con todas sus métricas consolidadas.
- El Paso 5.2 debe devolver aproximadamente el 25% del total de combinaciones vendedor-región.
- Los vendedores con el mismo `VENTAS_ANUALES` deben tener el mismo `RANKING_GLOBAL` con `RANK`.

#### Verificación

```sql
-- Verificar que NTILE distribuyó los cuartiles correctamente
WITH ventas_vendedor_anual AS (
    SELECT
        VENDEDOR_ID,
        REGION,
        SUM(MONTO_VENTA) AS VENTAS_ANUALES
    FROM VENTAS
    GROUP BY VENDEDOR_ID, REGION
),
cuartiles AS (
    SELECT
        VENDEDOR_ID,
        REGION,
        NTILE(4) OVER (ORDER BY VENTAS_ANUALES DESC) AS CUARTIL
    FROM ventas_vendedor_anual
)
SELECT
    CUARTIL,
    COUNT(*) AS NUM_VENDEDORES_REGION
FROM cuartiles
GROUP BY CUARTIL
ORDER BY CUARTIL;
```

Cada cuartil debe tener aproximadamente el mismo número de filas. La diferencia máxima entre cuartiles debe ser de una fila por efectos del redondeo de `NTILE`.

---

## Validación y Pruebas Finales

Ejecuta las siguientes consultas de validación para confirmar que todos los ejercicios fueron completados correctamente.

```sql
-- VALIDACIÓN 1: Rankings producen resultados válidos
-- Criterio: ROW_NUMBER siempre único dentro de cada región.
WITH base AS (
    SELECT
        p.NOMBRE_PRODUCTO,
        v.REGION,
        SUM(v.MONTO_VENTA) AS TOTAL
    FROM VENTAS v
    JOIN PRODUCTOS p
        ON v.PRODUCTO_ID = p.PRODUCTO_ID
    GROUP BY p.NOMBRE_PRODUCTO, v.REGION
),
rankings AS (
    SELECT
        NOMBRE_PRODUCTO,
        REGION,
        TOTAL,
        ROW_NUMBER() OVER (PARTITION BY REGION ORDER BY TOTAL DESC) AS ROW_NUM,
        RANK()       OVER (PARTITION BY REGION ORDER BY TOTAL DESC) AS RANK_POS,
        DENSE_RANK() OVER (PARTITION BY REGION ORDER BY TOTAL DESC) AS DENSE_RANK_POS
    FROM base
)
SELECT
    'Rankings correctos' AS VALIDACION,
    COUNT(DISTINCT REGION) AS REGIONES,
    COUNT(*) AS FILAS_RANKING,
    COUNT(DISTINCT REGION || '-' || ROW_NUM) = COUNT(*) AS ROW_NUMBER_UNICO_POR_REGION
FROM rankings;
```

```sql
-- VALIDACIÓN 2: Acumulado del último mes = Gran total
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', FECHA_VENTA) AS MES,
        SUM(MONTO_VENTA) AS TOTAL
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', FECHA_VENTA)
),
acumulados AS (
    SELECT
        MES,
        TOTAL,
        SUM(TOTAL) OVER (
            ORDER BY MES
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS ACUM
    FROM ventas_mensuales
)
SELECT
    'Acumulado correcto' AS VALIDACION,
    MAX(ACUM) AS ACUMULADO_ULTIMO_MES,
    SUM(TOTAL) AS GRAN_TOTAL,
    MAX(ACUM) = SUM(TOTAL) AS ES_IGUAL
FROM acumulados;
```

```sql
-- VALIDACIÓN 3: LAG con PARTITION BY no cruza regiones
-- El primer mes de cada región debe tener ventas_anterior = NULL
WITH ventas_region_mes AS (
    SELECT
        REGION,
        DATE_TRUNC('month', FECHA_VENTA) AS MES,
        SUM(MONTO_VENTA) AS TOTAL
    FROM VENTAS
    GROUP BY REGION, DATE_TRUNC('month', FECHA_VENTA)
),
con_lag AS (
    SELECT
        REGION,
        MES,
        TOTAL,
        LAG(TOTAL, 1, NULL) OVER (PARTITION BY REGION ORDER BY MES) AS ANTERIOR,
        ROW_NUMBER() OVER (PARTITION BY REGION ORDER BY MES) AS RN
    FROM ventas_region_mes
)
SELECT
    'LAG particionado correcto' AS VALIDACION,
    COUNT(DISTINCT REGION) AS TOTAL_REGIONES,
    SUM(CASE WHEN RN = 1 AND ANTERIOR IS NULL THEN 1 ELSE 0 END) AS PRIMEROS_MESES_SIN_ANTERIOR,
    SUM(CASE WHEN RN = 1 AND ANTERIOR IS NULL THEN 1 ELSE 0 END) = COUNT(DISTINCT REGION) AS OK
FROM con_lag;
```

**Criterio de éxito:** las validaciones deben devolver `TRUE` en sus columnas de verificación (`ROW_NUMBER_UNICO_POR_REGION`, `ES_IGUAL`, `OK`). Si alguna falla, revisa el ejercicio correspondiente.

---

## Resultados esperados clave con el dataset cargado

Estos resultados ayudan al instructor y al alumno a validar rápidamente que el laboratorio se está ejecutando sobre el dataset correcto.

| Consulta / Validación | Resultado esperado |
|---|---:|
| Conteo de `CLIENTES` | 16 filas |
| Conteo de `PRODUCTOS` | 8 filas |
| Conteo de `VENTAS` | 384 filas |
| Conteo de `PEDIDOS` | 384 filas |
| Meses con datos | 12 |
| Regiones con datos | 4 |
| Vendedores con datos | 8 |
| Productos con datos | 8 |
| Ventas por mes | 32 filas por mes |
| Top 5 por región | 5 productos por región, salvo empates |
| Primer mes por región con `LAG` | `NULL` en `VENTAS_MES_ANTERIOR` |
| Último acumulado mensual | Igual al total global de ventas |

---

## Solución de Problemas

### Problema 1 — Error: "Window function [X] requires ORDER BY in window specification"

**Síntoma:** Al ejecutar una consulta con `LAG`, `LEAD`, `RANK`, `ROW_NUMBER` o `DENSE_RANK`, Snowflake devuelve un error similar a:

```text
SQL compilation error: Window function [ROW_NUMBER] requires ORDER BY in window specification
```

**Causa:** Las funciones de numeración y desplazamiento (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`, `LEAD`) requieren una cláusula `ORDER BY` dentro de `OVER()`. Sin ella, Snowflake no puede determinar el orden en que asignar los números o buscar filas anteriores/siguientes.

**Solución:** Agrega siempre `ORDER BY` dentro del `OVER()` para estas funciones:

```sql
-- Incorrecto: falta ORDER BY
SELECT
    VENDEDOR_ID,
    ROW_NUMBER() OVER (PARTITION BY REGION) AS NUM
FROM VENTAS;

-- Correcto: ORDER BY incluido
SELECT
    VENDEDOR_ID,
    ROW_NUMBER() OVER (
        PARTITION BY REGION
        ORDER BY MONTO_VENTA DESC
    ) AS NUM
FROM VENTAS;
```

> **Nota:** `SUM() OVER()` y `AVG() OVER()` sí pueden usarse sin `ORDER BY` cuando quieres el total de toda la partición, no un acumulado. En ese caso, el marco de ventana por defecto incluye todas las filas de la partición.

---

### Problema 2 — `QUALIFY` no filtra los resultados o genera error de compilación

**Síntoma:** La consulta con `QUALIFY` devuelve todas las filas o genera el error:

```text
SQL compilation error: QUALIFY clause requires at least one window function
```

**Causa:** `QUALIFY` funciona para filtrar resultados de window functions. Si se usa con una columna calculada sin `OVER()`, Snowflake no puede procesarlo como filtro analítico.

**Solución:** Asegúrate de que la expresión en `QUALIFY` contiene explícitamente una window function o que filtra una columna calculada por una window function en la misma consulta.

```sql
-- Correcto: QUALIFY con la window function explícita
SELECT
    NOMBRE_PRODUCTO,
    TOTAL_VENTAS,
    DENSE_RANK() OVER (PARTITION BY REGION ORDER BY TOTAL_VENTAS DESC) AS POSICION
FROM ventas_por_producto_region
QUALIFY DENSE_RANK() OVER (PARTITION BY REGION ORDER BY TOTAL_VENTAS DESC) <= 5;
```

> ⚠️ **Recordatorio:** `QUALIFY` es exclusivo de Snowflake. Si necesitas portar esta consulta a PostgreSQL o MySQL, debes reemplazarla por una subconsulta o CTE que filtre por el alias del ranking.

---

### Problema 3 — El porcentaje de variación sale `NULL` en el primer mes

**Síntoma:** En los ejercicios con `LAG`, la columna de variación porcentual aparece como `NULL` en el primer mes.

**Causa:** El primer mes no tiene un mes anterior. Por eso `LAG(..., NULL)` devuelve `NULL`, y cualquier operación matemática con `NULL` devuelve `NULL`.

**Solución:** Este resultado es correcto. No lo reemplaces automáticamente por cero a menos que el negocio lo pida. Para evitar división entre cero se usa `NULLIF()`:

```sql
ROUND(
    (TOTAL_VENTAS - VENTAS_MES_ANTERIOR)
    / NULLIF(VENTAS_MES_ANTERIOR, 0) * 100,
    2
) AS VARIACION_PCT
```

---

## Limpieza del entorno

Al finalizar el laboratorio, ejecuta los siguientes comandos para liberar recursos y evitar consumo innecesario de créditos Snowflake:

```sql
-- Suspender el warehouse para detener el consumo de créditos
-- IMPORTANTE: ejecutar siempre al terminar la sesión
ALTER WAREHOUSE COMPUTE_WH SUSPEND;
```

```sql
-- Verificar que el warehouse quedó suspendido
SHOW WAREHOUSES LIKE 'COMPUTE_WH';
-- Busca la columna STATE; debe mostrar SUSPENDED.
```

> ⚠️ **Recordatorio de créditos:** Las cuentas trial de Snowflake tienen 400 USD de créditos. Un warehouse `X-SMALL` consume aproximadamente 1 crédito por hora de actividad. Suspenderlo al terminar cada sesión es una práctica obligatoria en este curso.

No es necesario eliminar tablas ni datos, ya que el schema `LAB_SQL_INTERMEDIO.VENTAS` es compartido por todos los laboratorios del curso y sus datos deben persistir para las sesiones siguientes.

---

## Resumen

En este laboratorio aplicaste funciones analíticas de Snowflake sobre datos de ventas mensuales, progresando desde conceptos básicos de ventanas hasta un reporte ejecutivo completo.

| Técnica | Función(es) usadas | Caso de uso aplicado |
|---|---|---|
| Rankings únicos | `ROW_NUMBER() OVER(PARTITION BY ... ORDER BY ...)` | Secuencias únicas por región |
| Rankings con empates | `RANK()`, `DENSE_RANK()` | Top 5 productos por región |
| Análisis temporal | `LAG()`, `LEAD()` | Variación mensual y detección de tendencias |
| Acumulados | `SUM() OVER(ORDER BY ... ROWS BETWEEN ...)` | Running total mensual |
| Promedios móviles | `AVG() OVER(ORDER BY ... ROWS BETWEEN ...)` | Suavizado de series temporales |
| Clasificación por cuartil | `NTILE(4) OVER(ORDER BY ...)` | Segmentación de vendedores por desempeño |
| Filtrado de rankings | `QUALIFY` | Selección directa de Top N sin subconsulta |

### Diferencias clave para recordar

1. **`RANK` vs. `DENSE_RANK`:** ambas asignan el mismo número a empates, pero `RANK` salta posiciones y `DENSE_RANK` no.
2. **`PARTITION BY` en `LAG/LEAD`:** sin `PARTITION BY`, `LAG` puede cruzar entre grupos. Con `PARTITION BY`, cada grupo tiene su propia secuencia.
3. **Marco de ventana (`ROWS BETWEEN`):** especificarlo explícitamente evita ambigüedades, especialmente cuando hay valores repetidos en el `ORDER BY`.
4. **`QUALIFY` es exclusivo de Snowflake:** mejora la legibilidad, pero no es portable a todos los motores SQL.

### Conexión con el siguiente laboratorio

En el **Laboratorio 5** aplicarás estas mismas técnicas a series temporales más complejas, incluyendo análisis de estacionalidad, detección de anomalías en tendencias de ventas y ventanas asimétricas combinadas con `CASE WHEN` para generar alertas automáticas.

### Recursos adicionales

| Recurso | URL |
|---|---|
| Documentación Snowflake: Window Functions | https://docs.snowflake.com/en/sql-reference/functions-analytic |
| Documentación Snowflake: QUALIFY | https://docs.snowflake.com/en/sql-reference/constructs/qualify |
| Documentación Snowflake: LAG | https://docs.snowflake.com/en/sql-reference/functions/lag |
| Documentación Snowflake: LEAD | https://docs.snowflake.com/en/sql-reference/functions/lead |
| Documentación Snowflake: Window frame syntax | https://docs.snowflake.com/en/sql-reference/functions-window-syntax |
| Mode Analytics: SQL Window Functions Tutorial | https://mode.com/sql-tutorial/sql-window-functions/ |

---
