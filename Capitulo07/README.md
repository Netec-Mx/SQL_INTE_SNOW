# Optimización y mejora de performance de queries

## Metadatos

| Campo            | Detalle                              |
|------------------|--------------------------------------|
| **Duración**     | 45 minutos                           |
| **Complejidad**  | Difícil                              |
| **Nivel Bloom**  | Aplicar (Apply)                      |
| **Módulo**       | 7 — Optimización y performance SQL   |
| **Plataforma**   | Snowflake (Snowsight)                |

---

## Descripción General

En este laboratorio aplicarás de forma integrada los principios de organización estructural, estilo SQL y optimización de performance aprendidos a lo largo del curso. Recibirás cinco queries funcionales pero mal escritos que resuelven problemas reales del dataset compartido (`LAB_SQL_INTERMEDIO`). Tu tarea es primero reformatearlos y documentarlos sin cambiar su lógica, luego identificar y corregir anti-patrones de performance, y finalmente medir el impacto de cada optimización usando el Query Profile de Snowsight. El laboratorio cierra con la construcción de una consulta final optimizada que integra todas las técnicas del curso.

---

## Objetivos de Aprendizaje

- [ ] Aplicar principios de organización por capas lógicas (CTEs con nombres descriptivos, indentación consistente y comentarios estratégicos) para refactorizar queries complejos sin alterar su lógica de negocio.
- [ ] Identificar y corregir anti-patrones de performance en Snowflake: `SELECT *`, funciones escalares en `WHERE`, CTEs repetidas innecesariamente y subqueries correlacionadas.
- [ ] Usar el Query Profile de Snowsight para comparar el plan de ejecución, tiempo por nodo y bytes escaneados antes y después de cada optimización.
- [ ] Consultar `INFORMATION_SCHEMA.QUERY_HISTORY` para obtener métricas objetivas de tiempo de ejecución y volumen de datos procesados.
- [ ] Construir una versión final optimizada de la consulta más compleja del curso aplicando todas las buenas prácticas consolidadas.

---

## Prerrequisitos

### Conocimiento previo
- Haber completado los Laboratorios 1 al 6 del curso (CTEs, subqueries, window functions, series temporales, reconciliación de datasets).
- Familiaridad con la interfaz de Snowsight: editor SQL, panel de resultados, pestaña **Query Profile** y **Query History**.
- Comprensión conceptual de cómo Snowflake ejecuta queries: virtual warehouses, micro-particiones y pruning automático.
- Conocimiento de `ROW_NUMBER()`, `RANK()`, `LAG()`/`LEAD()` y `QUALIFY` de laboratorios anteriores.

### Acceso requerido
- Cuenta Snowflake activa (trial o corporativa) con acceso a Snowsight.
- Base de datos `LAB_SQL_INTERMEDIO` creada por el script `00_setup_laboratorios.sql` (ejecutado por el instructor).
- Rol con privilegios `SELECT` sobre los schemas `VENTAS`, `VENTAS_ORIGEN` y `VENTAS_DESTINO`.
- Warehouse de tamaño `X-SMALL` disponible y activo.

---

## Entorno de Laboratorio

### Hardware recomendado

| Componente        | Mínimo                        | Recomendado                   |
|-------------------|-------------------------------|-------------------------------|
| Procesador        | Intel Core i5 / AMD Ryzen 5   | Intel Core i7 / AMD Ryzen 7   |
| RAM               | 8 GB                          | 16 GB                         |
| Almacenamiento    | 2 GB libres                   | 5 GB libres                   |
| Pantalla          | 1280×768                      | 1920×1080                     |
| Conexión          | 10 Mbps                       | 25 Mbps o superior            |

### Software requerido

| Software                        | Versión mínima       | Notas                                          |
|---------------------------------|----------------------|------------------------------------------------|
| Navegador web                   | Chrome/Firefox/Edge 110+ | Snowsight requiere JavaScript habilitado   |
| Snowflake (Snowsight)           | Cuenta activa        | Enterprise Edition o Trial                     |
| Visual Studio Code (opcional)   | 1.80+                | Para editar y guardar scripts localmente       |
| SnowSQL CLI (opcional)          | 1.2.x+               | Alternativa al editor web                     |

### Configuración inicial del entorno

Ejecuta el siguiente bloque al inicio de la sesión para establecer el contexto correcto. **No omitas este paso.**

```sql
-- ============================================================
-- Configuración de contexto para el Laboratorio 7
-- Ejecutar ANTES de cualquier ejercicio
-- ============================================================

USE ROLE     SYSADMIN;                      -- O el rol asignado por el instructor
USE WAREHOUSE LAB_WH;                       -- Warehouse X-SMALL del curso
USE DATABASE  LAB_SQL_INTERMEDIO;
USE SCHEMA    VENTAS;

-- Verificar que las tablas principales existen
SHOW TABLES;
```

