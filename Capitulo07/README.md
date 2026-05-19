# Optimización y mejora de performance de queries

## Metadatos

| Campo | Detalle |
|---|---|
| **Duración estimada** | 45 minutos |
| **Complejidad** | Difícil |
| **Nivel Bloom** | Aplicar (*Apply*) |
| **Módulo** | 7 — Optimización y performance SQL |
| **Plataforma** | Snowflake (Snowsight Worksheet) |
| **Schema de práctica** | `LAB_SQL_INTERMEDIO.VENTAS` |

---

## Descripción General

En este laboratorio aplicarás de forma integrada los principios de organización estructural, estilo SQL y optimización de performance aprendidos a lo largo del curso. Recibirás cinco queries funcionales pero mal escritos que resuelven problemas reales del dataset compartido (`LAB_SQL_INTERMEDIO`). Tu tarea es primero reformatearlos y documentarlos sin cambiar su lógica, luego identificar y corregir anti-patrones de performance, y finalmente medir el impacto de cada optimización usando el Query Profile de Snowsight. El laboratorio cierra con la construcción de una consulta final optimizada que integra todas las técnicas del curso.

A diferencia de un laboratorio puramente conceptual, aquí vas a trabajar como lo harías en un entorno profesional: primero ejecutas la versión original, registras sus métricas, analizas el Query Profile, reescribes el query con buenas prácticas y vuelves a medir. La meta no es memorizar reglas aisladas, sino construir criterio para decidir cuándo una consulta está escrita de forma mantenible y cuándo además está preparada para escalar sobre volúmenes mayores.

> 💡 **Nota importante:** En datasets pequeños, las diferencias de `bytes_scanned`, `partitions_scanned` o `tiempo_ms` pueden ser pequeñas o incluso iguales por caché, tamaño del warehouse o pocas micro-particiones. Eso no invalida el laboratorio. Lo importante es que aprendas el patrón correcto de medición y el razonamiento detrás de cada optimización.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Aplicar principios de organización por capas lógicas (CTEs con nombres descriptivos, indentación consistente y comentarios estratégicos) para refactorizar queries complejos sin alterar su lógica de negocio.
- [ ] Identificar y corregir anti-patrones de performance en Snowflake: `SELECT *`, funciones escalares en `WHERE`, CTEs repetidas innecesariamente y subqueries correlacionadas.
- [ ] Usar el Query Profile de Snowsight para comparar el plan de ejecución, tiempo por nodo y bytes escaneados antes y después de cada optimización.
- [ ] Consultar `INFORMATION_SCHEMA.QUERY_HISTORY` para obtener métricas objetivas de tiempo de ejecución y volumen de datos procesados.
- [ ] Construir una versión final optimizada de la consulta más compleja del curso aplicando todas las buenas prácticas consolidadas.

---

## Prerrequisitos

### Conocimientos previos

| Área | Nivel requerido |
|---|---|
| CTEs (`WITH ... AS`) y subqueries | Sólido — Laboratorios 1 al 6 |
| `JOIN` (`INNER JOIN`, `LEFT JOIN`) | Sólido |
| Funciones de agregación (`COUNT`, `SUM`, `AVG`) | Sólido |
| Window functions (`ROW_NUMBER`, `RANK`, `LAG`, `QUALIFY`) | Intermedio |
| Fechas y filtros por rango | Intermedio |
| Lectura básica de Query Profile en Snowsight | Básico |
| Concepto de micro-particiones y pruning en Snowflake | Conceptual |

### Acceso y configuración

| Requisito | Detalle |
|---|---|
| Cuenta Snowflake activa | Trial o corporativa con rol que pueda crear objetos en una base de laboratorio |
| Script de setup ejecutado | No se asume script previo. Esta práctica incluye el setup completo de base, schema, tablas y datos. |
| Database disponible | `LAB_SQL_INTERMEDIO` |
| Schema disponible | `LAB_SQL_INTERMEDIO.VENTAS` |
| Tablas requeridas | `CLIENTES`, `PRODUCTOS`, `PEDIDOS`, `VENTAS`, creadas en el Paso 0 |
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
| Visual Studio Code *(opcional)* | 1.80+ | Edición local de scripts SQL |
| SnowSQL *(opcional)* | 1.2.x+ | Ejecución desde terminal |

---

## Organización recomendada de Workspace en Snowsight

Para que la práctica sea ordenada y reutilizable, trabaja con un Workspace y 2 folders. En esta práctica se usa la palabra **workspace** como separación lógica de trabajo dentro de Snowsight.

| Workspace / Worksheet | Folder | Nombre sugerido | Uso |
|---|---|---|---|
| `SNOWLABS-INT` | `SETUP-LABS` | `07_SETUP_DATOS_PERFORMANCE` | Crear database, schema, tablas y dataset controlado para performance. Se ejecuta una vez al inicio o cuando quieras reiniciar el laboratorio. |
| `SNOWLABS-INT` | `SCRIPT-LABS` | `07_LAB_OPTIMIZACION_PERFORMANCE` | Ejecutar los ejercicios del laboratorio sin mezclar el script de carga de datos. |

### Paso 0.0 — Crear el workspace de las prácticas

1. Entra a **Snowsight**.
2. Da clic en la opción **Projects**.
3. Clic en **+**.
4. Selecciona la opción **Private workspace**.
5. Nómbralo: **`SNOWLABS-INT`**.
6. Clic en **Create**.

### Paso 0.0.1 — Crear el folder y script que carga los datos

1. Dentro del nuevo workspace da clic en **+ Add new**.
2. Clic en **Folder** y nómbralo: **`SETUP-LABS`**.
3. Dentro del folder **SETUP-LABS**, da clic en el símbolo **+**.
4. Crea un archivo de tipo **SQL**.
5. Nómbralo: **`07_SETUP_DATOS_PERFORMANCE`**.
6. Pega ahí el siguiente script completo.
7. Ejecuta el script completo antes de comenzar el laboratorio.

Este dataset está diseñado para activar todos los escenarios de la práctica:

- Ventas y pedidos de **2022 y 2023** para comparar periodos.
- Regiones con más de 10 pedidos completados para el query de ventas por región.
- Categorías `ELECTRONICA`, `ROPA`, `HOGAR` y `SOFTWARE` para ejercicios de agregación.
- Clientes activos e inactivos para el ejercicio de subquery correlacionada.
- Datos suficientes por región-categoría para que el reporte final de top 3 no devuelva vacío.
- Algunos pedidos cancelados, en proceso, montos cero y ajustes negativos para demostrar filtros de negocio.

