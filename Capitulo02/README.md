# Clasificación de clientes y reglas de negocio

## 1. Metadatos

| Atributo | Valor |
|---|---|
| **Duración estimada** | 60 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (Apply) |
| **Módulo** | 2 — Clasificación y segmentación con CASE WHEN |
| **Laboratorio previo requerido** | Lab 01 (CTEs y subqueries) |
| **Schema de datos** | `LAB_SQL_INTERMEDIO.VENTAS` |

---

## 2. Descripción General

En este laboratorio aplicarás la expresión `CASE WHEN` para construir un sistema de segmentación de clientes basado en reglas de negocio reales. Partirás de las tablas `CLIENTES` y `PEDIDOS` del schema de práctica y crearás clasificaciones progresivas: primero una segmentación simple por monto de compra, luego una clasificación multinivel que combina monto, frecuencia y antigüedad del cliente, y finalmente un resumen ejecutivo por segmento usando `GROUP BY` con métricas agregadas. El resultado final es un dataset enriquecido listo para reportes o para alimentar análisis posteriores.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Implementar `CASE WHEN` en forma buscada y simple para clasificar clientes según reglas de negocio definidas (segmentos GOLD, SILVER, BRONZE).
- [ ] Construir clasificaciones multinivel que combinen condiciones compuestas con `AND`, `OR` y `BETWEEN` sobre monto de compra, frecuencia y antigüedad.
- [ ] Integrar `CASE WHEN` dentro de funciones de agregación (`SUM`, `COUNT`, `AVG`) para generar métricas por segmento en una sola consulta.
- [ ] Preparar un dataset enriquecido con columnas derivadas de clasificación usando CTEs, listo para análisis posterior.
- [ ] Utilizar `IFF()` como alternativa simplificada de Snowflake para clasificaciones de dos ramas.

---

## 4. Prerrequisitos

### Conocimientos previos

| Área | Nivel requerido |
|---|---|
| CTEs (`WITH ... AS`) y subqueries | Completado (Lab 01) o equivalente |
| Operadores de comparación y lógicos (`AND`, `OR`, `NOT`, `BETWEEN`, `IN`) | Intermedio |
| Funciones de agregación (`SUM`, `COUNT`, `AVG`, `MAX`, `MIN`) con `GROUP BY` | Intermedio |
| Sintaxis básica de `SELECT`, `FROM`, `WHERE`, `ORDER BY` | Sólido |

### Acceso y configuración

| Recurso | Estado requerido |
|---|---|
| Cuenta Snowflake activa (trial o corporativa) | ✅ Activa y accesible |
| Script `00_setup_laboratorios.sql` ejecutado por el instructor | ✅ Ejecutado previamente |
| Base de datos `LAB_SQL_INTERMEDIO` visible en Snowsight | ✅ Confirmado |
| Schema `VENTAS` con tablas `CLIENTES`, `PEDIDOS`, `VENTAS`, `PRODUCTOS` | ✅ Poblado |
| Warehouse `LAB_WH` disponible (tamaño X-SMALL) | ✅ Activo o suspendido (se activa al ejecutar) |

---

## 5. Entorno del Laboratorio

### Hardware recomendado

| Componente | Mínimo | Recomendado |
|---|---|---|
| Procesador | Intel Core i5 / AMD Ryzen 5 (64-bit) | Intel Core i7 / AMD Ryzen 7 |
| RAM | 8 GB | 16 GB |
| Almacenamiento libre | 2 GB | 5 GB |
| Resolución de pantalla | 1280×768 | 1920×1080 |
| Conexión a Internet | 10 Mbps | 25 Mbps o superior |

### Software requerido

| Software | Versión mínima | Uso |
|---|---|---|
| Navegador web (Chrome / Firefox / Edge / Safari) | 110+ / 110+ / 110+ / 16+ | Acceso a Snowsight |
| Snowflake (Snowsight) | Enterprise o Trial (versión web actual) | Ejecución de consultas SQL |
| Visual Studio Code (opcional) | 1.80+ | Edición previa de scripts |
| SnowSQL CLI (opcional) | 1.2.x+ | Ejecución por línea de comandos |

### Configuración inicial del entorno

Antes de comenzar los ejercicios, ejecuta el siguiente bloque de configuración en Snowsight para establecer el contexto correcto. Abre una nueva hoja de trabajo (*Worksheet*) y pega estas instrucciones:

```sql
-- ============================================================
-- CONFIGURACIÓN INICIAL - Lab 02-00-01
-- Ejecutar este bloque COMPLETO antes de comenzar
-- ============================================================

-- 1. Seleccionar el rol de trabajo
USE ROLE LAB_ROLE;  -- Ajustar al rol asignado por el instructor

-- 2. Activar el warehouse de laboratorio (tamaño X-SMALL)
USE WAREHOUSE LAB_WH;

-- 3. Seleccionar la base de datos y schema de práctica
USE DATABASE LAB_SQL_INTERMEDIO;
USE SCHEMA VENTAS;

-- 4. Verificar que las tablas necesarias existen y tienen datos
SELECT 'CLIENTES'  AS tabla, COUNT(*) AS total_filas FROM CLIENTES
UNION ALL
SELECT 'PEDIDOS'   AS tabla, COUNT(*) AS total_filas FROM PEDIDOS
UNION ALL
SELECT 'VENTAS'    AS tabla, COUNT(*) AS total_filas FROM VENTAS
UNION ALL
SELECT 'PRODUCTOS' AS tabla, COUNT(*) AS total_filas FROM PRODUCTOS
ORDER BY tabla;
```

**Resultado esperado de la verificación:**

| TABLA | TOTAL_FILAS |
|---|---|
| CLIENTES | > 0 |
| PEDIDOS | > 0 |
| PRODUCTOS | > 0 |
| VENTAS | > 0 |

> ⚠️ **Si alguna tabla muestra 0 filas o aparece un error de objeto no encontrado**, detente y notifica al instructor. El script `00_setup_laboratorios.sql` debe ejecutarse nuevamente antes de continuar.

