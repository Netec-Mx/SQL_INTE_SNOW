# Análisis de tendencias y métricas por periodo

## Metadatos

| Atributo | Detalle |
|---|---|
| **Duración estimada** | 50 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (Apply) |
| **Módulo** | 5 — Análisis de tendencias y métricas por periodo |
| **Laboratorio previo requerido** | Lab 04 — Window Functions (LAG/LEAD, SUM OVER) |
| **Schema de trabajo** | `LAB_SQL_INTERMEDIO.VENTAS` |

---

## Descripción General

En este laboratorio aplicarás las funciones de fecha de Snowflake aprendidas en la Lección 5.1 para construir análisis temporales completos sobre datos reales de ventas. Comenzarás agrupando ventas por semana, mes y trimestre usando `DATE_TRUNC`, luego construirás comparaciones mes a mes con `LAG()`, calcularás variaciones porcentuales entre períodos y finalmente identificarás los mejores y peores meses por crecimiento. El laboratorio conecta directamente las funciones de fecha con las window functions del Laboratorio 4, consolidando ambas habilidades en un reporte de tendencias cohesivo.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:

- [ ] Aplicar `DATE_TRUNC`, `DATEADD`, `DATEDIFF`, `YEAR`, `MONTH` y `TO_CHAR` para agrupar y presentar datos por períodos temporales en Snowflake.
- [ ] Construir comparaciones de métricas entre períodos equivalentes (mes actual vs. mes anterior, año actual vs. año anterior) usando `LAG()` y CTEs.
- [ ] Calcular tasas de variación porcentual entre períodos consecutivos e identificar tendencias de crecimiento en series temporales de ventas.
- [ ] Generar un reporte de tendencias que clasifique los 3 mejores y 3 peores meses por variación porcentual de ventas.

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

### Acceso requerido

| Recurso | Detalle |
|---|---|
| Cuenta Snowflake activa | Trial o corporativa con permisos de lectura sobre `LAB_SQL_INTERMEDIO` |
| Warehouse disponible | `X-SMALL` suspendido al finalizar la sesión |
| Script de setup ejecutado | `00_setup_laboratorios.sql` ejecutado por el instructor previamente |
| Navegador web moderno | Chrome 110+, Firefox 110+, Edge 110+ o Safari 16+ |

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

### Configuración inicial del entorno

Antes de comenzar los ejercicios, ejecuta los siguientes comandos para establecer el contexto de trabajo correcto. Copia y pega este bloque completo en una nueva hoja de trabajo (worksheet) en Snowsight.

```sql
-- ============================================================
-- CONFIGURACIÓN INICIAL DEL LABORATORIO 05-00-01
-- Ejecutar este bloque COMPLETO antes de comenzar los pasos
-- ============================================================

-- 1. Seleccionar el rol adecuado
USE ROLE SYSADMIN;

-- 2. Activar el warehouse (tamaño X-SMALL para minimizar créditos)
USE WAREHOUSE LAB_WH;

-- 3. Establecer la base de datos y schema de trabajo
USE DATABASE LAB_SQL_INTERMEDIO;
USE SCHEMA VENTAS;

-- 4. Verificar que las tablas necesarias existen
SHOW TABLES LIKE '%VENTAS%';
SHOW TABLES LIKE '%PEDIDOS%';
SHOW TABLES LIKE '%CLIENTES%';
```

**Resultado esperado de la verificación:** Debes ver al menos las tablas `VENTAS`, `PEDIDOS` y `CLIENTES` listadas. Si no aparecen, contacta al instructor para que ejecute el script `00_setup_laboratorios.sql`.

> ⚠️ **Recordatorio de créditos:** Este laboratorio usa un warehouse `X-SMALL`. Al terminar la sesión, ejecuta `ALTER WAREHOUSE LAB_WH SUSPEND;` para evitar consumo innecesario de créditos.

---

## Pasos del Laboratorio

---

### Paso 1 — Exploración de la estructura temporal de los datos