**Salida esperada:** Debes ver al menos las tablas `CLIENTES`, `PEDIDOS`, `VENTAS` y `PRODUCTOS` en el resultado de `SHOW TABLES`.

> ⚠️ **Importante:** Al finalizar cada sesión ejecuta `ALTER WAREHOUSE LAB_WH SUSPEND;` para evitar consumo innecesario de créditos.

---

## Pasos del Laboratorio

---

### Paso 1 — Preparación: Registrar el estado inicial del warehouse

**Objetivo:** Establecer una línea base de métricas antes de ejecutar cualquier query optimizado, y familiarizarse con `QUERY_HISTORY` como herramienta de medición objetiva.

#### Instrucciones

1. En Snowsight, abre una nueva hoja de trabajo (worksheet) y nómbrala `Lab07_Optimizacion`.

2. Ejecuta el siguiente query para obtener las métricas de los últimos queries ejecutados en tu sesión. Lo usarás como referencia comparativa durante todo el laboratorio:

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
    partitions_scanned        AS particiones_escaneadas,
    partitions_total          AS particiones_totales,
    start_time
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION(
    RESULT_LIMIT => 20
))
WHERE query_type = 'SELECT'
ORDER BY start_time DESC;
```

3. Anota (o guarda en un documento aparte) los valores de `tiempo_ms` y `bytes_escaneados` de los últimos queries. Esta tabla será tu referencia durante el laboratorio.

#### Salida esperada

Una tabla con hasta 20 filas mostrando los queries recientes de tu sesión. Si acabas de iniciar sesión, puede aparecer vacía o con muy pocas filas — esto es normal.

#### Verificación

✅ La consulta se ejecuta sin errores y devuelve columnas con nombres legibles en español (alias aplicados correctamente).

---

### Paso 2 — Refactorización de estilo: Query 1 (Ventas por región)

**Objetivo:** Aplicar los principios de organización por capas, indentación consistente y comentarios estratégicos a un query desestructurado, sin cambiar su lógica.

#### Instrucciones

1. Lee y ejecuta el siguiente **query original** tal como está. Observa su resultado y anota el `query_id` desde `QUERY_HISTORY`:

```sql
select r.region_nombre,count(distinct p.pedido_id) as total_pedidos,sum(v.monto) as ingresos,avg(v.monto) as ticket_promedio from ventas v join pedidos p on v.pedido_id=p.pedido_id join clientes c on p.cliente_id=c.cliente_id join (select distinct cliente_id,region_nombre from clientes where region_nombre is not null) r on c.cliente_id=r.cliente_id where p.fecha_pedido>='2023-01-01' and p.estado='COMPLETADO' and v.monto>0 group by r.region_nombre having count(distinct p.pedido_id)>10 order by ingresos desc;
```

2. Ahora escribe la **versión refactorizada** aplicando todos los principios de la Lección 7.1. A continuación se muestra la solución de referencia — intenta escribirla tú primero antes de consultarla:

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
    FROM pedidos_completados  pc
    INNER JOIN ventas_positivas  vp ON pc.pedido_id  = vp.pedido_id
    INNER JOIN clientes           c  ON pc.cliente_id = c.cliente_id
    INNER JOIN regiones_validas   r  ON c.cliente_id  = r.cliente_id
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

3. Ejecuta la versión refactorizada y verifica que el resultado sea **idéntico** al del query original.

4. En Snowsight, haz clic en el ícono de **Query Profile** (ícono de gráfico junto al `query_id`) para ambas versiones y compara visualmente los nodos de ejecución.

#### Salida esperada

Ambos queries deben devolver exactamente el mismo número de filas y los mismos valores. El resultado es una tabla con columnas `REGION_NOMBRE`, `TOTAL_PEDIDOS`, `INGRESOS` y `TICKET_PROMEDIO`, ordenada de mayor a menor ingreso.

#### Verificación

✅ Los resultados de ambas versiones son idénticos (mismas filas, mismo orden, mismos valores).  
✅ La versión refactorizada tiene al menos 4 CTEs con nombres descriptivos en español.  
✅ Cada CTE tiene un comentario que explica el *por qué* del filtro, no solo el *qué*.

---

### Paso 3 — Anti-patrón 1: Eliminar SELECT * y columnas innecesarias

**Objetivo:** Identificar el impacto de `SELECT *` en queries sobre tablas grandes y reemplazarlo con selección explícita de columnas.

#### Instrucciones

1. Ejecuta el siguiente query que usa `SELECT *` y anota sus métricas de `bytes_scanned`:

```sql
-- QUERY ANTI-PATRÓN: SELECT * con join a tabla grande
-- Ejecutar para medir bytes_scanned ANTES de la corrección

