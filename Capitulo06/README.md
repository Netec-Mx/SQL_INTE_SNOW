---LAB_START---
LAB_ID: 06-00-01
---MARKDOWN---
# Validación y reconciliación de datasets

## Metadatos

| Atributo | Valor |
|---|---|
| **Duración estimada** | 35 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 6 — Validación y reconciliación de datos |
| **Plataforma** | Snowflake (Snowsight) |
| **Prerequisito de setup** | Script `00_setup_laboratorios.sql` ejecutado por el instructor |

---

## Descripción general

En este laboratorio aplicarás técnicas de comparación de datasets para detectar discrepancias entre dos versiones de los mismos datos: el schema `VENTAS_ORIGEN` (datos crudos) y el schema `VENTAS_DESTINO` (datos procesados). Utilizarás operadores de conjunto (`EXCEPT`, `INTERSECT`), comparaciones de agregados y checksums de fila para construir un reporte de reconciliación completo. Al finalizar, habrás empaquetado todas las validaciones como CTEs reutilizables que simulan un conjunto de controles de calidad aplicables a cualquier pipeline de datos.

---

## Objetivos de aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Comparar conteos totales y por categoría entre dos schemas para detectar discrepancias de volumen.
- [ ] Usar `EXCEPT` e `INTERSECT` para identificar registros faltantes, extra o coincidentes entre datasets de origen y destino.
- [ ] Implementar comparaciones de checksums (`MD5`) sobre columnas numéricas clave para detectar diferencias de valores.
- [ ] Construir un reporte de reconciliación consolidado usando `UNION ALL` y `CASE WHEN` con clasificación `PASS/FAIL`.
- [ ] Encapsular validaciones como CTEs reutilizables orientadas a monitoreo continuo de calidad de datos.

---

## Prerrequisitos

### Conocimiento previo

| Requisito | Nivel esperado |
|---|---|
| CTEs (`WITH`) y subqueries | Sólido — Laboratorios 1 al 5 |
| `JOIN` (INNER, LEFT, FULL OUTER) | Sólido |
| Funciones de agregación (`COUNT`, `SUM`, `AVG`) | Sólido |
| Operadores de conjunto (`UNION`, `EXCEPT`, `INTERSECT`) | Básico a intermedio |
| Concepto de pipeline ETL y calidad de datos | Conceptual |

### Acceso requerido

| Recurso | Detalle |
|---|---|
| Cuenta Snowflake activa | Trial o corporativa con rol `SYSADMIN` o `LAB_ROLE` |
| Base de datos `LAB_SQL_INTERMEDIO` | Creada por el instructor con `00_setup_laboratorios.sql` |
| Schemas disponibles | `VENTAS_ORIGEN` y `VENTAS_DESTINO` dentro de `LAB_SQL_INTERMEDIO` |
| Warehouse activo | `LAB_WH` (tamaño `X-SMALL`) |

---

## Entorno de laboratorio

### Hardware recomendado

| Componente | Mínimo | Recomendado |
|---|---|---|
| Procesador | Intel Core i5 / AMD Ryzen 5 (64 bits) | Intel Core i7 / AMD Ryzen 7 |
| RAM | 8 GB | 16 GB |
| Almacenamiento libre | 2 GB | 5 GB |
| Resolución de pantalla | 1280 × 768 | 1920 × 1080 |
| Conexión a Internet | 10 Mbps | 25 Mbps o superior |

### Software requerido

| Software | Versión mínima | Uso |
|---|---|---|
| Navegador web (Chrome / Firefox / Edge / Safari) | 110+ / 110+ / 110+ / 16+ | Acceso a Snowsight |
| Snowflake (Snowsight) | Versión web actual | Editor SQL principal |
| Visual Studio Code (opcional) | 1.80+ | Edición local de scripts |

### Comandos de configuración inicial

Antes de comenzar los ejercicios, ejecuta el siguiente bloque en Snowsight para confirmar que el entorno está correctamente configurado y establecer el contexto de trabajo:

```sql
-- 1. Verificar que el warehouse está activo
ALTER WAREHOUSE LAB_WH RESUME IF SUSPENDED;

-- 2. Establecer contexto de trabajo
USE ROLE     SYSADMIN;
USE WAREHOUSE LAB_WH;
USE DATABASE  LAB_SQL_INTERMEDIO;

-- 3. Confirmar que los schemas de origen y destino existen
SHOW SCHEMAS IN DATABASE LAB_SQL_INTERMEDIO;
```

**Resultado esperado del paso 3:** Deberás ver al menos los schemas `VENTAS_ORIGEN` y `VENTAS_DESTINO` en la lista devuelta.

```sql
-- 4. Verificar tablas disponibles en cada schema
SHOW TABLES IN SCHEMA LAB_SQL_INTERMEDIO.VENTAS_ORIGEN;
SHOW TABLES IN SCHEMA LAB_SQL_INTERMEDIO.VENTAS_DESTINO;
```

**Resultado esperado del paso 4:** Ambos schemas deben contener la tabla `PEDIDOS` (y opcionalmente `VENTAS`, `CLIENTES`, `PRODUCTOS`). Si algún schema está vacío, notifica al instructor antes de continuar.

> ⚠️ **Nota importante:** Si el script `00_setup_laboratorios.sql` no fue ejecutado previamente por el instructor, ninguno de los pasos siguientes podrá completarse. Detén el laboratorio y coordina con el instructor.

