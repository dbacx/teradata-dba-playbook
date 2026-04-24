# 02 — Performance & EXPLAIN

## ¿Por qué importa?

El 80% de los problemas de performance en Teradata tienen la misma causa raíz:
- PI mal elegido (skew)
- Estadísticas desactualizadas o faltantes
- Full Table Scan innecesario
- Producto cartesiano accidental

El `EXPLAIN` te dice exactamente qué va a hacer el optimizador **antes** de ejecutar la query.

---

## EXPLAIN — Cómo leerlo

```sql
EXPLAIN
SELECT *
FROM ventas.fact_orders o
JOIN ventas.dim_customer c ON o.customer_id = c.customer_id
WHERE o.order_date >= DATE '2024-01-01';
```

### Frases clave en el EXPLAIN y qué significan

| Frase en EXPLAIN | Qué significa | ¿Problema? |
|-----------------|---------------|------------|
| `we do an all-AMPs retrieve` | Full Table Scan | Depende del tamaño |
| `we do a single-AMP retrieve` | Acceso por PI exacto | ✅ Óptimo |
| `we do an all-AMPs JOIN` | Join sin redistribución por PI | Costoso en tablas grandes |
| `redistributing rows` | Los datos se mueven entre AMPs | Costoso en red |
| `duplicating rows` | Se replica tabla en todos los AMPs | Solo aceptable en tablas pequeñas |
| `with no residual conditions` | No hay filtro — lee todo | ⚠️ Revisar WHERE |
| `Confidence: none` | Sin estadísticas para esa columna | ⚠️ Colectar stats |
| `Confidence: low` | Stats desactualizadas | ⚠️ Refrescar stats |
| `Confidence: high` | Stats actualizadas y confiables | ✅ |

---

## Estadísticas

### Ver qué estadísticas tiene una tabla
```sql
SELECT
    DatabaseName,
    TableName,
    ColumnName,
    CollectTimeStamp,
    RowCount,
    UniqueValueCount
FROM DBC.StatsV
WHERE DatabaseName = 'nombre_base'
  AND TableName = 'nombre_tabla'
ORDER BY CollectTimeStamp DESC;
```
> **Qué validar:** Stats con más de 7 días en tablas de carga diaria → refrescar.

### Colectar estadísticas
```sql
-- En columna de join o filtro frecuente
COLLECT STATISTICS ON nombre_base.nombre_tabla COLUMN (columna);

-- En el Primary Index
COLLECT STATISTICS ON nombre_base.nombre_tabla INDEX (columna_pi);

-- En múltiples columnas (cuando el WHERE combina dos columnas)
COLLECT STATISTICS ON nombre_base.nombre_tabla COLUMN (col1, col2);
```

### Refrescar estadísticas de toda una tabla
```sql
COLLECT STATISTICS ON nombre_base.nombre_tabla;
```
> ⚠️ En tablas muy grandes puede ser costoso. Hazlo en ventana de mantenimiento.

---

## Skew

### Detectar skew en tablas grandes
```sql
SELECT
    DatabaseName,
    TableName,
    MAX(CurrentPerm) AS max_amp_bytes,
    AVG(CurrentPerm) AS avg_amp_bytes,
    (MAX(CurrentPerm) / NULLIFZERO(AVG(CurrentPerm))) AS skew_ratio
FROM DBC.TableSizeV
WHERE DatabaseName = 'nombre_base'
GROUP BY 1, 2
HAVING skew_ratio > 1.5
ORDER BY skew_ratio DESC;
```
> **Qué validar:**
> - `skew_ratio` 1.0–1.5 → aceptable
> - 1.5–3.0 → monitorear
> - > 3.0 → problema real, revisar PI

---

## Queries lentas — diagnóstico activo

### Ver queries que están corriendo ahora
```sql
SELECT
    s.UserName,
    s.SessionNo,
    q.QueryText,
    q.StartTime,
    (CAST(CURRENT_TIMESTAMP AS FLOAT) - CAST(q.StartTime AS FLOAT)) * 86400 AS segundos_corriendo
FROM DBC.SessionInfoV s
JOIN DBC.QryLogV q ON s.SessionNo = q.SessionNo
WHERE q.StatementType = 'Select'
ORDER BY segundos_corriendo DESC;
```

### Ver las 10 queries más costosas del día (requiere DBQL activo)
```sql
SELECT TOP 10
    UserName,
    StartTime,
    AMPCPUTime,
    TotalIOCount,
    SpoolUsage / 1024 / 1024 AS spool_mb,
    TRIM(QueryText) AS query
FROM DBC.QryLogV
WHERE CAST(StartTime AS DATE) = CURRENT_DATE
ORDER BY AMPCPUTime DESC;
```
> **Qué validar:** Queries con `spool_mb` alto + `AMPCPUTime` alto suelen ser producto cartesiano o full scan sin filtro útil.

### Ver queries que hicieron más I/O
```sql
SELECT TOP 10
    UserName,
    TotalIOCount,
    AMPCPUTime,
    SpoolUsage / 1024 / 1024 AS spool_mb,
    CAST(StartTime AS DATE) AS fecha,
    TRIM(SUBSTR(QueryText, 1, 200)) AS query_inicio
FROM DBC.QryLogV
WHERE CAST(StartTime AS DATE) >= CURRENT_DATE - 7
ORDER BY TotalIOCount DESC;
```

---

## Índices secundarios (SI)

### Ver índices de una tabla
```sql
SELECT
    IndexNumber,
    IndexType,
    ColumnName,
    UniqueFlag
FROM DBC.IndicesV
WHERE DatabaseName = 'nombre_base'
  AND TableName = 'nombre_tabla'
ORDER BY IndexNumber, ColumnPosition;
```

> **Tipos de índice:**
> - `P` = Primary Index
> - `S` = Secondary Index (non-unique)
> - `U` = Unique Secondary Index (USI)
> - `J` = Join Index

---

## Checklist de performance ante una query lenta

- [ ] Correr `EXPLAIN` y buscar `redistributing rows` o `Confidence: none`
- [ ] Verificar stats en columnas del WHERE y JOIN (`DBC.StatsV`)
- [ ] Revisar skew de las tablas involucradas
- [ ] Ver si el filtro del WHERE aplica sobre el PI o requiere full scan
- [ ] Revisar spool usage en DBQL — si es muy alto, posible producto cartesiano
- [ ] Comparar el plan con una query similar que sí sea rápida

---

## Siguiente módulo

→ [03 — Space Management](../03_space_management/README.md)