---

## 6. Desarrollo del Laboratorio

### Contexto del Negocio

La empresa **TechCommerce S.A.** necesita implementar un programa de lealtad para sus clientes. El equipo de marketing ha definido las siguientes reglas de segmentación:

| Segmento | Criterio principal |
|---|---|
| **GOLD** | Clientes con alto valor de compra y alta frecuencia |
| **SILVER** | Clientes con valor o frecuencia moderados |
| **BRONZE** | Clientes con compras bajas o poco frecuentes |
| **NEW** | Clientes sin historial de compras suficiente |

Tu trabajo como analista SQL es implementar estas reglas en Snowflake y generar los reportes que el equipo de marketing necesita para lanzar el programa.

---

### Paso 1: Exploración de las tablas base

**Objetivo:** Familiarizarte con la estructura y distribución de datos en `CLIENTES` y `PEDIDOS` antes de construir las clasificaciones.

#### Instrucciones

**1.1** Examina la estructura de la tabla `CLIENTES`:

```sql
-- Estructura de la tabla CLIENTES
DESC TABLE CLIENTES;
```

**1.2** Revisa una muestra de los datos de clientes:

```sql
-- Muestra de datos de clientes
SELECT
    ID_CLIENTE,
    NOMBRE,
    EMAIL,
    FECHA_REGISTRO,
    CIUDAD,
    PAIS
FROM CLIENTES
LIMIT 10;
```

**1.3** Examina la estructura y una muestra de `PEDIDOS`:

```sql
-- Muestra de datos de pedidos con métricas básicas
SELECT
    ID_PEDIDO,
    ID_CLIENTE,
    FECHA_PEDIDO,
    MONTO_TOTAL,
    ESTADO_PEDIDO
FROM PEDIDOS
LIMIT 10;
```

**1.4** Calcula estadísticas de distribución del monto de pedidos para entender los rangos de datos antes de definir los umbrales de clasificación:

```sql
-- Estadísticas descriptivas de montos de pedidos
SELECT
    COUNT(*)                        AS total_pedidos,
    ROUND(MIN(MONTO_TOTAL), 2)      AS monto_minimo,
    ROUND(MAX(MONTO_TOTAL), 2)      AS monto_maximo,
    ROUND(AVG(MONTO_TOTAL), 2)      AS monto_promedio,
    ROUND(MEDIAN(MONTO_TOTAL), 2)   AS monto_mediana,
    COUNT(DISTINCT ID_CLIENTE)      AS clientes_con_pedidos
FROM PEDIDOS
WHERE ESTADO_PEDIDO != 'CANCELADO';
```

**1.5** Calcula la distribución de frecuencia de compras por cliente:

```sql
-- Distribución de frecuencia: ¿cuántos pedidos tiene cada cliente?
SELECT
    pedidos_por_cliente,
    COUNT(*) AS cantidad_clientes
FROM (
    SELECT
        ID_CLIENTE,
        COUNT(*) AS pedidos_por_cliente
    FROM PEDIDOS
    WHERE ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY ID_CLIENTE
) sub
GROUP BY pedidos_por_cliente
ORDER BY pedidos_por_cliente;
```

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

### Paso 2: Clasificación simple con CASE WHEN — Segmentación por monto

**Objetivo:** Implementar una primera clasificación de pedidos usando `CASE WHEN` en forma buscada, asignando una categoría de valor a cada transacción.

#### Instrucciones

**2.1** Crea una clasificación simple de pedidos por monto usando `CASE WHEN` en forma buscada. Ajusta los umbrales según los valores que observaste en el Paso 1 si el instructor lo indica; de lo contrario, usa los valores del enunciado:

```sql
-- Clasificación de pedidos por monto (CASE WHEN forma buscada)
-- Umbrales: Bajo < 200 | Medio 200-999 | Alto 1000-2999 | Premium >= 3000
SELECT
    ID_PEDIDO,
    ID_CLIENTE,
    FECHA_PEDIDO,
    MONTO_TOTAL,
    CASE
        WHEN MONTO_TOTAL < 200              THEN 'Bajo'
        WHEN MONTO_TOTAL < 1000             THEN 'Medio'
        WHEN MONTO_TOTAL < 3000             THEN 'Alto'
        ELSE                                     'Premium'
    END AS categoria_monto
FROM PEDIDOS
WHERE ESTADO_PEDIDO != 'CANCELADO'
ORDER BY MONTO_TOTAL DESC;
```

**2.2** Verifica la distribución de categorías para confirmar que los umbrales son razonables:

```sql
-- Distribución de pedidos por categoría de monto
SELECT
    CASE
        WHEN MONTO_TOTAL < 200              THEN 'Bajo'
        WHEN MONTO_TOTAL < 1000             THEN 'Medio'
        WHEN MONTO_TOTAL < 3000             THEN 'Alto'
        ELSE                                     'Premium'
    END                                     AS categoria_monto,
    COUNT(*)                                AS cantidad_pedidos,
    ROUND(SUM(MONTO_TOTAL), 2)              AS monto_total_segmento,
    ROUND(AVG(MONTO_TOTAL), 2)              AS monto_promedio
FROM PEDIDOS
WHERE ESTADO_PEDIDO != 'CANCELADO'
GROUP BY
    CASE
        WHEN MONTO_TOTAL < 200              THEN 'Bajo'
        WHEN MONTO_TOTAL < 1000             THEN 'Medio'
        WHEN MONTO_TOTAL < 3000             THEN 'Alto'
        ELSE                                     'Premium'
    END
ORDER BY monto_promedio DESC;
```

**2.3** Ahora practica la **forma simple** de `CASE WHEN` para traducir el código de estado de pedido a una descripción legible:

```sql
-- CASE WHEN forma simple: traducción de códigos de estado
SELECT
    ID_PEDIDO,
    ESTADO_PEDIDO,
    CASE ESTADO_PEDIDO
        WHEN 'COMPLETADO'   THEN 'Entregado al cliente'
        WHEN 'EN_PROCESO'   THEN 'En preparación'
        WHEN 'ENVIADO'      THEN 'En camino'
        WHEN 'CANCELADO'    THEN 'Cancelado por cliente o sistema'
        ELSE                     'Estado desconocido: ' || ESTADO_PEDIDO
    END AS descripcion_estado,
    MONTO_TOTAL
FROM PEDIDOS
LIMIT 20;
```

> 📌 **Observa el patrón `ELSE 'Estado desconocido: ' || ESTADO_PEDIDO`**: concatena el valor original con un prefijo descriptivo. Esto es útil para detectar valores inesperados sin perder información.

**2.4** Practica `IFF()`, la alternativa simplificada de Snowflake para clasificaciones de dos ramas:

```sql
-- IFF() como alternativa a CASE WHEN de dos ramas (exclusivo de Snowflake)
-- Sintaxis: IFF(condición, valor_si_verdadero, valor_si_falso)
SELECT
    ID_PEDIDO,
    ID_CLIENTE,
    MONTO_TOTAL,
    IFF(MONTO_TOTAL >= 1000, 'Alto valor', 'Valor estándar') AS clasificacion_iff,
    -- Equivalente con CASE WHEN para comparación:
    CASE
        WHEN MONTO_TOTAL >= 1000 THEN 'Alto valor'
        ELSE                          'Valor estándar'
    END AS clasificacion_case
FROM PEDIDOS
WHERE ESTADO_PEDIDO != 'CANCELADO'
LIMIT 15;
```

> ⚠️ **Nota sobre portabilidad:** `IFF()` es una función exclusiva de Snowflake. No existe en PostgreSQL, MySQL ni SQL Server. Úsala cuando la simplicidad del código sea prioritaria, pero prefiere `CASE WHEN` cuando necesites portabilidad entre motores SQL.

#### Resultado esperado

- La consulta 2.1 debe mostrar cada pedido con su columna `categoria_monto` asignada correctamente.
- La consulta 2.2 debe mostrar exactamente 4 filas (una por categoría), con sus conteos y montos.
- Las columnas `clasificacion_iff` y `clasificacion_case` en la consulta 2.4 deben mostrar valores idénticos.

#### Verificación

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

---

### Paso 3: Construcción de métricas por cliente — Base para la segmentación

**Objetivo:** Calcular las métricas agregadas por cliente (monto total, frecuencia de compra, antigüedad) que servirán como insumo para la clasificación multinivel del Paso 4.

#### Instrucciones

**3.1** Crea una CTE que calcule las métricas base de cada cliente. Este será el fundamento de toda la segmentación posterior:

```sql
-- Métricas base por cliente: monto total, frecuencia y antigüedad
WITH metricas_cliente AS (
    SELECT
        c.ID_CLIENTE,
        c.NOMBRE,
        c.FECHA_REGISTRO,
        -- Antigüedad en días desde el registro
        DATEDIFF('day', c.FECHA_REGISTRO, CURRENT_DATE())    AS dias_como_cliente,
        -- Cantidad de pedidos completados
        COUNT(p.ID_PEDIDO)                                    AS total_pedidos,
        -- Monto total gastado (excluyendo cancelados)
        ROUND(SUM(p.MONTO_TOTAL), 2)                         AS monto_total_gastado,
        -- Monto promedio por pedido
        ROUND(AVG(p.MONTO_TOTAL), 2)                         AS monto_promedio_pedido,
        -- Fecha del último pedido
        MAX(p.FECHA_PEDIDO)                                   AS fecha_ultimo_pedido
    FROM CLIENTES c
    LEFT JOIN PEDIDOS p
           ON c.ID_CLIENTE = p.ID_CLIENTE
          AND p.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY
        c.ID_CLIENTE,
        c.NOMBRE,
        c.FECHA_REGISTRO
)
SELECT *
FROM metricas_cliente
ORDER BY monto_total_gastado DESC NULLS LAST
LIMIT 20;
```

> 📌 **Nota sobre `LEFT JOIN`:** Usamos `LEFT JOIN` para incluir también a los clientes que aún no tienen pedidos (o cuyo único pedido fue cancelado). Aparecerán con `total_pedidos = 0` y `monto_total_gastado = NULL`. El `CASE WHEN` del siguiente paso manejará estos casos.

**3.2** Verifica cuántos clientes tienen métricas nulas (sin pedidos válidos):

```sql
-- Clientes sin pedidos válidos
WITH metricas_cliente AS (
    SELECT
        c.ID_CLIENTE,
        COUNT(p.ID_PEDIDO)             AS total_pedidos,
        SUM(p.MONTO_TOTAL)             AS monto_total_gastado
    FROM CLIENTES c
    LEFT JOIN PEDIDOS p
           ON c.ID_CLIENTE = p.ID_CLIENTE
          AND p.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY c.ID_CLIENTE
)
SELECT
    COUNT(*)                                          AS total_clientes,
    COUNT(CASE WHEN total_pedidos = 0
               THEN 1 END)                            AS clientes_sin_pedidos,
    COUNT(CASE WHEN total_pedidos > 0
               THEN 1 END)                            AS clientes_con_pedidos
FROM metricas_cliente;
```

#### Resultado esperado

- La consulta 3.1 muestra una fila por cliente con sus métricas calculadas.
- Los clientes sin pedidos aparecen con `total_pedidos = 0` y `monto_total_gastado = NULL`.
- La consulta 3.2 muestra el desglose entre clientes con y sin historial de compras.

#### Verificación

Confirma que el total de clientes en la CTE coincide con el total de la tabla `CLIENTES`:

```sql
-- La CTE debe retornar exactamente el mismo número de clientes que la tabla base
SELECT COUNT(*) AS total_en_tabla FROM CLIENTES;
-- Compara este número con total_clientes de la consulta 3.2
```

---

### Paso 4: Clasificación multinivel — Sistema de segmentación GOLD / SILVER / BRONZE