```sql
-- 07_setup_datos_performance_snowflake.sql
-- Práctica Snowflake Intermedio
-- Dataset controlado para completar el laboratorio:
-- Optimización y mejora de performance de queries
--
-- Objetivo del dataset:
-- 1) Tener datos de 2022 y 2023 para comparaciones anuales.
-- 2) Tener suficientes pedidos completados por región y categoría.
-- 3) Tener clientes activos e inactivos para filtros de negocio.
-- 4) Tener ventas positivas, cero y negativas para demostrar filtros.
-- 5) Tener volumen suficiente para usar Query Profile y QUERY_HISTORY.

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
    CLIENTE_ID NUMBER(10,0) NOT NULL,
    NOMBRE VARCHAR(120) NOT NULL,
    EMAIL VARCHAR(150),
    REGION_NOMBRE VARCHAR(80),
    CIUDAD VARCHAR(80),
    PAIS VARCHAR(80),
    FECHA_REGISTRO DATE,
    ACTIVO BOOLEAN,
    CONSTRAINT PK_CLIENTES PRIMARY KEY (CLIENTE_ID)
);

CREATE OR REPLACE TABLE PRODUCTOS (
    PRODUCTO_ID NUMBER(10,0) NOT NULL,
    NOMBRE_PRODUCTO VARCHAR(120) NOT NULL,
    CATEGORIA_PRODUCTO VARCHAR(80) NOT NULL,
    PRECIO_UNITARIO NUMBER(12,2) NOT NULL,
    ACTIVO BOOLEAN,
    CONSTRAINT PK_PRODUCTOS PRIMARY KEY (PRODUCTO_ID)
);

CREATE OR REPLACE TABLE PEDIDOS (
    PEDIDO_ID NUMBER(10,0) NOT NULL,
    CLIENTE_ID NUMBER(10,0) NOT NULL,
    FECHA_PEDIDO DATE NOT NULL,
    ESTADO VARCHAR(30) NOT NULL,
    MONTO_TOTAL NUMBER(12,2) NOT NULL,
    CANAL VARCHAR(40),
    CONSTRAINT PK_PEDIDOS PRIMARY KEY (PEDIDO_ID),
    CONSTRAINT FK_PEDIDOS_CLIENTES FOREIGN KEY (CLIENTE_ID) REFERENCES CLIENTES(CLIENTE_ID)
);

CREATE OR REPLACE TABLE VENTAS (
    VENTA_ID NUMBER(10,0) NOT NULL,
    PEDIDO_ID NUMBER(10,0) NOT NULL,
    PRODUCTO_ID NUMBER(10,0) NOT NULL,
    VENDEDOR_ID NUMBER(10,0) NOT NULL,
    FECHA_VENTA DATE NOT NULL,
    MONTO NUMBER(12,2) NOT NULL,
    CATEGORIA_PRODUCTO VARCHAR(80) NOT NULL,
    REGION VARCHAR(80),
    CANAL VARCHAR(40),
    DESCRIPCION_DETALLE VARCHAR(500),
    CONSTRAINT PK_VENTAS PRIMARY KEY (VENTA_ID),
    CONSTRAINT FK_VENTAS_PEDIDOS FOREIGN KEY (PEDIDO_ID) REFERENCES PEDIDOS(PEDIDO_ID),
    CONSTRAINT FK_VENTAS_PRODUCTOS FOREIGN KEY (PRODUCTO_ID) REFERENCES PRODUCTOS(PRODUCTO_ID)
);

INSERT INTO CLIENTES
    (CLIENTE_ID, NOMBRE, EMAIL, REGION_NOMBRE, CIUDAD, PAIS, FECHA_REGISTRO, ACTIVO)
VALUES
    (1,  'Cliente Norte 01',      'cliente01@techcommerce.com', 'NORTE',      'Monterrey',    'México', '2021-01-15', TRUE),
    (2,  'Cliente Norte 02',      'cliente02@techcommerce.com', 'NORTE',      'Saltillo',     'México', '2021-02-10', TRUE),
    (3,  'Cliente Norte 03',      'cliente03@techcommerce.com', 'NORTE',      'Torreón',      'México', '2021-03-12', TRUE),
    (4,  'Cliente Norte 04',      'cliente04@techcommerce.com', 'NORTE',      'Chihuahua',    'México', '2021-04-08', TRUE),
    (5,  'Cliente Norte 05',      'cliente05@techcommerce.com', 'NORTE',      'Monterrey',    'México', '2021-05-19', TRUE),
    (6,  'Cliente Norte 06',      'cliente06@techcommerce.com', 'NORTE',      'Saltillo',     'México', '2021-06-21', FALSE),

    (7,  'Cliente Centro 01',     'cliente07@techcommerce.com', 'CENTRO',     'CDMX',         'México', '2021-01-22', TRUE),
    (8,  'Cliente Centro 02',     'cliente08@techcommerce.com', 'CENTRO',     'Toluca',       'México', '2021-02-18', TRUE),
    (9,  'Cliente Centro 03',     'cliente09@techcommerce.com', 'CENTRO',     'Querétaro',    'México', '2021-03-26', TRUE),
    (10, 'Cliente Centro 04',     'cliente10@techcommerce.com', 'CENTRO',     'CDMX',         'México', '2021-04-14', TRUE),
    (11, 'Cliente Centro 05',     'cliente11@techcommerce.com', 'CENTRO',     'Pachuca',      'México', '2021-05-17', TRUE),
    (12, 'Cliente Centro 06',     'cliente12@techcommerce.com', 'CENTRO',     'Cuernavaca',   'México', '2021-06-29', FALSE),

    (13, 'Cliente Occidente 01',  'cliente13@techcommerce.com', 'OCCIDENTE',  'Guadalajara',  'México', '2021-01-30', TRUE),
    (14, 'Cliente Occidente 02',  'cliente14@techcommerce.com', 'OCCIDENTE',  'León',         'México', '2021-02-13', TRUE),
    (15, 'Cliente Occidente 03',  'cliente15@techcommerce.com', 'OCCIDENTE',  'Aguascalientes','México','2021-03-11', TRUE),
    (16, 'Cliente Occidente 04',  'cliente16@techcommerce.com', 'OCCIDENTE',  'Guadalajara',  'México', '2021-04-23', TRUE),
    (17, 'Cliente Occidente 05',  'cliente17@techcommerce.com', 'OCCIDENTE',  'Morelia',      'México', '2021-05-05', TRUE),
    (18, 'Cliente Occidente 06',  'cliente18@techcommerce.com', 'OCCIDENTE',  'Colima',       'México', '2021-06-16', FALSE),

    (19, 'Cliente Sur 01',        'cliente19@techcommerce.com', 'SUR',        'Mérida',       'México', '2021-01-09', TRUE),
    (20, 'Cliente Sur 02',        'cliente20@techcommerce.com', 'SUR',        'Cancún',       'México', '2021-02-24', TRUE),
    (21, 'Cliente Sur 03',        'cliente21@techcommerce.com', 'SUR',        'Puebla',       'México', '2021-03-07', TRUE),
    (22, 'Cliente Sur 04',        'cliente22@techcommerce.com', 'SUR',        'Veracruz',     'México', '2021-04-27', TRUE),
    (23, 'Cliente Sur 05',        'cliente23@techcommerce.com', 'SUR',        'Mérida',       'México', '2021-05-31', TRUE),
    (24, 'Cliente Sur 06',        'cliente24@techcommerce.com', 'SUR',        'Oaxaca',       'México', '2021-06-03', FALSE);

INSERT INTO PRODUCTOS
    (PRODUCTO_ID, NOMBRE_PRODUCTO, CATEGORIA_PRODUCTO, PRECIO_UNITARIO, ACTIVO)
VALUES
    (1, 'Laptop Pro 14',             'ELECTRONICA',  1800.00, TRUE),
    (2, 'Monitor UltraWide',         'ELECTRONICA',   700.00, TRUE),
    (3, 'Camisa Ejecutiva',          'ROPA',          120.00, TRUE),
    (4, 'Chamarra Técnica',          'ROPA',          260.00, TRUE),
    (5, 'Silla Ergonómica',          'HOGAR',         520.00, TRUE),
    (6, 'Escritorio Ajustable',      'HOGAR',         680.00, TRUE),
    (7, 'Licencia BI Empresarial',   'SOFTWARE',     1500.00, TRUE),
    (8, 'Licencia CRM Empresarial',  'SOFTWARE',     2100.00, TRUE);

-- Generar 360 pedidos distribuidos en 24 meses:
-- 15 pedidos por mes desde enero 2022 hasta diciembre 2023.
-- El patrón asegura suficientes filas por región, categoría y cliente.
INSERT INTO PEDIDOS (PEDIDO_ID, CLIENTE_ID, FECHA_PEDIDO, ESTADO, MONTO_TOTAL, CANAL)
WITH numeros AS (
    SELECT SEQ4() AS n
    FROM TABLE(GENERATOR(ROWCOUNT => 360))
),
base AS (
    SELECT
        n,
        10000 + n AS pedido_id,
        MOD(n, 24) + 1 AS cliente_id,
        DATEADD(
            day,
            MOD(n * 3, 27),
            DATEADD(month, FLOOR(n / 15), DATE '2022-01-01')
        ) AS fecha_pedido,
        CASE
            WHEN MOD(n, 20) = 0 THEN 'CANCELADO'
            WHEN MOD(n, 17) = 0 THEN 'EN_PROCESO'
            ELSE 'COMPLETADO'
        END AS estado,
        CASE
            WHEN MOD(n, 55) = 0 THEN 0
            WHEN MOD(n, 71) = 0 THEN -25
            ELSE 250 + MOD(n * 137, 4200)
        END AS monto_total,
        CASE MOD(n, 4)
            WHEN 0 THEN 'Web'
            WHEN 1 THEN 'Ejecutivo'
            WHEN 2 THEN 'Marketplace'
            ELSE 'Partner'
        END AS canal
    FROM numeros
)
SELECT
    pedido_id,
    cliente_id,
    fecha_pedido,
    estado,
    monto_total,
    canal
FROM base;

-- Generar una venta por pedido.
-- La categoría se deriva de PRODUCTOS para mantener consistencia.
INSERT INTO VENTAS
    (VENTA_ID, PEDIDO_ID, PRODUCTO_ID, VENDEDOR_ID, FECHA_VENTA, MONTO, CATEGORIA_PRODUCTO, REGION, CANAL, DESCRIPCION_DETALLE)
WITH pedidos_con_producto AS (
    SELECT
        p.PEDIDO_ID,
        p.CLIENTE_ID,
        p.FECHA_PEDIDO,
        p.MONTO_TOTAL,
        p.CANAL,
        MOD(FLOOR((p.PEDIDO_ID - 10000) / 3), 8) + 1 AS producto_id,
        MOD((p.PEDIDO_ID - 10000), 12) + 1 AS vendedor_id
    FROM PEDIDOS p
)
SELECT
    20000 + ROW_NUMBER() OVER (ORDER BY p.PEDIDO_ID) AS venta_id,
    p.PEDIDO_ID,
    p.PRODUCTO_ID,
    p.VENDEDOR_ID,
    p.FECHA_PEDIDO AS fecha_venta,
    p.MONTO_TOTAL  AS monto,
    pr.CATEGORIA_PRODUCTO,
    c.REGION_NOMBRE AS region,
    p.CANAL,
    'Venta generada para práctica de performance. Incluye texto descriptivo para demostrar el costo potencial de SELECT *.' AS descripcion_detalle
FROM pedidos_con_producto p
INNER JOIN PRODUCTOS pr
    ON p.PRODUCTO_ID = pr.PRODUCTO_ID
INNER JOIN CLIENTES c
    ON p.CLIENTE_ID = c.CLIENTE_ID;

-- Validación rápida del dataset.
SELECT 'CLIENTES' AS TABLA, COUNT(*) AS FILAS FROM CLIENTES
UNION ALL
SELECT 'PRODUCTOS' AS TABLA, COUNT(*) AS FILAS FROM PRODUCTOS
UNION ALL
SELECT 'PEDIDOS' AS TABLA, COUNT(*) AS FILAS FROM PEDIDOS
UNION ALL
SELECT 'VENTAS' AS TABLA, COUNT(*) AS FILAS FROM VENTAS;

-- Resultado esperado:
-- CLIENTES  = 24
-- PRODUCTOS = 8
-- PEDIDOS   = 360
-- VENTAS    = 360

-- Validación de distribución por año.
SELECT
    YEAR(FECHA_PEDIDO) AS ANIO,
    COUNT(*) AS TOTAL_PEDIDOS,
    COUNT_IF(ESTADO = 'COMPLETADO') AS PEDIDOS_COMPLETADOS,
    ROUND(SUM(MONTO_TOTAL), 2) AS MONTO_TOTAL
FROM PEDIDOS
GROUP BY YEAR(FECHA_PEDIDO)
ORDER BY ANIO;

-- Resultado esperado: 2022 y 2023, con 180 pedidos por año.

-- Validación de distribución por región y categoría para el reporte final.
SELECT
    c.REGION_NOMBRE,
    v.CATEGORIA_PRODUCTO,
    COUNT(DISTINCT p.CLIENTE_ID) AS CLIENTES_DISTINTOS,
    COUNT(*) AS TOTAL_VENTAS,
    ROUND(SUM(v.MONTO), 2) AS MONTO_TOTAL
FROM VENTAS v
INNER JOIN PEDIDOS p
    ON v.PEDIDO_ID = p.PEDIDO_ID
INNER JOIN CLIENTES c
    ON p.CLIENTE_ID = c.CLIENTE_ID
WHERE
    p.FECHA_PEDIDO >= '2023-01-01'
    AND p.FECHA_PEDIDO < '2024-01-01'
    AND p.ESTADO = 'COMPLETADO'
    AND v.MONTO > 0
GROUP BY c.REGION_NOMBRE, v.CATEGORIA_PRODUCTO
ORDER BY c.REGION_NOMBRE, v.CATEGORIA_PRODUCTO;

-- Resultado esperado: deben existir varias combinaciones región-categoría
-- con al menos 5 clientes distintos para que el reporte final devuelva datos.
```