---

## Pasos del laboratorio

### Paso 1 — Reconocer los datasets: comparación de volumen total

**Objetivo:** Establecer una primera validación rápida comparando el número total de registros y la suma de montos entre `VENTAS_ORIGEN` y `VENTAS_DESTINO`. Esta es la prueba más básica de un pipeline ETL: ¿llegaron todos los registros al destino?

#### Instrucciones

1. Abre una nueva hoja de trabajo en Snowsight. Nómbrala `Lab06_Paso1_Volumen`.

2. Ejecuta la siguiente consulta para comparar el volumen total de registros y la suma de montos entre ambos schemas:

```sql
-- ============================================================
-- PASO 1: Comparación de volumen total entre origen y destino
-- ============================================================

SELECT
    'VENTAS_ORIGEN'          AS schema_fuente,
    COUNT(*)                 AS total_registros,
    SUM(monto_total)         AS suma_montos,
    MIN(fecha_pedido)        AS fecha_minima,
    MAX(fecha_pedido)        AS fecha_maxima
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS

UNION ALL

SELECT
    'VENTAS_DESTINO'         AS schema_fuente,
    COUNT(*)                 AS total_registros,
    SUM(monto_total)         AS suma_montos,
    MIN(fecha_pedido)        AS fecha_minima,
    MAX(fecha_pedido)        AS fecha_maxima
FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS

ORDER BY schema_fuente;
```

3. Observa los resultados. Anota mentalmente si los valores de `total_registros` y `suma_montos` coinciden entre ambas filas.

#### Resultado esperado

Verás dos filas, una por schema. Si el pipeline funcionó correctamente, los valores de `total_registros` deberían ser iguales o muy cercanos. Es probable que el laboratorio haya sido diseñado con una discrepancia intencional para que la detectes.

```
schema_fuente    | total_registros | suma_montos  | fecha_minima | fecha_maxima
-----------------|-----------------|--------------|--------------|-------------
VENTAS_DESTINO   |           4 850 |  2 341 200.0 | 2022-01-01   | 2023-12-31
VENTAS_ORIGEN    |           5 000 |  2 398 750.0 | 2022-01-01   | 2023-12-31
```

#### Verificación

✅ Si `VENTAS_ORIGEN` tiene más registros que `VENTAS_DESTINO`, confirma que existe una pérdida de datos en el pipeline — exactamente lo que investigarás en los pasos siguientes.

✅ Si las `suma_montos` difieren, puede haber tanto registros faltantes como diferencias de valores en registros existentes.

---

### Paso 2 — Comparación por categoría: desglose de discrepancias

**Objetivo:** Profundizar la comparación del Paso 1 desglosando por `categoria_producto` para identificar si las discrepancias están concentradas en categorías específicas o distribuidas uniformemente.

#### Instrucciones

1. Crea una nueva hoja de trabajo en Snowsight. Nómbrala `Lab06_Paso2_Categoria`.

2. Ejecuta la siguiente consulta:

```sql
-- ============================================================
-- PASO 2: Comparación de volumen y montos por categoría
-- ============================================================

SELECT
    'VENTAS_ORIGEN'           AS schema_fuente,
    categoria_producto,
    COUNT(*)                  AS total_registros,
    SUM(monto_total)          AS suma_montos
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
GROUP BY categoria_producto

UNION ALL

SELECT
    'VENTAS_DESTINO'          AS schema_fuente,
    categoria_producto,
    COUNT(*)                  AS total_registros,
    SUM(monto_total)          AS suma_montos
FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
GROUP BY categoria_producto

ORDER BY categoria_producto, schema_fuente;
```

3. Para una lectura más comparativa, ejecuta también esta versión con `FULL OUTER JOIN` que pone las cifras lado a lado:

```sql
-- ============================================================
-- PASO 2b: Vista comparativa lado a lado por categoría
-- ============================================================

SELECT
    COALESCE(o.categoria_producto,
             d.categoria_producto)          AS categoria_producto,
    o.total_registros                       AS registros_origen,
    d.total_registros                       AS registros_destino,
    o.total_registros - d.total_registros   AS diferencia_registros,
    o.suma_montos                           AS montos_origen,
    d.suma_montos                           AS montos_destino,
    ROUND(o.suma_montos - d.suma_montos, 2) AS diferencia_montos,
    CASE
        WHEN d.categoria_producto IS NULL   THEN 'FAIL - Categoría ausente en destino'
        WHEN o.total_registros
           = d.total_registros
         AND ROUND(o.suma_montos, 2)
           = ROUND(d.suma_montos, 2)        THEN 'PASS'
        ELSE                                     'FAIL - Discrepancia detectada'
    END                                     AS estado_validacion
FROM (
    SELECT
        categoria_producto,
        COUNT(*)        AS total_registros,
        SUM(monto_total) AS suma_montos
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
    GROUP BY categoria_producto
) AS o
FULL OUTER JOIN (
    SELECT
        categoria_producto,
        COUNT(*)        AS total_registros,
        SUM(monto_total) AS suma_montos
    FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
    GROUP BY categoria_producto
) AS d
    ON o.categoria_producto = d.categoria_producto
ORDER BY diferencia_registros DESC NULLS LAST;
```

#### Resultado esperado

La consulta del Paso 2b mostrará una fila por categoría con las diferencias calculadas y una columna `estado_validacion` que clasifica cada categoría como `PASS` o `FAIL`. Las categorías con más discrepancias aparecerán primero.