SELECT *
FROM ventas v
JOIN pedidos p
    ON v.pedido_id = p.pedido_id
WHERE p.fecha_pedido BETWEEN '2023-01-01' AND '2023-12-31';
```

2. Inmediatamente después, consulta `QUERY_HISTORY` para capturar los bytes escaneados:

```sql
-- Capturar métricas del query anterior
SELECT
    query_id,
    SUBSTR(query_text, 1, 80)    AS query_resumen,
    bytes_scanned                AS bytes_escaneados,
    total_elapsed_time           AS tiempo_ms,
    partitions_scanned           AS particiones_escaneadas
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION(
    RESULT_LIMIT => 5
))
WHERE query_type = 'SELECT'
ORDER BY start_time DESC;
```

3. Ahora ejecuta la versión corregida, seleccionando **solo las columnas necesarias para un reporte de ventas**:

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
FROM ventas  v
INNER JOIN pedidos p
    ON v.pedido_id = p.pedido_id
WHERE
    p.fecha_pedido BETWEEN '2023-01-01' AND '2023-12-31';
```

4. Vuelve a capturar las métricas con `QUERY_HISTORY` y **compara** `bytes_scanned` entre ambas versiones.

5. Abre el **Query Profile** de ambas ejecuciones en Snowsight. Observa la diferencia en el nodo **"TableScan"**: el número de columnas proyectadas debe ser menor en la versión optimizada.

#### Salida esperada

La versión con `SELECT *` mostrará un valor de `bytes_escaneados` mayor o igual al de la versión optimizada. En tablas con muchas columnas anchas (como `VENTAS` con campos de texto largos), la diferencia puede ser significativa. Ambas versiones deben devolver el mismo número de filas.

#### Verificación

✅ La versión optimizada selecciona exactamente 7 columnas con nombres explícitos.  
✅ El `bytes_scanned` de la versión optimizada es ≤ al de la versión con `SELECT *`.  
✅ En el Query Profile, el nodo TableScan de la versión optimizada muestra menos columnas proyectadas.

> 📝 **Nota pedagógica:** En Snowflake, el impacto de `SELECT *` es menor que en bases de datos relacionales tradicionales porque el almacenamiento es columnar. Sin embargo, sigue siendo una mala práctica porque: (1) dificulta la lectura, (2) puede romper pipelines si cambia el esquema de la tabla, y (3) transfiere datos innecesarios a la capa de resultado.

---

### Paso 4 — Anti-patrón 2: Funciones escalares en WHERE que bloquean el pruning

**Objetivo:** Identificar cómo el uso de funciones sobre columnas en condiciones `WHERE` impide que Snowflake aplique pruning de micro-particiones, y reescribir los filtros para aprovechar esta optimización.

#### Instrucciones

1. Ejecuta el siguiente query que aplica `YEAR()` directamente sobre la columna de fecha en el `WHERE`. Anota `partitions_scanned` y `tiempo_ms`:

```sql
-- ANTI-PATRÓN: Función escalar sobre columna en WHERE
-- Snowflake no puede hacer pruning efectivo cuando la columna
-- está envuelta en una función

SELECT
    p.pedido_id,
    p.cliente_id,
    p.fecha_pedido,
    p.monto_total
FROM pedidos p
WHERE YEAR(p.fecha_pedido)  = 2023
  AND MONTH(p.fecha_pedido) = 6;
```

2. Captura las métricas:

```sql
SELECT
    query_id,
    total_elapsed_time    AS tiempo_ms,
    partitions_scanned    AS particiones_escaneadas,
    partitions_total      AS particiones_totales,
    bytes_scanned         AS bytes_escaneados
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION(
    RESULT_LIMIT => 3
))
WHERE query_type = 'SELECT'
ORDER BY start_time DESC
LIMIT 1;
```

3. Ahora ejecuta la versión corregida usando un rango de fechas explícito con `BETWEEN`:

```sql
-- ============================================================
-- Pedidos de junio 2023: versión optimizada con rango de fechas
-- Usar BETWEEN permite a Snowflake aplicar partition pruning
-- y escanear solo las micro-particiones del rango solicitado
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

4. Captura las métricas nuevamente y compara `partitions_scanned` entre ambas versiones.

5. En el Query Profile, busca el nodo **"Partition Pruning"** o revisa en el nodo TableScan la relación `partitions_scanned / partitions_total`. Una relación más baja en la versión optimizada indica mejor pruning.

#### Salida esperada

Ambas versiones deben devolver exactamente las mismas filas. La versión con `BETWEEN` debe mostrar un valor de `partitions_scanned` igual o menor al de la versión con `YEAR()`/`MONTH()`, especialmente si la tabla `PEDIDOS` tiene datos de múltiples años.

#### Verificación

✅ Ambas versiones devuelven el mismo conjunto de filas.  
✅ La versión con `BETWEEN` tiene `partitions_scanned` ≤ a la versión con funciones en `WHERE`.  
✅ El Query Profile de la versión optimizada muestra una mejor relación `partitions_scanned / partitions_total`.

> ⚠️ **Nota sobre QUALIFY:** En laboratorios anteriores usaste `QUALIFY` para filtrar sobre window functions. Esta cláusula es exclusiva de Snowflake y no existe en SQL estándar (PostgreSQL, MySQL). El mismo principio aplica aquí: cuando sea posible, filtra sobre la columna directamente, no sobre una función que la envuelva.

---

### Paso 5 — Anti-patrón 3: Subquery correlacionada vs. JOIN

**Objetivo:** Identificar una subquery correlacionada que se ejecuta una vez por cada fila del query exterior, y reemplazarla con un `JOIN` equivalente que Snowflake puede optimizar como una sola operación.

#### Instrucciones

1. Ejecuta el siguiente query con subquery correlacionada. Anota `tiempo_ms` y `bytes_scanned`:

```sql
-- ANTI-PATRÓN: Subquery correlacionada
-- Este patrón ejecuta la subquery UNA VEZ POR CADA FILA de clientes
-- En tablas grandes, esto puede multiplicar el tiempo de ejecución

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

2. Captura las métricas de esta ejecución.

3. Ahora reescribe el mismo resultado usando un `LEFT JOIN` con agregación previa en una CTE:

```sql
-- ============================================================
-- Total de compras 2023 por cliente activo
-- Versión optimizada: JOIN reemplaza subquery correlacionada
-- Snowflake puede ejecutar la agregación como un solo hash join
-- en lugar de una subquery repetida por cada cliente
-- ============================================================

WITH compras_2023 AS (
    -- Pre-agregamos por cliente ANTES del join
    -- Esto evita que Snowflake repita el cálculo por cada fila de clientes
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

4. Captura las métricas de la versión con `JOIN` y compara con la subquery correlacionada.

5. En el **Query Profile**, compara la estructura de nodos entre ambas versiones:
   - La subquery correlacionada puede mostrar un nodo **"NestedLoop"** o múltiples nodos de escaneo repetidos.
   - La versión con JOIN debe mostrar un nodo **"HashJoin"** más eficiente.

#### Salida esperada

Ambas versiones deben devolver exactamente las mismas filas y valores. La versión con `LEFT JOIN` debe tener un `tiempo_ms` igual o menor. Los clientes sin compras en 2023 aparecen con `NULL` en `total_compras_2023` en ambas versiones.

#### Verificación

✅ Ambas versiones devuelven el mismo número de filas con los mismos valores.  
✅ La versión con JOIN tiene `tiempo_ms` ≤ a la versión con subquery correlacionada.  
✅ El Query Profile de la versión optimizada muestra un nodo HashJoin en lugar de NestedLoop.

---

### Paso 6 — Anti-patrón 4: CTE referenciada múltiples veces

**Objetivo:** Identificar una CTE que se referencia más de una vez en el mismo query (lo que puede implicar recálculo), y evaluar si materializarla o reestructurar el query mejora el performance.

#### Instrucciones

1. Ejecuta el siguiente query donde la CTE `ventas_base` se usa **tres veces** en el query final:

```sql
-- ANTI-PATRÓN: CTE referenciada múltiples veces
-- En Snowflake, una CTE puede recalcularse cada vez que se referencia.
-- Si la CTE es costosa, esto puede triplicar el trabajo.