### Paso 0.0.2 — Crear el folder y script de laboratorio

1. Da clic en el botón **+ Add new**.
2. Clic en **Folder** y nómbralo: **`SCRIPT-LABS`**.
3. Dentro de **SCRIPT-LABS**, crea un archivo de tipo **SQL**.
4. Nómbralo: **`07_LAB_OPTIMIZACION_PERFORMANCE`**.
5. Usa este archivo para ejecutar los ejercicios 1 al 7.
6. **No pegues aquí el script de carga completo; solo usa las consultas de análisis del laboratorio.**

---

### Paso 0.1 — Confirmar que las tablas quedaron disponibles

Dentro del archivo **`07_LAB_OPTIMIZACION_PERFORMANCE`**, ejecuta lo siguiente:

```sql
USE WAREHOUSE COMPUTE_WH;
USE DATABASE LAB_SQL_INTERMEDIO;
USE SCHEMA VENTAS;

SHOW TABLES;
```

**Resultado esperado:** deben aparecer al menos estas tablas:

| Tabla | Uso en la práctica |
|---|---|
| `CLIENTES` | Datos maestros de clientes, región, estado activo y fechas de registro. |
| `PRODUCTOS` | Catálogo de productos y categorías. |
| `PEDIDOS` | Hechos transaccionales: cliente, fecha, estado, monto total y canal. |
| `VENTAS` | Hechos de venta con producto, categoría, región, vendedor y monto. |

### Paso 0.2 — Validar volumen mínimo de datos

Ejecuta:

```sql
SELECT 'CLIENTES' AS TABLA, COUNT(*) AS FILAS FROM CLIENTES
UNION ALL
SELECT 'PRODUCTOS' AS TABLA, COUNT(*) AS FILAS FROM PRODUCTOS
UNION ALL
SELECT 'PEDIDOS' AS TABLA, COUNT(*) AS FILAS FROM PEDIDOS
UNION ALL
SELECT 'VENTAS' AS TABLA, COUNT(*) AS FILAS FROM VENTAS;
```

**Resultado esperado:**

| TABLA | FILAS |
|---|---:|
| CLIENTES | 24 |
| PRODUCTOS | 8 |
| PEDIDOS | 360 |
| VENTAS | 360 |

### Paso 0.3 — Validar distribución por año, región y categoría

Ejecuta:

```sql
SELECT
    YEAR(FECHA_PEDIDO) AS ANIO,
    ESTADO,
    COUNT(*) AS TOTAL_PEDIDOS
FROM PEDIDOS
GROUP BY YEAR(FECHA_PEDIDO), ESTADO
ORDER BY ANIO, ESTADO;
```

**Resultado esperado:** deben aparecer registros para `2022` y `2023`, incluyendo estados `COMPLETADO`, `CANCELADO` y `EN_PROCESO`.

Ejecuta también:

```sql
SELECT
    REGION,
    CATEGORIA_PRODUCTO,
    COUNT(*) AS TOTAL_VENTAS,
    COUNT(DISTINCT PEDIDO_ID) AS PEDIDOS_DISTINTOS,
    ROUND(SUM(MONTO), 2) AS MONTO_TOTAL
FROM VENTAS
GROUP BY REGION, CATEGORIA_PRODUCTO
ORDER BY REGION, CATEGORIA_PRODUCTO;
```

**Resultado esperado:** deben aparecer 4 regiones y 4 categorías.

### Paso 0.4 — Validar que existen datos suficientes para la consulta final

```sql
SELECT
    c.REGION_NOMBRE,
    v.CATEGORIA_PRODUCTO,
    COUNT(DISTINCT p.CLIENTE_ID) AS CLIENTES_DISTINTOS,
    COUNT(*) AS TOTAL_VENTAS,
    ROUND(SUM(v.MONTO), 2) AS INGRESOS_2023
FROM VENTAS v
INNER JOIN PEDIDOS p
    ON v.PEDIDO_ID = p.PEDIDO_ID
INNER JOIN CLIENTES c
    ON p.CLIENTE_ID = c.CLIENTE_ID
WHERE
    p.FECHA_PEDIDO >= '2023-01-01'
    AND p.FECHA_PEDIDO < '2024-01-01'
    AND p.ESTADO = 'COMPLETADO'
    AND c.REGION_NOMBRE IS NOT NULL
    AND v.MONTO > 0
GROUP BY c.REGION_NOMBRE, v.CATEGORIA_PRODUCTO
HAVING COUNT(DISTINCT p.CLIENTE_ID) >= 5
ORDER BY c.REGION_NOMBRE, v.CATEGORIA_PRODUCTO;
```

**Resultado esperado:** debe devolver al menos una combinación región-categoría. Si devuelve 0 filas, vuelve a ejecutar el setup completo.

---

## Ejercicios Paso a Paso

---

### Ejercicio 1 — Preparación: registrar el estado inicial del warehouse

**Objetivo:** Establecer una línea base de métricas antes de ejecutar cualquier query optimizado, y familiarizarte con `QUERY_HISTORY` como herramienta de medición objetiva.

#### Instrucciones

**Paso 1.1 — Abrir el script de laboratorio**

En Snowsight, abre el worksheet **`07_LAB_OPTIMIZACION_PERFORMANCE`** dentro del folder **`SCRIPT-LABS`**.

**Paso 1.2 — Registrar métricas recientes de la sesión**

Ejecuta el siguiente query para obtener las métricas de los últimos queries ejecutados en tu sesión. Lo usarás como referencia comparativa durante todo el laboratorio:

```sql
-- ============================================================
-- Vista base de métricas de ejecución
-- Usaremos esta consulta antes y después de cada optimización
-- ============================================================

SELECT
    query_id,
    query_text,
    total_elapsed_time        AS tiempo_ms,
    bytes_scanned             AS bytes_escaneados,
    rows_produced             AS filas_resultado,
    compilation_time          AS compilacion_ms,
    execution_time            AS ejecucion_ms,
    start_time
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION(
    RESULT_LIMIT => 20
))
WHERE query_type = 'SELECT'
ORDER BY start_time DESC;
```

**Paso 1.3 — Medición por `QUERY_ID`**

Cuando compares dos queries, es recomendable copiar el `QUERY_ID` desde Snowsight y consultar sus métricas de forma explícita. Esto evita capturar accidentalmente la consulta de `QUERY_HISTORY` en lugar del query que quieres medir.

**Nota:** Si el query id usado no funciona prueba otro, inclusive puede que probando todos no de resultados.

```sql
-- Plantilla recomendada para medir un query específico.
-- Reemplaza <PEGAR_QUERY_ID> por el query_id real copiado desde Snowsight.

SELECT
    query_id,
    total_elapsed_time AS tiempo_ms,
    bytes_scanned AS bytes_escaneados,
    rows_produced AS filas_resultado,
    partitions_scanned AS particiones_escaneadas,
    partitions_total AS particiones_totales,
    ROUND(
        partitions_scanned * 100.0 / NULLIF(partitions_total, 0),
        2
    ) AS pct_particiones_escaneadas
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_id = '<PEGAR_QUERY_ID>';
```

#### Salida esperada

Una tabla con hasta 20 filas mostrando los queries recientes de tu sesión. Si acabas de iniciar sesión, puede aparecer vacía o con muy pocas filas — esto es normal.

#### Verificación

✅ La consulta se ejecuta sin errores y devuelve columnas con nombres legibles en español.  
✅ Puedes identificar `query_id`, `tiempo_ms`, `bytes_escaneados` y `particiones_escaneadas`.  
✅ Entiendes que las métricas pueden variar por caché, concurrencia, tamaño del warehouse y tamaño real de micro-particiones.

---

### Ejercicio 2 — Refactorización de estilo: Query 1 ventas por región

**Objetivo:** Aplicar los principios de organización por capas, indentación consistente y comentarios estratégicos a un query desestructurado, sin cambiar su lógica.

#### Instrucciones

**Paso 2.1 — Ejecutar el query original**

Lee y ejecuta el siguiente **query original** tal como está. Observa su resultado y anota el `query_id` desde `QUERY_HISTORY`:

```sql
select r.region_nombre,count(distinct p.pedido_id) as total_pedidos,sum(v.monto) as ingresos,avg(v.monto) as ticket_promedio from ventas v join pedidos p on v.pedido_id=p.pedido_id join clientes c on p.cliente_id=c.cliente_id join (select distinct cliente_id,region_nombre from clientes where region_nombre is not null) r on c.cliente_id=r.cliente_id where p.fecha_pedido>='2023-01-01' and p.estado='COMPLETADO' and v.monto>0 group by r.region_nombre having count(distinct p.pedido_id)>10 order by ingresos desc;
```

**Paso 2.2 — Ejecutar la versión refactorizada**

Ahora escribe la **versión refactorizada** aplicando principios de organización por capas, nombres descriptivos, indentación consistente y comentarios estratégicos. A continuación se muestra la solución de referencia — intenta escribirla tú primero antes de consultarla:

```sql
-- ============================================================
-- Resumen de ventas por región geográfica
-- Periodo: desde 2023-01-01 | Solo pedidos COMPLETADOS
-- Excluye regiones sin nombre y pedidos con monto <= 0
-- Autor: Equipo Analytics | Laboratorio 7
-- ============================================================

WITH regiones_validas AS (
    -- Solo clientes con región asignada (excluye registros legacy sin región)
    SELECT DISTINCT
        cliente_id,
        region_nombre
    FROM clientes
    WHERE region_nombre IS NOT NULL
),

pedidos_completados AS (
    -- Pedidos completados desde el inicio del año fiscal 2023
    SELECT
        p.pedido_id,
        p.cliente_id,
        p.fecha_pedido
    FROM pedidos p
    WHERE
        p.fecha_pedido >= '2023-01-01'
        AND p.estado    = 'COMPLETADO'
),

ventas_positivas AS (
    -- Excluimos ajustes contables con monto negativo o cero
    SELECT
        v.pedido_id,
        v.monto
    FROM ventas v
    WHERE v.monto > 0
),

metricas_por_region AS (
    SELECT
        r.region_nombre,
        COUNT(DISTINCT pc.pedido_id) AS total_pedidos,
        SUM(vp.monto)                AS ingresos,
        AVG(vp.monto)                AS ticket_promedio
    FROM pedidos_completados pc
    INNER JOIN ventas_positivas vp
        ON pc.pedido_id = vp.pedido_id
    INNER JOIN clientes c
        ON pc.cliente_id = c.cliente_id
    INNER JOIN regiones_validas r
        ON c.cliente_id = r.cliente_id
    GROUP BY r.region_nombre
    HAVING COUNT(DISTINCT pc.pedido_id) > 10
)

-- Resultado final: regiones ordenadas por ingresos totales descendente
SELECT
    region_nombre,
    total_pedidos,
    ingresos,
    ROUND(ticket_promedio, 2) AS ticket_promedio
FROM metricas_por_region
ORDER BY ingresos DESC;
```