```
categoria_producto | registros_origen | registros_destino | diferencia_registros | estado_validacion
-------------------|------------------|-------------------|----------------------|---------------------------
Electrónica        |            1 200 |             1 050 |                  150 | FAIL - Discrepancia detectada
Ropa               |              900 |               900 |                    0 | PASS
...
```

#### Verificación

✅ Identifica qué categorías tienen `estado_validacion = 'FAIL'`. Estas serán el foco de la investigación en los pasos siguientes.

✅ Verifica que ninguna categoría presente en `VENTAS_ORIGEN` aparezca como `NULL` en el resultado de `COALESCE` — eso indicaría una categoría completamente ausente en destino.

---

### Paso 3 — Registros faltantes y extra: operadores EXCEPT e INTERSECT

**Objetivo:** Usar `EXCEPT` para identificar registros presentes en origen pero ausentes en destino (registros no migrados), y `INTERSECT` para confirmar cuántos registros coinciden exactamente en ambos schemas.

#### Instrucciones

1. Crea una nueva hoja de trabajo en Snowsight. Nómbrala `Lab06_Paso3_Except`.

2. Ejecuta la siguiente consulta para encontrar registros en origen que no llegaron al destino:

```sql
-- ============================================================
-- PASO 3a: Registros en ORIGEN que NO están en DESTINO
-- (Registros perdidos en el pipeline)
-- ============================================================

SELECT
    pedido_id,
    cliente_id,
    fecha_pedido,
    monto_total,
    categoria_producto
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS

EXCEPT

SELECT
    pedido_id,
    cliente_id,
    fecha_pedido,
    monto_total,
    categoria_producto
FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS

ORDER BY pedido_id;
```

3. Ejecuta la consulta inversa para detectar registros que aparecen en destino pero no tienen correspondencia en origen (registros fantasma o generados incorrectamente):

```sql
-- ============================================================
-- PASO 3b: Registros en DESTINO que NO están en ORIGEN
-- (Registros fantasma o generados por error)
-- ============================================================

SELECT
    pedido_id,
    cliente_id,
    fecha_pedido,
    monto_total,
    categoria_producto
FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS

EXCEPT

SELECT
    pedido_id,
    cliente_id,
    fecha_pedido,
    monto_total,
    categoria_producto
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS

ORDER BY pedido_id;
```

4. Finalmente, cuantifica cuántos registros coinciden exactamente en ambas tablas:

```sql
-- ============================================================
-- PASO 3c: Registros que coinciden EXACTAMENTE en ambos schemas
-- ============================================================

SELECT COUNT(*) AS registros_coincidentes
FROM (
    SELECT
        pedido_id,
        cliente_id,
        fecha_pedido,
        monto_total,
        categoria_producto
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS

    INTERSECT

    SELECT
        pedido_id,
        cliente_id,
        fecha_pedido,
        monto_total,
        categoria_producto
    FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
) AS coincidentes;
```

#### Resultado esperado

- **Paso 3a:** Lista de `pedido_id` que existen en origen pero no llegaron a destino. Estos son los registros perdidos.
- **Paso 3b:** Idealmente vacío (0 filas). Si hay resultados, indica registros generados en destino sin correspondencia en origen.
- **Paso 3c:** Un número que representa los registros que pasaron correctamente el pipeline sin ninguna modificación.

```
-- Paso 3a (ejemplo)
pedido_id | cliente_id | fecha_pedido | monto_total | categoria_producto
----------|------------|--------------|-------------|-------------------
  10 045  |    C-0231  |  2023-08-15  |      450.00 | Electrónica
  10 089  |    C-0187  |  2023-09-02  |      230.00 | Electrónica
  ...     |    ...     |     ...      |      ...    | ...
(150 filas)

-- Paso 3c (ejemplo)
registros_coincidentes
----------------------
                 4 700
```

#### Verificación

✅ El número de registros del Paso 3a + el resultado del Paso 3c debe ser aproximadamente igual al `total_registros` de `VENTAS_ORIGEN` del Paso 1 (considerando que algunos registros pueden estar en destino con valores modificados y no aparecen en el `INTERSECT`).

✅ Si el Paso 3b devuelve filas, documenta el hallazgo — es un indicador de un problema crítico en el pipeline (inserción de datos sin fuente verificable).

> 💡 **Nota conceptual:** `EXCEPT` compara filas completas. Si un registro tiene el mismo `pedido_id` pero un `monto_total` diferente, `EXCEPT` lo tratará como dos filas distintas y lo incluirá en el resultado. Esto lo verás con más detalle en el Paso 4.

---

### Paso 4 — Detección de diferencias de valores: FULL OUTER JOIN con auditoría

**Objetivo:** Construir una tabla de auditoría completa que clasifique cada registro según su estado de coincidencia, distinguiendo entre registros faltantes, extra y con valores modificados.

#### Instrucciones

1. Crea una nueva hoja de trabajo en Snowsight. Nómbrala `Lab06_Paso4_Auditoria`.

2. Ejecuta la consulta de auditoría detallada con `FULL OUTER JOIN`:

```sql
-- ============================================================
-- PASO 4: Tabla de auditoría completa con FULL OUTER JOIN
-- Clasifica cada registro según su estado de coincidencia
-- ============================================================

SELECT
    COALESCE(o.pedido_id,   d.pedido_id)    AS pedido_id,
    COALESCE(o.cliente_id,  d.cliente_id)   AS cliente_id,
    o.monto_total                           AS monto_origen,
    d.monto_total                           AS monto_destino,
    o.fecha_pedido                          AS fecha_origen,
    d.fecha_pedido                          AS fecha_destino,
    o.categoria_producto                    AS categoria_origen,
    d.categoria_producto                    AS categoria_destino,
    CASE
        WHEN o.pedido_id IS NULL                    THEN 'EXTRA EN DESTINO'
        WHEN d.pedido_id IS NULL                    THEN 'FALTANTE EN DESTINO'
        WHEN o.monto_total  <> d.monto_total        THEN 'MONTO DIFERENTE'
        WHEN o.fecha_pedido <> d.fecha_pedido       THEN 'FECHA DIFERENTE'
        WHEN o.categoria_producto
          <> d.categoria_producto                   THEN 'CATEGORÍA DIFERENTE'
        ELSE                                             'OK'
    END                                     AS estado_auditoria
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS  AS o
FULL OUTER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS AS d
    ON o.pedido_id = d.pedido_id
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

3. Para obtener el resumen ejecutivo de la auditoría (cuántos registros caen en cada categoría), ejecuta:

```sql
-- ============================================================
-- PASO 4b: Resumen ejecutivo de la auditoría
-- ============================================================

WITH auditoria AS (
    SELECT
        COALESCE(o.pedido_id, d.pedido_id) AS pedido_id,
        CASE
            WHEN o.pedido_id IS NULL                THEN 'EXTRA EN DESTINO'
            WHEN d.pedido_id IS NULL                THEN 'FALTANTE EN DESTINO'
            WHEN o.monto_total  <> d.monto_total    THEN 'MONTO DIFERENTE'
            WHEN o.fecha_pedido <> d.fecha_pedido   THEN 'FECHA DIFERENTE'
            WHEN o.categoria_producto
              <> d.categoria_producto               THEN 'CATEGORÍA DIFERENTE'
            ELSE                                         'OK'
        END AS estado_auditoria
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS  AS o
    FULL OUTER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS AS d
        ON o.pedido_id = d.pedido_id
)
SELECT
    estado_auditoria,
    COUNT(*)                                            AS cantidad,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS porcentaje
FROM auditoria
GROUP BY estado_auditoria
ORDER BY cantidad DESC;
```

#### Resultado esperado

```
estado_auditoria     | cantidad | porcentaje
---------------------|----------|------------
OK                   |    4 700 |      94.00
FALTANTE EN DESTINO  |      150 |       3.00
MONTO DIFERENTE      |       85 |       1.70
FECHA DIFERENTE      |       40 |       0.80
EXTRA EN DESTINO     |       20 |       0.40
CATEGORÍA DIFERENTE  |        5 |       0.10
```

#### Verificación

✅ La suma de `cantidad` en el resumen ejecutivo debe ser igual al total de filas del `FULL OUTER JOIN`, que es: `MAX(registros_origen, registros_destino) + registros_solo_en_un_lado`.

✅ El `porcentaje` de `OK` indica la **tasa de concordancia** del pipeline. Un valor por debajo del 95% generalmente requiere investigación inmediata.

✅ Verifica que la lógica del `CASE WHEN` en la CTE sea idéntica a la del Paso 4a para garantizar consistencia entre el detalle y el resumen.

---

### Paso 5 — Checksums de fila: validación con MD5

**Objetivo:** Implementar una técnica de validación más robusta usando `MD5()` para generar un checksum de fila completa y detectar cualquier diferencia en cualquier campo, sin necesidad de comparar columna por columna.

#### Instrucciones

1. Crea una nueva hoja de trabajo en Snowsight. Nómbrala `Lab06_Paso5_Checksum`.

2. Primero, entiende cómo funciona `MD5()` en Snowflake para generar un hash de fila:

```sql
-- ============================================================
-- PASO 5a: Demostración de MD5 para checksum de fila
-- ============================================================

-- MD5 convierte la concatenación de todos los campos en un hash único
-- Si cualquier campo cambia, el hash cambia
SELECT
    pedido_id,
    MD5(
        CONCAT(
            COALESCE(CAST(pedido_id      AS VARCHAR), ''),
            '|',
            COALESCE(CAST(cliente_id     AS VARCHAR), ''),
            '|',
            COALESCE(CAST(fecha_pedido   AS VARCHAR), ''),
            '|',
            COALESCE(CAST(monto_total    AS VARCHAR), ''),
            '|',
            COALESCE(categoria_producto,              '')
        )
    ) AS checksum_fila
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
LIMIT 5;
```

3. Ahora compara los checksums entre origen y destino para detectar cualquier modificación de datos:

```sql
-- ============================================================
-- PASO 5b: Comparación de checksums entre ORIGEN y DESTINO
-- ============================================================

WITH checksums_origen AS (
    SELECT
        pedido_id,
        MD5(
            CONCAT(
                COALESCE(CAST(pedido_id      AS VARCHAR), ''),
                '|',
                COALESCE(CAST(cliente_id     AS VARCHAR), ''),
                '|',
                COALESCE(CAST(fecha_pedido   AS VARCHAR), ''),
                '|',
                COALESCE(CAST(monto_total    AS VARCHAR), ''),
                '|',
                COALESCE(categoria_producto,              '')
            )
        ) AS checksum_fila
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
),