**Objetivo:** Familiarizarte con el rango de fechas disponible en la tabla `VENTAS` y validar que existen al menos 24 meses de datos históricos para el análisis de tendencias.

#### Instrucciones

1. Abre una nueva worksheet en Snowsight y asegúrate de que el contexto esté configurado con `LAB_SQL_INTERMEDIO.VENTAS`.

2. Ejecuta la siguiente consulta para explorar la estructura y el rango temporal de los datos:

```sql
-- ============================================================
-- PASO 1: Exploración del rango temporal de VENTAS
-- ============================================================

-- 1a. Verificar estructura de la tabla
DESCRIBE TABLE VENTAS;
```

3. Ejecuta la consulta de rango temporal:

```sql
-- 1b. Rango de fechas y volumen de datos disponibles
SELECT
    MIN(FECHA_VENTA)                                    AS fecha_mas_antigua,
    MAX(FECHA_VENTA)                                    AS fecha_mas_reciente,
    DATEDIFF('month', MIN(FECHA_VENTA), MAX(FECHA_VENTA)) AS meses_de_historia,
    DATEDIFF('day',  MIN(FECHA_VENTA), MAX(FECHA_VENTA)) AS dias_de_historia,
    COUNT(*)                                            AS total_registros,
    COUNT(DISTINCT DATE_TRUNC('month', FECHA_VENTA))    AS meses_con_datos
FROM VENTAS;
```

4. Ejecuta una exploración rápida de los primeros registros para entender las columnas disponibles:

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
|---|---|---|---|---|---|
| 2022-01-01 | 2024-12-31 | 36 | ~1095 | ~50,000+ | 36 |

> Si `meses_de_historia` es menor a 24, notifica al instructor antes de continuar.

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

Debes ver al menos 2 años distintos con registros. Si ves 3 años, el dataset está completo para todos los ejercicios de este laboratorio.

---

### Paso 2 — Agrupación temporal con DATE_TRUNC

**Objetivo:** Practicar el uso de `DATE_TRUNC` para agregar métricas de ventas a diferentes niveles de granularidad temporal: semanal, mensual y trimestral.

#### Instrucciones

1. Construye primero la agrupación **mensual**, que será la base de los análisis siguientes:

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

2. Construye la agrupación **trimestral** para identificar estacionalidad a nivel macro:

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

3. Construye la agrupación **semanal** para detectar patrones de corto plazo (limita el resultado a los últimos 3 meses para legibilidad):

```sql
-- ============================================================
-- PASO 2C: Ventas por SEMANA (últimos 3 meses)
-- ============================================================
SELECT
    DATE_TRUNC('week', FECHA_VENTA)             AS inicio_semana,
    DATEADD('day', 6,
        DATE_TRUNC('week', FECHA_VENTA))        AS fin_semana,
    COUNT(*)                                    AS cantidad_ventas,
    ROUND(SUM(MONTO_TOTAL), 2)                  AS ventas_totales
FROM VENTAS
WHERE FECHA_VENTA >= DATEADD('month', -3, CURRENT_DATE)
GROUP BY DATE_TRUNC('week', FECHA_VENTA)
ORDER BY inicio_semana DESC;
```

#### Resultado esperado

- **2A (mensual):** Una fila por mes con ventas agregadas. El número de filas debe coincidir con `meses_con_datos` del Paso 1.
- **2B (trimestral):** Una fila por trimestre. Si tienes 3 años de datos, verás ~12 filas.
- **2C (semanal):** Aproximadamente 12–13 semanas listadas en orden descendente.

> **Observa** cómo `DATE_TRUNC('month', fecha)` siempre devuelve el primer día del mes (ej. `2024-03-01`), no una cadena de texto. Esto permite ordenar correctamente con `ORDER BY`.

#### Verificación

```sql
-- Verificar que DATE_TRUNC produce el tipo de dato correcto
SELECT
    DATE_TRUNC('month', CURRENT_DATE)          AS tipo_date,
    TO_CHAR(DATE_TRUNC('month', CURRENT_DATE),
            'YYYY-MM')                         AS tipo_varchar,
    TYPEOF(DATE_TRUNC('month', CURRENT_DATE))  AS tipo_de_dato
FROM DUAL;
```

