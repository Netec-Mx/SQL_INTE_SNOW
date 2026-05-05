# Análisis de rankings y secuencias con window functions

## 1. Metadatos

| Campo             | Detalle                                      |
|-------------------|----------------------------------------------|
| **Duración**      | 60 minutos                                   |
| **Complejidad**   | Alta (Hard)                                  |
| **Nivel Bloom**   | Aplicar (Apply)                              |
| **Módulo**        | Capítulo 4 — Funciones Analíticas en Snowflake |
| **Laboratorio**   | 04-00-01                                     |

---

## 2. Descripción General

En este laboratorio aplicarás las funciones analíticas (window functions) más importantes de Snowflake sobre datos reales de ventas y pedidos. Trabajarás con `RANK`, `DENSE_RANK` y `ROW_NUMBER` para construir rankings de productos y vendedores por región, observando cómo cada función trata los empates de manera diferente. Luego usarás `LAG` y `LEAD` para comparar ventas entre períodos consecutivos, y finalizarás construyendo acumulados y promedios móviles con `SUM() OVER()` y `AVG() OVER()` usando marcos de ventana (`ROWS BETWEEN`). Cada ejercicio incluye una variante con `PARTITION BY` para segmentar el análisis por región o categoría.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Aplicar `ROW_NUMBER`, `RANK` y `DENSE_RANK` para generar rankings de productos, vendedores y clientes según métricas de negocio, identificando las diferencias en el manejo de empates.
- [ ] Usar `LAG` y `LEAD` para calcular la variación de ventas respecto al período anterior y siguiente en una serie temporal.
- [ ] Dominar la sintaxis `OVER(PARTITION BY ... ORDER BY ...)` para controlar el alcance y orden de las funciones analíticas dentro de grupos de datos.
- [ ] Calcular totales acumulados y promedios móviles usando `SUM() OVER()` y `AVG() OVER()` con marcos de ventana (`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`).
- [ ] Filtrar resultados de window functions directamente con la cláusula `QUALIFY`, reconociéndola como una extensión exclusiva de Snowflake.

---

## 4. Prerrequisitos

### Conocimiento previo requerido

- Haber completado el **Laboratorio 3** (introducción a `ROW_NUMBER` en contexto de deduplicación).
- Comprensión sólida de `ORDER BY`, `GROUP BY` y funciones de agregación (`SUM`, `AVG`, `COUNT`).
- Manejo de **CTEs** (`WITH ... AS (...)`) para organizar consultas complejas.
- Lectura de la lección **4.1: Introducción a funciones analíticas** (estructura `OVER`, diferencia entre agregación y función analítica).

### Acceso requerido

- Cuenta activa en **Snowflake** (Trial o corporativa) con acceso a Snowsight.
- El instructor debe haber ejecutado el script `00_setup_laboratorios.sql` que crea la base de datos `LAB_SQL_INTERMEDIO` con los schemas `VENTAS`, las tablas `CLIENTES`, `PEDIDOS`, `VENTAS` y `PRODUCTOS`.
- Rol asignado con permisos de `SELECT` sobre `LAB_SQL_INTERMEDIO.VENTAS.*`.

---

## 5. Entorno del Laboratorio

### Hardware recomendado

| Componente       | Mínimo                          | Recomendado                     |
|------------------|---------------------------------|---------------------------------|
| Procesador       | Intel Core i5 / AMD Ryzen 5     | Intel Core i7 / AMD Ryzen 7     |
| RAM              | 8 GB                            | 16 GB                           |
| Almacenamiento   | 2 GB libres                     | 5 GB libres                     |
| Pantalla         | 1280×768                        | 1920×1080                       |
| Conexión         | 10 Mbps                         | 25 Mbps o superior              |

### Software requerido

| Software                  | Versión mínima         | Uso en el lab                          |
|---------------------------|------------------------|----------------------------------------|
| Snowflake (Snowsight)     | Enterprise / Trial     | Ejecución de todas las consultas SQL   |
| Navegador web             | Chrome 110+ / Edge 110+| Acceso a la interfaz Snowsight         |
| Visual Studio Code        | 1.80+ (opcional)       | Edición local de scripts SQL           |
| SnowSQL CLI               | 1.2.x (opcional)       | Ejecución alternativa por terminal     |

### Configuración inicial del entorno

Antes de comenzar los ejercicios, ejecuta los siguientes comandos en una hoja de trabajo nueva en Snowsight para establecer el contexto correcto:

```sql
-- Paso 1: Seleccionar el warehouse de tamaño X-SMALL para minimizar consumo de créditos
USE WAREHOUSE LAB_WH;

-- Paso 2: Seleccionar la base de datos y schema del laboratorio
USE DATABASE LAB_SQL_INTERMEDIO;
USE SCHEMA VENTAS;

-- Paso 3: Verificar que las tablas necesarias existen y tienen datos
SELECT 'VENTAS'    AS tabla, COUNT(*) AS filas FROM VENTAS
UNION ALL
SELECT 'PEDIDOS'   AS tabla, COUNT(*) AS filas FROM PEDIDOS
UNION ALL
SELECT 'PRODUCTOS' AS tabla, COUNT(*) AS filas FROM PRODUCTOS
UNION ALL
SELECT 'CLIENTES'  AS tabla, COUNT(*) AS filas FROM CLIENTES;
```

**Resultado esperado de la verificación:** Deberías ver las cuatro tablas con conteos mayores a cero. Si alguna tabla aparece con 0 filas o arroja un error, avisa al instructor para que ejecute nuevamente el script de setup.