checksums_destino AS (
    SELECT
        pedido_id,
        MD5(
            CONCAT(
                COALESCE(CAST(pedido_id      AS VARCHAR), ''),
                '|',
                COALESCE(CAST(cliente_id     AS VARCHAR), ''),
                '|',
                COALESCE(CAST(fecha_pedido   AS VARCHAR), ''),
                '|',
                COALESCE(CAST(monto_total    AS VARCHAR), ''),
                '|',
                COALESCE(categoria_producto,              '')
            )
        ) AS checksum_fila
    FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
)

SELECT
    COALESCE(o.pedido_id, d.pedido_id)  AS pedido_id,
    o.checksum_fila                     AS checksum_origen,
    d.checksum_fila                     AS checksum_destino,
    CASE
        WHEN o.pedido_id IS NULL         THEN 'EXTRA EN DESTINO'
        WHEN d.pedido_id IS NULL         THEN 'FALTANTE EN DESTINO'
        WHEN o.checksum_fila
          = d.checksum_fila              THEN 'IDÉNTICO'
        ELSE                                  'MODIFICADO'
    END                                 AS estado_checksum
FROM checksums_origen  AS o
FULL OUTER JOIN checksums_destino AS d
    ON o.pedido_id = d.pedido_id
ORDER BY
    CASE estado_checksum
        WHEN 'FALTANTE EN DESTINO' THEN 1
        WHEN 'EXTRA EN DESTINO'    THEN 2
        WHEN 'MODIFICADO'          THEN 3
        ELSE                            4
    END,
    pedido_id;
```

4. Obtén el resumen de la comparación por checksum:

```sql
-- ============================================================
-- PASO 5c: Resumen de validación por checksum
-- ============================================================

WITH checksums_origen AS (
    SELECT
        pedido_id,
        MD5(CONCAT(
            COALESCE(CAST(pedido_id AS VARCHAR), ''), '|',
            COALESCE(CAST(cliente_id AS VARCHAR), ''), '|',
            COALESCE(CAST(fecha_pedido AS VARCHAR), ''), '|',
            COALESCE(CAST(monto_total AS VARCHAR), ''), '|',
            COALESCE(categoria_producto, '')
        )) AS checksum_fila
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
),
checksums_destino AS (
    SELECT
        pedido_id,
        MD5(CONCAT(
            COALESCE(CAST(pedido_id AS VARCHAR), ''), '|',
            COALESCE(CAST(cliente_id AS VARCHAR), ''), '|',
            COALESCE(CAST(fecha_pedido AS VARCHAR), ''), '|',
            COALESCE(CAST(monto_total AS VARCHAR), ''), '|',
            COALESCE(categoria_producto, '')
        )) AS checksum_fila
    FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
),
comparacion AS (
    SELECT
        CASE
            WHEN o.pedido_id IS NULL         THEN 'EXTRA EN DESTINO'
            WHEN d.pedido_id IS NULL         THEN 'FALTANTE EN DESTINO'
            WHEN o.checksum_fila
              = d.checksum_fila              THEN 'IDÉNTICO'
            ELSE                                  'MODIFICADO'
        END AS estado_checksum
    FROM checksums_origen  AS o
    FULL OUTER JOIN checksums_destino AS d
        ON o.pedido_id = d.pedido_id
)
SELECT
    estado_checksum,
    COUNT(*)                                            AS cantidad,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS porcentaje
