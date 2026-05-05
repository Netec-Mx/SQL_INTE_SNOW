# Detección de duplicados y registros inconsistentes

## Metadatos

| Atributo | Detalle |
|---|---|
| **Duración estimada** | 60 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar |
| **Módulo** | 3 — Calidad de datos y detección de anomalías |
| **Laboratorio previo requerido** | Lab 01 y Lab 02 (o equivalente) |
| **Plataforma** | Snowflake (Snowsight) |

---

## Descripción General

En este laboratorio trabajarás con versiones intencionalmente degradadas de las tablas del schema de práctica (`CLIENTES_DIRTY` y `PEDIDOS_DIRTY`) que contienen duplicados, valores nulos críticos y referencias inválidas. Aplicarás tres técnicas complementarias: detección de duplicados con `GROUP BY` + `HAVING`, marcado y aislamiento de duplicados con `ROW_NUMBER()`, y validación de integridad referencial con `LEFT JOIN` + `IS NULL`. Al finalizar, construirás un reporte consolidado de calidad de datos que integra todos los hallazgos. Este laboratorio introduce formalmente las **window functions** como puente conceptual hacia el Laboratorio 4.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Detectar registros duplicados usando `GROUP BY` con `HAVING COUNT(*) > 1` sobre claves simples y compuestas
- [ ] Aplicar `ROW_NUMBER() OVER(PARTITION BY ... ORDER BY ...)` para numerar duplicados e identificar el registro canónico a conservar
- [ ] Utilizar la cláusula `QUALIFY` de Snowflake para filtrar resultados de window functions directamente en la consulta
- [ ] Validar integridad referencial entre tablas mediante `LEFT JOIN` con filtro `WHERE ... IS NULL` para detectar registros huérfanos
- [ ] Construir un reporte de calidad de datos que consolide duplicados, nulos críticos e inconsistencias de referencia

---

## Prerrequisitos

### Conocimientos Previos

| Área | Nivel Requerido |
|---|---|
| `SELECT`, `FROM`, `WHERE`, `ORDER BY` | Sólido |
| `GROUP BY` con funciones de agregación (`COUNT`, `SUM`) | Sólido |
| `HAVING` para filtrado de grupos | Sólido |
| `JOIN` (especialmente `LEFT JOIN`) | Intermedio |
| Concepto de `NULL` y operadores `IS NULL` / `IS NOT NULL` | Intermedio |
| Concepto introductorio de window functions | Básico (suficiente con la introducción conceptual) |

### Acceso y Configuración

- Cuenta activa en Snowflake (trial o corporativa) con acceso a Snowsight
- Script `00_setup_laboratorios.sql` ejecutado previamente por el instructor
- Rol con permisos `SELECT` sobre el schema `LAB_SQL_INTERMEDIO.VENTAS`
- Warehouse `X-SMALL` disponible y en estado `STARTED` o `SUSPENDED` (se activa automáticamente)

---

## Entorno del Laboratorio

### Requisitos de Hardware y Software

| Componente | Mínimo | Recomendado |
|---|---|---|
| RAM | 8 GB | 16 GB |
| Procesador | Intel Core i5 / AMD Ryzen 5 (64-bit) | Intel Core i7 / AMD Ryzen 7 |
| Resolución | 1280×768 | 1920×1080 |
| Conexión a Internet | 10 Mbps | 25 Mbps |
| Navegador | Chrome 110+, Firefox 110+, Edge 110+, Safari 16+ | Chrome última versión |
| Snowflake Edition | Trial o Enterprise | Enterprise |

### Estructura de Datos del Laboratorio

Este laboratorio utiliza las siguientes tablas del schema `LAB_SQL_INTERMEDIO.VENTAS`:

| Tabla | Descripción | Problema intencionado |
|---|---|---|
| `CLIENTES_DIRTY` | Versión degradada de CLIENTES | Duplicados por `EMAIL` y `CLIENTE_ID` |
| `PEDIDOS_DIRTY` | Versión degradada de PEDIDOS | Duplicados por `PEDIDO_ID`, nulos en `CLIENTE_ID`, referencias inválidas |
| `CLIENTES` | Tabla limpia de referencia | Sin problemas (usada para comparación) |
| `PEDIDOS` | Tabla limpia de referencia | Sin problemas (usada para comparación) |

### Comandos de Configuración Inicial

Ejecuta los siguientes comandos al inicio del laboratorio para establecer el contexto correcto:

```sql
-- Establecer contexto de trabajo
USE ROLE ANALYST;
USE WAREHOUSE LAB_WH;
USE DATABASE LAB_SQL_INTERMEDIO;
USE SCHEMA VENTAS;

-- Verificar que el warehouse está activo
SELECT CURRENT_WAREHOUSE(), CURRENT_DATABASE(), CURRENT_SCHEMA();
```

**Resultado esperado:**

| CURRENT_WAREHOUSE() | CURRENT_DATABASE() | CURRENT_SCHEMA() |
|---|---|---|
| LAB_WH | LAB_SQL_INTERMEDIO | VENTAS |

> ⚠️ **Importante sobre créditos:** Este laboratorio usa un warehouse `X-SMALL`. Al finalizar la sesión ejecuta `ALTER WAREHOUSE LAB_WH SUSPEND;` para evitar consumo innecesario de créditos de tu cuenta trial.

---

## Pasos del Laboratorio

---

### Paso 1: Exploración Inicial de las Tablas con Problemas

**Objetivo:** Familiarizarse con la estructura y el contenido de las tablas `CLIENTES_DIRTY` y `PEDIDOS_DIRTY` antes de comenzar el análisis de calidad.

#### Instrucciones

**1.1** Examina la estructura y primeras filas de `CLIENTES_DIRTY`:

```sql
-- Vista previa de CLIENTES_DIRTY
SELECT *
FROM CLIENTES_DIRTY
LIMIT 20;
```

**1.2** Examina la estructura y primeras filas de `PEDIDOS_DIRTY`:

```sql
-- Vista previa de PEDIDOS_DIRTY
SELECT *
FROM PEDIDOS_DIRTY
LIMIT 20;
```

**1.3** Obtén un conteo general de ambas tablas para tener una línea base:

```sql
-- Conteo de registros en ambas tablas
SELECT
    'CLIENTES_DIRTY'  AS tabla,
    COUNT(*)          AS total_registros
FROM CLIENTES_DIRTY

UNION ALL

SELECT
    'PEDIDOS_DIRTY'   AS tabla,
    COUNT(*)          AS total_registros
FROM PEDIDOS_DIRTY

UNION ALL

SELECT
    'CLIENTES (limpia)' AS tabla,
    COUNT(*)            AS total_registros
FROM CLIENTES

UNION ALL

SELECT
    'PEDIDOS (limpia)'  AS tabla,
    COUNT(*)            AS total_registros
FROM PEDIDOS;
```

**1.4** Identifica cuántos valores nulos existen en columnas críticas de cada tabla:

```sql
-- Diagnóstico de nulos en CLIENTES_DIRTY
SELECT
    COUNT(*)                                         AS total_filas,
    COUNT(CLIENTE_ID)                                AS cliente_id_no_nulos,
    COUNT(*) - COUNT(CLIENTE_ID)                     AS cliente_id_nulos,
    COUNT(EMAIL)                                     AS email_no_nulos,
    COUNT(*) - COUNT(EMAIL)                          AS email_nulos,
    COUNT(NOMBRE)                                    AS nombre_no_nulos,
    COUNT(*) - COUNT(NOMBRE)                         AS nombre_nulos
FROM CLIENTES_DIRTY;
```

```sql
-- Diagnóstico de nulos en PEDIDOS_DIRTY
SELECT
    COUNT(*)                                         AS total_filas,
    COUNT(PEDIDO_ID)                                 AS pedido_id_no_nulos,
    COUNT(*) - COUNT(PEDIDO_ID)                      AS pedido_id_nulos,
    COUNT(CLIENTE_ID)                                AS cliente_id_no_nulos,
    COUNT(*) - COUNT(CLIENTE_ID)                     AS cliente_id_nulos,
    COUNT(MONTO_TOTAL)                               AS monto_no_nulos,
    COUNT(*) - COUNT(MONTO_TOTAL)                    AS monto_nulos
FROM PEDIDOS_DIRTY;
```

#### Resultado Esperado

- `CLIENTES_DIRTY` debería tener más registros que `CLIENTES` (indicando duplicados introducidos)
- `PEDIDOS_DIRTY` debería tener más registros que `PEDIDOS`
- Deberías observar valores nulos en al menos `EMAIL` en `CLIENTES_DIRTY` y en `CLIENTE_ID` en `PEDIDOS_DIRTY`
- La diferencia entre `COUNT(*)` y `COUNT(columna)` revela exactamente cuántos nulos hay en cada campo

#### Verificación

Anota los siguientes valores en tu hoja de trabajo (los usarás en el Paso 5):

| Métrica | Valor observado |
|---|---|
| Total filas `CLIENTES_DIRTY` | \_\_\_\_\_ |
| Total filas `CLIENTES` (limpia) | \_\_\_\_\_ |
| Nulos en `EMAIL` de `CLIENTES_DIRTY` | \_\_\_\_\_ |
| Total filas `PEDIDOS_DIRTY` | \_\_\_\_\_ |
| Nulos en `CLIENTE_ID` de `PEDIDOS_DIRTY` | \_\_\_\_\_ |

---

### Paso 2: Detección de Duplicados con GROUP BY + HAVING

**Objetivo:** Aplicar la técnica de `GROUP BY` con `HAVING COUNT(*) > 1` para identificar claves repetidas en `CLIENTES_DIRTY` y `PEDIDOS_DIRTY`, tanto por clave simple como compuesta.

#### Instrucciones

**2.1** Detecta duplicados de `CLIENTE_ID` en `CLIENTES_DIRTY` (clave primaria que debería ser única):

```sql
-- Detección de CLIENTE_ID duplicados
SELECT
    CLIENTE_ID,
    COUNT(*) AS veces_registrado
FROM CLIENTES_DIRTY
GROUP BY CLIENTE_ID
HAVING COUNT(*) > 1
ORDER BY veces_registrado DESC;
```

**2.2** Detecta duplicados por `EMAIL` (campo de negocio que también debería ser único):

```sql
-- Detección de EMAIL duplicados
SELECT
    EMAIL,
    COUNT(*) AS veces_registrado
FROM CLIENTES_DIRTY
GROUP BY EMAIL
HAVING COUNT(*) > 1
ORDER BY veces_registrado DESC;
```

> 💡 **Observa la diferencia:** Es posible que encuentres más duplicados por `EMAIL` que por `CLIENTE_ID`, o viceversa. Esto refleja diferentes tipos de problemas de carga: en algunos casos se duplicó el registro completo (mismo ID), en otros se crearon registros nuevos para el mismo cliente real (mismo email, ID diferente).

**2.3** Calcula el total de registros de más que deben eliminarse en `CLIENTES_DIRTY`:

```sql
-- Total de registros duplicados (sobrantes) por CLIENTE_ID
SELECT
    SUM(veces_registrado - 1) AS registros_sobrantes
FROM (
    SELECT
        CLIENTE_ID,
        COUNT(*) AS veces_registrado
    FROM CLIENTES_DIRTY
    GROUP BY CLIENTE_ID
    HAVING COUNT(*) > 1
) AS duplicados_cliente;
```