**Objetivo:** Implementar la clasificación principal del laboratorio combinando múltiples condiciones (`AND`, `BETWEEN`, comparaciones de frecuencia y antigüedad) en un `CASE WHEN` de múltiples niveles.

#### Instrucciones

**4.1** Implementa la clasificación completa de tres niveles. Las reglas de negocio son:

| Segmento | Regla |
|---|---|
| **GOLD** | Monto total ≥ 5,000 **Y** total de pedidos ≥ 5 |
| **SILVER** | Monto total ≥ 1,500 **Y** total de pedidos ≥ 2 (no clasificados como GOLD) |
| **BRONZE** | Tiene al menos 1 pedido (no clasificado en niveles superiores) |
| **NEW** | Sin pedidos válidos o cliente registrado hace menos de 30 días |

```sql
-- Clasificación multinivel: GOLD / SILVER / BRONZE / NEW
WITH metricas_cliente AS (
    SELECT
        c.ID_CLIENTE,
        c.NOMBRE,
        c.FECHA_REGISTRO,
        DATEDIFF('day', c.FECHA_REGISTRO, CURRENT_DATE())    AS dias_como_cliente,
        COUNT(p.ID_PEDIDO)                                    AS total_pedidos,
        COALESCE(SUM(p.MONTO_TOTAL), 0)                      AS monto_total_gastado,
        COALESCE(AVG(p.MONTO_TOTAL), 0)                      AS monto_promedio_pedido,
        MAX(p.FECHA_PEDIDO)                                   AS fecha_ultimo_pedido
    FROM CLIENTES c
    LEFT JOIN PEDIDOS p
           ON c.ID_CLIENTE = p.ID_CLIENTE
          AND p.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY
        c.ID_CLIENTE,
        c.NOMBRE,
        c.FECHA_REGISTRO
),
clientes_segmentados AS (
    SELECT
        ID_CLIENTE,
        NOMBRE,
        FECHA_REGISTRO,
        dias_como_cliente,
        total_pedidos,
        monto_total_gastado,
        monto_promedio_pedido,
        fecha_ultimo_pedido,
        -- Clasificación multinivel con CASE WHEN
        CASE
            -- Segmento NEW: sin pedidos o cliente muy reciente
            WHEN total_pedidos = 0
              OR dias_como_cliente < 30                       THEN 'NEW'
            -- Segmento GOLD: alto valor Y alta frecuencia
            WHEN monto_total_gastado >= 5000
             AND total_pedidos >= 5                           THEN 'GOLD'
            -- Segmento SILVER: valor medio O frecuencia moderada
            WHEN monto_total_gastado >= 1500
             AND total_pedidos >= 2                           THEN 'SILVER'
            -- Segmento BRONZE: tiene al menos un pedido
            WHEN total_pedidos >= 1                           THEN 'BRONZE'
            -- Fallback de seguridad
            ELSE                                                   'NEW'
        END AS segmento_cliente
    FROM metricas_cliente
)
SELECT
    ID_CLIENTE,
    NOMBRE,
    segmento_cliente,
    total_pedidos,
    monto_total_gastado,
    monto_promedio_pedido,
    dias_como_cliente,
    fecha_ultimo_pedido
FROM clientes_segmentados
ORDER BY
    CASE segmento_cliente
        WHEN 'GOLD'   THEN 1
        WHEN 'SILVER' THEN 2
        WHEN 'BRONZE' THEN 3
        WHEN 'NEW'    THEN 4
    END,
    monto_total_gastado DESC;
```

> 📌 **Observa el `ORDER BY` con `CASE WHEN`:** Usamos `CASE WHEN` dentro del `ORDER BY` para controlar el orden de los segmentos (GOLD primero, luego SILVER, etc.). Esta es una técnica muy útil para presentar resultados en un orden lógico de negocio en lugar de orden alfabético.

**4.2** Agrega una columna adicional que clasifique también la **antigüedad del cliente** como una segunda dimensión de análisis:

```sql
-- Clasificación enriquecida: segmento de valor + segmento de antigüedad
WITH metricas_cliente AS (
    SELECT
        c.ID_CLIENTE,
        c.NOMBRE,
        c.FECHA_REGISTRO,
        DATEDIFF('day', c.FECHA_REGISTRO, CURRENT_DATE())    AS dias_como_cliente,
        COUNT(p.ID_PEDIDO)                                    AS total_pedidos,
        COALESCE(SUM(p.MONTO_TOTAL), 0)                      AS monto_total_gastado,
        COALESCE(AVG(p.MONTO_TOTAL), 0)                      AS monto_promedio_pedido
    FROM CLIENTES c
    LEFT JOIN PEDIDOS p
           ON c.ID_CLIENTE = p.ID_CLIENTE
          AND p.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY
        c.ID_CLIENTE,
        c.NOMBRE,
        c.FECHA_REGISTRO
)
SELECT
    ID_CLIENTE,
    NOMBRE,
    total_pedidos,
    monto_total_gastado,
    dias_como_cliente,
    -- Segmento de valor (clasificación principal)
    CASE
        WHEN total_pedidos = 0
          OR dias_como_cliente < 30                           THEN 'NEW'
        WHEN monto_total_gastado >= 5000
         AND total_pedidos >= 5                               THEN 'GOLD'
        WHEN monto_total_gastado >= 1500
         AND total_pedidos >= 2                               THEN 'SILVER'
        WHEN total_pedidos >= 1                               THEN 'BRONZE'
        ELSE                                                       'NEW'
    END AS segmento_valor,
    -- Segmento de antigüedad (segunda dimensión)
    CASE
        WHEN dias_como_cliente < 30                           THEN 'Nuevo (< 1 mes)'
        WHEN dias_como_cliente BETWEEN 30 AND 179             THEN 'Reciente (1-6 meses)'
        WHEN dias_como_cliente BETWEEN 180 AND 364            THEN 'Establecido (6-12 meses)'
        WHEN dias_como_cliente >= 365                         THEN 'Veterano (> 1 año)'
        ELSE                                                       'Sin clasificar'
    END AS segmento_antiguedad,
    -- Indicador binario con IFF: ¿es cliente de alto valor?
    IFF(monto_total_gastado >= 3000, 'Sí', 'No')             AS es_alto_valor
FROM metricas_cliente
ORDER BY monto_total_gastado DESC;
```