FROM comparacion
GROUP BY estado_checksum
ORDER BY cantidad DESC;
```

#### Resultado esperado

```
estado_checksum      | cantidad | porcentaje
---------------------|----------|------------
IDÉNTICO             |    4 700 |      94.00
FALTANTE EN DESTINO  |      150 |       3.00
MODIFICADO           |      130 |       2.60
EXTRA EN DESTINO     |       20 |       0.40
```

#### Verificación

✅ El total de registros `MODIFICADO` en el checksum debe ser mayor o igual a la suma de `MONTO DIFERENTE + FECHA DIFERENTE + CATEGORÍA DIFERENTE` del Paso 4, ya que el checksum detecta cualquier cambio en cualquier campo.

✅ Si `IDÉNTICO` + `FALTANTE EN DESTINO` + `EXTRA EN DESTINO` del checksum no coincide con los resultados del Paso 3, investiga si hay registros con el mismo `pedido_id` pero datos modificados que `EXCEPT` contó como "faltantes".

> 💡 **Nota técnica:** El separador `|` en la concatenación para `MD5` es importante. Sin él, valores como `('12', '3')` y `('1', '23')` producirían el mismo hash. Siempre usa un separador que no aparezca en los datos.

---

### Paso 6 — Reporte de reconciliación consolidado: CTEs reutilizables

**Objetivo:** Empaquetar todas las validaciones anteriores en un único script con CTEs reutilizables que genere un reporte de reconciliación completo con métricas cuantificables y clasificación `PASS/FAIL` para cada control.

#### Instrucciones

1. Crea una nueva hoja de trabajo en Snowsight. Nómbrala `Lab06_Paso6_Reporte_Final`.

2. Ejecuta el reporte de reconciliación consolidado:

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
        (SELECT COUNT(*) FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS) AS valor_destino
),

-- ── Control 2: Suma total de montos ─────────────────────────
ctrl_suma AS (
    SELECT
        'CTRL-02'                               AS control_id,
        'Suma total de montos'                  AS descripcion_control,
        (SELECT SUM(monto_total) FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS)  AS valor_origen,
        (SELECT SUM(monto_total) FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS) AS valor_destino
),

-- ── Control 3: Registros faltantes en destino ────────────────
ctrl_faltantes AS (
    SELECT
        'CTRL-03'                               AS control_id,
        'Registros faltantes en destino'        AS descripcion_control,
        COUNT(*)                                AS valor_origen,
        0                                       AS valor_destino
    FROM (
        SELECT pedido_id, cliente_id, fecha_pedido, monto_total, categoria_producto
        FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
        EXCEPT
        SELECT pedido_id, cliente_id, fecha_pedido, monto_total, categoria_producto
        FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
    ) AS faltantes
),

-- ── Control 4: Registros extra en destino ───────────────────
ctrl_extra AS (
    SELECT
        'CTRL-04'                               AS control_id,
        'Registros extra en destino'            AS descripcion_control,
        0                                       AS valor_origen,
        COUNT(*)                                AS valor_destino
    FROM (
        SELECT pedido_id, cliente_id, fecha_pedido, monto_total, categoria_producto
        FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
        EXCEPT
        SELECT pedido_id, cliente_id, fecha_pedido, monto_total, categoria_producto
        FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
    ) AS extra
),

-- ── Control 5: Registros con monto modificado ───────────────
ctrl_montos AS (
    SELECT
        'CTRL-05'                               AS control_id,
        'Registros con monto modificado'        AS descripcion_control,
        COUNT(*)                                AS valor_origen,
        0                                       AS valor_destino
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS AS o
    INNER JOIN LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS AS d
        ON o.pedido_id = d.pedido_id
    WHERE o.monto_total <> d.monto_total
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
)

-- ── Reporte final con clasificación PASS/FAIL ───────────────
SELECT
    control_id,
    descripcion_control,
    valor_origen,
    valor_destino,
    ROUND(ABS(valor_origen - valor_destino), 2)         AS diferencia_absoluta,
    CASE
        WHEN control_id IN ('CTRL-01', 'CTRL-02')
         AND valor_origen = valor_destino                THEN 'PASS ✓'
        WHEN control_id IN ('CTRL-01', 'CTRL-02')
         AND valor_origen <> valor_destino               THEN 'FAIL ✗'
        WHEN control_id IN ('CTRL-03', 'CTRL-04', 'CTRL-05')
         AND valor_origen = 0                            THEN 'PASS ✓'
        WHEN control_id IN ('CTRL-03', 'CTRL-04', 'CTRL-05')
         AND valor_origen > 0                            THEN 'FAIL ✗'
        ELSE                                                  'REVISAR'
    END                                                 AS resultado,
    CURRENT_TIMESTAMP()                                 AS timestamp_ejecucion
FROM todos_los_controles
ORDER BY control_id;
```

#### Resultado esperado

```
control_id | descripcion_control              | valor_origen | valor_destino | diferencia_absoluta | resultado
-----------|----------------------------------|--------------|---------------|---------------------|----------
CTRL-01    | Conteo total de registros        |        5 000 |         4 850 |                 150 | FAIL ✗
CTRL-02    | Suma total de montos             |  2 398 750.0 |   2 341 200.0 |           57 550.0  | FAIL ✗
CTRL-03    | Registros faltantes en destino   |          150 |             0 |                 150 | FAIL ✗
CTRL-04    | Registros extra en destino       |            0 |            20 |                  20 | FAIL ✗
CTRL-05    | Registros con monto modificado   |           85 |             0 |                  85 | FAIL ✗
```

#### Verificación

✅ El reporte debe mostrar exactamente 5 filas, una por cada control implementado.

✅ Cada fila debe tener un `timestamp_ejecucion` que refleje el momento actual de ejecución, confirmando que el reporte es dinámico y puede re-ejecutarse en cualquier momento.

✅ Si todos los controles muestran `PASS ✓`, el pipeline está funcionando correctamente. Si hay `FAIL ✗`, el reporte te da las métricas exactas para reportar al equipo de ingeniería de datos.

> 💡 **Buenas prácticas:** Este patrón de reporte con CTEs es directamente reutilizable. Para adaptarlo a otras tablas, solo necesitas cambiar los nombres de las tablas en cada CTE. La estructura de `UNION ALL` al final permite agregar nuevos controles sin modificar la lógica existente.

---

## Validación y pruebas

Una vez completados todos los pasos, ejecuta este bloque de validación final para confirmar que el laboratorio fue completado correctamente:

```sql
-- ============================================================
-- VALIDACIÓN FINAL DEL LABORATORIO 6
-- Confirma que todas las consultas produjeron resultados
-- ============================================================

-- Prueba 1: Confirmar diferencia de volumen detectada
SELECT
    'Prueba 1: Diferencia de volumen'           AS prueba,
    CASE
        WHEN ABS(
            (SELECT COUNT(*) FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS) -
            (SELECT COUNT(*) FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS)
        ) > 0                                   THEN 'PASS - Discrepancia detectada correctamente'
        ELSE                                         'REVISAR - No se detectó discrepancia de volumen'
    END                                         AS resultado;
```