**2.4** Detecta duplicados de `PEDIDO_ID` en `PEDIDOS_DIRTY`:

```sql
-- Detección de PEDIDO_ID duplicados
SELECT
    PEDIDO_ID,
    COUNT(*) AS veces_registrado
FROM PEDIDOS_DIRTY
GROUP BY PEDIDO_ID
HAVING COUNT(*) > 1
ORDER BY veces_registrado DESC;
```

**2.5** Detecta duplicados compuestos en `PEDIDOS_DIRTY`: misma combinación de `CLIENTE_ID` + `FECHA_PEDIDO` + `MONTO_TOTAL` (posible doble procesamiento del mismo pedido):

```sql
-- Duplicados compuestos: mismo cliente, misma fecha, mismo monto
SELECT
    CLIENTE_ID,
    FECHA_PEDIDO,
    MONTO_TOTAL,
    COUNT(*) AS veces_registrado
FROM PEDIDOS_DIRTY
WHERE CLIENTE_ID   IS NOT NULL
  AND FECHA_PEDIDO IS NOT NULL
  AND MONTO_TOTAL  IS NOT NULL
GROUP BY CLIENTE_ID, FECHA_PEDIDO, MONTO_TOTAL
HAVING COUNT(*) > 1
ORDER BY veces_registrado DESC, FECHA_PEDIDO DESC;
```

**2.6** Visualiza el detalle completo de los registros duplicados para poder compararlos. Usa un `INNER JOIN` contra el resultado de la detección:

```sql
-- Ver filas completas de clientes con CLIENTE_ID duplicado
SELECT
    c.*
FROM CLIENTES_DIRTY c
INNER JOIN (
    SELECT CLIENTE_ID
    FROM CLIENTES_DIRTY
    GROUP BY CLIENTE_ID
    HAVING COUNT(*) > 1
) dup ON c.CLIENTE_ID = dup.CLIENTE_ID
ORDER BY c.CLIENTE_ID, c.FECHA_REGISTRO;
```

#### Resultado Esperado

- El paso 2.1 debe mostrar una lista de `CLIENTE_ID` que aparecen más de una vez
- El paso 2.3 devuelve un número entero: la cantidad exacta de filas que sobran
- El paso 2.5 puede revelar duplicados que no se detectarían solo por `PEDIDO_ID` (casos donde el ID es diferente pero el contenido es idéntico)
- El paso 2.6 muestra pares o tríos de filas que representan el mismo cliente, permitiendo comparar qué campos difieren entre duplicados

#### Verificación

Responde las siguientes preguntas con base en los resultados:

1. ¿Cuántos `CLIENTE_ID` distintos tienen duplicados en `CLIENTES_DIRTY`?
2. ¿El número de duplicados por `EMAIL` coincide con el de duplicados por `CLIENTE_ID`? ¿Qué indica si no coinciden?
3. ¿Cuántos registros sobrantes calculó la subconsulta del paso 2.3?

---

### Paso 3: Deduplicación con ROW_NUMBER()

**Objetivo:** Aplicar la función de ventana `ROW_NUMBER()` para numerar los duplicados dentro de cada grupo y aislar el registro canónico (el que se debe conservar) según un criterio de negocio definido.

> 📘 **Concepto clave — Window Functions:** A diferencia de las funciones de agregación que colapsan múltiples filas en una sola, las window functions calculan un valor **para cada fila** considerando un conjunto de filas relacionadas (la "ventana"). `ROW_NUMBER()` asigna un número secuencial único a cada fila dentro de una partición. La sintaxis es: `ROW_NUMBER() OVER(PARTITION BY columna ORDER BY columna)`. La cláusula `PARTITION BY` define los grupos (como `GROUP BY`), y `ORDER BY` define el criterio de numeración.

#### Instrucciones

**3.1** Aplica `ROW_NUMBER()` para numerar los duplicados de `CLIENTE_ID`, ordenando por `FECHA_REGISTRO` ascendente (conservar el registro más antiguo como canónico):

```sql
-- Numerar duplicados de CLIENTE_ID con ROW_NUMBER()
-- El registro con rn = 1 es el que conservaríamos (más antiguo)
SELECT
    CLIENTE_ID,
    NOMBRE,
    EMAIL,
    FECHA_REGISTRO,
    ROW_NUMBER() OVER (
        PARTITION BY CLIENTE_ID
        ORDER BY FECHA_REGISTRO ASC
    ) AS rn
FROM CLIENTES_DIRTY
ORDER BY CLIENTE_ID, rn;
```

Observa cómo cada grupo de `CLIENTE_ID` recibe numeración independiente comenzando en 1.

**3.2** Construye una consulta que aísle **solo los duplicados** (filas con `rn > 1`), usando una subconsulta:

```sql
-- Aislar únicamente los registros duplicados (los que se eliminarían)
SELECT *
FROM (
    SELECT
        CLIENTE_ID,
        NOMBRE,
        EMAIL,
        FECHA_REGISTRO,
        ROW_NUMBER() OVER (
            PARTITION BY CLIENTE_ID
            ORDER BY FECHA_REGISTRO ASC
        ) AS rn
    FROM CLIENTES_DIRTY
) AS numerados
WHERE rn > 1
ORDER BY CLIENTE_ID, rn;
```

**3.3** Usa la cláusula `QUALIFY` de Snowflake para lograr el mismo resultado de forma más concisa. `QUALIFY` permite filtrar directamente sobre el resultado de una window function sin necesidad de subconsulta:

```sql
-- Equivalente con QUALIFY (sintaxis exclusiva de Snowflake)
SELECT
    CLIENTE_ID,
    NOMBRE,
    EMAIL,
    FECHA_REGISTRO,
    ROW_NUMBER() OVER (
        PARTITION BY CLIENTE_ID
        ORDER BY FECHA_REGISTRO ASC
    ) AS rn
FROM CLIENTES_DIRTY
QUALIFY rn > 1
ORDER BY CLIENTE_ID, rn;
```