> ⚠️ **Importante:** Usa siempre un warehouse de tamaño `X-SMALL`. Las cuentas Trial tienen 400 USD de créditos; warehouses más grandes consumen créditos innecesariamente para este laboratorio.

---

## 6. Ejercicios Paso a Paso

---

### Ejercicio 1 — Exploración inicial: entendiendo la ventana de datos

**Objetivo:** Familiarizarte con la estructura de los datos de ventas mensuales y observar la diferencia fundamental entre una agregación con `GROUP BY` y una función analítica con `OVER()`.

#### Instrucciones

**1.1** Abre una nueva hoja de trabajo en Snowsight. Nombra la pestaña `Lab04 - Ejercicio 1`.

**1.2** Ejecuta la siguiente consulta para explorar la estructura de la tabla `VENTAS`:

```sql
-- Exploración de la tabla VENTAS
SELECT
    v.venta_id,
    v.producto_id,
    p.nombre_producto,
    p.categoria,
    v.region,
    v.vendedor_id,
    v.fecha_venta,
    DATE_TRUNC('month', v.fecha_venta) AS mes_venta,
    v.monto_venta
FROM VENTAS v
JOIN PRODUCTOS p ON v.producto_id = p.producto_id
ORDER BY v.fecha_venta
LIMIT 20;
```

**1.3** Ahora compara el comportamiento de `GROUP BY` versus una window function ejecutando ambas consultas en paralelo:

```sql
-- CONSULTA A: Agregación tradicional con GROUP BY
-- Resultado: UNA fila por región (detalle de ventas individuales PERDIDO)
SELECT
    region,
    SUM(monto_venta) AS total_ventas_region
FROM VENTAS
GROUP BY region
ORDER BY total_ventas_region DESC;
```

```sql
-- CONSULTA B: Función analítica con OVER()
-- Resultado: TODAS las filas originales + columna con el total de la región
SELECT
    venta_id,
    region,
    vendedor_id,
    monto_venta,
    SUM(monto_venta) OVER (PARTITION BY region) AS total_ventas_region,
    ROUND(monto_venta / SUM(monto_venta) OVER (PARTITION BY region) * 100, 2) AS pct_contribucion
FROM VENTAS
ORDER BY region, monto_venta DESC
LIMIT 30;
```

#### Resultado esperado

- La **Consulta A** devuelve una fila por región (por ejemplo, 4 filas si hay 4 regiones), sin detalle individual.
- La **Consulta B** devuelve todas las filas de `VENTAS`, pero cada una tiene la columna `total_ventas_region` (el mismo valor para todas las filas de la misma región) y el porcentaje de contribución de esa venta al total de su región.

#### Verificación

Confirma que en la Consulta B el porcentaje (`pct_contribucion`) de todas las filas de una misma región suma aproximadamente 100%. Puedes verificarlo con:

```sql
SELECT
    region,
    SUM(ROUND(monto_venta / SUM(monto_venta) OVER (PARTITION BY region) * 100, 2)) AS suma_porcentajes
FROM VENTAS
GROUP BY region;
```

> 💡 **Punto clave:** `PARTITION BY region` en la cláusula `OVER()` actúa como un "GROUP BY interno" que no colapsa las filas. Cada fila conserva su identidad y además recibe el valor calculado sobre su partición.

---

### Ejercicio 2 — Rankings con RANK, DENSE_RANK y ROW_NUMBER

**Objetivo:** Construir un ranking de los top 5 productos por región usando las tres funciones de numeración, observando cómo cada una maneja los empates de forma diferente.

#### Instrucciones

**2.1** Abre una nueva hoja de trabajo. Nombra la pestaña `Lab04 - Ejercicio 2`.

**2.2** Primero construye la base del ranking: ventas totales por producto y región:

```sql
-- CTE base: total de ventas por producto y región
WITH ventas_por_producto_region AS (
    SELECT
        p.nombre_producto,
        p.categoria,
        v.region,
        SUM(v.monto_venta) AS total_ventas
    FROM VENTAS v
    JOIN PRODUCTOS p ON v.producto_id = p.producto_id
    GROUP BY p.nombre_producto, p.categoria, v.region
)
SELECT *
FROM ventas_por_producto_region
ORDER BY region, total_ventas DESC;
```

**2.3** Ahora aplica las tres funciones de ranking sobre esa base y observa las diferencias:

```sql
-- Comparación de ROW_NUMBER, RANK y DENSE_RANK
WITH ventas_por_producto_region AS (
    SELECT
        p.nombre_producto,
        p.categoria,
        v.region,
        SUM(v.monto_venta) AS total_ventas
    FROM VENTAS v
    JOIN PRODUCTOS p ON v.producto_id = p.producto_id
    GROUP BY p.nombre_producto, p.categoria, v.region
),
rankings AS (
    SELECT
        nombre_producto,
        categoria,
        region,
        total_ventas,
        ROW_NUMBER() OVER (PARTITION BY region ORDER BY total_ventas DESC) AS row_num,
        RANK()       OVER (PARTITION BY region ORDER BY total_ventas DESC) AS rank_pos,
        DENSE_RANK() OVER (PARTITION BY region ORDER BY total_ventas DESC) AS dense_rank_pos
    FROM ventas_por_producto_region
)
SELECT *
FROM rankings
ORDER BY region, total_ventas DESC;
```

**2.4** Filtra para ver solo el **Top 5 por región** usando `QUALIFY` (extensión de Snowflake):