**Paso 2.3 — Validar equivalencia con `EXCEPT`**

Para confirmar que la refactorización no cambió la lógica de negocio, puedes comparar ambos resultados con `EXCEPT` en ambas direcciones.

```sql
-- Validación A: original menos refactorizado
WITH resultado_original AS (
    SELECT
        r.region_nombre,
        COUNT(DISTINCT p.pedido_id) AS total_pedidos,
        SUM(v.monto) AS ingresos,
        ROUND(AVG(v.monto), 2) AS ticket_promedio
    FROM ventas v
    JOIN pedidos p
        ON v.pedido_id = p.pedido_id
    JOIN clientes c
        ON p.cliente_id = c.cliente_id
    JOIN (
        SELECT DISTINCT
            cliente_id,
            region_nombre
        FROM clientes
        WHERE region_nombre IS NOT NULL
    ) r
        ON c.cliente_id = r.cliente_id
    WHERE
        p.fecha_pedido >= '2023-01-01'
        AND p.estado = 'COMPLETADO'
        AND v.monto > 0
    GROUP BY r.region_nombre
    HAVING COUNT(DISTINCT p.pedido_id) > 10
),
resultado_refactorizado AS (
    WITH regiones_validas AS (
        SELECT DISTINCT cliente_id, region_nombre
        FROM clientes
        WHERE region_nombre IS NOT NULL
    ),
    pedidos_completados AS (
        SELECT pedido_id, cliente_id, fecha_pedido
        FROM pedidos
        WHERE fecha_pedido >= '2023-01-01'
          AND estado = 'COMPLETADO'
    ),
    ventas_positivas AS (
        SELECT pedido_id, monto
        FROM ventas
        WHERE monto > 0
    )
    SELECT
        r.region_nombre,
        COUNT(DISTINCT pc.pedido_id) AS total_pedidos,
        SUM(vp.monto) AS ingresos,
        ROUND(AVG(vp.monto), 2) AS ticket_promedio
    FROM pedidos_completados pc
    INNER JOIN ventas_positivas vp
        ON pc.pedido_id = vp.pedido_id
    INNER JOIN clientes c
        ON pc.cliente_id = c.cliente_id
    INNER JOIN regiones_validas r
        ON c.cliente_id = r.cliente_id
    GROUP BY r.region_nombre
    HAVING COUNT(DISTINCT pc.pedido_id) > 10
)
SELECT *
FROM resultado_original
EXCEPT
SELECT *
FROM resultado_refactorizado;
```

```sql
-- Validación B: refactorizado menos original
WITH resultado_original AS (
    SELECT
        r.region_nombre,
        COUNT(DISTINCT p.pedido_id) AS total_pedidos,
        SUM(v.monto) AS ingresos,
        ROUND(AVG(v.monto), 2) AS ticket_promedio
    FROM ventas v
    JOIN pedidos p
        ON v.pedido_id = p.pedido_id
    JOIN clientes c
        ON p.cliente_id = c.cliente_id
    JOIN (
        SELECT DISTINCT
            cliente_id,
            region_nombre
        FROM clientes
        WHERE region_nombre IS NOT NULL
    ) r
        ON c.cliente_id = r.cliente_id
    WHERE
        p.fecha_pedido >= '2023-01-01'
        AND p.estado = 'COMPLETADO'
        AND v.monto > 0
    GROUP BY r.region_nombre
    HAVING COUNT(DISTINCT p.pedido_id) > 10
),
resultado_refactorizado AS (
    WITH regiones_validas AS (
        SELECT DISTINCT cliente_id, region_nombre
        FROM clientes
        WHERE region_nombre IS NOT NULL
    ),
    pedidos_completados AS (
        SELECT pedido_id, cliente_id, fecha_pedido
        FROM pedidos
        WHERE fecha_pedido >= '2023-01-01'
          AND estado = 'COMPLETADO'
    ),
    ventas_positivas AS (
        SELECT pedido_id, monto
        FROM ventas
        WHERE monto > 0
    )
    SELECT
        r.region_nombre,
        COUNT(DISTINCT pc.pedido_id) AS total_pedidos,
        SUM(vp.monto) AS ingresos,
        ROUND(AVG(vp.monto), 2) AS ticket_promedio
    FROM pedidos_completados pc
    INNER JOIN ventas_positivas vp
        ON pc.pedido_id = vp.pedido_id
    INNER JOIN clientes c
        ON pc.cliente_id = c.cliente_id
    INNER JOIN regiones_validas r
        ON c.cliente_id = r.cliente_id
    GROUP BY r.region_nombre
    HAVING COUNT(DISTINCT pc.pedido_id) > 10
)
SELECT *
FROM resultado_refactorizado
EXCEPT
SELECT *
FROM resultado_original;
```

**Resultado esperado:** ambas validaciones deben devolver 0 filas.

**Paso 2.4 — Revisar Query Profile**

En Snowsight, haz clic en el ícono de **Query Profile** para ambas versiones y compara visualmente los nodos de ejecución.

#### Salida esperada

Ambos queries deben devolver exactamente el mismo número de filas y los mismos valores. El resultado es una tabla con columnas `REGION_NOMBRE`, `TOTAL_PEDIDOS`, `INGRESOS` y `TICKET_PROMEDIO`, ordenada de mayor a menor ingreso.

#### Verificación

✅ Los resultados de ambas versiones son idénticos.  
✅ La versión refactorizada tiene al menos 4 CTEs con nombres descriptivos en español.  
✅ Cada CTE tiene un comentario que explica el *por qué* del filtro, no solo el *qué*.  
✅ Las validaciones con `EXCEPT` devuelven 0 filas.

---

### Ejercicio 3 — Anti-patrón 1: eliminar `SELECT *` y columnas innecesarias

**Objetivo:** Identificar el impacto de `SELECT *` en queries sobre tablas grandes y reemplazarlo con selección explícita de columnas.

#### Instrucciones

**Paso 3.1 — Ejecutar query con anti-patrón**

Ejecuta el siguiente query que usa `SELECT *` y anota sus métricas de `bytes_scanned`:

```sql
-- QUERY ANTI-PATRÓN: SELECT * con join a tabla grande
-- Ejecutar para medir bytes_scanned ANTES de la corrección

SELECT *
FROM ventas v
JOIN pedidos p
    ON v.pedido_id = p.pedido_id
WHERE p.fecha_pedido BETWEEN '2023-01-01' AND '2023-12-31';
```

**Paso 3.2 — Capturar métricas**

Inmediatamente después, consulta QUERY_HISTORY para capturar los bytes escaneados:

```sql
-- Capturar métricas recientes.
-- Si aparece primero esta misma consulta de QUERY_HISTORY,
-- copia el query_id del SELECT * desde Snowsight y usa la plantilla del Paso 1.4.

SELECT
    query_id,
    SUBSTR(query_text, 1, 80)    AS query_resumen,
    bytes_scanned                AS bytes_escaneados,
    total_elapsed_time           AS tiempo_ms,
    compilation_time             AS compilacion_ms,
    rows_produced                AS filas_resultado
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION(
    RESULT_LIMIT => 10
))
WHERE query_type = 'SELECT'
ORDER BY start_time DESC;
```

**Paso 3.3 — Ejecutar versión optimizada**

Ahora ejecuta la versión corregida, seleccionando **solo las columnas necesarias para un reporte de ventas**:

```sql
-- ============================================================
-- Reporte de ventas 2023: versión optimizada sin SELECT *
-- Solo columnas necesarias para el análisis de negocio
-- ============================================================

SELECT
    v.venta_id,
    v.pedido_id,
    v.monto,
    v.fecha_venta,
    p.cliente_id,
    p.estado           AS estado_pedido,
    p.fecha_pedido
FROM ventas v
INNER JOIN pedidos p
    ON v.pedido_id = p.pedido_id
WHERE
    p.fecha_pedido BETWEEN '2023-01-01' AND '2023-12-31';
```

**Paso 3.4 — Comparar métricas**

Vuelve a capturar las métricas con `QUERY_HISTORY` y **compara** `bytes_scanned` entre ambas versiones.

**Paso 3.5 — Revisar TableScan en Query Profile**

Abre el **Query Profile** de ambas ejecuciones en Snowsight. Observa la diferencia en el nodo **TableScan**: el número de columnas proyectadas debe ser menor en la versión optimizada.

#### Salida esperada

La versión con `SELECT *` mostrará un valor de `bytes_escaneados` mayor o igual al de la versión optimizada. En tablas con muchas columnas anchas, la diferencia puede ser significativa. Ambas versiones deben devolver el mismo número de filas.

#### Verificación

✅ La versión optimizada selecciona exactamente 7 columnas con nombres explícitos.  
✅ El `bytes_scanned` de la versión optimizada es ≤ al de la versión con `SELECT *`.  
✅ En el Query Profile, el nodo TableScan de la versión optimizada muestra menos columnas proyectadas.

> 📝 **Nota pedagógica:** En Snowflake, el impacto de `SELECT *` es menor que en bases de datos relacionales tradicionales porque el almacenamiento es columnar. Sin embargo, sigue siendo una mala práctica porque: (1) dificulta la lectura, (2) puede romper pipelines si cambia el esquema de la tabla, y (3) transfiere datos innecesarios a la capa de resultado.

---

### Ejercicio 4 — Anti-patrón 2: funciones escalares en `WHERE` que bloquean el pruning