Debes ver `DATE` o `TIMESTAMP_NTZ` en la columna `tipo_de_dato`. Esto confirma que `DATE_TRUNC` mantiene el tipo nativo de fecha, lo que es esencial para los cálculos del siguiente paso.

---

### Paso 3 — Comparación mes a mes con LAG()

**Objetivo:** Construir una comparación de ventas entre el mes actual y el mes anterior usando la función de ventana `LAG()` sobre la serie temporal mensual creada en el Paso 2.

#### Instrucciones

1. Construye la consulta base con `LAG()` para traer el valor del mes anterior junto al mes actual. Este patrón conecta directamente con lo aprendido en el Laboratorio 4:

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

2. Refactoriza la consulta anterior usando una CTE para mayor legibilidad (buena práctica recomendada en el curso):

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
|---|---|---|---|---|
| 2022-01 | 125,430.00 | NULL | NULL | N/A (primer mes) |
| 2022-02 | 118,920.50 | 125,430.00 | -6,509.50 | ▼ Caída |
| 2022-03 | 134,210.75 | 118,920.50 | 15,290.25 | ▲ Crecimiento |
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
)
SELECT
    COUNT(*)                                                    AS total_filas,
    COUNT(LAG(ventas_totales) OVER (ORDER BY periodo_mes))      AS filas_con_mes_anterior,
    COUNT(*) - COUNT(LAG(ventas_totales)
        OVER (ORDER BY periodo_mes))                            AS filas_null_esperadas
FROM ventas_mensuales;
```

`filas_null_esperadas` debe ser exactamente `1` (el primer mes del dataset).

---

### Paso 4 — Cálculo de variación porcentual

**Objetivo:** Extender la comparación del Paso 3 para calcular la tasa de variación porcentual entre períodos consecutivos, identificando los meses de mayor crecimiento y mayor caída.

#### Instrucciones

1. Agrega el cálculo de variación porcentual a la CTE del Paso 3. La fórmula es: `((actual - anterior) / anterior) * 100`:

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
        WHEN variacion_pct IS NULL       THEN 'Sin referencia'
        WHEN variacion_pct >= 10         THEN '🚀 Alto crecimiento'
        WHEN variacion_pct BETWEEN 0 AND 9.99  THEN '📈 Crecimiento moderado'
        WHEN variacion_pct BETWEEN -9.99 AND -0.01 THEN '📉 Caída moderada'
        WHEN variacion_pct <= -10        THEN '⚠️ Caída significativa'
        ELSE 'Sin cambio'
    END AS clasificacion_variacion
FROM comparacion_mensual
ORDER BY periodo_mes;
```

2. Analiza la distribución de las variaciones para tener una vista estadística de la volatilidad mensual:

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
    COUNT(variacion_pct)        AS meses_con_variacion,
    ROUND(AVG(variacion_pct), 2) AS variacion_promedio_pct,
    ROUND(MAX(variacion_pct), 2) AS mayor_crecimiento_pct,
    ROUND(MIN(variacion_pct), 2) AS mayor_caida_pct,
    ROUND(STDDEV(variacion_pct), 2) AS desviacion_estandar_pct
FROM variaciones
WHERE variacion_pct IS NOT NULL;
```

#### Resultado esperado

- **4A:** Tabla con una fila por mes mostrando la variación porcentual y su clasificación. Los valores de `variacion_pct` deben ser números decimales (positivos = crecimiento, negativos = caída).
- **4B:** Una sola fila con estadísticas descriptivas de la volatilidad mensual.

> **Importante:** El uso de `NULLIF(..., 0)` en el denominador previene errores de división por cero si algún mes tiene ventas registradas en cero. Esta es una práctica de escritura defensiva que debes aplicar siempre en cálculos de porcentajes.

#### Verificación

```sql
-- Verificar que no hay errores de división por cero
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