**4.3** Analiza la combinación de segmentos para identificar patrones:

```sql
-- Cruce de segmento de valor vs. segmento de antigüedad
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
),
clientes_segmentados AS (
    SELECT
        CASE
            WHEN total_pedidos = 0 OR dias_como_cliente < 30  THEN 'NEW'
            WHEN monto_total_gastado >= 5000 AND total_pedidos >= 5 THEN 'GOLD'
            WHEN monto_total_gastado >= 1500 AND total_pedidos >= 2 THEN 'SILVER'
            WHEN total_pedidos >= 1                            THEN 'BRONZE'
            ELSE 'NEW'
        END AS segmento_valor,
        CASE
            WHEN dias_como_cliente < 30                        THEN 'Nuevo'
            WHEN dias_como_cliente BETWEEN 30 AND 179          THEN 'Reciente'
            WHEN dias_como_cliente BETWEEN 180 AND 364         THEN 'Establecido'
            WHEN dias_como_cliente >= 365                      THEN 'Veterano'
            ELSE 'Sin clasificar'
        END AS segmento_antiguedad
    FROM metricas_cliente
)
SELECT
    segmento_valor,
    segmento_antiguedad,
    COUNT(*) AS cantidad_clientes
FROM clientes_segmentados
GROUP BY segmento_valor, segmento_antiguedad
ORDER BY
    CASE segmento_valor
        WHEN 'GOLD' THEN 1 WHEN 'SILVER' THEN 2
        WHEN 'BRONZE' THEN 3 WHEN 'NEW' THEN 4
    END,
    segmento_antiguedad;
```

#### Resultado esperado

- La consulta 4.1 muestra todos los clientes con su segmento asignado, ordenados de GOLD a NEW.
- La consulta 4.2 agrega dos dimensiones de clasificación por cliente: segmento de valor y segmento de antigüedad.
- La consulta 4.3 muestra una tabla cruzada con la distribución de clientes por combinación de segmentos.

#### Verificación

```sql
-- Verificar que TODOS los clientes tienen un segmento asignado (no NULL)
WITH metricas_cliente AS (
    SELECT
        c.ID_CLIENTE,
        DATEDIFF('day', c.FECHA_REGISTRO, CURRENT_DATE()) AS dias_como_cliente,
        COUNT(p.ID_PEDIDO)                                 AS total_pedidos,
        COALESCE(SUM(p.MONTO_TOTAL), 0)                   AS monto_total_gastado
    FROM CLIENTES c
    LEFT JOIN PEDIDOS p
           ON c.ID_CLIENTE = p.ID_CLIENTE
          AND p.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY c.ID_CLIENTE, c.FECHA_REGISTRO
)
SELECT
    COUNT(*) AS total_clientes,
    COUNT(CASE
              WHEN total_pedidos = 0 OR dias_como_cliente < 30  THEN 'NEW'
              WHEN monto_total_gastado >= 5000 AND total_pedidos >= 5 THEN 'GOLD'
              WHEN monto_total_gastado >= 1500 AND total_pedidos >= 2 THEN 'SILVER'
              WHEN total_pedidos >= 1 THEN 'BRONZE'
              ELSE 'NEW'
          END) AS clientes_con_segmento
FROM metricas_cliente;
-- total_clientes debe ser igual a clientes_con_segmento
```

---

### Paso 5: Resumen ejecutivo por segmento con CASE WHEN en agregaciones

**Objetivo:** Generar una tabla resumen con métricas clave por segmento usando `CASE WHEN` dentro de funciones de agregación, produciendo el reporte ejecutivo que el equipo de marketing necesita.

#### Instrucciones

**5.1** Construye el reporte ejecutivo completo por segmento usando `CASE WHEN` dentro de `SUM()` y `COUNT()`:

```sql
-- Reporte ejecutivo por segmento de cliente
-- Usa CASE WHEN dentro de funciones de agregación para métricas condicionales
WITH metricas_cliente AS (
    SELECT
        c.ID_CLIENTE,
        c.NOMBRE,
        c.FECHA_REGISTRO,
        DATEDIFF('day', c.FECHA_REGISTRO, CURRENT_DATE())    AS dias_como_cliente,
        COUNT(p.ID_PEDIDO)                                    AS total_pedidos,
        COALESCE(SUM(p.MONTO_TOTAL), 0)                      AS monto_total_gastado,
        COALESCE(AVG(p.MONTO_TOTAL), 0)                      AS monto_promedio_pedido
    FROM CLIENTES c
    LEFT JOIN PEDIDOS p
           ON c.ID_CLIENTE = p.ID_CLIENTE
          AND p.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY
        c.ID_CLIENTE,
        c.NOMBRE,
        c.FECHA_REGISTRO
),
clientes_segmentados AS (
    SELECT
        ID_CLIENTE,
        NOMBRE,
        total_pedidos,
        monto_total_gastado,
        monto_promedio_pedido,
        dias_como_cliente,
        CASE
            WHEN total_pedidos = 0
              OR dias_como_cliente < 30                       THEN 'NEW'
            WHEN monto_total_gastado >= 5000
             AND total_pedidos >= 5                           THEN 'GOLD'
            WHEN monto_total_gastado >= 1500
             AND total_pedidos >= 2                           THEN 'SILVER'
            WHEN total_pedidos >= 1                           THEN 'BRONZE'
            ELSE                                                   'NEW'
        END AS segmento_cliente
    FROM metricas_cliente
)
-- Reporte final: métricas agregadas por segmento
SELECT
    segmento_cliente,
    COUNT(*)                                                  AS cantidad_clientes,
    -- Porcentaje del total de clientes
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 1)        AS pct_clientes,
    -- Métricas de gasto
    ROUND(SUM(monto_total_gastado), 2)                        AS revenue_total_segmento,
    ROUND(AVG(monto_total_gastado), 2)                        AS gasto_promedio_cliente,
    ROUND(MAX(monto_total_gastado), 2)                        AS gasto_maximo_cliente,
    -- Métricas de frecuencia
    ROUND(AVG(total_pedidos), 1)                              AS pedidos_promedio,
    -- Antigüedad promedio
    ROUND(AVG(dias_como_cliente), 0)                          AS antiguedad_promedio_dias,
    -- Porcentaje del revenue total
    ROUND(SUM(monto_total_gastado) * 100.0 /
          SUM(SUM(monto_total_gastado)) OVER(), 1)            AS pct_revenue_total
FROM clientes_segmentados
GROUP BY segmento_cliente
ORDER BY
    CASE segmento_cliente
        WHEN 'GOLD'   THEN 1
        WHEN 'SILVER' THEN 2
        WHEN 'BRONZE' THEN 3
        WHEN 'NEW'    THEN 4
    END;
```