**Objetivo:** Identificar cómo el uso de funciones sobre columnas en condiciones `WHERE` puede impedir que Snowflake aplique pruning de micro-particiones de forma óptima, y reescribir los filtros para aprovechar esta optimización.

#### Instrucciones

**Paso 4.1 — Ejecutar query con función escalar en `WHERE`**

Ejecuta el siguiente query que aplica `YEAR()` y `MONTH()` directamente sobre la columna de fecha en el `WHERE`. Anota `partitions_scanned` y `tiempo_ms`:

```sql
-- ANTI-PATRÓN: Función escalar sobre columna en WHERE
-- Snowflake puede tener menos oportunidades de pruning cuando la columna
-- está envuelta en una función, especialmente en tablas grandes.

SELECT
    p.pedido_id,
    p.cliente_id,
    p.fecha_pedido,
    p.monto_total
FROM pedidos p
WHERE YEAR(p.fecha_pedido)  = 2023
  AND MONTH(p.fecha_pedido) = 6;
```

**Paso 4.2 — Capturar métricas**

```sql
SELECT
    query_id,
    total_elapsed_time    AS tiempo_ms,
    compilation_time      AS compilacion_ms,
    execution_time        AS ejecucion_ms,
    bytes_scanned         AS bytes_escaneados,
    rows_produced         AS filas_resultado
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION(
    RESULT_LIMIT => 10
))
WHERE query_type = 'SELECT'
ORDER BY start_time DESC;
```

**Paso 4.3 — Ejecutar versión optimizada con rango de fechas**

```sql
-- ============================================================
-- Pedidos de junio 2023: versión optimizada con rango de fechas
-- Usar rangos permite a Snowflake aplicar partition pruning
-- y escanear solo las micro-particiones del rango solicitado
-- cuando el volumen y la distribución lo permiten.
-- ============================================================

SELECT
    p.pedido_id,
    p.cliente_id,
    p.fecha_pedido,
    p.monto_total
FROM pedidos p
WHERE
    p.fecha_pedido >= '2023-06-01'
    AND p.fecha_pedido  < '2023-07-01';
```

**Paso 4.4 — Comparar Query Profile**

Captura las métricas nuevamente y compara `partitions_scanned` entre ambas versiones.

En el Query Profile, busca el nodo **TableScan** y revisa la relación `partitions_scanned / partitions_total`. Una relación más baja en la versión optimizada indica mejor pruning.

#### Salida esperada

Ambas versiones deben devolver exactamente las mismas filas. La versión con rango de fechas debe mostrar un valor de `partitions_scanned` igual o menor al de la versión con `YEAR()`/`MONTH()`, especialmente si la tabla tiene datos de múltiples meses y múltiples años.

#### Verificación

✅ Ambas versiones devuelven el mismo conjunto de filas.  
✅ La versión con rango de fechas tiene `partitions_scanned` ≤ a la versión con funciones en `WHERE`.  
✅ El Query Profile de la versión optimizada muestra una mejor relación `partitions_scanned / partitions_total`, si el dataset tiene suficientes micro-particiones.

> ⚠️ **Nota realista:** En un dataset de laboratorio pequeño, Snowflake puede escanear las mismas particiones en ambos casos. El principio sigue siendo correcto: filtra por rangos sobre la columna original para maximizar el pruning cuando el volumen crece.

---

### Ejercicio 5 — Anti-patrón 3: subquery correlacionada vs. `JOIN`

**Objetivo:** Identificar una subquery correlacionada que se ejecuta conceptualmente una vez por cada fila del query exterior, y reemplazarla con un `JOIN` equivalente que Snowflake puede optimizar como una sola operación.

#### Instrucciones

**Paso 5.1 — Ejecutar query con subquery correlacionada**

Ejecuta el siguiente query con subquery correlacionada. Anota `tiempo_ms` y `bytes_scanned`:

```sql
-- ANTI-PATRÓN: Subquery correlacionada
-- Este patrón calcula el total de compras por cada cliente activo.
-- En tablas grandes, este estilo puede multiplicar el trabajo lógico.

SELECT
    c.cliente_id,
    c.nombre,
    c.email,
    (
        SELECT SUM(p.monto_total)
        FROM pedidos p
        WHERE p.cliente_id = c.cliente_id       -- correlación: referencia al exterior
          AND p.estado = 'COMPLETADO'
          AND p.fecha_pedido >= '2023-01-01'
    ) AS total_compras_2023
FROM clientes c
WHERE c.activo = TRUE
ORDER BY total_compras_2023 DESC NULLS LAST;
```

**Paso 5.2 — Capturar métricas**

Usa `QUERY_HISTORY` o copia el `query_id` desde Snowsight para capturar sus métricas.

**Paso 5.3 — Ejecutar versión optimizada con agregación previa y `LEFT JOIN`**

```sql
-- ============================================================
-- Total de compras 2023 por cliente activo
-- Versión optimizada: JOIN reemplaza subquery correlacionada
-- Snowflake puede ejecutar la agregación como un solo bloque
-- antes de unirla con la dimensión de clientes.
-- ============================================================

WITH compras_2023 AS (
    -- Pre-agregamos por cliente ANTES del join
    -- Esto evita repetir la lógica de agregación por cada fila de clientes
    SELECT
        cliente_id,
        SUM(monto_total) AS total_compras_2023
    FROM pedidos
    WHERE
        estado       = 'COMPLETADO'
        AND fecha_pedido >= '2023-01-01'
    GROUP BY cliente_id
)

SELECT
    c.cliente_id,
    c.nombre,
    c.email,
    co.total_compras_2023
FROM clientes c
LEFT JOIN compras_2023 co
    ON c.cliente_id = co.cliente_id
WHERE c.activo = TRUE
ORDER BY co.total_compras_2023 DESC NULLS LAST;
```

**Paso 5.4 — Validar equivalencia**

```sql
-- Validación: ambas versiones deben producir el mismo número de clientes activos.
WITH version_correlacionada AS (
    SELECT
        c.cliente_id,
        c.nombre,
        c.email,
        (
            SELECT SUM(p.monto_total)
            FROM pedidos p
            WHERE p.cliente_id = c.cliente_id
              AND p.estado = 'COMPLETADO'
              AND p.fecha_pedido >= '2023-01-01'
        ) AS total_compras_2023
    FROM clientes c
    WHERE c.activo = TRUE
),
version_join AS (
    WITH compras_2023 AS (
        SELECT
            cliente_id,
            SUM(monto_total) AS total_compras_2023
        FROM pedidos
        WHERE
            estado = 'COMPLETADO'
            AND fecha_pedido >= '2023-01-01'
        GROUP BY cliente_id
    )
    SELECT
        c.cliente_id,
        c.nombre,
        c.email,
        co.total_compras_2023
    FROM clientes c
    LEFT JOIN compras_2023 co
        ON c.cliente_id = co.cliente_id
    WHERE c.activo = TRUE
)
SELECT
    (SELECT COUNT(*) FROM version_correlacionada) AS filas_version_correlacionada,
    (SELECT COUNT(*) FROM version_join)           AS filas_version_join;
```

#### Salida esperada

Ambas versiones deben devolver exactamente las mismas filas y valores. La versión con `LEFT JOIN` debe tener un `tiempo_ms` igual o menor, especialmente conforme crece el volumen de pedidos. Los clientes sin compras en 2023 aparecen con `NULL` en `total_compras_2023` en ambas versiones.

#### Verificación

✅ Ambas versiones devuelven el mismo número de filas con los mismos valores.  
✅ La versión con `JOIN` tiene `tiempo_ms` ≤ a la versión con subquery correlacionada, si el volumen permite observar diferencias.  
✅ El Query Profile de la versión optimizada debería mostrar un plan más simple y fácil de interpretar.

---

### Ejercicio 6 — Anti-patrón 4: CTE referenciada múltiples veces

**Objetivo:** Identificar una CTE que se referencia más de una vez en el mismo query, lo que puede implicar recálculo o un plan menos legible, y reestructurar el query para calcular las métricas con una sola definición lógica.

#### Instrucciones

**Paso 6.1 — Ejecutar query con CTE referenciada múltiples veces**

Ejecuta el siguiente query donde la CTE `ventas_base` se usa **tres veces** en el query final:

```sql
-- ANTI-PATRÓN: CTE referenciada múltiples veces
-- En Snowflake, una CTE puede ser optimizada por el motor,
-- pero si la lógica es costosa y se referencia varias veces,
-- el query se vuelve menos legible y puede implicar más trabajo.
-- Lo importante aquí es aprender a reconocer el patrón.

WITH ventas_base AS (
    SELECT
        v.venta_id,
        v.pedido_id,
        v.monto,
        v.categoria_producto,
        p.fecha_pedido,
        p.cliente_id
    FROM ventas v
    INNER JOIN pedidos p
        ON v.pedido_id = p.pedido_id
    WHERE p.fecha_pedido >= '2023-01-01'
      AND p.estado = 'COMPLETADO'
)

SELECT
    'Total general'                   AS metrica,
    COUNT(*)                          AS valor_conteo,
    SUM(monto)                        AS valor_suma
FROM ventas_base

UNION ALL

SELECT
    'Categoría: ELECTRONICA'          AS metrica,
    COUNT(*)                          AS valor_conteo,
    SUM(monto)                        AS valor_suma
FROM ventas_base                      -- segunda referencia
WHERE categoria_producto = 'ELECTRONICA'

UNION ALL

SELECT
    'Categoría: ROPA'                 AS metrica,
    COUNT(*)                          AS valor_conteo,
    SUM(monto)                        AS valor_suma
FROM ventas_base                      -- tercera referencia
WHERE categoria_producto = 'ROPA';
```

**Paso 6.2 — Capturar métricas**

Captura las métricas de esta ejecución.

**Paso 6.3 — Ejecutar versión optimizada con `GROUPING SETS`**

La versión con `GROUPING SETS` calcula subtotales por categoría y total general en una sola estructura de agregación. Esta forma es más limpia para múltiples categorías y evita repetir la misma CTE en varios bloques con `UNION ALL`.

