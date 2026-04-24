# 09 — Troubleshooting

> Casos reales organizados por síntoma. Cada caso tiene: síntoma → diagnóstico → solución.

---

## 🔴 CASO 1: Query que no termina nunca

**Síntoma:** Una query lleva horas corriendo y normalmente tarda minutos.

### Paso 1 — Verificar que sigue activa
```sql
SELECT
    SessionNo,
    UserName,
    StartTime,
    CAST(CURRENT_TIMESTAMP - StartTime AS INTERVAL MINUTE(4)) AS minutos
FROM DBC.QryLogV
WHERE ErrorCode = 0
  AND StartTime < CURRENT_TIMESTAMP - INTERVAL '60' MINUTE
ORDER BY minutos DESC;
```

### Paso 2 — Ver el EXPLAIN de la query
Pídele al usuario que la ejecute con EXPLAIN primero y busca:
- `redistributing rows` en tablas grandes
- `Confidence: none` (sin estadísticas)
- `duplicating rows` en una tabla grande

### Paso 3 — Ver si hay lock esperando
```sql
SELECT
    DatabaseName, TableName, LockType, LockStatus, BlockingSessionNo
FROM DBC.LockLogShredV
WHERE SessionNo = <session_number>;
```

### Solución según causa
| Causa | Solución |
|-------|----------|
| Sin estadísticas | `COLLECT STATISTICS ON tabla COLUMN (col)` |
| Lock esperando | Identificar y liberar la sesión bloqueante |
| Skew | Evaluar cambio de PI o agregar PPI |
| Spool lleno | Ampliar spool del usuario o reescribir query |

---

## 🔴 CASO 2: Espacio lleno — "No more room in database"

**Síntoma:** Error `2646 - No more spool space` o `2664 - No more room in database`.

### Error 2646 — Spool lleno
```sql
-- Ver spool asignado vs usado por el usuario
SELECT
    UserName,
    SpoolSpace / 1024 / 1024 / 1024 AS spool_max_gb
FROM DBC.UsersV
WHERE UserName = 'nombre_usuario';

-- Ampliar temporalmente
MODIFY USER nombre_usuario AS SPOOL = 100000000000; -- 100 GB
```

### Error 2664 — Perm space lleno
```sql
-- Identificar qué base de datos está llena
SELECT
    DatabaseName,
    SUM(CurrentPerm) / 1024 / 1024 / 1024 AS used_gb,
    SUM(MaxPerm) / 1024 / 1024 / 1024 AS max_gb,
    CAST(SUM(CurrentPerm) * 100.0 / NULLIFZERO(SUM(MaxPerm)) AS DECIMAL(5,2)) AS pct
FROM DBC.DiskSpaceV
GROUP BY 1
HAVING pct > 90
ORDER BY pct DESC;

-- Ampliar perm space
MODIFY DATABASE nombre_base AS PERM = 200000000000; -- 200 GB
```
> ⚠️ Antes de ampliar, identifica por qué se llenó. Puede ser una tabla que creció sin control.

---

## 🔴 CASO 3: Sistema lento — todos se quejan al mismo tiempo

**Síntoma:** Múltiples usuarios reportan lentitud simultánea.

### Paso 1 — Ver carga de AMPs
```sql
SELECT
    NodeID,
    AMPNumber,
    CPUTime,
    IOCount
FROM DBC.AMPUsage
ORDER BY CPUTime DESC;
```
> Si 1-2 AMPs tienen CPU 10x mayor que el resto → skew en alguna query activa.

### Paso 2 — Identificar la query culpable
```sql
SELECT TOP 5
    UserName,
    SessionNo,
    AMPCPUTime,
    TotalIOCount,
    TRIM(SUBSTR(QueryText, 1, 300)) AS query
FROM DBC.QryLogV
WHERE CAST(StartTime AS DATE) = CURRENT_DATE
ORDER BY AMPCPUTime DESC;
```

### Paso 3 — Ver si hay workload saturado
```sql
SELECT
    WorkloadName,
    ActiveRequests,
    QueuedRequests
FROM DBC.WorkloadStatusV
ORDER BY QueuedRequests DESC;
```

### Paso 4 — Acción de emergencia (abortar sesión problemática)
```sql
-- Primero identificar la sesión
SELECT SessionNo, UserName FROM DBC.SessionInfoV WHERE UserName = 'usuario_problematico';

-- Abortar la sesión
ABORT SESSION <session_number>;
```

---

## 🟡 CASO 4: FastLoad falla a mitad de la carga

**Síntoma:** Job de FastLoad termina con error. La tabla queda en estado "locked".

### Diagnóstico
```sql
-- Ver si la tabla tiene locks residuales
SELECT *
FROM DBC.LockLogShredV
WHERE TableName = 'nombre_tabla';

-- Ver las tablas de error
SELECT COUNT(*), ErrorCode
FROM nombre_base.ET_nombre_tabla
GROUP BY 2;
```