> ⚠️ **Nota de portabilidad:** `QUALIFY` es una cláusula **exclusiva de Snowflake** y no existe en SQL estándar ni en motores como PostgreSQL o MySQL. Es una ventaja de la plataforma que mejora la legibilidad, pero si necesitas portar este código a otro motor, deberás usar la versión con subconsulta del paso 3.2.

**3.4** Construye la consulta de deduplicación: obtén **solo el registro canónico** de cada `CLIENTE_ID` (el que se conservaría tras una limpieza):

```sql
-- Dataset deduplicado: un registro por CLIENTE_ID (el más antiguo)
SELECT
    CLIENTE_ID,
    NOMBRE,
    EMAIL,
    FECHA_REGISTRO
FROM CLIENTES_DIRTY
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY CLIENTE_ID
    ORDER BY FECHA_REGISTRO ASC
) = 1
ORDER BY CLIENTE_ID;
```

**3.5** Verifica que el resultado deduplicado tiene el número correcto de filas. Debería coincidir aproximadamente con la tabla limpia `CLIENTES`:

```sql
-- Comparar conteos: deduplicado vs. tabla limpia
SELECT
    'CLIENTES_DIRTY deduplicada' AS origen,
    COUNT(*)                     AS total_registros
FROM (
    SELECT CLIENTE_ID
    FROM CLIENTES_DIRTY
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY CLIENTE_ID
        ORDER BY FECHA_REGISTRO ASC
    ) = 1
)

UNION ALL

SELECT
    'CLIENTES (limpia)' AS origen,
    COUNT(*)            AS total_registros
FROM CLIENTES;
```

**3.6** Aplica el mismo enfoque a `PEDIDOS_DIRTY`, esta vez usando como criterio de conservación el registro con `FECHA_PEDIDO` más reciente en caso de duplicado por `PEDIDO_ID`:

```sql
-- Deduplicación de PEDIDOS_DIRTY: conservar el registro más reciente
SELECT
    PEDIDO_ID,
    CLIENTE_ID,
    FECHA_PEDIDO,
    MONTO_TOTAL,
    ESTADO,
    ROW_NUMBER() OVER (
        PARTITION BY PEDIDO_ID
        ORDER BY FECHA_PEDIDO DESC
    ) AS rn
FROM PEDIDOS_DIRTY
QUALIFY rn = 1
ORDER BY PEDIDO_ID;
```

#### Resultado Esperado

- El paso 3.1 muestra todas las filas con su número de fila dentro del grupo. Los clientes sin duplicados tienen solo `rn = 1`; los duplicados tienen `rn = 2`, `rn = 3`, etc.
- Los pasos 3.2 y 3.3 deben devolver **exactamente el mismo número de filas** (verificación de equivalencia entre subconsulta y `QUALIFY`)
- El paso 3.5 debe mostrar conteos iguales o muy cercanos entre la tabla deduplicada y la tabla limpia
- El paso 3.6 produce un dataset de pedidos sin duplicados de `PEDIDO_ID`

#### Verificación

Confirma los siguientes puntos antes de continuar:

- [ ] El resultado del paso 3.2 y del paso 3.3 tienen el mismo número de filas
- [ ] La consulta del paso 3.4 devuelve exactamente 1 fila por cada `CLIENTE_ID` único
- [ ] El conteo del paso 3.5 muestra valores iguales o con diferencia explicable (clientes en la tabla dirty que no están en la limpia, o viceversa)

---

### Paso 4: Validación de Integridad Referencial

**Objetivo:** Detectar registros huérfanos usando `LEFT JOIN` con filtro `WHERE IS NULL`, identificando pedidos sin cliente válido y clientes sin ningún pedido asociado.

#### Instrucciones

**4.1** Encuentra pedidos en `PEDIDOS_DIRTY` cuyo `CLIENTE_ID` no existe en `CLIENTES_DIRTY` (pedidos huérfanos — referencias rotas):

```sql
-- Pedidos con CLIENTE_ID que no existe en CLIENTES_DIRTY
SELECT
    p.PEDIDO_ID,
    p.CLIENTE_ID        AS cliente_id_en_pedido,
    p.FECHA_PEDIDO,
    p.MONTO_TOTAL,
    c.CLIENTE_ID        AS cliente_id_en_clientes
FROM PEDIDOS_DIRTY p
LEFT JOIN CLIENTES_DIRTY c ON p.CLIENTE_ID = c.CLIENTE_ID
WHERE c.CLIENTE_ID IS NULL
ORDER BY p.PEDIDO_ID;
```

> 💡 **¿Por qué funciona este patrón?** El `LEFT JOIN` mantiene **todas** las filas de `PEDIDOS_DIRTY`, independientemente de si tienen coincidencia en `CLIENTES_DIRTY`. Cuando no hay coincidencia, las columnas de `CLIENTES_DIRTY` aparecen como `NULL`. Filtrar por `WHERE c.CLIENTE_ID IS NULL` selecciona precisamente esos pedidos sin cliente válido.

**4.2** Distingue entre pedidos huérfanos por referencia rota vs. pedidos con `CLIENTE_ID` nulo (son problemas diferentes):

```sql
-- Categorizar los pedidos problemáticos
SELECT
    CASE
        WHEN p.CLIENTE_ID IS NULL          THEN 'CLIENTE_ID nulo en pedido'
        WHEN c.CLIENTE_ID IS NULL          THEN 'CLIENTE_ID no existe en clientes'
        ELSE                                    'Referencia válida'
    END                                    AS tipo_problema,
    COUNT(*)                               AS cantidad
FROM PEDIDOS_DIRTY p
LEFT JOIN CLIENTES_DIRTY c ON p.CLIENTE_ID = c.CLIENTE_ID
GROUP BY tipo_problema
ORDER BY cantidad DESC;
```