> 📌 **Nota sobre `SUM(COUNT(*)) OVER()`:** Esta es una window function básica que calcula el total general para calcular porcentajes. Las window functions se estudiarán en profundidad en el Laboratorio 4; por ahora, úsala como está sin modificarla.

**5.2** Genera un reporte de métricas condicionales usando `CASE WHEN` dentro de `SUM()` — el patrón de "pivot condicional":

```sql
-- Pivot condicional: métricas de revenue por segmento en columnas separadas
-- Patrón: SUM(CASE WHEN condición THEN valor ELSE 0 END)
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
),
clientes_segmentados AS (
    SELECT
        ID_CLIENTE,
        monto_total_gastado,
        total_pedidos,
        CASE
            WHEN total_pedidos = 0 OR dias_como_cliente < 30  THEN 'NEW'
            WHEN monto_total_gastado >= 5000 AND total_pedidos >= 5 THEN 'GOLD'
            WHEN monto_total_gastado >= 1500 AND total_pedidos >= 2 THEN 'SILVER'
            WHEN total_pedidos >= 1 THEN 'BRONZE'
            ELSE 'NEW'
        END AS segmento_cliente
    FROM metricas_cliente
)
-- Resumen global con columnas por segmento (pivot condicional)
SELECT
    COUNT(*)                                                   AS total_clientes,
    -- Conteo por segmento usando COUNT con CASE WHEN
    COUNT(CASE WHEN segmento_cliente = 'GOLD'   THEN 1 END)   AS clientes_gold,
    COUNT(CASE WHEN segmento_cliente = 'SILVER' THEN 1 END)   AS clientes_silver,
    COUNT(CASE WHEN segmento_cliente = 'BRONZE' THEN 1 END)   AS clientes_bronze,
    COUNT(CASE WHEN segmento_cliente = 'NEW'    THEN 1 END)   AS clientes_new,
    -- Revenue por segmento usando SUM con CASE WHEN
    ROUND(SUM(CASE WHEN segmento_cliente = 'GOLD'
                   THEN monto_total_gastado ELSE 0 END), 2)   AS revenue_gold,
    ROUND(SUM(CASE WHEN segmento_cliente = 'SILVER'
                   THEN monto_total_gastado ELSE 0 END), 2)   AS revenue_silver,
    ROUND(SUM(CASE WHEN segmento_cliente = 'BRONZE'
                   THEN monto_total_gastado ELSE 0 END), 2)   AS revenue_bronze,
    -- Revenue total para verificación
    ROUND(SUM(monto_total_gastado), 2)                         AS revenue_total
FROM clientes_segmentados;
```

#### Resultado esperado

- La consulta 5.1 debe producir exactamente 4 filas (una por segmento: GOLD, SILVER, BRONZE, NEW) con todas las métricas calculadas.
- La consulta 5.2 debe producir exactamente **1 fila** con columnas separadas para cada segmento — el patrón pivot condicional.
- La suma de `clientes_gold + clientes_silver + clientes_bronze + clientes_new` debe ser igual a `total_clientes`.
- La suma de `revenue_gold + revenue_silver + revenue_bronze` debe ser igual a `revenue_total`.

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

---

### Paso 6: Dataset enriquecido para análisis posterior — Integración con CTE

**Objetivo:** Producir el dataset final enriquecido con todas las clasificaciones, listo para ser consumido por reportes o análisis posteriores, usando una estructura de CTEs encadenadas aprendida en el Laboratorio 1.

#### Instrucciones

**6.1** Construye la consulta final que integra todo lo aprendido en este laboratorio: métricas base, clasificación multinivel, segmentación de antigüedad y resumen ejecutivo en una estructura de CTEs encadenadas:

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

**6.2** Genera el resumen ejecutivo final del dataset enriquecido:

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

#### Resultado esperado

- La consulta 6.1 produce el dataset enriquecido completo con 5 columnas de clasificación por cliente.
- La consulta 6.2 muestra la distribución cruzada de segmento vs. riesgo de abandono.
- Todos los clientes tienen valores asignados en todas las columnas de clasificación (sin `NULL` en las columnas `CASE WHEN`).

---

## 7. Validación y Pruebas

Ejecuta las siguientes consultas de validación para confirmar que todos los resultados son correctos antes de cerrar el laboratorio:

```sql
-- ============================================================
-- SUITE DE VALIDACIÓN FINAL - Lab 02-00-01
-- Ejecutar todas las consultas y verificar los resultados
-- ============================================================

-- VALIDACIÓN 1: Conteo total de clientes debe ser consistente
SELECT
    'Clientes en tabla base'    AS verificacion,
    COUNT(*)                    AS valor
FROM CLIENTES
UNION ALL
SELECT
    'Clientes en métricas CTE'  AS verificacion,
    COUNT(DISTINCT c.ID_CLIENTE) AS valor
FROM CLIENTES c
LEFT JOIN PEDIDOS p ON c.ID_CLIENTE = p.ID_CLIENTE;
-- Ambos valores deben ser iguales

-- VALIDACIÓN 2: Ningún cliente debe tener segmento NULL
WITH metricas AS (
    SELECT
        c.ID_CLIENTE,
        DATEDIFF('day', c.FECHA_REGISTRO, CURRENT_DATE()) AS dias_cliente,
        COUNT(p.ID_PEDIDO)                                 AS n_pedidos,
        COALESCE(SUM(p.MONTO_TOTAL), 0)                   AS monto
    FROM CLIENTES c
    LEFT JOIN PEDIDOS p
           ON c.ID_CLIENTE = p.ID_CLIENTE
          AND p.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY c.ID_CLIENTE, c.FECHA_REGISTRO
)
SELECT
    COUNT(*)                                               AS total_clientes,
    COUNT(CASE WHEN n_pedidos = 0 OR dias_cliente < 30    THEN 'NEW'
               WHEN monto >= 5000 AND n_pedidos >= 5      THEN 'GOLD'
               WHEN monto >= 1500 AND n_pedidos >= 2      THEN 'SILVER'
               WHEN n_pedidos >= 1                        THEN 'BRONZE'
               ELSE 'NEW' END)                             AS clientes_con_segmento
FROM metricas;
-- total_clientes debe ser igual a clientes_con_segmento

-- VALIDACIÓN 3: La suma de clientes por segmento debe igualar el total
WITH metricas AS (
    SELECT
        c.ID_CLIENTE,
        DATEDIFF('day', c.FECHA_REGISTRO, CURRENT_DATE()) AS dias_cliente,
        COUNT(p.ID_PEDIDO)                                 AS n_pedidos,
        COALESCE(SUM(p.MONTO_TOTAL), 0)                   AS monto
    FROM CLIENTES c
    LEFT JOIN PEDIDOS p
           ON c.ID_CLIENTE = p.ID_CLIENTE
          AND p.ESTADO_PEDIDO != 'CANCELADO'
    GROUP BY c.ID_CLIENTE, c.FECHA_REGISTRO
),
segmentados AS (
    SELECT
        CASE
            WHEN n_pedidos = 0 OR dias_cliente < 30       THEN 'NEW'
            WHEN monto >= 5000 AND n_pedidos >= 5         THEN 'GOLD'
            WHEN monto >= 1500 AND n_pedidos >= 2         THEN 'SILVER'
            WHEN n_pedidos >= 1                           THEN 'BRONZE'
            ELSE 'NEW'
        END AS segmento
    FROM metricas
)
SELECT
    COUNT(*)                                               AS total_general,
    COUNT(CASE WHEN segmento = 'GOLD'   THEN 1 END)       AS gold,
    COUNT(CASE WHEN segmento = 'SILVER' THEN 1 END)       AS silver,
    COUNT(CASE WHEN segmento = 'BRONZE' THEN 1 END)       AS bronze,
    COUNT(CASE WHEN segmento = 'NEW'    THEN 1 END)       AS new_clientes,
    -- La siguiente columna debe ser 0 si todos están clasificados
    COUNT(CASE WHEN segmento IS NULL    THEN 1 END)       AS sin_segmento
FROM segmentados;
-- sin_segmento debe ser 0
-- gold + silver + bronze + new_clientes debe igualar total_general
```

**Criterios de éxito:**

| Validación | Resultado esperado |
|---|---|
| Validación 1 | Ambos conteos son iguales |
| Validación 2 | `total_clientes = clientes_con_segmento` |
| Validación 3 | `sin_segmento = 0`; suma de segmentos = total general |

---

## 8. Resolución de Problemas

### Problema 1: Error "Object 'CLIENTES' does not exist or not authorized"

**Síntomas:**
Al ejecutar cualquier consulta que referencia `CLIENTES` o `PEDIDOS`, Snowflake retorna:
```
SQL compilation error: Object 'LAB_SQL_INTERMEDIO.VENTAS.CLIENTES' does not exist or not authorized.
```

**Causa:**
El contexto de la sesión no está configurado correctamente. El warehouse, la base de datos o el schema activo no corresponden al entorno del laboratorio. Esto ocurre cuando se abre una nueva worksheet sin ejecutar el bloque de configuración inicial, o cuando el rol activo no tiene permisos sobre los objetos.

**Solución:**
1. Verifica el contexto activo en la barra superior de Snowsight (muestra el rol, warehouse, base de datos y schema seleccionados).
2. Ejecuta nuevamente el bloque de configuración inicial del Paso 5 de este laboratorio:
   ```sql
   USE ROLE LAB_ROLE;
   USE WAREHOUSE LAB_WH;
   USE DATABASE LAB_SQL_INTERMEDIO;
   USE SCHEMA VENTAS;
   ```
3. Si el error persiste después de configurar el contexto, verifica con el instructor que el script `00_setup_laboratorios.sql` fue ejecutado y que tu usuario tiene los permisos correctos:
   ```sql
   SHOW GRANTS TO ROLE LAB_ROLE;
   ```

---

### Problema 2: La clasificación CASE WHEN produce resultados inesperados — clientes clasificados en el segmento incorrecto

**Síntomas:**
Algunos clientes aparecen clasificados como `BRONZE` cuando deberían ser `GOLD`, o aparecen en `NEW` a pesar de tener historial de compras. Al revisar los datos del cliente, los valores de `monto_total_gastado` y `total_pedidos` parecen correctos.

**Causa:**
El error más frecuente es el **orden de evaluación de condiciones en `CASE WHEN`**. Si la condición de `BRONZE` (`total_pedidos >= 1`) aparece antes que la condición de `GOLD` o `SILVER`, todos los clientes con al menos un pedido serán clasificados como `BRONZE` sin llegar a evaluar las condiciones superiores. También puede ocurrir cuando `monto_total_gastado` tiene valores `NULL` (no tratados con `COALESCE`) que hacen que las comparaciones numéricas retornen `NULL` en lugar de `FALSE`.