### Solución
```sql
-- 1. Liberar los locks del FastLoad fallido
RELEASE MLOAD nombre_base.nombre_tabla;

-- 2. Limpiar tablas de error
DROP TABLE nombre_base.ET_nombre_tabla;
DROP TABLE nombre_base.UV_nombre_tabla;

-- 3. Vaciar la tabla destino
DELETE FROM nombre_base.nombre_tabla ALL;

-- 4. Volver a ejecutar el FastLoad
```

---

## 🟡 CASO 5: Estadísticas desactualizadas causan plan de ejecución malo

**Síntoma:** Una query que antes era rápida ahora tarda mucho. El EXPLAIN muestra `Confidence: low` o `Confidence: none`.

### Diagnóstico
```sql
-- Ver cuándo se colectaron las stats de la tabla
SELECT
    ColumnName,
    CollectTimeStamp,
    RowCount,
    UniqueValueCount
FROM DBC.StatsV
WHERE DatabaseName = 'nombre_base'
  AND TableName = 'nombre_tabla'
ORDER BY CollectTimeStamp;
```

### Solución
```sql
-- Refrescar stats en columnas del WHERE y JOIN
COLLECT STATISTICS ON nombre_base.nombre_tabla COLUMN (columna_filtro);
COLLECT STATISTICS ON nombre_base.nombre_tabla COLUMN (columna_join);
COLLECT STATISTICS ON nombre_base.nombre_tabla INDEX (columna_pi);

-- Volver a ejecutar el EXPLAIN para verificar que mejoró
EXPLAIN SELECT ...;
```

---

## 🟡 CASO 6: Usuario no puede conectarse

**Síntoma:** Error de autenticación o "max sessions exceeded".

### Diagnóstico
```sql
-- Ver si el usuario existe
SELECT UserName, CreateTimeStamp, LastAccessTimeStamp
FROM DBC.UsersV
WHERE UserName = 'nombre_usuario';

-- Ver cuántas sesiones activas tiene
SELECT COUNT(*) AS sesiones_activas
FROM DBC.SessionInfoV
WHERE UserName = 'nombre_usuario';

-- Ver el límite de sesiones del sistema
SELECT InfoKey, InfoData
FROM DBC.DBCInfoV
WHERE InfoKey LIKE '%SESSION%';
```

### Solución según causa
| Causa | Solución |
|-------|----------|
| Contraseña expirada | `MODIFY USER nombre_usuario AS PASSWORD = 'nueva';` |
| Demasiadas sesiones abiertas | Identificar sesiones colgadas y abortarlas |
| Usuario bloqueado (perfil) | Revisar perfil y resetear con `MODIFY USER` |
| No tiene acceso a la base | `GRANT SELECT ON nombre_base TO nombre_usuario;` |

---

## 🟢 CASO 7: Tabla que creció inesperadamente

**Síntoma:** El espacio de una base de datos se llenó y no se sabe por qué.

### Diagnóstico
```sql
-- Ver crecimiento reciente (tablas más grandes)
SELECT TOP 20
    DatabaseName,
    TableName,
    SUM(CurrentPerm) / 1024 / 1024 AS perm_mb
FROM DBC.TableSizeV
WHERE DatabaseName = 'nombre_base'
GROUP BY 1, 2
ORDER BY perm_mb DESC;

-- Ver si hay tablas temporales o de error olvidadas
SELECT
    TableName,
    TableKind,
    CreateTimeStamp
FROM DBC.TablesV
WHERE DatabaseName = 'nombre_base'
  AND (TableName LIKE 'ET_%'
    OR TableName LIKE 'UV_%'
    OR TableName LIKE 'TMP_%'
    OR TableName LIKE 'TEMP_%')
ORDER BY CreateTimeStamp DESC;
```

### Solución
```sql
-- Limpiar tablas temporales o de error que ya no se necesiten
DROP TABLE nombre_base.TMP_tabla_vieja;

-- Si hay datos duplicados por carga fallida
DELETE FROM nombre_base.nombre_tabla WHERE fecha_carga = DATE '2024-01-15';
```

---

## 🔵 Referencia rápida de errores comunes

| Código | Mensaje | Causa más común |
|--------|---------|-----------------|
| 2646 | No more spool space | Query genera demasiados datos intermedios |
| 2664 | No more room in database | Perm space lleno |
| 3523 | The user does not have... | Falta de privilegio |
| 3706 | Syntax error | Error en el SQL |
| 3807 | Object does not exist | Tabla o base de datos no existe |
| 3928 | The maximum number of sessions | Demasiadas sesiones abiertas |
| 5628 | COLLECT STATISTICS not allowed | Stats en tabla sin datos o vista |
| 2673 | The source parcel length | Tipos de datos no coinciden en carga |

---

## Checklist de troubleshooting

- [ ] ¿Tengo el session number o el texto de la query?
- [ ] ¿Revisé los locks antes de intervenir?
- [ ] ¿Corrí EXPLAIN antes de ejecutar la solución?
- [ ] ¿Documenté el caso (qué pasó, causa raíz, solución)?
- [ ] ¿Notifiqué al usuario el resultado?

---

> Este es el último módulo de contenido.  
> Si tienes un caso real que no está acá, abre un [Issue](../../issues) o PR.