**4.3** Encuentra clientes en `CLIENTES_DIRTY` que no tienen ningún pedido en `PEDIDOS_DIRTY` (clientes sin actividad — posibles registros fantasma o clientes nuevos sin compras):

```sql
-- Clientes sin ningún pedido asociado
SELECT
    c.CLIENTE_ID,
    c.NOMBRE,
    c.EMAIL,
    c.FECHA_REGISTRO,
    p.PEDIDO_ID         AS pedido_id_encontrado
FROM CLIENTES_DIRTY c
LEFT JOIN PEDIDOS_DIRTY p ON c.CLIENTE_ID = p.CLIENTE_ID
WHERE p.PEDIDO_ID IS NULL
ORDER BY c.FECHA_REGISTRO DESC;
```

**4.4** Valida contra las tablas limpias: ¿los pedidos huérfanos de `PEDIDOS_DIRTY` también son huérfanos en la tabla limpia `PEDIDOS`?

```sql
-- ¿Los pedidos huérfanos en DIRTY también son problemáticos en la tabla limpia?
SELECT
    p_dirty.PEDIDO_ID,
    p_dirty.CLIENTE_ID,
    CASE
        WHEN p_limpia.PEDIDO_ID IS NOT NULL THEN 'Existe en tabla limpia'
        ELSE                                     'No existe en tabla limpia'
    END AS estado_en_limpia
FROM (
    -- Subconsulta: pedidos huérfanos en DIRTY
    SELECT p.PEDIDO_ID, p.CLIENTE_ID
    FROM PEDIDOS_DIRTY p
    LEFT JOIN CLIENTES_DIRTY c ON p.CLIENTE_ID = c.CLIENTE_ID
    WHERE c.CLIENTE_ID IS NULL
      AND p.CLIENTE_ID IS NOT NULL
) p_dirty
LEFT JOIN PEDIDOS p_limpia ON p_dirty.PEDIDO_ID = p_limpia.PEDIDO_ID
ORDER BY p_dirty.PEDIDO_ID;
```

**4.5** Obtén un resumen cuantitativo de los problemas de integridad referencial encontrados:

```sql
-- Resumen de integridad referencial
SELECT 'Pedidos con CLIENTE_ID nulo'                     AS problema,
       COUNT(*)                                           AS cantidad
FROM PEDIDOS_DIRTY
WHERE CLIENTE_ID IS NULL

UNION ALL

SELECT 'Pedidos con referencia a cliente inexistente'     AS problema,
       COUNT(*)                                           AS cantidad
FROM PEDIDOS_DIRTY p
LEFT JOIN CLIENTES_DIRTY c ON p.CLIENTE_ID = c.CLIENTE_ID
WHERE c.CLIENTE_ID IS NULL
  AND p.CLIENTE_ID IS NOT NULL

UNION ALL

SELECT 'Clientes sin ningún pedido'                       AS problema,
       COUNT(*)                                           AS cantidad
FROM CLIENTES_DIRTY c
LEFT JOIN PEDIDOS_DIRTY p ON c.CLIENTE_ID = p.CLIENTE_ID
WHERE p.PEDIDO_ID IS NULL;
```

#### Resultado Esperado

- El paso 4.1 devuelve los pedidos cuyos `CLIENTE_ID` no existen en `CLIENTES_DIRTY`
- El paso 4.2 categoriza claramente cuántos problemas son de tipo "nulo" vs. "referencia rota" — son causas distintas que requieren soluciones distintas
- El paso 4.3 puede devolver clientes legítimos nuevos sin compras, o bien registros duplicados que quedaron sin pedidos asociados
- El paso 4.5 produce una tabla resumen de 3 filas con los conteos de cada tipo de problema

#### Verificación

- [ ] El paso 4.1 devuelve al menos 1 pedido huérfano (confirma que los datos dirty tienen el problema esperado)
- [ ] El paso 4.2 distingue correctamente entre nulos y referencias rotas (categorías mutuamente excluyentes)
- [ ] Los conteos del paso 4.5 son consistentes con los diagnósticos del Paso 1

---

### Paso 5: Construcción del Reporte de Calidad de Datos

**Objetivo:** Consolidar todos los hallazgos de los pasos anteriores en un único reporte de calidad de datos que pueda presentarse a stakeholders técnicos y de negocio.

#### Instrucciones

**5.1** Construye el reporte consolidado usando `UNION ALL` para agregar todas las dimensiones de calidad analizadas:

```sql
-- ============================================================
-- REPORTE DE CALIDAD DE DATOS - TABLAS DIRTY
-- Consolida: duplicados, nulos críticos e integridad referencial
-- ============================================================

-- DIMENSIÓN 1: Duplicados en CLIENTES_DIRTY
SELECT
    'CLIENTES_DIRTY'                                    AS tabla,
    'Duplicados por CLIENTE_ID'                         AS dimension_calidad,
    COUNT(DISTINCT CLIENTE_ID)                          AS claves_afectadas,
    SUM(cnt - 1)                                        AS registros_sobrantes
FROM (
    SELECT CLIENTE_ID, COUNT(*) AS cnt
    FROM CLIENTES_DIRTY
    GROUP BY CLIENTE_ID
    HAVING COUNT(*) > 1
) dup_clientes

UNION ALL

-- DIMENSIÓN 2: Duplicados en PEDIDOS_DIRTY
SELECT
    'PEDIDOS_DIRTY'                                     AS tabla,
    'Duplicados por PEDIDO_ID'                          AS dimension_calidad,
    COUNT(DISTINCT PEDIDO_ID)                           AS claves_afectadas,
    SUM(cnt - 1)                                        AS registros_sobrantes
FROM (
    SELECT PEDIDO_ID, COUNT(*) AS cnt
    FROM PEDIDOS_DIRTY
    GROUP BY PEDIDO_ID
    HAVING COUNT(*) > 1
) dup_pedidos

UNION ALL

-- DIMENSIÓN 3: Nulos en EMAIL de CLIENTES_DIRTY
SELECT
    'CLIENTES_DIRTY'                                    AS tabla,
    'Nulos en columna EMAIL'                            AS dimension_calidad,
    COUNT(*) - COUNT(EMAIL)                             AS claves_afectadas,
    COUNT(*) - COUNT(EMAIL)                             AS registros_sobrantes
FROM CLIENTES_DIRTY

UNION ALL

-- DIMENSIÓN 4: Nulos en CLIENTE_ID de PEDIDOS_DIRTY
SELECT
    'PEDIDOS_DIRTY'                                     AS tabla,
    'Nulos en columna CLIENTE_ID'                       AS dimension_calidad,
    COUNT(*) - COUNT(CLIENTE_ID)                        AS claves_afectadas,
    COUNT(*) - COUNT(CLIENTE_ID)                        AS registros_sobrantes
FROM PEDIDOS_DIRTY

UNION ALL

-- DIMENSIÓN 5: Pedidos con referencia a cliente inexistente
SELECT
    'PEDIDOS_DIRTY'                                     AS tabla,
    'Referencia a CLIENTE_ID inexistente'               AS dimension_calidad,
    COUNT(*)                                            AS claves_afectadas,
    COUNT(*)                                            AS registros_sobrantes
FROM PEDIDOS_DIRTY p
LEFT JOIN CLIENTES_DIRTY c ON p.CLIENTE_ID = c.CLIENTE_ID
WHERE c.CLIENTE_ID IS NULL
  AND p.CLIENTE_ID IS NOT NULL

ORDER BY tabla, dimension_calidad;
```

**5.2** Agrega una columna de severidad y porcentaje de impacto al reporte:

```sql
-- Reporte de calidad con severidad y porcentaje de impacto
WITH totales AS (
    SELECT
        (SELECT COUNT(*) FROM CLIENTES_DIRTY) AS total_clientes,
        (SELECT COUNT(*) FROM PEDIDOS_DIRTY)  AS total_pedidos
),

hallazgos AS (
    -- Duplicados CLIENTES
    SELECT 'CLIENTES_DIRTY' AS tabla, 'Duplicados por CLIENTE_ID' AS problema,
           SUM(cnt - 1) AS registros_afectados
    FROM (SELECT CLIENTE_ID, COUNT(*) AS cnt FROM CLIENTES_DIRTY
          GROUP BY CLIENTE_ID HAVING COUNT(*) > 1) d

    UNION ALL

    -- Duplicados PEDIDOS
    SELECT 'PEDIDOS_DIRTY', 'Duplicados por PEDIDO_ID',
           SUM(cnt - 1)
    FROM (SELECT PEDIDO_ID, COUNT(*) AS cnt FROM PEDIDOS_DIRTY
          GROUP BY PEDIDO_ID HAVING COUNT(*) > 1) d

    UNION ALL

    -- Nulos EMAIL en CLIENTES
    SELECT 'CLIENTES_DIRTY', 'Nulos en EMAIL',
           COUNT(*) - COUNT(EMAIL)
    FROM CLIENTES_DIRTY

    UNION ALL

    -- Nulos CLIENTE_ID en PEDIDOS
    SELECT 'PEDIDOS_DIRTY', 'Nulos en CLIENTE_ID',
           COUNT(*) - COUNT(CLIENTE_ID)
    FROM PEDIDOS_DIRTY

    UNION ALL

    -- Referencias rotas en PEDIDOS
    SELECT 'PEDIDOS_DIRTY', 'Referencia a CLIENTE_ID inexistente',
           COUNT(*)
    FROM PEDIDOS_DIRTY p
    LEFT JOIN CLIENTES_DIRTY c ON p.CLIENTE_ID = c.CLIENTE_ID
    WHERE c.CLIENTE_ID IS NULL AND p.CLIENTE_ID IS NOT NULL
)

SELECT
    h.tabla,
    h.problema,
    h.registros_afectados,
    CASE h.tabla
        WHEN 'CLIENTES_DIRTY' THEN t.total_clientes
        ELSE t.total_pedidos
    END                                                         AS total_tabla,
    ROUND(
        h.registros_afectados * 100.0 /
        CASE h.tabla
            WHEN 'CLIENTES_DIRTY' THEN t.total_clientes
            ELSE t.total_pedidos
        END, 2
    )                                                           AS pct_impacto,
    CASE
        WHEN h.registros_afectados * 100.0 /
             CASE h.tabla WHEN 'CLIENTES_DIRTY'
                 THEN t.total_clientes ELSE t.total_pedidos END > 10
            THEN '🔴 CRÍTICO'
        WHEN h.registros_afectados * 100.0 /
             CASE h.tabla WHEN 'CLIENTES_DIRTY'
                 THEN t.total_clientes ELSE t.total_pedidos END > 3
            THEN '🟡 MODERADO'
        ELSE '🟢 BAJO'
    END                                                         AS severidad
FROM hallazgos h
CROSS JOIN totales t
ORDER BY pct_impacto DESC;
```

#### Resultado Esperado

El reporte final debe mostrar una tabla con las siguientes columnas:

| tabla | problema | registros_afectados | total_tabla | pct_impacto | severidad |
|---|---|---|---|---|---|
| PEDIDOS_DIRTY | Duplicados por PEDIDO_ID | _N_ | _N_ | _N%_ | 🔴/🟡/🟢 |
| CLIENTES_DIRTY | Duplicados por CLIENTE_ID | _N_ | _N_ | _N%_ | 🔴/🟡/🟢 |
| PEDIDOS_DIRTY | Nulos en CLIENTE_ID | _N_ | _N_ | _N%_ | 🔴/🟡/🟢 |
| ... | ... | ... | ... | ... | ... |