```sql
-- Top 5 productos por región usando QUALIFY con DENSE_RANK
-- QUALIFY es exclusivo de Snowflake; no existe en PostgreSQL ni MySQL
WITH ventas_por_producto_region AS (
    SELECT
        p.nombre_producto,
        p.categoria,
        v.region,
        SUM(v.monto_venta)                                                    AS total_ventas
    FROM VENTAS v
    JOIN PRODUCTOS p ON v.producto_id = p.producto_id
    GROUP BY p.nombre_producto, p.categoria, v.region
)
SELECT
    nombre_producto,
    categoria,
    region,
    total_ventas,
    DENSE_RANK() OVER (PARTITION BY region ORDER BY total_ventas DESC)        AS posicion
FROM ventas_por_producto_region
QUALIFY DENSE_RANK() OVER (PARTITION BY region ORDER BY total_ventas DESC) <= 5
ORDER BY region, posicion;
```

**2.5** Para entender la diferencia entre las tres funciones, busca deliberadamente un caso de empate. Ejecuta esta consulta que fuerza un escenario de empate artificial para visualizar el comportamiento:

```sql
-- Demostración de empates: mismo total_ventas para dos productos
-- Observa cómo cada función asigna números distintos
WITH datos_empate AS (
    SELECT 'Producto A' AS producto, 'Norte' AS region, 5000 AS total_ventas
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
    producto,
    region,
    total_ventas,
    ROW_NUMBER() OVER (PARTITION BY region ORDER BY total_ventas DESC) AS row_num,
    RANK()       OVER (PARTITION BY region ORDER BY total_ventas DESC) AS rank_pos,
    DENSE_RANK() OVER (PARTITION BY region ORDER BY total_ventas DESC) AS dense_rank_pos
FROM datos_empate;
```

#### Resultado esperado del paso 2.5

| producto   | total_ventas | row_num | rank_pos | dense_rank_pos |
|------------|-------------|---------|----------|----------------|
| Producto A | 5000        | 1       | 1        | 1              |
| Producto B | 5000        | 2       | 1        | 1              |
| Producto C | 4800        | 3       | 3        | 2              |
| Producto D | 4200        | 4       | 4        | 3              |
| Producto E | 4200        | 5       | 4        | 3              |
| Producto F | 3900        | 6       | 6        | 4              |

> 🔍 **Diferencias clave:**
> - `ROW_NUMBER`: siempre asigna números únicos consecutivos, incluso en empates. El orden entre empatados es arbitrario.
> - `RANK`: en empates asigna el mismo número, pero **salta** posiciones (después de dos posición 1, la siguiente es la 3).
> - `DENSE_RANK`: en empates asigna el mismo número y **no salta** posiciones (después de dos posición 1, la siguiente es la 2).

> ⚠️ **Nota de portabilidad:** La cláusula `QUALIFY` del paso 2.4 es exclusiva de Snowflake. En otros motores como PostgreSQL o MySQL deberías envolver la consulta en una subconsulta o CTE adicional para filtrar por el ranking.

#### Verificación

Confirma que en el resultado del paso 2.4, cada región tiene exactamente 5 filas (o más si hay empates en la posición 5 con `DENSE_RANK`). Si una región tiene 6 filas, significa que dos productos empataron en el puesto 5.

---

### Ejercicio 3 — Análisis temporal con LAG y LEAD

**Objetivo:** Usar `LAG` y `LEAD` para calcular la variación de ventas respecto al mes anterior y siguiente, identificando tendencias de crecimiento o caída.

#### Instrucciones

**3.1** Abre una nueva hoja de trabajo. Nombra la pestaña `Lab04 - Ejercicio 3`.

**3.2** Construye primero la serie temporal de ventas mensuales totales:

```sql
-- Serie temporal de ventas mensuales agregadas
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', fecha_venta)  AS mes,
        SUM(monto_venta)                   AS total_ventas,
        COUNT(DISTINCT venta_id)           AS num_transacciones,
        COUNT(DISTINCT vendedor_id)        AS num_vendedores_activos
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', fecha_venta)
)
SELECT
    mes,
    total_ventas,
    num_transacciones,
    num_vendedores_activos
FROM ventas_mensuales
ORDER BY mes;
```

**3.3** Aplica `LAG` para obtener el valor del mes anterior y calcular la variación:

```sql
-- LAG: comparar ventas con el mes anterior
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', fecha_venta)  AS mes,
        SUM(monto_venta)                   AS total_ventas
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', fecha_venta)
)
SELECT
    mes,
    total_ventas,
    -- LAG(columna, offset, valor_default) OVER(ORDER BY ...)
    LAG(total_ventas, 1, 0)  OVER (ORDER BY mes)  AS ventas_mes_anterior,
    total_ventas - LAG(total_ventas, 1, 0) OVER (ORDER BY mes)
                                                   AS variacion_absoluta,
    ROUND(
        (total_ventas - LAG(total_ventas, 1, 0) OVER (ORDER BY mes))
        / NULLIF(LAG(total_ventas, 1, 0) OVER (ORDER BY mes), 0) * 100,
        2
    )                                              AS variacion_pct
FROM ventas_mensuales
ORDER BY mes;
```

**3.4** Agrega `LEAD` para ver también el mes siguiente y construir una vista completa de contexto temporal:

```sql
-- LAG y LEAD combinados: contexto temporal completo
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', fecha_venta)  AS mes,
        SUM(monto_venta)                   AS total_ventas
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', fecha_venta)
)
SELECT
    mes,
    total_ventas,
    LAG(total_ventas,  1, NULL) OVER (ORDER BY mes) AS ventas_mes_anterior,
    LEAD(total_ventas, 1, NULL) OVER (ORDER BY mes) AS ventas_mes_siguiente,
    -- Variación vs mes anterior
    ROUND(
        (total_ventas - LAG(total_ventas, 1, NULL) OVER (ORDER BY mes))
        / NULLIF(LAG(total_ventas, 1, NULL) OVER (ORDER BY mes), 0) * 100,
        2
    )                                               AS var_pct_vs_anterior,
    -- Clasificación de tendencia
    CASE
        WHEN total_ventas > LAG(total_ventas, 1, NULL) OVER (ORDER BY mes)
             AND total_ventas > LEAD(total_ventas, 1, NULL) OVER (ORDER BY mes)
             THEN '🔺 Pico local'
        WHEN total_ventas < LAG(total_ventas, 1, NULL) OVER (ORDER BY mes)
             AND total_ventas < LEAD(total_ventas, 1, NULL) OVER (ORDER BY mes)
             THEN '🔻 Valle local'
        WHEN total_ventas > LAG(total_ventas, 1, NULL) OVER (ORDER BY mes)
             THEN '📈 Crecimiento'
        WHEN total_ventas < LAG(total_ventas, 1, NULL) OVER (ORDER BY mes)
             THEN '📉 Caída'
        ELSE '➡️ Sin cambio / Primer mes'
    END                                             AS tendencia
FROM ventas_mensuales
ORDER BY mes;
```

**3.5** Aplica el mismo análisis **segmentado por región** usando `PARTITION BY`:

```sql
-- LAG particionado por región: cada región tiene su propia serie temporal
WITH ventas_mensuales_region AS (
    SELECT
        DATE_TRUNC('month', fecha_venta)  AS mes,
        region,
        SUM(monto_venta)                   AS total_ventas
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', fecha_venta), region
)
SELECT
    mes,
    region,
    total_ventas,
    -- PARTITION BY region garantiza que LAG no "cruce" entre regiones
    LAG(total_ventas, 1, NULL) OVER (PARTITION BY region ORDER BY mes) AS ventas_mes_anterior,
    ROUND(
        (total_ventas - LAG(total_ventas, 1, NULL) OVER (PARTITION BY region ORDER BY mes))
        / NULLIF(LAG(total_ventas, 1, NULL) OVER (PARTITION BY region ORDER BY mes), 0) * 100,
        2
    )                                               AS var_pct_vs_anterior
FROM ventas_mensuales_region
ORDER BY region, mes;
```

#### Resultado esperado

- En el paso 3.3, el primer mes tendrá `ventas_mes_anterior = 0` (por el valor default especificado) y `variacion_pct = NULL` o 0.
- En el paso 3.4, el último mes tendrá `ventas_mes_siguiente = NULL` (no hay mes siguiente).
- En el paso 3.5, el primer mes de **cada región** tendrá `ventas_mes_anterior = NULL`, ya que `PARTITION BY region` reinicia la secuencia para cada región.

#### Verificación

Confirma que en el paso 3.5, el primer registro de cada región tiene `ventas_mes_anterior = NULL`. Esto demuestra que `PARTITION BY` funcionó correctamente: LAG no "cruzó" de una región a otra.

> 💡 **Concepto clave:** El tercer argumento de `LAG(columna, offset, default)` define qué valor devolver cuando no existe una fila anterior (por ejemplo, el primer mes). Usar `NULL` es más preciso que `0` para cálculos de porcentajes, ya que `0` puede generar divisiones por cero o variaciones del 100% artificiales.

---

### Ejercicio 4 — Acumulados y promedios móviles con marcos de ventana

**Objetivo:** Calcular totales acumulados y promedios móviles usando `SUM() OVER()` y `AVG() OVER()` con la cláusula `ROWS BETWEEN`, dominando el concepto de marco de ventana (*window frame*).

#### Instrucciones

**4.1** Abre una nueva hoja de trabajo. Nombra la pestaña `Lab04 - Ejercicio 4`.

**4.2** Construye un acumulado de ventas por mes (total acumulado desde el inicio del período):

```sql
-- Acumulado de ventas mensuales (running total)
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', fecha_venta)  AS mes,
        SUM(monto_venta)                   AS total_ventas
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', fecha_venta)
)
SELECT
    mes,
    total_ventas,
    -- Marco: desde el inicio del dataset hasta la fila actual
    SUM(total_ventas) OVER (
        ORDER BY mes
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )                                           AS ventas_acumuladas,
    -- Porcentaje del acumulado sobre el gran total
    ROUND(
        SUM(total_ventas) OVER (
            ORDER BY mes
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        )
        / SUM(total_ventas) OVER () * 100,
        2
    )                                           AS pct_acumulado
FROM ventas_mensuales
ORDER BY mes;
```

**4.3** Implementa un promedio móvil de 3 meses para suavizar las fluctuaciones estacionales:

```sql
-- Promedio móvil de 3 meses (mes actual + 2 anteriores)
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', fecha_venta)  AS mes,
        SUM(monto_venta)                   AS total_ventas
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', fecha_venta)
)
SELECT
    mes,
    total_ventas,
    -- Promedio móvil: ventana de 3 meses (1 anterior + actual + 0 siguientes)
    ROUND(
        AVG(total_ventas) OVER (
            ORDER BY mes
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ),
        2
    )                                           AS promedio_movil_3m,
    -- Promedio móvil: ventana de 5 meses para comparar
    ROUND(
        AVG(total_ventas) OVER (
            ORDER BY mes
            ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
        ),
        2
    )                                           AS promedio_movil_5m
FROM ventas_mensuales
ORDER BY mes;
```