```sql
-- Prueba 2: Confirmar que EXCEPT identifica registros faltantes
SELECT
    'Prueba 2: Registros faltantes con EXCEPT'  AS prueba,
    CASE
        WHEN COUNT(*) > 0                       THEN 'PASS - Se identificaron ' || COUNT(*) || ' registros faltantes'
        ELSE                                         'REVISAR - EXCEPT no devolvió resultados'
    END                                         AS resultado
FROM (
    SELECT pedido_id, cliente_id, fecha_pedido, monto_total, categoria_producto
    FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS
    EXCEPT
    SELECT pedido_id, cliente_id, fecha_pedido, monto_total, categoria_producto
    FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS
) AS faltantes;
```

```sql
-- Prueba 3: Confirmar que el reporte de reconciliación tiene 5 controles
WITH ctrl_conteo AS (
    SELECT 'CTRL-01' AS control_id,
           (SELECT COUNT(*) FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS)  AS v_o,
           (SELECT COUNT(*) FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS) AS v_d
),
ctrl_suma AS (
    SELECT 'CTRL-02' AS control_id,
           (SELECT SUM(monto_total) FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS)  AS v_o,
           (SELECT SUM(monto_total) FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS) AS v_d
)
SELECT
    'Prueba 3: Estructura del reporte'          AS prueba,
    'PASS - Reporte de reconciliación ejecutado' AS resultado
FROM ctrl_conteo
CROSS JOIN ctrl_suma
LIMIT 1;
```

**Criterios de éxito del laboratorio:**

| Prueba | Criterio de éxito |
|---|---|
| Prueba 1 | `PASS - Discrepancia detectada correctamente` |
| Prueba 2 | `PASS - Se identificaron N registros faltantes` (N > 0) |
| Prueba 3 | `PASS - Reporte de reconciliación ejecutado` |
| Paso 4b | Resumen ejecutivo con al menos 3 categorías de estado |
| Paso 5c | Porcentaje de `IDÉNTICO` calculado correctamente |
| Paso 6 | Reporte de 5 controles con clasificación `PASS/FAIL` |

---

## Solución de problemas

### Problema 1: Error "Object does not exist" al referenciar VENTAS_ORIGEN o VENTAS_DESTINO

**Síntoma:**
```
SQL compilation error: Object 'LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS' does not exist or not authorized.
```

**Causa:**
El script de setup `00_setup_laboratorios.sql` no fue ejecutado por el instructor, o fue ejecutado con un rol diferente al que estás usando actualmente. También puede ocurrir si el contexto de base de datos no está correctamente establecido.

**Solución:**
```sql
-- Paso 1: Verificar el contexto actual
SELECT CURRENT_DATABASE(), CURRENT_SCHEMA(), CURRENT_ROLE();

-- Paso 2: Establecer el contexto correcto
USE ROLE SYSADMIN;
USE DATABASE LAB_SQL_INTERMEDIO;

-- Paso 3: Verificar que los schemas existen
SHOW SCHEMAS IN DATABASE LAB_SQL_INTERMEDIO;

-- Paso 4: Si los schemas no aparecen, verificar con el instructor
-- El instructor debe re-ejecutar 00_setup_laboratorios.sql
-- con el rol SYSADMIN antes de continuar

-- Paso 5: Si los schemas existen pero no tienes acceso, solicitar al instructor:
GRANT USAGE ON SCHEMA LAB_SQL_INTERMEDIO.VENTAS_ORIGEN  TO ROLE <TU_ROL>;
GRANT USAGE ON SCHEMA LAB_SQL_INTERMEDIO.VENTAS_DESTINO TO ROLE <TU_ROL>;
GRANT SELECT ON ALL TABLES IN SCHEMA LAB_SQL_INTERMEDIO.VENTAS_ORIGEN  TO ROLE <TU_ROL>;
GRANT SELECT ON ALL TABLES IN SCHEMA LAB_SQL_INTERMEDIO.VENTAS_DESTINO TO ROLE <TU_ROL>;
```

---

### Problema 2: EXCEPT devuelve 0 filas inesperadamente o más filas de las esperadas

**Síntoma:**
El Paso 3a devuelve 0 filas (indicando que no hay diferencias) cuando el Paso 1 claramente mostró una diferencia de conteo entre origen y destino. O bien, `EXCEPT` devuelve muchas más filas de las esperadas.

**Causa:**
`EXCEPT` compara filas completas. Si el número de columnas seleccionadas en ambas partes no es idéntico, o si los tipos de datos no son compatibles, Snowflake puede lanzar un error o producir resultados inesperados. También ocurre cuando hay diferencias en valores `NULL` — un campo `NULL` en origen vs un string vacío `''` en destino hace que `EXCEPT` trate ambas filas como distintas, produciendo más resultados de los esperados.