Si el resultado es `0`, no hay riesgo de división por cero en los datos actuales. Si es mayor a `0`, el uso de `NULLIF` en el Paso 4A es imprescindible.

---

### Paso 5 — Comparación año sobre año (YoY)

**Objetivo:** Construir una comparación Year-over-Year (YoY) que permita comparar el desempeño de cada mes en el año actual contra el mismo mes del año anterior, usando CTEs y `DATEADD`.

#### Instrucciones

1. Construye la comparación YoY usando un self-join sobre la CTE de ventas mensuales. Este enfoque es portable y muy legible:

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

2. Construye una vista pivotada alternativa que muestre los años como columnas para comparar visualmente (útil para presentaciones):

```sql
-- ============================================================
-- PASO 5B: Vista pivotada de ventas por mes y año
-- ============================================================
SELECT
    MONTH(FECHA_VENTA)                      AS mes_numero,
    TO_CHAR(DATE_TRUNC('month',
        DATE_FROM_PARTS(2000, MONTH(FECHA_VENTA), 1)),
        'Month')                            AS nombre_mes,
    ROUND(SUM(CASE WHEN YEAR(FECHA_VENTA) = 2022
                   THEN MONTO_TOTAL ELSE 0 END), 2) AS ventas_2022,
    ROUND(SUM(CASE WHEN YEAR(FECHA_VENTA) = 2023
                   THEN MONTO_TOTAL ELSE 0 END), 2) AS ventas_2023,
    ROUND(SUM(CASE WHEN YEAR(FECHA_VENTA) = 2024
                   THEN MONTO_TOTAL ELSE 0 END), 2) AS ventas_2024
FROM VENTAS
GROUP BY
    MONTH(FECHA_VENTA),
    TO_CHAR(DATE_TRUNC('month',
        DATE_FROM_PARTS(2000, MONTH(FECHA_VENTA), 1)),
        'Month')
ORDER BY mes_numero;
```

> **Nota:** El pivote manual con `CASE WHEN` es la técnica estándar en Snowflake para comparaciones YoY cuando los años son conocidos. Para pivotes dinámicos se puede usar `PIVOT`, pero eso está fuera del alcance de este laboratorio.

#### Resultado esperado

- **5A:** Una fila por mes mostrando ventas del año actual vs. año anterior y la variación YoY. Los meses del primer año del dataset tendrán `NULL` en `ventas_anio_anterior` (comportamiento correcto del `LEFT JOIN`).
- **5B:** 12 filas (una por mes del año), con columnas separadas por año, permitiendo comparar visualmente el desempeño estacional.

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
    COUNT(*)                                   AS total_filas_yoy,
    COUNT(anio_anterior.ventas_totales)        AS filas_con_comparacion,
    COUNT(*) - COUNT(anio_anterior.ventas_totales) AS filas_sin_anio_anterior
FROM ventas_mensuales AS anio_actual
LEFT JOIN ventas_mensuales AS anio_anterior
    ON anio_actual.mes_numero = anio_anterior.mes_numero
   AND anio_actual.anio       = anio_anterior.anio + 1;