**4.4** Combina acumulados con `PARTITION BY` para calcular el acumulado **por región**:

```sql
-- Acumulado mensual particionado por región
WITH ventas_mensuales_region AS (
    SELECT
        DATE_TRUNC('month', fecha_venta)  AS mes,
        region,
        SUM(monto_venta)                   AS total_ventas
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', fecha_venta), region
)
SELECT
    mes,
    region,
    total_ventas,
    -- Acumulado reinicia para cada región gracias a PARTITION BY
    SUM(total_ventas) OVER (
        PARTITION BY region
        ORDER BY mes
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )                                           AS ventas_acumuladas_region,
    -- Promedio móvil de 3 meses por región
    ROUND(
        AVG(total_ventas) OVER (
            PARTITION BY region
            ORDER BY mes
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ),
        2
    )                                           AS promedio_movil_3m_region
FROM ventas_mensuales_region
ORDER BY region, mes;
```

**4.5** Construye la consulta completa de análisis integrado que combina todas las técnicas del ejercicio:

```sql
-- CONSULTA INTEGRADORA: acumulado + promedio móvil + variación vs mes anterior
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', fecha_venta)  AS mes,
        SUM(monto_venta)                   AS total_ventas,
        COUNT(DISTINCT venta_id)           AS num_transacciones
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', fecha_venta)
)
SELECT
    mes,
    total_ventas,
    num_transacciones,
    -- Acumulado total
    SUM(total_ventas) OVER (
        ORDER BY mes
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )                                           AS ventas_acumuladas,
    -- Promedio móvil 3 meses
    ROUND(
        AVG(total_ventas) OVER (
            ORDER BY mes
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ),
        2
    )                                           AS promedio_movil_3m,
    -- Variación vs mes anterior
    LAG(total_ventas, 1, NULL) OVER (ORDER BY mes)
                                                AS ventas_mes_anterior,
    ROUND(
        (total_ventas - LAG(total_ventas, 1, NULL) OVER (ORDER BY mes))
        / NULLIF(LAG(total_ventas, 1, NULL) OVER (ORDER BY mes), 0) * 100,
        2
    )                                           AS variacion_pct_mom,
    -- Ranking del mes por volumen de ventas
    RANK() OVER (ORDER BY total_ventas DESC)    AS ranking_mes
FROM ventas_mensuales
ORDER BY mes;
```

#### Resultado esperado

- En el paso 4.2, `ventas_acumuladas` debe aumentar monotónicamente (nunca decrecer) mes a mes, y el último mes debe mostrar `pct_acumulado = 100.00`.
- En el paso 4.3, los primeros 2 meses tendrán `promedio_movil_3m` calculado sobre menos de 3 meses (ventana incompleta), lo cual es el comportamiento correcto de `ROWS BETWEEN`.
- En el paso 4.4, el acumulado de cada región reinicia en su primer mes, demostrando que `PARTITION BY` funciona correctamente.

#### Verificación

```sql
-- Verificación: el acumulado del último mes debe igualar el gran total
WITH ventas_mensuales AS (
    SELECT
        DATE_TRUNC('month', fecha_venta) AS mes,
        SUM(monto_venta)                  AS total_ventas
    FROM VENTAS
    GROUP BY DATE_TRUNC('month', fecha_venta)
)
SELECT
    MAX(SUM(total_ventas) OVER (
        ORDER BY mes
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ))                                    AS acumulado_ultimo_mes,
    SUM(total_ventas)                     AS gran_total_directo
FROM ventas_mensuales;
```

Ambas columnas deben mostrar el mismo valor. Si difieren, hay un error en el marco de ventana.

> 💡 **Anatomía del marco de ventana:**
> - `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` → desde la primera fila hasta la actual (acumulado clásico).
> - `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` → las 2 filas anteriores más la actual (promedio móvil de 3).
> - `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` → todas las filas (equivalente a `OVER()` sin marco).

---

### Ejercicio 5 — Desafío integrador: Reporte ejecutivo de desempeño

**Objetivo:** Combinar todas las técnicas del laboratorio en una consulta analítica compleja que genere un reporte ejecutivo de desempeño de vendedores con ranking, tendencia y clasificación por cuartil.

#### Instrucciones

**5.1** Abre una nueva hoja de trabajo. Nombra la pestaña `Lab04 - Ejercicio 5 (Desafío)`.

**5.2** Construye el reporte ejecutivo completo en una sola consulta con CTEs:

```sql
-- REPORTE EJECUTIVO DE DESEMPEÑO DE VENDEDORES
-- Combina: RANK, DENSE_RANK, LAG, SUM acumulado y clasificación por cuartil

WITH
-- CTE 1: Ventas totales por vendedor y mes
ventas_vendedor_mes AS (
    SELECT
        v.vendedor_id,
        DATE_TRUNC('month', v.fecha_venta)  AS mes,
        v.region,
        SUM(v.monto_venta)                   AS total_ventas,
        COUNT(DISTINCT v.venta_id)           AS num_transacciones
    FROM VENTAS v
    GROUP BY v.vendedor_id, DATE_TRUNC('month', v.fecha_venta), v.region
),

-- CTE 2: Ventas totales anuales por vendedor (para ranking global)
ventas_vendedor_anual AS (
    SELECT
        vendedor_id,
        region,
        SUM(total_ventas)                    AS ventas_anuales,
        SUM(num_transacciones)               AS transacciones_anuales,
        COUNT(DISTINCT mes)                  AS meses_activo
    FROM ventas_vendedor_mes
    GROUP BY vendedor_id, region
),

-- CTE 3: Métricas de análisis temporal por vendedor
metricas_temporales AS (
    SELECT
        vendedor_id,
        mes,
        region,
        total_ventas,
        num_transacciones,
        -- Ventas del mes anterior para este vendedor en esta región
        LAG(total_ventas, 1, NULL) OVER (
            PARTITION BY vendedor_id, region
            ORDER BY mes
        )                                    AS ventas_mes_anterior,
        -- Acumulado del vendedor en la región
        SUM(total_ventas) OVER (
            PARTITION BY vendedor_id, region
            ORDER BY mes
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        )                                    AS ventas_acumuladas,
        -- Promedio móvil 3 meses del vendedor
        ROUND(
            AVG(total_ventas) OVER (
                PARTITION BY vendedor_id, region
                ORDER BY mes
                ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
            ),
            2
        )                                    AS promedio_movil_3m
    FROM ventas_vendedor_mes
),

-- CTE 4: Ranking y clasificación por cuartil (sobre ventas anuales)
ranking_vendedores AS (
    SELECT
        vendedor_id,
        region,
        ventas_anuales,
        transacciones_anuales,
        meses_activo,
        -- Ranking global (todos los vendedores)
        RANK()       OVER (ORDER BY ventas_anuales DESC)              AS ranking_global,
        DENSE_RANK() OVER (ORDER BY ventas_anuales DESC)              AS dense_ranking_global,
        -- Ranking dentro de la región
        RANK()       OVER (PARTITION BY region ORDER BY ventas_anuales DESC) AS ranking_region,
        -- Clasificación por cuartil usando NTILE
        NTILE(4)     OVER (ORDER BY ventas_anuales DESC)              AS cuartil,
        -- Etiqueta del cuartil
        CASE NTILE(4) OVER (ORDER BY ventas_anuales DESC)
            WHEN 1 THEN 'Top 25% — Élite'
            WHEN 2 THEN 'Top 50% — Alto'
            WHEN 3 THEN 'Top 75% — Medio'
            WHEN 4 THEN 'Bottom 25% — En desarrollo'
        END                                                           AS categoria_desempeno
    FROM ventas_vendedor_anual
)

-- CONSULTA FINAL: Unir todas las métricas
SELECT
    r.vendedor_id,
    r.region,
    r.ventas_anuales,
    r.transacciones_anuales,
    r.meses_activo,
    r.ranking_global,
    r.ranking_region,
    r.cuartil,
    r.categoria_desempeno,
    -- Último mes con datos (para contexto de tendencia reciente)
    MAX(mt.mes)                                                       AS ultimo_mes_con_datos,
    -- Ventas del último mes registrado
    MAX(mt.total_ventas)                                              AS ventas_ultimo_mes,
    -- Promedio móvil del último mes
    MAX(mt.promedio_movil_3m)                                         AS promedio_movil_ultimo_mes
FROM ranking_vendedores r
JOIN metricas_temporales mt ON r.vendedor_id = mt.vendedor_id
                            AND r.region     = mt.region
GROUP BY
    r.vendedor_id, r.region, r.ventas_anuales, r.transacciones_anuales,
    r.meses_activo, r.ranking_global, r.dense_ranking_global,
    r.ranking_region, r.cuartil, r.categoria_desempeno
ORDER BY r.ranking_global;
```

**5.3** Filtra el resultado para ver solo los vendedores del **Top 25% (Élite)** usando `QUALIFY`:

```sql
-- Versión con QUALIFY para filtrar directamente por cuartil
-- QUALIFY es exclusivo de Snowflake
WITH
ventas_vendedor_anual AS (
    SELECT
        vendedor_id,
        region,
        SUM(monto_venta)         AS ventas_anuales,
        COUNT(DISTINCT venta_id) AS transacciones_anuales
    FROM VENTAS
    GROUP BY vendedor_id, region
)
SELECT
    vendedor_id,
    region,
    ventas_anuales,
    transacciones_anuales,
    RANK()   OVER (ORDER BY ventas_anuales DESC)                      AS ranking_global,
    RANK()   OVER (PARTITION BY region ORDER BY ventas_anuales DESC)  AS ranking_region,
    NTILE(4) OVER (ORDER BY ventas_anuales DESC)                      AS cuartil
FROM ventas_vendedor_anual
QUALIFY NTILE(4) OVER (ORDER BY ventas_anuales DESC) = 1  -- Solo Top 25%
ORDER BY ranking_global;
```

#### Resultado esperado

- El reporte del paso 5.2 debe mostrar una fila por vendedor con todas sus métricas consolidadas.
- El paso 5.3 debe devolver aproximadamente el 25% del total de vendedores (los de mayor desempeño).
- Los vendedores con el mismo `ventas_anuales` deben tener el mismo `ranking_global` con `RANK`.

#### Verificación

```sql
-- Verificar que NTILE distribuyó los cuartiles correctamente
WITH ventas_vendedor_anual AS (
    SELECT
        vendedor_id,
        SUM(monto_venta) AS ventas_anuales
    FROM VENTAS
    GROUP BY vendedor_id
)
SELECT
    NTILE(4) OVER (ORDER BY ventas_anuales DESC) AS cuartil,
    COUNT(*)                                      AS num_vendedores
FROM ventas_vendedor_anual
GROUP BY cuartil
ORDER BY cuartil;
```

Cada cuartil debe tener aproximadamente el mismo número de vendedores (diferencia máxima de 1 fila por efectos del redondeo de `NTILE`).

---

## 7. Validación y Pruebas Finales

Ejecuta las siguientes consultas de validación para confirmar que todos los ejercicios fueron completados correctamente:

```sql
-- VALIDACIÓN GLOBAL DEL LABORATORIO
-- Ejecuta cada bloque por separado y verifica los criterios indicados

-- ✅ Validación 1: Rankings producen resultados distintos con empates
-- Criterio: row_num siempre único; rank y dense_rank pueden repetirse
SELECT
    'Rankings correctos' AS validacion,
    COUNT(DISTINCT row_num) = COUNT(*)                    AS row_num_siempre_unico,
    COUNT(*) >= COUNT(DISTINCT rank_pos)                  AS rank_puede_repetir,
    COUNT(*) >= COUNT(DISTINCT dense_rank_pos)            AS dense_rank_puede_repetir
FROM (
    WITH base AS (
        SELECT p.nombre_producto, v.region, SUM(v.monto_venta) AS total
        FROM VENTAS v JOIN PRODUCTOS p ON v.producto_id = p.producto_id
        GROUP BY p.nombre_producto, v.region
    )
    SELECT
        nombre_producto, region, total,
        ROW_NUMBER() OVER (PARTITION BY region ORDER BY total DESC) AS row_num,
        RANK()       OVER (PARTITION BY region ORDER BY total DESC) AS rank_pos,
        DENSE_RANK() OVER (PARTITION BY region ORDER BY total DESC) AS dense_rank_pos
    FROM base
);
```

```sql
-- ✅ Validación 2: Acumulado del último mes = Gran total
WITH ventas_mensuales AS (
    SELECT DATE_TRUNC('month', fecha_venta) AS mes, SUM(monto_venta) AS total
    FROM VENTAS GROUP BY 1
),
acumulados AS (
    SELECT mes, total,
           SUM(total) OVER (ORDER BY mes ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS acum
    FROM ventas_mensuales
)
SELECT
    'Acumulado correcto' AS validacion,
    MAX(acum)            AS acumulado_ultimo_mes,
    SUM(total)           AS gran_total,
    MAX(acum) = SUM(total) AS es_igual
FROM acumulados;
```

```sql
-- ✅ Validación 3: LAG con PARTITION BY no cruza regiones
-- El primer mes de cada región debe tener ventas_anterior = NULL
WITH ventas_region_mes AS (
    SELECT region, DATE_TRUNC('month', fecha_venta) AS mes, SUM(monto_venta) AS total
    FROM VENTAS GROUP BY 1, 2
),
con_lag AS (
    SELECT region, mes, total,
           LAG(total, 1, NULL) OVER (PARTITION BY region ORDER BY mes) AS anterior,
           ROW_NUMBER() OVER (PARTITION BY region ORDER BY mes) AS rn
    FROM ventas_region_mes
)
SELECT
    'LAG particionado correcto' AS validacion,
    COUNT(*)                    AS regiones_verificadas,
    SUM(CASE WHEN rn = 1 AND anterior IS NULL THEN 1 ELSE 0 END) AS primeros_meses_sin_anterior,
    COUNT(DISTINCT region)      AS total_regiones,
    SUM(CASE WHEN rn = 1 AND anterior IS NULL THEN 1 ELSE 0 END) = COUNT(DISTINCT region) AS ok
FROM con_lag;
```

**Criterio de éxito:** Las tres validaciones deben devolver `TRUE` o `1` en su columna de verificación (`es_igual`, `ok`, etc.). Si alguna falla, revisa el ejercicio correspondiente con ayuda del instructor.

---

## 8. Resolución de Problemas

### Problema 1: Error "Window function [X] requires ORDER BY in window specification"

**Síntoma:** Al ejecutar una consulta con `LAG`, `LEAD`, `RANK`, `ROW_NUMBER` o `DENSE_RANK`, Snowflake devuelve un error similar a:
```
SQL compilation error: Window function [ROW_NUMBER] requires ORDER BY in window specification
```