**Solución:**
1. **Verifica el orden de las condiciones:** Las condiciones más restrictivas (GOLD) deben ir primero. `CASE WHEN` retorna el resultado de la **primera condición verdadera**:
   ```sql
   -- CORRECTO: de más restrictivo a menos restrictivo
   CASE
       WHEN total_pedidos = 0 OR dias_como_cliente < 30  THEN 'NEW'    -- primero
       WHEN monto >= 5000 AND total_pedidos >= 5         THEN 'GOLD'   -- segundo
       WHEN monto >= 1500 AND total_pedidos >= 2         THEN 'SILVER' -- tercero
       WHEN total_pedidos >= 1                           THEN 'BRONZE' -- último
       ELSE 'NEW'
   END
   ```
2. **Trata los valores NULL con `COALESCE`** en la CTE de métricas base:
   ```sql
   COALESCE(SUM(p.MONTO_TOTAL), 0) AS monto_total_gastado
   -- Convierte NULL a 0 para que las comparaciones numéricas funcionen correctamente
   ```
3. Para diagnosticar, agrega una columna de depuración que muestre los valores crudos junto con la clasificación y verifica manualmente algunos registros:
   ```sql
   SELECT ID_CLIENTE, total_pedidos, monto_total_gastado, segmento_cliente
   FROM clientes_segmentados
   WHERE segmento_cliente = 'BRONZE'
   ORDER BY monto_total_gastado DESC
   LIMIT 10;
   -- Si ves clientes con monto >= 5000 y pedidos >= 5, el orden de condiciones está incorrecto
   ```

---

## 9. Limpieza del Entorno

Al finalizar el laboratorio, ejecuta los siguientes comandos para liberar recursos y evitar consumo innecesario de créditos Snowflake:

```sql
-- ============================================================
-- LIMPIEZA POST-LABORATORIO - Lab 02-00-01
-- ============================================================

-- 1. Suspender el warehouse para detener el consumo de créditos
--    IMPORTANTE: Ejecutar SIEMPRE al terminar la sesión
ALTER WAREHOUSE LAB_WH SUSPEND;

-- 2. Verificar que el warehouse quedó suspendido
SHOW WAREHOUSES LIKE 'LAB_WH';
-- El campo STATE debe mostrar 'SUSPENDED'
```

> ⚠️ **Recordatorio importante sobre créditos:** Las cuentas de prueba (trial) de Snowflake tienen un límite de 400 USD de créditos. Un warehouse X-SMALL consume aproximadamente 1 crédito por hora de uso activo. Suspender el warehouse al terminar cada sesión es una práctica obligatoria en este curso. Si olvidas suspenderlo, el warehouse se suspende automáticamente después del período de inactividad configurado (generalmente 5-10 minutos), pero es mejor hacerlo manualmente.

**No es necesario eliminar ningún objeto de base de datos.** Las tablas `CLIENTES`, `PEDIDOS`, `VENTAS` y `PRODUCTOS` son compartidas entre todos los laboratorios del curso y deben permanecer intactas para los laboratorios 3 al 7.

---

## 10. Resumen y Próximos Pasos

### Lo que aprendiste en este laboratorio

En este laboratorio implementaste un sistema completo de segmentación de clientes usando `CASE WHEN` en Snowflake. Los conceptos clave que practicaste:

| Concepto | Aplicación en el laboratorio |
|---|---|
| `CASE WHEN` forma buscada | Clasificación de pedidos por monto (Paso 2) |
| `CASE WHEN` forma simple | Traducción de códigos de estado (Paso 2) |
| `CASE WHEN` con condiciones compuestas (`AND`, `BETWEEN`) | Clasificación multinivel GOLD/SILVER/BRONZE (Paso 4) |
| `CASE WHEN` dentro de `SUM()` y `COUNT()` | Pivot condicional de métricas por segmento (Paso 5) |
| `CASE WHEN` dentro de `ORDER BY` | Control de orden lógico de resultados (Pasos 4 y 5) |
| `IFF()` de Snowflake | Clasificación binaria simplificada (Pasos 2 y 6) |
| CTEs encadenadas con clasificaciones | Dataset enriquecido final (Paso 6) |
| Manejo de `NULL` con `COALESCE` en clasificaciones | Tratamiento de clientes sin pedidos (Pasos 3-6) |

### Puntos clave para recordar

- **El orden de las condiciones en `CASE WHEN` es crítico:** se evalúan de arriba hacia abajo y se retorna la primera condición verdadera. Coloca siempre las condiciones más restrictivas primero.
- **`IFF()` es exclusivo de Snowflake:** no es portable a PostgreSQL, MySQL ni SQL Server. Úsalo cuando la simplicidad sea prioritaria y el código no necesite ser portable.
- **`CASE WHEN` dentro de `SUM()` o `COUNT()` permite calcular métricas condicionales en una sola pasada** sobre los datos, lo que es más eficiente que múltiples subconsultas o uniones.
- **`COALESCE()` es tu aliado:** siempre trata los `NULL` antes de usarlos en comparaciones dentro de `CASE WHEN`.

### Próximos pasos

En el **Laboratorio 3 (Lab 02-01-01: Detección y manejo de duplicados)** aplicarás las clasificaciones construidas en este laboratorio como punto de partida para identificar y resolver registros duplicados en los datasets. Además, se introducirá `ROW_NUMBER()` como primera window function, en el contexto específico de deduplicación de datos.

### Recursos adicionales

| Recurso | URL | Utilidad |
|---|---|---|
| Documentación oficial Snowflake: CASE | https://docs.snowflake.com/en/sql-reference/functions/case | Referencia técnica completa con ejemplos |
| Documentación oficial Snowflake: IFF | https://docs.snowflake.com/en/sql-reference/functions/iff | Referencia de la función IFF exclusiva de Snowflake |
| Mode Analytics: SQL CASE Tutorial | https://mode.com/sql-tutorial/sql-case/ | Guía práctica orientada a análisis de datos |
| W3Schools: SQL CASE Statement | https://www.w3schools.com/sql/sql_case.asp | Referencia rápida con ejemplos interactivos |

---