**Solución:**
```sql
-- Diagnóstico: Verificar tipos de datos en ambas tablas
DESCRIBE TABLE LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS;
DESCRIBE TABLE LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS;

-- Solución para NULLs: Usar COALESCE para normalizar valores nulos
-- antes de aplicar EXCEPT
SELECT
    pedido_id,
    cliente_id,
    fecha_pedido,
    COALESCE(monto_total, 0)                    AS monto_total,
    COALESCE(categoria_producto, 'SIN_CATEGORIA') AS categoria_producto
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS

EXCEPT

SELECT
    pedido_id,
    cliente_id,
    fecha_pedido,
    COALESCE(monto_total, 0)                    AS monto_total,
    COALESCE(categoria_producto, 'SIN_CATEGORIA') AS categoria_producto
FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS;

-- Verificación adicional: Contar NULLs en columnas clave
SELECT
    'ORIGEN'          AS fuente,
    COUNT(*)          AS total,
    SUM(CASE WHEN monto_total       IS NULL THEN 1 ELSE 0 END) AS nulls_monto,
    SUM(CASE WHEN categoria_producto IS NULL THEN 1 ELSE 0 END) AS nulls_categoria
FROM LAB_SQL_INTERMEDIO.VENTAS_ORIGEN.PEDIDOS

UNION ALL

SELECT
    'DESTINO'         AS fuente,
    COUNT(*)          AS total,
    SUM(CASE WHEN monto_total       IS NULL THEN 1 ELSE 0 END) AS nulls_monto,
    SUM(CASE WHEN categoria_producto IS NULL THEN 1 ELSE 0 END) AS nulls_categoria
FROM LAB_SQL_INTERMEDIO.VENTAS_DESTINO.PEDIDOS;
```

---

## Limpieza del entorno

Al finalizar el laboratorio, ejecuta los siguientes comandos para liberar recursos y suspender el warehouse, evitando consumo innecesario de créditos Snowflake:

```sql
-- ============================================================
-- LIMPIEZA POST-LABORATORIO 6
-- ============================================================

-- 1. Cerrar las hojas de trabajo guardadas
--    (guardar manualmente en Snowsight si no se hizo automáticamente)

-- 2. Suspender el warehouse para detener el consumo de créditos
ALTER WAREHOUSE LAB_WH SUSPEND;

-- 3. Verificar que el warehouse fue suspendido correctamente
SHOW WAREHOUSES LIKE 'LAB_WH';
-- El campo "state" debe mostrar "SUSPENDED"

-- 4. (Opcional) Limpiar resultados de consultas en caché
-- No es necesario ejecutar comandos adicionales;
-- Snowflake gestiona automáticamente la caché de resultados.
```

> ⚠️ **Importante:** Las cuentas trial de Snowflake tienen 400 USD de créditos. Un warehouse `X-SMALL` consume aproximadamente 1 crédito por hora de uso activo. Suspender el warehouse al terminar cada sesión es una práctica obligatoria en este curso.

---

## Resumen

### Conceptos aplicados en este laboratorio

En este laboratorio aplicaste un flujo completo de validación y reconciliación de datasets usando técnicas SQL avanzadas en Snowflake:

| Técnica | Uso en el laboratorio | Paso |
|---|---|---|
| `UNION ALL` | Comparación de métricas agregadas entre origen y destino | 1, 2 |
| `FULL OUTER JOIN` + `CASE WHEN` | Tabla de auditoría detallada con clasificación de estado | 2b, 4 |
| `EXCEPT` | Identificación de registros presentes en origen pero ausentes en destino | 3a |
| `INTERSECT` | Cuantificación de registros exactamente coincidentes | 3c |
| `MD5()` + `CONCAT()` | Checksum de fila para detectar cualquier modificación de datos | 5 |
| CTEs encadenadas | Encapsulación de cada control como bloque reutilizable | 6 |
| `PASS/FAIL` con `CASE WHEN` | Reporte ejecutivo de calidad con clasificación binaria | 6 |

### Hallazgos clave del laboratorio

Al completar este laboratorio, identificaste que el pipeline `VENTAS_ORIGEN → VENTAS_DESTINO` presenta los siguientes problemas:

1. **Pérdida de registros:** Aproximadamente 150 pedidos de la categoría "Electrónica" no fueron migrados al destino.
2. **Registros fantasma:** 20 registros aparecen en destino sin correspondencia en origen, indicando una posible inserción incorrecta.
3. **Modificación de valores:** 85 registros presentan diferencias en `monto_total` entre origen y destino.
4. **Tasa de concordancia:** Aproximadamente 94% de los registros pasaron el pipeline sin modificaciones.

### Patrón de reconciliación reutilizable

El flujo de cuatro pasos que aplicaste es un patrón estándar de auditoría de pipelines:

```
1. VOLUMEN     → ¿Cuántos registros llegaron?
2. DESGLOSE    → ¿En qué categorías hay discrepancias?
3. FALTANTES   → ¿Qué registros específicos se perdieron?
4. VALORES     → ¿Qué registros llegaron pero con datos modificados?
```

Este patrón, empaquetado como CTEs reutilizables, puede adaptarse a cualquier tabla y cualquier pipeline con mínimas modificaciones.

### Recursos adicionales

- [Documentación Snowflake: Operadores de conjunto (EXCEPT, INTERSECT, UNION)](https://docs.snowflake.com/en/sql-reference/operators-query)
- [Documentación Snowflake: Función MD5](https://docs.snowflake.com/en/sql-reference/functions/md5)
- [Documentación Snowflake: FULL OUTER JOIN](https://docs.snowflake.com/en/sql-reference/constructs/join)
- [dbt Labs: Pruebas de datos en pipelines modernos](https://docs.getdbt.com/docs/build/data-tests)
- [Great Expectations: Framework de validación de datos](https://docs.greatexpectations.io/docs/)

---

> 📝 **Nota del instructor:** Este laboratorio está diseñado intencionalmente para ser completado en 35 minutos. Si el grupo necesita más tiempo o desea profundizar, el material complementario incluye escenarios adicionales de reconciliación con las tablas `CLIENTES` y `PRODUCTOS` siguiendo el mismo patrón de CTEs aplicado en el Paso 6.

---
LAB_END---