```

`filas_sin_anio_anterior` debe corresponder a los 12 meses del primer año del dataset (que no tienen año previo para comparar).

---

### Paso 6 — Reporte de tendencias: Top 3 mejores y peores meses

**Objetivo:** Construir el reporte final que identifique los 3 meses con mayor crecimiento y los 3 meses con mayor caída en ventas, consolidando todas las técnicas del laboratorio en una consulta analítica completa.

#### Instrucciones

1. Construye el reporte completo de tendencias usando múltiples CTEs encadenadas:

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

2. Complementa el reporte con un análisis de patrones estacionales por trimestre:

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
- **6B:** 4 filas (una por trimestre) mostrando si cada trimestre está por encima o por debajo del promedio, permitiendo identificar estacionalidad clara.

Ejemplo parcial del resultado de 6A:

| categoria | posicion | etiqueta_mes | ventas_totales | ventas_mes_anterior | variacion_pct |
|---|---|---|---|---|---|
| TOP CAÍDA | 1 | 2022-08 | 89,340.00 | 142,120.50 | -37.13 |
| TOP CAÍDA | 2 | 2023-02 | 95,670.25 | 138,450.00 | -30.90 |
| TOP CAÍDA | 3 | ... | ... | ... | ... |
| TOP CRECIMIENTO | 1 | 2022-12 | 198,450.75 | 124,310.00 | 59.63 |
| TOP CRECIMIENTO | 2 | ... | ... | ... | ... |

#### Verificación

```sql
-- Verificar que el reporte tiene exactamente 6 filas
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
    QUALIFY variacion_pct IS NOT NULL
),
ranking AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY variacion_pct DESC) AS rank_crec,
        ROW_NUMBER() OVER (ORDER BY variacion_pct ASC)  AS rank_caida
    FROM variaciones
)
SELECT COUNT(*) AS filas_reporte_final
FROM ranking
WHERE rank_crec <= 3 OR rank_caida <= 3;
```

> **Nota sobre QUALIFY:** Esta cláusula es exclusiva de Snowflake y permite filtrar directamente sobre el resultado de una window function sin necesidad de una subconsulta adicional. No es portable a PostgreSQL ni MySQL. Aquí se usa como alternativa compacta al `WHERE variacion_pct IS NOT NULL` de la CTE.

El resultado debe ser `6`.

---

## Validación y Pruebas

Una vez completados todos los pasos, ejecuta el siguiente script de validación integral para confirmar que las consultas funcionan correctamente y producen resultados coherentes:

```sql
-- ============================================================
-- VALIDACIÓN INTEGRAL - Lab 05-00-01
-- Ejecutar completo para verificar todos los pasos
-- ============================================================

-- TEST 1: DATE_TRUNC produce tipo de dato correcto (Paso 2)
SELECT
    'TEST 1 - DATE_TRUNC tipo correcto' AS prueba,
    CASE
        WHEN TYPEOF(DATE_TRUNC('month', CURRENT_DATE))
             IN ('DATE', 'TIMESTAMP_NTZ')
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
    SELECT LAG(ventas) OVER (ORDER BY periodo_mes) AS lag_val
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
    QUALIFY variacion_pct IS NOT NULL
),
ranking AS (
    SELECT
        ROW_NUMBER() OVER (ORDER BY variacion_pct DESC) AS rank_crec,
        ROW_NUMBER() OVER (ORDER BY variacion_pct ASC)  AS rank_caida
    FROM variaciones
)
SELECT
    'TEST 5 - Reporte top 3 produce 6 filas' AS prueba,
    CASE
        WHEN COUNT(*) = 6 THEN '✅ PASÓ'
        ELSE '❌ FALLÓ - Se esperaban 6 filas, se obtuvieron: '
             || COUNT(*) || '. Verificar si hay empates en variacion_pct.'
    END AS resultado
FROM ranking
WHERE rank_crec <= 3 OR rank_caida <= 3;
```

**Resultado esperado de la validación:** Los 5 tests deben mostrar `✅ PASÓ`. Si alguno falla, revisa el paso correspondiente antes de continuar.

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

Si esta consulta no devuelve filas, el problema está en el contexto de base de datos o schema. Verifica con `SELECT CURRENT_DATABASE(), CURRENT_SCHEMA();`.

---

### Problema 2: La variación porcentual YoY muestra valores inesperadamente extremos (ej. 10,000%)

**Síntoma:** Al ejecutar el Paso 5A, algunas filas muestran variaciones YoY de miles de por ciento, lo que no tiene sentido de negocio.

**Causa probable:** El `LEFT JOIN` en la comparación YoY está produciendo múltiples coincidencias por `mes_numero`, lo que ocurre cuando existen registros duplicados en la CTE `ventas_mensuales` o cuando el `GROUP BY` no está incluyendo todas las columnas necesarias. Si hay dos filas para el mismo mes en la CTE (por ejemplo, si `DATE_TRUNC` y `YEAR/MONTH` no coinciden exactamente en el `GROUP BY`), el join multiplica los valores y produce sumas infladas.

**Solución:** Verifica que la CTE base no tiene duplicados de periodo:

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

## Limpieza del Entorno

Al finalizar el laboratorio, ejecuta los siguientes comandos para liberar recursos y evitar consumo innecesario de créditos Snowflake:

```sql
-- ============================================================
-- LIMPIEZA POST-LABORATORIO
-- Ejecutar obligatoriamente al terminar la sesión
-- ============================================================