Los valores exactos dependerán de los datos generados por el script de setup. Lo importante es que:
- Todos los problemas conocidos aparecen en el reporte
- Los porcentajes son coherentes con los conteos individuales de pasos anteriores
- La columna de severidad clasifica correctamente según los umbrales definidos

#### Verificación

- [ ] El reporte muestra exactamente 5 filas (una por cada dimensión de calidad analizada)
- [ ] La suma de `registros_afectados` de las dimensiones de duplicados coincide con los valores calculados en el Paso 2
- [ ] Los porcentajes son coherentes: `registros_afectados / total_tabla * 100`

---

## Validación y Pruebas Finales

Una vez completados todos los pasos, ejecuta las siguientes consultas de validación para confirmar que tus resultados son consistentes:

### Prueba de Consistencia 1: Verificar la Equivalencia entre Subconsulta y QUALIFY

```sql
-- Ambas consultas deben devolver el mismo número de filas
-- Método 1: Subconsulta
SELECT COUNT(*) AS metodo_subconsulta
FROM (
    SELECT
        CLIENTE_ID,
        ROW_NUMBER() OVER (PARTITION BY CLIENTE_ID ORDER BY FECHA_REGISTRO ASC) AS rn
    FROM CLIENTES_DIRTY
) t
WHERE rn > 1;

-- Método 2: QUALIFY
SELECT COUNT(*) AS metodo_qualify
FROM CLIENTES_DIRTY
WHERE 1=1
QUALIFY ROW_NUMBER() OVER (PARTITION BY CLIENTE_ID ORDER BY FECHA_REGISTRO ASC) > 1;
```

**Resultado esperado:** Ambos conteos deben ser **idénticos**.

### Prueba de Consistencia 2: Verificar que la Deduplicación no Pierde Clientes Únicos

```sql
-- Los clientes que NO tienen duplicados deben aparecer en ambos datasets
SELECT COUNT(*) AS clientes_sin_duplicados_original
FROM CLIENTES_DIRTY
WHERE CLIENTE_ID NOT IN (
    SELECT CLIENTE_ID
    FROM CLIENTES_DIRTY
    GROUP BY CLIENTE_ID
    HAVING COUNT(*) > 1
);

-- El mismo conteo debe aparecer en el dataset deduplicado
-- (porque esos registros tienen rn = 1 por defecto)
```

### Prueba de Consistencia 3: Validar el Reporte Final

```sql
-- La suma de registros_afectados de duplicados debe coincidir
-- con los cálculos individuales del Paso 2
SELECT
    'Duplicados CLIENTES (Paso 2.3)' AS origen,
    SUM(veces_registrado - 1)        AS total
FROM (
    SELECT CLIENTE_ID, COUNT(*) AS veces_registrado
    FROM CLIENTES_DIRTY
    GROUP BY CLIENTE_ID
    HAVING COUNT(*) > 1
)

UNION ALL

SELECT
    'Duplicados PEDIDOS (Paso 2.4)' AS origen,
    SUM(veces_registrado - 1)       AS total
FROM (
    SELECT PEDIDO_ID, COUNT(*) AS veces_registrado
    FROM PEDIDOS_DIRTY
    GROUP BY PEDIDO_ID
    HAVING COUNT(*) > 1
);
```

**Resultado esperado:** Los valores deben coincidir exactamente con los obtenidos en el Paso 2 y con los del reporte del Paso 5.

---

## Resolución de Problemas

### Problema 1: Error "Object does not exist" al consultar CLIENTES_DIRTY o PEDIDOS_DIRTY

**Síntoma:** Al ejecutar cualquier consulta del laboratorio, Snowflake devuelve el error:
```
SQL compilation error: Object 'LAB_SQL_INTERMEDIO.VENTAS.CLIENTES_DIRTY' does not exist or not authorized.
```

**Causa:** El script de setup `00_setup_laboratorios.sql` no fue ejecutado antes del laboratorio, o fue ejecutado en un database/schema diferente al esperado. También puede ocurrir si el rol activo no tiene permisos `SELECT` sobre las tablas del schema `VENTAS`.

**Solución:**

1. Verifica el contexto activo:
   ```sql
   SELECT CURRENT_DATABASE(), CURRENT_SCHEMA(), CURRENT_ROLE();
   ```
2. Si el database o schema no es correcto, ejecuta los comandos de configuración inicial del Paso 0:
   ```sql
   USE DATABASE LAB_SQL_INTERMEDIO;
   USE SCHEMA VENTAS;
   ```
3. Si el problema persiste, verifica que las tablas existen en el schema:
   ```sql
   SHOW TABLES IN SCHEMA LAB_SQL_INTERMEDIO.VENTAS;
   ```
4. Si `CLIENTES_DIRTY` no aparece en la lista, contacta al instructor para que ejecute el script de setup. No es posible continuar el laboratorio sin este paso previo.

---

### Problema 2: QUALIFY no es reconocida y genera error de sintaxis

**Síntoma:** Al ejecutar una consulta con `QUALIFY`, Snowflake devuelve:
```
SQL compilation error: syntax error line X at position Y unexpected 'QUALIFY'.
```
O bien, el editor de Snowsight marca `QUALIFY` como palabra no reconocida.

**Causa:** Esto ocurre cuando la sesión está usando una versión de compatibilidad de SQL que no soporta `QUALIFY`, o cuando el estudiante está intentando ejecutar la consulta en otro motor (por ejemplo, en una instancia de PostgreSQL o MySQL que no tiene esta cláusula). También puede ocurrir si hay un error de escritura en la palabra clave.