WITH ventas_base AS (
    SELECT
        v.venta_id,
        v.pedido_id,
        v.monto,
        v.categoria_producto,
        p.fecha_pedido,
        p.cliente_id
    FROM ventas  v
    INNER JOIN pedidos p ON v.pedido_id = p.pedido_id
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

2. Captura las métricas de esta ejecución.

3. Ahora reescribe el query usando una sola pasada con `GROUPING SETS` o con `CASE WHEN` en la agregación, eliminando las referencias múltiples a la CTE:

```sql
-- ============================================================
-- Resumen de ventas 2023 por categoría seleccionada
-- Versión optimizada: una sola pasada sobre ventas_base
-- Reemplaza UNION ALL con CASE WHEN en la agregación
-- ============================================================

WITH ventas_base AS (
    -- Fuente única: ventas completadas desde inicio de 2023
    SELECT
        v.venta_id,
        v.pedido_id,
        v.monto,
        v.categoria_producto,
        p.fecha_pedido,
        p.cliente_id
    FROM ventas  v
    INNER JOIN pedidos p
        ON v.pedido_id = p.pedido_id
    WHERE
        p.fecha_pedido >= '2023-01-01'
        AND p.estado    = 'COMPLETADO'
)

-- Una sola pasada: calculamos todas las métricas en un GROUP BY
SELECT
    CASE
        WHEN categoria_producto = 'ELECTRONICA' THEN 'Categoría: ELECTRONICA'
        WHEN categoria_producto = 'ROPA'        THEN 'Categoría: ROPA'
        ELSE                                         'Total general'
    END                                              AS metrica,
    COUNT(*)                                         AS valor_conteo,
    SUM(monto)                                       AS valor_suma
FROM ventas_base
WHERE
    categoria_producto IN ('ELECTRONICA', 'ROPA')
    OR TRUE   -- incluye todas las filas para el total general

GROUP BY
    CASE
        WHEN categoria_producto = 'ELECTRONICA' THEN 'Categoría: ELECTRONICA'
        WHEN categoria_producto = 'ROPA'        THEN 'Categoría: ROPA'
        ELSE                                         'Total general'
    END
ORDER BY valor_suma DESC;
```

> 📝 **Alternativa con GROUPING SETS** (más elegante para múltiples categorías):

```sql
-- ============================================================
-- Alternativa con GROUPING SETS: más limpia para N categorías
-- ============================================================

WITH ventas_base AS (
    SELECT
        v.monto,
        v.categoria_producto
    FROM ventas  v
    INNER JOIN pedidos p
        ON v.pedido_id = p.pedido_id
    WHERE
        p.fecha_pedido >= '2023-01-01'
        AND p.estado    = 'COMPLETADO'
        AND v.categoria_producto IN ('ELECTRONICA', 'ROPA')
)

SELECT
    COALESCE(categoria_producto, 'Total general') AS metrica,
    COUNT(*)                                       AS valor_conteo,
    SUM(monto)                                     AS valor_suma
FROM ventas_base
GROUP BY GROUPING SETS (
    (categoria_producto),   -- subtotal por categoría
    ()                      -- total general (fila con NULL en categoria)
)
ORDER BY valor_suma DESC;
```

4. Compara las métricas de ejecución entre la versión con `UNION ALL` y la versión con `GROUPING SETS`.

#### Salida esperada

La versión con `GROUPING SETS` debe devolver 3 filas: una para `ELECTRONICA`, una para `ROPA` y una para `Total general`. Los valores deben coincidir con los del query original con `UNION ALL`.

#### Verificación

✅ Ambas versiones producen el mismo resultado (mismas 3 filas, mismos valores).  
✅ La versión con `GROUPING SETS` referencia la CTE **una sola vez**.  
✅ Las métricas de `bytes_scanned` de la versión optimizada son ≤ a la versión con `UNION ALL`.

---

### Paso 7 — Consulta final integrada: Síntesis de todos los módulos

**Objetivo:** Construir desde cero la consulta más compleja del curso, integrando CTEs organizadas por capas, window functions, filtros optimizados y documentación completa, aplicando todas las buenas prácticas consolidadas.

#### Instrucciones

El siguiente es el **requerimiento de negocio** que debes implementar:

> *"Necesitamos un reporte que muestre, para cada región y categoría de producto, el top 3 de clientes por ingresos en el año 2023, junto con su variación porcentual respecto al año anterior (2022). Solo incluir combinaciones región-categoría con al menos 5 clientes distintos. Ordenar por región, categoría y ranking."*

1. Antes de escribir el query, diseña la estructura de capas en comentarios:

```sql
-- DISEÑO DE CAPAS:
-- CTE 1: ventas_por_anio    → Ventas de 2022 y 2023 con cliente y categoría
-- CTE 2: ingresos_2022      → Agregado por cliente-categoría-región para 2022
-- CTE 3: ingresos_2023      → Agregado por cliente-categoría-región para 2023
-- CTE 4: comparacion        → JOIN entre 2022 y 2023 + cálculo de variación %
-- CTE 5: ranking_por_region → ROW_NUMBER() por región-categoría
-- CTE 6: regiones_validas   → Filtro: solo combinaciones con >= 5 clientes
-- SELECT final              → Top 3 por región-categoría
```

2. Implementa el query completo:

```sql
-- ============================================================
-- Top 3 clientes por región-categoría con variación anual
-- Requerimiento: Análisis comparativo 2022 vs 2023
-- Solo combinaciones con >= 5 clientes distintos
-- Técnicas: CTEs, window functions, LEFT JOIN, filtros optimizados
-- Autor: [Tu nombre] | Laboratorio 7 — Consulta Final
-- ============================================================

WITH ventas_con_contexto AS (
    -- Base: ventas de 2022 y 2023 enriquecidas con cliente y región
    -- Filtro por rango de fechas para aprovechar partition pruning
    SELECT
        v.monto,
        v.categoria_producto,
        p.cliente_id,
        p.fecha_pedido,
        YEAR(p.fecha_pedido)  AS anio,
        c.region_nombre
    FROM ventas  v
    INNER JOIN pedidos  p ON v.pedido_id  = p.pedido_id
    INNER JOIN clientes c ON p.cliente_id = c.cliente_id
    WHERE
        p.fecha_pedido >= '2022-01-01'          -- inicio del rango comparativo
        AND p.fecha_pedido  < '2024-01-01'      -- cierre del rango comparativo
        AND p.estado        = 'COMPLETADO'
        AND c.region_nombre IS NOT NULL          -- excluye registros legacy sin región
        AND v.monto         > 0                  -- excluye ajustes contables negativos
),

ingresos_2022 AS (
    -- Ingresos por cliente, categoría y región en el año base (2022)
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
    -- Ingresos por cliente, categoría y región en el año de análisis (2023)
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
    -- Comparación 2022 vs 2023 con variación porcentual
    -- LEFT JOIN para incluir clientes nuevos en 2023 (sin historial en 2022)
    SELECT
        i23.cliente_id,
        i23.categoria_producto,
        i23.region_nombre,
        i23.ingresos_2023,
        i22.ingresos_2022,
        CASE
            WHEN i22.ingresos_2022 IS NULL OR i22.ingresos_2022 = 0
                THEN NULL   -- cliente nuevo: variación no aplica
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
    -- Ranking de clientes dentro de cada región-categoría por ingresos 2023
    -- ROW_NUMBER garantiza ranking único sin empates
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
    -- Solo combinaciones región-categoría con al menos 5 clientes distintos
    -- Esto filtra segmentos con muestra estadística insuficiente
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
    -- Aplicamos el filtro de top 3 Y el filtro de combinaciones válidas
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
    c.nombre                                            AS cliente,
    ROUND(ts.ingresos_2023, 2)                          AS ingresos_2023,
    ROUND(COALESCE(ts.ingresos_2022, 0), 2)             AS ingresos_2022,
    COALESCE(CAST(ts.variacion_pct AS VARCHAR), 'Nuevo') AS variacion_pct
FROM top3_por_segmento ts
INNER JOIN clientes c
    ON ts.cliente_id = c.cliente_id
ORDER BY
    ts.region_nombre       ASC,
    ts.categoria_producto  ASC,
    ts.ranking             ASC;
```

3. Ejecuta el query y verifica que devuelve resultados coherentes.

4. Abre el **Query Profile** de esta ejecución. Identifica y anota:
   - El nodo que más tiempo consume (generalmente el primer `TableScan` o el `HashJoin` más grande).
   - La relación `partitions_scanned / partitions_total` en los nodos TableScan.
   - El número total de bytes escaneados.

5. Captura las métricas finales con `QUERY_HISTORY`:

```sql
-- Métricas de la consulta final integrada
SELECT
    query_id,
    total_elapsed_time           AS tiempo_ms,
    bytes_scanned                AS bytes_escaneados,
    rows_produced                AS filas_resultado,
    partitions_scanned           AS particiones_escaneadas,
    partitions_total             AS particiones_totales,
    ROUND(
        partitions_scanned * 100.0 / NULLIF(partitions_total, 0),
        1
    )                            AS pct_particiones_escaneadas
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION(
    RESULT_LIMIT => 3
))
WHERE query_type = 'SELECT'
ORDER BY start_time DESC
LIMIT 1;
```

#### Salida esperada

Una tabla con columnas `REGION_NOMBRE`, `CATEGORIA_PRODUCTO`, `RANKING` (1, 2 o 3), `CLIENTE`, `INGRESOS_2023`, `INGRESOS_2022` y `VARIACION_PCT`. Solo aparecen combinaciones región-categoría con al menos 5 clientes. Los clientes nuevos en 2023 muestran `'Nuevo'` en la columna de variación.

#### Verificación

✅ El query tiene al menos 6 CTEs con nombres descriptivos en español.  
✅ Cada CTE tiene un comentario de bloque explicando su propósito de negocio.  
✅ No hay `SELECT *` en ninguna capa.  
✅ Los filtros de fecha usan rangos explícitos (`>=` y `<`) en lugar de funciones como `YEAR()`.  
✅ El `ranking` por región-categoría va de 1 a 3 sin saltos.  
✅ Las métricas de `QUERY_HISTORY` muestran un `pct_particiones_escaneadas` razonable (idealmente < 50% si la tabla tiene datos de múltiples años).

---

## Validación y Pruebas Finales

Ejecuta el siguiente bloque de validación al finalizar todos los pasos. Cada sentencia debe devolver el resultado indicado:

```sql
-- ============================================================
-- BLOQUE DE VALIDACIÓN FINAL — Laboratorio 7
-- Ejecutar completo al terminar todos los ejercicios
-- ============================================================

-- VALIDACIÓN 1: La consulta final devuelve solo rankings 1, 2 y 3
-- Resultado esperado: 0 filas (no debe haber rankings fuera de rango)
WITH top3_check AS (
    SELECT DISTINCT ranking
    FROM (
        -- Pega aquí el SELECT final del Paso 7 sin el ORDER BY
        -- o referencia el query guardado en tu worksheet
        SELECT 1 AS ranking  -- placeholder: reemplazar con query real
    )
)
SELECT ranking
FROM top3_check
WHERE ranking NOT IN (1, 2, 3);
-- Resultado esperado: 0 filas

-- VALIDACIÓN 2: QUERY_HISTORY muestra al menos 8 queries SELECT en la sesión
SELECT COUNT(*) AS total_queries_select
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION(
    RESULT_LIMIT => 50
))
WHERE query_type = 'SELECT';
-- Resultado esperado: >= 8

-- VALIDACIÓN 3: Comparar bytes_scanned entre SELECT * y SELECT explícito
-- (queries ejecutados en el Paso 3)
SELECT
    SUBSTR(query_text, 1, 50)  AS query_inicio,
    bytes_scanned               AS bytes_escaneados,
    total_elapsed_time          AS tiempo_ms
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION(
    RESULT_LIMIT => 30
))
WHERE
    query_type = 'SELECT'
    AND query_text ILIKE '%ventas%pedidos%'
    AND query_text ILIKE '%fecha_pedido%BETWEEN%'
ORDER BY start_time DESC
LIMIT 4;
-- Resultado esperado: filas con variación en bytes_scanned entre versiones

-- VALIDACIÓN 4: El warehouse sigue siendo X-SMALL
SHOW WAREHOUSES LIKE 'LAB_WH';
-- Resultado esperado: columna SIZE = 'X-Small'
```

---

## Solución de Problemas

### Problema 1: El Query Profile no muestra diferencias de particiones entre la versión con funciones y la versión con BETWEEN

**Síntomas:** Ambas versiones del Paso 4 muestran exactamente el mismo `partitions_scanned`, y el Query Profile no refleja ninguna mejora de pruning.

**Causa:** La tabla `PEDIDOS` en el entorno de laboratorio puede contener datos de un solo año o estar almacenada en pocas micro-particiones. El pruning de particiones solo es visible cuando los datos están distribuidos en múltiples particiones temporales y la tabla tiene suficiente volumen. En datasets pequeños (< 100,000 filas), todas las particiones son escaneadas de todas formas porque Snowflake decide que el overhead del pruning no vale la pena.

**Solución:**
1. Verifica el volumen de la tabla con `SELECT COUNT(*), MIN(fecha_pedido), MAX(fecha_pedido) FROM pedidos;`. Si el rango de fechas es menor a 6 meses, el efecto de pruning será mínimo.
2. Acepta el resultado como válido conceptualmente: el principio de evitar funciones en `WHERE` es correcto y su impacto escala con el volumen de datos.
3. El instructor puede demostrar el efecto en una tabla más grande si está disponible en el entorno.
4. Documenta en tus notas que este anti-patrón tiene impacto proporcional al volumen de datos y al rango de fechas cubierto.

---

### Problema 2: La consulta final del Paso 7 devuelve 0 filas aunque los datos existen

**Síntomas:** El query del Paso 7 se ejecuta sin errores pero devuelve una tabla vacía. Al ejecutar CTEs individuales sí aparecen datos.

**Causa:** El filtro `HAVING COUNT(DISTINCT cliente_id) >= 5` en la CTE `combinaciones_validas` es demasiado restrictivo para el dataset de laboratorio. Si el dataset tiene pocas filas por combinación región-categoría, ninguna combinación supera el umbral de 5 clientes distintos, y el `INNER JOIN` final elimina todas las filas.

**Solución:**
1. Ejecuta primero la CTE `combinaciones_validas` de forma aislada para ver cuántos clientes hay por combinación:
```sql
-- Diagnóstico: ¿cuántas combinaciones superan el umbral?
SELECT
    region_nombre,
    categoria_producto,
    COUNT(DISTINCT cliente_id) AS clientes_distintos
FROM (
    SELECT
        p.cliente_id,
        v.categoria_producto,
        c.region_nombre
    FROM ventas  v
    INNER JOIN pedidos  p ON v.pedido_id  = p.pedido_id
    INNER JOIN clientes c ON p.cliente_id = c.cliente_id
    WHERE
        p.fecha_pedido >= '2023-01-01'
        AND p.fecha_pedido  < '2024-01-01'
        AND p.estado        = 'COMPLETADO'
        AND c.region_nombre IS NOT NULL
) base
GROUP BY region_nombre, categoria_producto
ORDER BY clientes_distintos DESC;
```
2. Si el máximo de `clientes_distintos` es menor a 5, ajusta el umbral a `>= 2` o `>= 3` en la CTE `combinaciones_validas` para que el laboratorio produzca resultados visibles.
3. Documenta el cambio con un comentario: `-- Umbral ajustado a 2 por volumen reducido del dataset de laboratorio`.

---

## Limpieza del Entorno

Ejecuta el siguiente bloque al finalizar el laboratorio para liberar recursos y evitar consumo de créditos:

```sql
-- ============================================================
-- LIMPIEZA POST-LABORATORIO 7
-- Ejecutar obligatoriamente al terminar la sesión
-- ============================================================

-- 1. Suspender el warehouse para detener el consumo de créditos
ALTER WAREHOUSE LAB_WH SUSPEND;

-- 2. Verificar que el warehouse quedó suspendido
SHOW WAREHOUSES LIKE 'LAB_WH';
-- Confirmar que la columna STATE = 'SUSPENDED'

-- 3. (Opcional) Limpiar worksheets temporales si se crearon objetos de prueba
-- No se crearon tablas ni vistas en este laboratorio, solo queries SELECT.
-- No se requiere DROP de objetos.

-- 4. Cerrar sesión en Snowsight o mantenerla activa solo si se continúa trabajando
-- Una sesión inactiva sin warehouse activo NO consume créditos.
```

> ✅ **Confirmación de limpieza:** El warehouse `LAB_WH` debe aparecer con `STATE = SUSPENDED` en el resultado de `SHOW WAREHOUSES`. Si el estado sigue siendo `STARTED`, repite el comando `ALTER WAREHOUSE LAB_WH SUSPEND`.

---

## Resumen

### Lo que aprendiste en este laboratorio

En este laboratorio aplicaste de forma integrada las técnicas de optimización SQL más importantes para entornos Snowflake profesionales:

| Técnica aplicada                          | Beneficio principal                                              |
|-------------------------------------------|------------------------------------------------------------------|
| Organización por capas con CTEs           | Legibilidad, mantenibilidad y depuración más rápida              |
| Indentación y comentarios estratégicos    | Comunicación efectiva con el equipo y con tu yo futuro           |
| Eliminación de SELECT *                   | Menor transferencia de datos y esquemas más robustos             |
| Filtros con rangos en lugar de funciones  | Mejor partition pruning → menos bytes escaneados                 |
| JOIN en lugar de subqueries correlacionadas | HashJoin vs NestedLoop → escalabilidad en tablas grandes       |
| CTE referenciada una sola vez             | Evita recálculos innecesarios en datasets grandes                |
| Medición con Query Profile y QUERY_HISTORY | Decisiones de optimización basadas en evidencia, no en intuición |

### Principios consolidados del curso

1. **Escribe para humanos primero, para la máquina segundo.** Un query que nadie entiende no puede mantenerse ni auditarse.
2. **Mide antes de optimizar.** El Query Profile y `QUERY_HISTORY` son tus aliados para tomar decisiones informadas.
3. **Filtra temprano y con precisión.** Los filtros sobre columnas directas (sin funciones) permiten que Snowflake haga su trabajo de pruning.
4. **Una CTE, una responsabilidad.** Cada bloque debe tener un nombre que describa exactamente qué contiene.
5. **`QUALIFY` es una ventaja de Snowflake, no SQL estándar.** Úsala en Snowflake con confianza, pero documenta que no es portable a PostgreSQL o MySQL.

### Recursos de referencia

- [Snowflake: Query Best Practices](https://docs.snowflake.com/en/user-guide/query-best-practices)
- [Snowflake: Understanding Query Profile](https://docs.snowflake.com/en/user-guide/ui-query-profile)
- [Snowflake: INFORMATION_SCHEMA.QUERY_HISTORY](https://docs.snowflake.com/en/sql-reference/functions/query_history)
- [Snowflake: Common Table Expressions](https://docs.snowflake.com/en/user-guide/queries-cte)
- [SQL Style Guide — Simon Holywell](https://www.sqlstyle.guide/)
- [dbt Labs: How we style our SQL](https://docs.getdbt.com/best-practices/how-we-style/1-how-we-style-our-dbt-models)
- [Snowflake: Micro-partition and Data Clustering](https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions)

---