-- 1. Suspender el warehouse para detener el consumo de créditos
--    (OBLIGATORIO en cuentas trial)
ALTER WAREHOUSE LAB_WH SUSPEND;

-- 2. Verificar que el warehouse quedó suspendido
SHOW WAREHOUSES LIKE 'LAB_WH';
-- Confirmar que el campo "state" muestra SUSPENDED

-- 3. No es necesario eliminar objetos: este laboratorio solo
--    ejecutó consultas SELECT sobre tablas existentes.
--    No se crearon tablas, vistas ni objetos permanentes.
```

> ✅ **Confirmación:** Si el campo `state` del warehouse muestra `SUSPENDED`, el laboratorio ha concluido correctamente y no se generarán cargos adicionales.

---

## Resumen

En este laboratorio aplicaste un conjunto completo de técnicas de análisis temporal en Snowflake, construyendo progresivamente desde la exploración básica hasta un reporte ejecutivo de tendencias. Los conceptos clave que practicaste:

| Técnica | Función(es) usada(s) | Paso donde se aplicó |
|---|---|---|
| Agrupación temporal por granularidad | `DATE_TRUNC('month'/'week'/'quarter', ...)` | Paso 2 |
| Extracción de componentes de fecha | `YEAR()`, `MONTH()`, `QUARTER()`, `DAYOFWEEK()` | Pasos 1, 2, 5 |
| Desplazamiento temporal en filtros | `DATEADD('month', -3, CURRENT_DATE)` | Paso 2C |
| Comparación período anterior | `LAG() OVER (ORDER BY periodo_mes)` | Pasos 3, 4, 6 |
| Variación porcentual segura | `(actual - anterior) / NULLIF(anterior, 0) * 100` | Pasos 4, 5, 6 |
| Comparación Year-over-Year | Self-join en CTE + pivote con `CASE WHEN` | Paso 5 |
| Reporte de rankings temporales | `ROW_NUMBER() OVER (ORDER BY variacion_pct)` | Paso 6 |
| Formateo de fechas para presentación | `TO_CHAR(fecha, 'YYYY-MM')` | Pasos 2, 3, 5, 6 |
| Filtrado de window functions (Snowflake) | `QUALIFY` | Paso 6 verificación |

### Conexión con el siguiente módulo

Las técnicas de este laboratorio son la base directa del **Laboratorio 6**, donde aplicarás reconciliación de datasets comparando métricas entre dos fuentes de datos (`VENTAS_ORIGEN` vs. `VENTAS_DESTINO`). Las CTEs encadenadas y los patrones de comparación entre períodos que practicaste aquí se reutilizarán directamente en ese contexto de validación de datos.

### Recursos adicionales

- [Documentación Snowflake: Date & Time Functions](https://docs.snowflake.com/en/sql-reference/functions-date-time)
- [Documentación Snowflake: DATE_TRUNC](https://docs.snowflake.com/en/sql-reference/functions/date_trunc)
- [Documentación Snowflake: DATEDIFF](https://docs.snowflake.com/en/sql-reference/functions/datediff)
- [Documentación Snowflake: DATEADD](https://docs.snowflake.com/en/sql-reference/functions/dateadd)
- [Documentación Snowflake: Window Functions (LAG/LEAD)](https://docs.snowflake.com/en/sql-reference/functions-analytic)
- [Documentación Snowflake: QUALIFY clause](https://docs.snowflake.com/en/sql-reference/constructs/qualify)

---
*Lab 05-00-01 — Análisis de tendencias y métricas por periodo | LAB_SQL_INTERMEDIO | Módulo 5*