**Solución:**

1. Verifica que estás en Snowflake y no en otro motor:
   ```sql
   SELECT CURRENT_ACCOUNT(), CURRENT_REGION();
   -- Si esto falla, no estás en Snowflake
   ```
2. Verifica la ortografía: `QUALIFY` (no `QUALIFIY`, `QUALIFY:`, ni `QUALIFY;` a mitad de consulta)
3. Si confirmas que estás en Snowflake y el error persiste, usa la alternativa con subconsulta (Paso 3.2) que es funcionalmente equivalente:
   ```sql
   -- Reemplaza cualquier uso de QUALIFY con esta estructura:
   SELECT * FROM (
       SELECT columnas,
              ROW_NUMBER() OVER (...) AS rn
       FROM tabla
   ) t
   WHERE rn = 1;  -- o rn > 1, según el caso
   ```
4. Recuerda: `QUALIFY` es **exclusivo de Snowflake**. Si en el futuro necesitas portar este código a PostgreSQL, BigQuery estándar o MySQL, deberás usar siempre la versión con subconsulta.

---

## Limpieza del Entorno

Al finalizar el laboratorio, ejecuta los siguientes comandos para liberar recursos y evitar consumo innecesario de créditos en tu cuenta trial:

```sql
-- Suspender el warehouse para detener el consumo de créditos
-- IMPORTANTE: Ejecutar siempre al terminar la sesión
ALTER WAREHOUSE LAB_WH SUSPEND;
```

> ✅ **Verificación de suspensión:** En Snowsight, navega a **Admin > Warehouses** y confirma que `LAB_WH` muestra el estado **Suspended**. Las cuentas trial tienen 400 USD de créditos; un warehouse X-SMALL consume aproximadamente 1 crédito por hora de uso activo.

Las tablas `CLIENTES_DIRTY` y `PEDIDOS_DIRTY` **no deben modificarse ni eliminarse**, ya que son compartidas entre todos los estudiantes del curso y son necesarias para los laboratorios posteriores. Este laboratorio es de solo lectura (`SELECT`).

```sql
-- Confirmación: verificar que no realizaste cambios (solo lectura)
-- Esta consulta debe ejecutarse sin errores si no modificaste nada
SELECT 'Laboratorio completado en modo solo lectura' AS estado;
```

---

## Resumen

### Lo que Aprendiste en este Laboratorio

En este laboratorio aplicaste tres técnicas fundamentales de auditoría y calidad de datos en Snowflake:

| Técnica | Cuándo usarla | Limitación |
|---|---|---|
| `GROUP BY` + `HAVING COUNT(*) > 1` | Detectar qué claves tienen duplicados y cuántos | No indica cuál registro conservar |
| `ROW_NUMBER() OVER(PARTITION BY ...)` | Numerar duplicados para seleccionar el canónico | Requiere definir un criterio de ordenación |
| `QUALIFY` (Snowflake) | Filtrar resultados de window functions sin subconsulta | Exclusivo de Snowflake, no portable |
| `LEFT JOIN` + `WHERE IS NULL` | Detectar registros huérfanos (referencias rotas) | Requiere identificar la tabla "padre" correcta |

### Conceptos Clave para Recordar

- **`HAVING` vs. `WHERE`:** `WHERE` filtra filas individuales antes del agrupamiento; `HAVING` filtra grupos después de aplicar funciones de agregación. Para trabajar con `COUNT()`, siempre necesitas `HAVING`.
- **`COUNT(*) vs. COUNT(columna)`:** `COUNT(*)` cuenta todas las filas incluyendo nulos; `COUNT(columna)` solo cuenta filas donde esa columna no es nula. La diferencia entre ambos revela los nulos.
- **`ROW_NUMBER()` como puente:** Esta función de ventana es la introducción formal a las window functions. En el Laboratorio 4 explorarás `RANK()`, `DENSE_RANK()`, `LAG()` y `LEAD()` con mayor profundidad.
- **`QUALIFY` es una ventaja de Snowflake:** Mejora la legibilidad al evitar subconsultas, pero recuerda que no es SQL estándar. Siempre documenta su uso si compartes código con equipos que usan otros motores.

### Conexión con los Próximos Laboratorios

```
Lab 01-02 (Fundamentos)
       ↓
Lab 03 (Este laboratorio)
  ├── GROUP BY + HAVING → técnica base de auditoría
  ├── ROW_NUMBER() → introducción a window functions
  └── LEFT JOIN + IS NULL → integridad referencial
       ↓
Lab 04 → Window functions avanzadas (RANK, DENSE_RANK, LAG, LEAD)
       ↓
Lab 05 → Series temporales con window functions
```

### Recursos de Referencia

| Recurso | URL |
|---|---|
| Documentación Snowflake: GROUP BY | https://docs.snowflake.com/en/sql-reference/constructs/group-by |
| Documentación Snowflake: HAVING | https://docs.snowflake.com/en/sql-reference/constructs/having |
| Documentación Snowflake: ROW_NUMBER | https://docs.snowflake.com/en/sql-reference/functions/row_number |
| Documentación Snowflake: QUALIFY | https://docs.snowflake.com/en/sql-reference/constructs/qualify |
| Documentación Snowflake: Window Functions | https://docs.snowflake.com/en/sql-reference/functions-analytic |

---

> 📌 **Nota para el instructor:** Si el grupo termina antes de los 60 minutos, propón como extensión que los estudiantes intenten construir una vista (`CREATE OR REPLACE VIEW`) que consolide el reporte de calidad del Paso 5.2, de forma que pueda ser consultada en cualquier momento sin re-ejecutar la lógica. Esto refuerza el concepto de reutilización de consultas complejas y conecta con las buenas prácticas de escritura SQL del objetivo del curso.