```sql
-- ============================================================
-- Resumen de ventas 2023 por categoría seleccionada
-- Versión optimizada: GROUPING SETS para subtotales y total general
-- Reemplaza múltiples referencias a la misma CTE con una sola consulta
-- de agregación reutilizable.
-- ============================================================

WITH ventas_base AS (
    -- Fuente única: ventas completadas desde inicio de 2023
    SELECT
        v.monto,
        v.categoria_producto
    FROM ventas v
    INNER JOIN pedidos p
        ON v.pedido_id = p.pedido_id
    WHERE
        p.fecha_pedido >= '2023-01-01'
        AND p.estado    = 'COMPLETADO'
)

SELECT
    CASE
        WHEN GROUPING(categoria_producto) = 1 THEN 'Total general'
        ELSE 'Categoría: ' || categoria_producto
    END AS metrica,
    COUNT(*) AS valor_conteo,
    SUM(monto) AS valor_suma
FROM ventas_base
GROUP BY GROUPING SETS (
    (categoria_producto),   -- subtotal por categoría
    ()                      -- total general
)
HAVING
    GROUPING(categoria_producto) = 1
    OR categoria_producto IN ('ELECTRONICA', 'ROPA')
ORDER BY
    CASE
        WHEN metrica = 'Total general' THEN 1
        WHEN metrica = 'Categoría: ELECTRONICA' THEN 2
        WHEN metrica = 'Categoría: ROPA' THEN 3
        ELSE 4
    END;
```

**Paso 6.4 — Validar equivalencia de resultados**

```sql
-- Validación de equivalencia entre UNION ALL y GROUPING SETS.
-- Ambas versiones deben devolver las mismas 3 filas:
-- Total general, Categoría: ELECTRONICA y Categoría: ROPA.

WITH ventas_base AS (
    SELECT
        v.monto,
        v.categoria_producto,
        p.fecha_pedido,
        p.estado
    FROM ventas v
    INNER JOIN pedidos p
        ON v.pedido_id = p.pedido_id
    WHERE p.fecha_pedido >= '2023-01-01'
      AND p.estado = 'COMPLETADO'
),

resultado_union AS (
    SELECT
        'Total general' AS metrica,
        COUNT(*) AS valor_conteo,
        SUM(monto) AS valor_suma
    FROM ventas_base

    UNION ALL

    SELECT
        'Categoría: ELECTRONICA' AS metrica,
        COUNT(*) AS valor_conteo,
        SUM(monto) AS valor_suma
    FROM ventas_base
    WHERE categoria_producto = 'ELECTRONICA'

    UNION ALL

    SELECT
        'Categoría: ROPA' AS metrica,
        COUNT(*) AS valor_conteo,
        SUM(monto) AS valor_suma
    FROM ventas_base
    WHERE categoria_producto = 'ROPA'
),

resultado_grouping_sets AS (
    SELECT
        CASE
            WHEN GROUPING(categoria_producto) = 1 THEN 'Total general'
            ELSE 'Categoría: ' || categoria_producto
        END AS metrica,
        COUNT(*) AS valor_conteo,
        SUM(monto) AS valor_suma
    FROM ventas_base
    GROUP BY GROUPING SETS (
        (categoria_producto),
        ()
    )
    HAVING
        GROUPING(categoria_producto) = 1
        OR categoria_producto IN ('ELECTRONICA', 'ROPA')
)

SELECT *
FROM resultado_union
EXCEPT
SELECT *
FROM resultado_grouping_sets;
```

**Resultado esperado:** 0 filas ó sin resultados.

Ejecuta también la comparación inversa:

```sql
WITH ventas_base AS (
    SELECT
        v.monto,
        v.categoria_producto,
        p.fecha_pedido,
        p.estado
    FROM ventas v
    INNER JOIN pedidos p
        ON v.pedido_id = p.pedido_id
    WHERE p.fecha_pedido >= '2023-01-01'
      AND p.estado = 'COMPLETADO'
),

resultado_union AS (
    SELECT
        'Total general' AS metrica,
        COUNT(*) AS valor_conteo,
        SUM(monto) AS valor_suma
    FROM ventas_base

    UNION ALL

    SELECT
        'Categoría: ELECTRONICA' AS metrica,
        COUNT(*) AS valor_conteo,
        SUM(monto) AS valor_suma
    FROM ventas_base
    WHERE categoria_producto = 'ELECTRONICA'

    UNION ALL

    SELECT
        'Categoría: ROPA' AS metrica,
        COUNT(*) AS valor_conteo,
        SUM(monto) AS valor_suma
    FROM ventas_base
    WHERE categoria_producto = 'ROPA'
),

resultado_grouping_sets AS (
    SELECT
        CASE
            WHEN GROUPING(categoria_producto) = 1 THEN 'Total general'
            ELSE 'Categoría: ' || categoria_producto
        END AS metrica,
        COUNT(*) AS valor_conteo,
        SUM(monto) AS valor_suma
    FROM ventas_base
    GROUP BY GROUPING SETS (
        (categoria_producto),
        ()
    )
    HAVING
        GROUPING(categoria_producto) = 1
        OR categoria_producto IN ('ELECTRONICA', 'ROPA')
)

SELECT *
FROM resultado_grouping_sets
EXCEPT
SELECT *
FROM resultado_union;
```

**Resultado esperado:** 0 filas o sin resultados.

#### Salida esperada

La versión con `GROUPING SETS` debe devolver 3 filas: una para `Total general`, una para `ELECTRONICA` y una para `ROPA`. Los valores deben coincidir con los del query original con `UNION ALL`.

#### Verificación

✅ Ambas versiones producen el mismo resultado.  
✅ La versión con `GROUPING SETS` expresa subtotales y total general en una sola consulta de agregación.  
✅ Las métricas de `bytes_scanned` de la versión optimizada son ≤ a la versión con `UNION ALL`, si el optimizador y el volumen del dataset permiten observar diferencias.

---

### Ejercicio 7 — Consulta final integrada: síntesis de todos los módulos

**Objetivo:** Construir desde cero la consulta más compleja del curso, integrando CTEs organizadas por capas, window functions, filtros optimizados y documentación completa, aplicando todas las buenas prácticas consolidadas.

#### Instrucciones

El siguiente es el **requerimiento de negocio** que debes implementar:

> *"Necesitamos un reporte que muestre, para cada región y categoría de producto, el top 3 de clientes por ingresos en el año 2023, junto con su variación porcentual respecto al año anterior (2022). Solo incluir combinaciones región-categoría con al menos 5 clientes distintos. Ordenar por región, categoría y ranking."*

**Paso 7.1 — Diseñar las capas antes de escribir el SQL**

Antes de escribir el query, diseña la estructura de capas en comentarios:

```sql
-- DISEÑO DE CAPAS:
-- CTE 1: ventas_con_contexto → Ventas de 2022 y 2023 con cliente, región y categoría
-- CTE 2: ingresos_2022      → Agregado por cliente-categoría-región para 2022
-- CTE 3: ingresos_2023      → Agregado por cliente-categoría-región para 2023
-- CTE 4: comparacion_anual  → JOIN entre 2022 y 2023 + cálculo de variación %
-- CTE 5: ranking_clientes   → ROW_NUMBER() por región-categoría
-- CTE 6: combinaciones_validas → Filtro: solo combinaciones con >= 5 clientes
-- CTE 7: top3_por_segmento  → Top 3 por región-categoría
-- SELECT final              → Presentación ejecutiva
```

**Paso 7.2 — Implementar el query completo**

```sql
-- ============================================================
-- Top 3 clientes por región-categoría con variación anual
-- Requerimiento: Análisis comparativo 2022 vs 2023
-- Solo combinaciones con >= 5 clientes distintos
-- Técnicas: CTEs, window functions, LEFT JOIN, filtros optimizados
-- Autor: [Tu nombre] | Laboratorio 7 — Consulta Final
-- ============================================================

WITH ventas_con_contexto AS (
    -- Base: ventas de 2022 y 2023 enriquecidas con cliente y región.
    -- Filtro por rango de fechas para aprovechar partition pruning.
    SELECT
        v.monto,
        v.categoria_producto,
        p.cliente_id,
        p.fecha_pedido,
        YEAR(p.fecha_pedido) AS anio,
        c.region_nombre
    FROM ventas v
    INNER JOIN pedidos p
        ON v.pedido_id = p.pedido_id
    INNER JOIN clientes c
        ON p.cliente_id = c.cliente_id
    WHERE
        p.fecha_pedido >= '2022-01-01'
        AND p.fecha_pedido  < '2024-01-01'
        AND p.estado        = 'COMPLETADO'
        AND c.region_nombre IS NOT NULL
        AND v.monto         > 0
),

ingresos_2022 AS (
    -- Ingresos por cliente, categoría y región en el año base (2022).
    SELECT
        cliente_id,
        categoria_producto,
        region_nombre,
        SUM(monto) AS ingresos_2022
    FROM ventas_con_contexto
    WHERE anio = 2022
    GROUP BY
        cliente_id,
        categoria_producto,
        region_nombre
),

ingresos_2023 AS (
    -- Ingresos por cliente, categoría y región en el año de análisis (2023).
    SELECT
        cliente_id,
        categoria_producto,
        region_nombre,
        SUM(monto) AS ingresos_2023
    FROM ventas_con_contexto
    WHERE anio = 2023
    GROUP BY
        cliente_id,
        categoria_producto,
        region_nombre
),

comparacion_anual AS (
    -- Comparación 2022 vs 2023 con variación porcentual.
    -- LEFT JOIN para incluir clientes nuevos en 2023 sin historial en 2022.
    SELECT
        i23.cliente_id,
        i23.categoria_producto,
        i23.region_nombre,
        i23.ingresos_2023,
        i22.ingresos_2022,
        CASE
            WHEN i22.ingresos_2022 IS NULL
              OR i22.ingresos_2022 = 0
                THEN NULL
            ELSE
                ROUND(
                    (i23.ingresos_2023 - i22.ingresos_2022)
                    / i22.ingresos_2022 * 100,
                    2
                )
        END AS variacion_pct
    FROM ingresos_2023 i23
    LEFT JOIN ingresos_2022 i22
        ON  i23.cliente_id         = i22.cliente_id
        AND i23.categoria_producto = i22.categoria_producto
        AND i23.region_nombre      = i22.region_nombre
),

ranking_clientes AS (
    -- Ranking de clientes dentro de cada región-categoría por ingresos 2023.
    -- ROW_NUMBER garantiza ranking único sin empates.
    SELECT
        cliente_id,
        categoria_producto,
        region_nombre,
        ingresos_2023,
        ingresos_2022,
        variacion_pct,
        ROW_NUMBER() OVER (
            PARTITION BY region_nombre, categoria_producto
            ORDER BY ingresos_2023 DESC
        ) AS ranking
    FROM comparacion_anual
),

combinaciones_validas AS (
    -- Solo combinaciones región-categoría con al menos 5 clientes distintos.
    -- Esto filtra segmentos con muestra estadística insuficiente.
    SELECT
        region_nombre,
        categoria_producto
    FROM ingresos_2023
    GROUP BY
        region_nombre,
        categoria_producto
    HAVING COUNT(DISTINCT cliente_id) >= 5
),

top3_por_segmento AS (
    -- Aplicamos el filtro de top 3 y el filtro de combinaciones válidas.
    SELECT
        rc.region_nombre,
        rc.categoria_producto,
        rc.cliente_id,
        rc.ingresos_2023,
        rc.ingresos_2022,
        rc.variacion_pct,
        rc.ranking
    FROM ranking_clientes rc
    INNER JOIN combinaciones_validas cv
        ON  rc.region_nombre      = cv.region_nombre
        AND rc.categoria_producto = cv.categoria_producto
    WHERE rc.ranking <= 3
)

-- ============================================================
-- Resultado final: top 3 clientes por región-categoría
-- Enriquecido con nombre de cliente para presentación ejecutiva
-- ============================================================
SELECT
    ts.region_nombre,
    ts.categoria_producto,
    ts.ranking,
    c.nombre                                             AS cliente,
    ROUND(ts.ingresos_2023, 2)                           AS ingresos_2023,
    ROUND(COALESCE(ts.ingresos_2022, 0), 2)              AS ingresos_2022,
    COALESCE(CAST(ts.variacion_pct AS VARCHAR), 'Nuevo') AS variacion_pct
FROM top3_por_segmento ts
INNER JOIN clientes c
    ON ts.cliente_id = c.cliente_id
ORDER BY
    ts.region_nombre      ASC,
    ts.categoria_producto ASC,
    ts.ranking            ASC;
```