**Causa:** Las funciones de numeración y desplazamiento (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`, `LEAD`) **requieren obligatoriamente** una cláusula `ORDER BY` dentro del `OVER()`. Sin ella, Snowflake no puede determinar el orden en que asignar los números o buscar filas anteriores/siguientes.

**Solución:** Agrega siempre `ORDER BY` dentro del `OVER()` para estas funciones:

```sql
-- ❌ Incorrecto — falta ORDER BY
SELECT vendedor_id, ROW_NUMBER() OVER (PARTITION BY region) AS num
FROM VENTAS;

-- ✅ Correcto — ORDER BY incluido
SELECT vendedor_id, ROW_NUMBER() OVER (PARTITION BY region ORDER BY monto_venta DESC) AS num
FROM VENTAS;
```

> **Nota:** `SUM() OVER()` y `AVG() OVER()` **sí pueden** usarse sin `ORDER BY` cuando quieres el total de toda la partición (no un acumulado). En ese caso, el marco de ventana por defecto incluye todas las filas de la partición.

---

### Problema 2: QUALIFY no filtra los resultados o genera error de compilación

**Síntoma:** La consulta con `QUALIFY` devuelve todas las filas (sin filtrar) o genera el error:
```
SQL compilation error: QUALIFY clause requires at least one window function
```

**Causa A:** `QUALIFY` solo funciona cuando la expresión de filtro hace referencia a una window function. Si se usa con una columna calculada sin `OVER()`, Snowflake no puede procesarlo.

**Causa B:** En algunos casos, el alias de la columna de la window function se usa en `QUALIFY` pero Snowflake no puede resolverlo si la función no está también en el `SELECT`.

**Solución:** Asegúrate de que la expresión en `QUALIFY` contiene explícitamente la window function (no solo el alias):

```sql
-- ❌ Incorrecto — QUALIFY referencia un alias que no existe en el contexto
SELECT nombre_producto, total_ventas, posicion
FROM (SELECT ..., DENSE_RANK() OVER (...) AS posicion FROM ...)
QUALIFY posicion <= 5;  -- Esto requeriría una subconsulta, no QUALIFY directo

-- ✅ Correcto — QUALIFY con la window function explícita en la misma consulta
SELECT
    nombre_producto,
    total_ventas,
    DENSE_RANK() OVER (PARTITION BY region ORDER BY total_ventas DESC) AS posicion
FROM ventas_por_producto_region
QUALIFY DENSE_RANK() OVER (PARTITION BY region ORDER BY total_ventas DESC) <= 5;
```

> ⚠️ **Recordatorio:** `QUALIFY` es exclusivo de Snowflake. Si necesitas portar esta consulta a PostgreSQL o MySQL, debes reemplazarla por una subconsulta o CTE que filtre por el alias del ranking.

---

## 9. Limpieza del Entorno

Al finalizar el laboratorio, ejecuta los siguientes comandos para liberar recursos y evitar consumo innecesario de créditos de Snowflake:

```sql
-- Paso 1: Suspender el warehouse manualmente para detener el consumo de créditos
-- Reemplaza LAB_WH con el nombre de tu warehouse si es diferente
ALTER WAREHOUSE LAB_WH SUSPEND;
```

```sql
-- Paso 2: Verificar que el warehouse quedó suspendido
SHOW WAREHOUSES LIKE 'LAB_WH';
-- Busca la columna "state" — debe mostrar "SUSPENDED"
```

> ⚠️ **Importante:** Las cuentas Trial tienen 400 USD de créditos. Un warehouse `X-SMALL` activo consume créditos incluso sin ejecutar consultas. Siempre suspéndelo al terminar la sesión.

> 💡 **Opcional:** Si quieres que el warehouse se suspenda automáticamente después de inactividad, configura:
> ```sql
> ALTER WAREHOUSE LAB_WH SET AUTO_SUSPEND = 60;  -- Suspende tras 60 segundos de inactividad
> ```

No es necesario eliminar ninguna tabla ni objeto creado en este laboratorio, ya que todas las consultas fueron de tipo `SELECT` (solo lectura). Los objetos del schema `VENTAS` son compartidos con los demás laboratorios del curso.

---

## 10. Resumen

### Lo que aprendiste en este laboratorio

En este laboratorio aplicaste las funciones analíticas más importantes de Snowflake sobre datos reales de ventas y pedidos:

| Técnica                      | Función(es) usadas                                    | Caso de uso aplicado                              |
|------------------------------|-------------------------------------------------------|---------------------------------------------------|
| **Rankings únicos**          | `ROW_NUMBER() OVER(PARTITION BY ... ORDER BY ...)`    | Secuencias únicas por región, sin empates         |
| **Rankings con empates**     | `RANK()`, `DENSE_RANK()`                              | Top 5 productos por región, con manejo de empates |
| **Análisis temporal**        | `LAG(col, offset, default)`, `LEAD(col, offset, default)` | Variación MoM de ventas, detección de picos/valles |
| **Acumulados**               | `SUM() OVER(ORDER BY ... ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` | Running total de ventas mensual |
| **Promedios móviles**        | `AVG() OVER(ORDER BY ... ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)` | Suavizado de series temporales (3 y 5 meses) |
| **Clasificación por cuartil**| `NTILE(4) OVER(ORDER BY ...)`                         | Segmentación de vendedores por desempeño          |
| **Filtrado de rankings**     | `QUALIFY`                                             | Selección directa de Top N sin subconsulta        |

### Diferencias clave para recordar

- **`RANK` vs `DENSE_RANK`:** Ambas asignan el mismo número a empates, pero `RANK` salta posiciones y `DENSE_RANK` no.
- **`PARTITION BY` en `LAG/LEAD`:** Sin `PARTITION BY`, LAG puede "cruzar" entre grupos. Con `PARTITION BY`, cada grupo tiene su propia secuencia.
- **Marco de ventana (`ROWS BETWEEN`):** Sin él, `SUM() OVER(ORDER BY ...)` usa el marco por defecto (`RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`), que puede producir resultados inesperados con valores duplicados. Especificar `ROWS BETWEEN` explícitamente es una buena práctica.
- **`QUALIFY` es exclusivo de Snowflake:** No es portable a PostgreSQL, MySQL ni otros motores SQL estándar.

### Conexión con el siguiente laboratorio

En el **Laboratorio 5** aplicarás estas mismas técnicas a series temporales más complejas, incluyendo análisis de estacionalidad y detección de anomalías en tendencias de ventas, usando `ROWS BETWEEN` con ventanas asimétricas y combinando window functions con `CASE WHEN` para generar alertas automáticas.

### Recursos adicionales

- [Documentación oficial de Snowflake: Window Functions](https://docs.snowflake.com/en/sql-reference/functions-analytic)
- [Documentación oficial de Snowflake: QUALIFY](https://docs.snowflake.com/en/sql-reference/constructs/qualify)
- [Documentación oficial de Snowflake: LAG y LEAD](https://docs.snowflake.com/en/sql-reference/functions/lag)
- [Documentación oficial de Snowflake: Marcos de ventana (ROWS/RANGE)](https://docs.snowflake.com/en/sql-reference/functions/over#window-frame)
- [Mode Analytics: SQL Window Functions Tutorial](https://mode.com/sql-tutorial/sql-window-functions/)

---