**Paso 7.3 — Revisar Query Profile**

Ejecuta el query y verifica que devuelve resultados coherentes. Después abre el **Query Profile** de esta ejecución. Identifica y anota:

- El nodo que más tiempo consume.
- La relación `partitions_scanned / partitions_total` en los nodos TableScan.
- El número total de bytes escaneados.
- Si hay `HashJoin`, `Aggregate` o `WindowFunction` en el plan.
- Si el filtro de fecha aparece temprano en el plan.

**Paso 7.4 — Capturar métricas finales con `QUERY_HISTORY`**

```sql
-- Métricas de la consulta final integrada.
-- Si necesitas precisión, reemplaza por la plantilla de query_id del Paso 1.4.

SELECT
    query_id,
    total_elapsed_time           AS tiempo_ms,
    bytes_scanned                AS bytes_escaneados,
    rows_produced                AS filas_resultado,
    compilation_time             AS compilacion_ms,
    execution_time               AS ejecucion_ms,
    ROUND(
        execution_time * 100.0 / NULLIF(total_elapsed_time, 0),
        1
    )                            AS pct_ejecucion
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION(
    RESULT_LIMIT => 10
))
WHERE query_type = 'SELECT'
ORDER BY start_time DESC;
```

#### Salida esperada

Una tabla con columnas `REGION_NOMBRE`, `CATEGORIA_PRODUCTO`, `RANKING` (1, 2 o 3), `CLIENTE`, `INGRESOS_2023`, `INGRESOS_2022` y `VARIACION_PCT`. Solo aparecen combinaciones región-categoría con al menos 5 clientes. Los clientes nuevos en 2023 muestran `'Nuevo'` en la columna de variación.

#### Verificación

✅ El query tiene al menos 7 CTEs con nombres descriptivos en español.  
✅ Cada CTE tiene un comentario de bloque explicando su propósito de negocio.  
✅ No hay `SELECT *` en ninguna capa.  
✅ Los filtros de fecha usan rangos explícitos (`>=` y `<`) en lugar de funciones como `YEAR()` en `WHERE`.  
✅ El `ranking` por región-categoría va de 1 a 3 sin saltos.  
✅ El reporte devuelve filas con combinaciones región-categoría válidas.

---

## Validación y Pruebas Finales

Ejecuta el siguiente bloque de validación al finalizar todos los pasos. Cada sentencia debe devolver el resultado indicado.

### Validación 1 — La consulta final devuelve solo rankings 1, 2 y 3

```sql
-- VALIDACIÓN 1: La consulta final devuelve solo rankings 1, 2 y 3.
-- Resultado esperado: 0 filas.

WITH ventas_con_contexto AS (
    SELECT
        v.monto,
        v.categoria_producto,
        p.cliente_id,
        p.fecha_pedido,
        YEAR(p.fecha_pedido) AS anio,
        c.region_nombre
    FROM ventas v
    INNER JOIN pedidos p
        ON v.pedido_id = p.pedido_id
    INNER JOIN clientes c
        ON p.cliente_id = c.cliente_id
    WHERE
        p.fecha_pedido >= '2022-01-01'
        AND p.fecha_pedido < '2024-01-01'
        AND p.estado = 'COMPLETADO'
        AND c.region_nombre IS NOT NULL
        AND v.monto > 0
),
ingresos_2022 AS (
    SELECT
        cliente_id,
        categoria_producto,
        region_nombre,
        SUM(monto) AS ingresos_2022
    FROM ventas_con_contexto
    WHERE anio = 2022
    GROUP BY cliente_id, categoria_producto, region_nombre
),
ingresos_2023 AS (
    SELECT
        cliente_id,
        categoria_producto,
        region_nombre,
        SUM(monto) AS ingresos_2023
    FROM ventas_con_contexto
    WHERE anio = 2023
    GROUP BY cliente_id, categoria_producto, region_nombre
),
comparacion_anual AS (
    SELECT
        i23.cliente_id,
        i23.categoria_producto,
        i23.region_nombre,
        i23.ingresos_2023,
        i22.ingresos_2022,
        CASE
            WHEN i22.ingresos_2022 IS NULL OR i22.ingresos_2022 = 0 THEN NULL
            ELSE ROUND((i23.ingresos_2023 - i22.ingresos_2022) / i22.ingresos_2022 * 100, 2)
        END AS variacion_pct
    FROM ingresos_2023 i23
    LEFT JOIN ingresos_2022 i22
        ON  i23.cliente_id = i22.cliente_id
        AND i23.categoria_producto = i22.categoria_producto
        AND i23.region_nombre = i22.region_nombre
),
ranking_clientes AS (
    SELECT
        cliente_id,
        categoria_producto,
        region_nombre,
        ingresos_2023,
        ingresos_2022,
        variacion_pct,
        ROW_NUMBER() OVER (
            PARTITION BY region_nombre, categoria_producto
            ORDER BY ingresos_2023 DESC
        ) AS ranking
    FROM comparacion_anual
),
combinaciones_validas AS (
    SELECT
        region_nombre,
        categoria_producto
    FROM ingresos_2023
    GROUP BY region_nombre, categoria_producto
    HAVING COUNT(DISTINCT cliente_id) >= 5
),
top3_por_segmento AS (
    SELECT
        rc.region_nombre,
        rc.categoria_producto,
        rc.cliente_id,
        rc.ranking
    FROM ranking_clientes rc
    INNER JOIN combinaciones_validas cv
        ON rc.region_nombre = cv.region_nombre
       AND rc.categoria_producto = cv.categoria_producto
    WHERE rc.ranking <= 3
)
SELECT DISTINCT ranking
FROM top3_por_segmento
WHERE ranking NOT IN (1, 2, 3);
```

**Resultado esperado:** 0 filas o sin resultados.


### Validación 2 — La sesión ejecutó suficientes queries `SELECT`

```sql
-- VALIDACIÓN 2: QUERY_HISTORY muestra al menos 8 queries SELECT en la sesión.
-- Resultado esperado: >= 8.

SELECT COUNT(*) AS total_queries_select
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION(
    RESULT_LIMIT => 50
))
WHERE query_type = 'SELECT';
```

### Validación 3 — Hay datos suficientes para el reporte final

```sql
-- VALIDACIÓN 3: Confirmar combinaciones región-categoría válidas.
-- Resultado esperado: al menos 1 fila.

WITH ingresos_2023 AS (
    SELECT
        c.region_nombre,
        v.categoria_producto,
        p.cliente_id,
        SUM(v.monto) AS ingresos_2023
    FROM ventas v
    INNER JOIN pedidos p
        ON v.pedido_id = p.pedido_id
    INNER JOIN clientes c
        ON p.cliente_id = c.cliente_id
    WHERE
        p.fecha_pedido >= '2023-01-01'
        AND p.fecha_pedido < '2024-01-01'
        AND p.estado = 'COMPLETADO'
        AND v.monto > 0
        AND c.region_nombre IS NOT NULL
    GROUP BY
        c.region_nombre,
        v.categoria_producto,
        p.cliente_id
)
SELECT
    region_nombre,
    categoria_producto,
    COUNT(DISTINCT cliente_id) AS clientes_distintos
FROM ingresos_2023
GROUP BY region_nombre, categoria_producto
HAVING COUNT(DISTINCT cliente_id) >= 5
ORDER BY region_nombre, categoria_producto;
```

### Validación 4 — El warehouse usado es `COMPUTE_WH`

```sql
-- VALIDACIÓN 4: Confirmar warehouse actual.
-- Resultado esperado: COMPUTE_WH.

SELECT CURRENT_WAREHOUSE() AS warehouse_actual;
```

---

## Resultados esperados clave con el dataset cargado

Estos resultados ayudan al instructor y al alumno a validar rápidamente que el laboratorio se está ejecutando sobre el dataset correcto.

| Consulta / Validación | Resultado esperado |
|---|---:|
| Conteo de `CLIENTES` | 24 filas |
| Conteo de `PRODUCTOS` | 8 filas |
| Conteo de `PEDIDOS` | 360 filas |
| Conteo de `VENTAS` | 360 filas |
| Años disponibles en `PEDIDOS` | 2022 y 2023 |
| Regiones disponibles | 4 |
| Categorías disponibles | 4 |
| Query de ventas por región | Debe devolver regiones con más de 10 pedidos |
| Query final integrada | Debe devolver rankings 1, 2 y 3 |
| Validación de rankings fuera de rango | 0 filas |
| Query final | No debe usar `SELECT *` |

---

## Solución de Problemas

### Problema 1 — El Query Profile no muestra diferencias de particiones entre la versión con funciones y la versión con rango de fechas

**Síntoma:** Ambas versiones del Ejercicio 4 muestran exactamente el mismo `partitions_scanned`, y el Query Profile no refleja ninguna mejora de pruning.

**Causa:** La tabla `PEDIDOS` en el entorno de laboratorio puede contener pocos datos o estar almacenada en pocas micro-particiones. El pruning de particiones solo es visible cuando los datos están distribuidos en múltiples particiones temporales y la tabla tiene suficiente volumen. En datasets pequeños, Snowflake puede decidir escanear todas las particiones porque el costo total sigue siendo muy bajo.

**Solución:**

1. Verifica el volumen y rango de fechas de la tabla:

```sql
SELECT
    COUNT(*) AS total_pedidos,
    MIN(fecha_pedido) AS fecha_minima,
    MAX(fecha_pedido) AS fecha_maxima,
    COUNT(DISTINCT DATE_TRUNC('month', fecha_pedido)) AS meses_con_datos
FROM pedidos;
```

2. Si el rango de fechas es menor a 6 meses o el volumen es bajo, el efecto de pruning será mínimo.
3. Acepta el resultado como válido conceptualmente: el principio de evitar funciones en `WHERE` es correcto y su impacto escala con el volumen de datos.
4. Documenta en tus notas que este anti-patrón tiene impacto proporcional al volumen de datos y al rango de fechas cubierto.

---

### Problema 2 — La consulta final del Ejercicio 7 devuelve 0 filas aunque los datos existen

**Síntoma:** El query del Ejercicio 7 se ejecuta sin errores pero devuelve una tabla vacía. Al ejecutar CTEs individuales sí aparecen datos.

**Causa:** El filtro `HAVING COUNT(DISTINCT cliente_id) >= 5` en la CTE `combinaciones_validas` puede ser demasiado restrictivo si el dataset fue modificado o si el setup no se ejecutó completo. Si ninguna combinación región-categoría supera el umbral de 5 clientes distintos, el `INNER JOIN` final elimina todas las filas.

**Solución:**

1. Ejecuta primero la validación de combinaciones:

```sql
SELECT
    c.region_nombre,
    v.categoria_producto,
    COUNT(DISTINCT p.cliente_id) AS clientes_distintos
FROM ventas v
INNER JOIN pedidos p
    ON v.pedido_id = p.pedido_id
INNER JOIN clientes c
    ON p.cliente_id = c.cliente_id
WHERE
    p.fecha_pedido >= '2023-01-01'
    AND p.fecha_pedido < '2024-01-01'
    AND p.estado = 'COMPLETADO'
    AND v.monto > 0
    AND c.region_nombre IS NOT NULL
GROUP BY
    c.region_nombre,
    v.categoria_producto
ORDER BY clientes_distintos DESC;
```

2. Si el máximo de `clientes_distintos` es menor a 5, ajusta temporalmente el umbral a `>= 2` o `>= 3` para que el laboratorio produzca resultados visibles.
3. Documenta el cambio con un comentario:

```sql
-- Umbral ajustado a 3 por volumen reducido del dataset de laboratorio.
```

4. Si estás usando el dataset oficial de esta práctica, vuelve a ejecutar el setup completo.

---

### Problema 3 — `QUERY_HISTORY` muestra primero la consulta de medición, no la consulta que querías medir

**Síntoma:** Después de ejecutar una consulta de medición, el resultado de `QUERY_HISTORY_BY_SESSION` muestra como primera fila la propia consulta de `QUERY_HISTORY`.

**Causa:** `QUERY_HISTORY_BY_SESSION` devuelve los queries más recientes de la sesión. Si ejecutas la medición inmediatamente después del query que quieres analizar, es posible que la medición aparezca también en la lista.

**Solución:**

1. Copia el `query_id` del query real desde Snowsight.
2. Usa la plantilla del Paso 1.4.
3. Filtra por ese `query_id`:

```sql
SELECT
    query_id,
    total_elapsed_time AS tiempo_ms,
    bytes_scanned AS bytes_escaneados,
    rows_produced AS filas_resultado,
    partitions_scanned AS particiones_escaneadas,
    partitions_total AS particiones_totales
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_id = '<PEGAR_QUERY_ID>';
```

---

### Problema 4 — La versión optimizada no es más rápida que la original

**Síntoma:** El query optimizado tiene el mismo tiempo de ejecución o incluso más tiempo que el query original.

**Causa:** En Snowflake, el tiempo de ejecución puede variar por caché de resultados, tamaño del warehouse, concurrencia, compilación del query, tamaño del dataset, optimizaciones internas y micro-particiones. En datasets pequeños, la diferencia puede ser imperceptible.

**Solución:**

1. Compara no solo `tiempo_ms`, sino también:
   - `bytes_scanned`
   - `partitions_scanned`
   - `rows_produced`
   - número de columnas proyectadas
   - simplicidad del plan en Query Profile

2. Desactiva mentalmente la expectativa de que toda optimización reduzca tiempo en un dataset pequeño. Algunas optimizaciones son principalmente de:
   - legibilidad;
   - mantenibilidad;
   - seguridad ante cambios de esquema;
   - escalabilidad futura.

3. Documenta el resultado observado. Una optimización responsable siempre se mide, aunque el impacto en laboratorio sea pequeño.

---

## Limpieza del Entorno

Al finalizar el laboratorio, ejecuta los siguientes comandos para liberar recursos y evitar consumo innecesario de créditos Snowflake:

```sql
-- ============================================================
-- LIMPIEZA POST-LABORATORIO 7
-- Ejecutar obligatoriamente al terminar la sesión
-- ============================================================

-- 1. Suspender el warehouse para detener el consumo de créditos
ALTER WAREHOUSE COMPUTE_WH SUSPEND;

-- 2. Verificar que el warehouse quedó suspendido
SHOW WAREHOUSES LIKE 'COMPUTE_WH';
-- Confirmar que la columna STATE = 'SUSPENDED'

-- 3. No se requiere DROP de objetos.
-- Las tablas de este laboratorio pueden reutilizarse para repaso.
```

> ✅ **Confirmación de limpieza:** El warehouse `COMPUTE_WH` debe aparecer con `STATE = 'SUSPENDED'` en el resultado de `SHOW WAREHOUSES`. Si el estado sigue siendo `STARTED`, repite el comando `ALTER WAREHOUSE COMPUTE_WH SUSPEND`.

---

## Resumen

En este laboratorio aplicaste de forma integrada las técnicas de optimización SQL más importantes para entornos Snowflake profesionales:

| Técnica aplicada | Beneficio principal |
|---|---|
| Organización por capas con CTEs | Legibilidad, mantenibilidad y depuración más rápida |
| Indentación y comentarios estratégicos | Comunicación efectiva con el equipo y contigo mismo en el futuro |
| Eliminación de `SELECT *` | Menor transferencia de datos y esquemas más robustos |
| Filtros con rangos en lugar de funciones | Mejor partition pruning y menos bytes escaneados cuando el volumen lo permite |
| `JOIN` en lugar de subqueries correlacionadas | Planes más claros y escalabilidad en tablas grandes |
| `GROUPING SETS` para subtotales y total general | Menos repetición lógica y consultas más expresivas |
| Medición con Query Profile y `QUERY_HISTORY` | Decisiones de optimización basadas en evidencia, no en intuición |

### Principios consolidados del curso

1. **Escribe para humanos primero, para la máquina segundo.** Un query que nadie entiende no puede mantenerse ni auditarse.
2. **Mide antes de optimizar.** El Query Profile y `QUERY_HISTORY` son tus aliados para tomar decisiones informadas.
3. **Filtra temprano y con precisión.** Los filtros sobre columnas directas, sin funciones innecesarias, permiten que Snowflake haga su trabajo de pruning.
4. **Una CTE, una responsabilidad.** Cada bloque debe tener un nombre que describa exactamente qué contiene.
5. **No confundas legibilidad con lentitud.** Una consulta bien estructurada puede ser igual de eficiente y mucho más mantenible.
6. **No todo cambio se mide en milisegundos.** Algunas mejoras reducen riesgo, deuda técnica y costo futuro de mantenimiento.
7. **`QUALIFY` es una ventaja de Snowflake, no SQL estándar.** Úsala en Snowflake con confianza, pero documenta que no es portable a PostgreSQL o MySQL.

### Conexión con el trabajo real

Este laboratorio representa una situación común en proyectos de datos: una consulta funciona, pero es difícil de mantener, difícil de medir y difícil de escalar. Tu trabajo como analista o ingeniero de datos no es solo "hacer que corra", sino dejar evidencia de que:

- la lógica es correcta;
- la consulta es entendible;
- el plan de ejecución fue revisado;
- los filtros se aplican de forma eficiente;
- el resultado puede sostenerse cuando crezca el volumen de datos.

### Recursos de referencia

| Recurso | URL |
|---|---|
| Snowflake: Query Best Practices | https://docs.snowflake.com/en/user-guide/querying-best-practices |
| Snowflake: Understanding Query Profile | https://docs.snowflake.com/en/user-guide/ui-query-profile |
| Snowflake: Query History Table Function | https://docs.snowflake.com/en/sql-reference/functions/query_history |
| Snowflake: Common Table Expressions | https://docs.snowflake.com/en/user-guide/queries-cte |
| Snowflake: Micro-partitions and Data Clustering | https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions |
| SQL Style Guide — Simon Holywell | https://www.sqlstyle.guide/ |
| dbt Labs: How we style our SQL | https://docs.getdbt.com/best-practices/how-we-style/1-how-we-style-our-dbt-models |

---

*Lab 07-00-01 — Optimización y mejora de performance de queries | LAB_SQL_INTERMEDIO | Módulo 7*
