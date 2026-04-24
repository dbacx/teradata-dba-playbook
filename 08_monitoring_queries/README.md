# 08 — Queries de Monitoreo

> Queries listas para copiar y ejecutar. Organizadas por categoría.  
> Requieren acceso de lectura a vistas `DBC.*` y `DBC.QryLogV` (DBQL activo).

---

## 🟢 Sistema General

### Estado general del sistema
```sql
SELECT
    InfoKey,
    InfoData
FROM DBC.DBCInfoV
ORDER BY InfoKey;
```

### Versión de Teradata
```sql
SELECT InfoData
FROM DBC.DBCInfoV
WHERE InfoKey = 'VERSION';
```

### Nodos activos
```sql
SELECT
    NodeID,
    NodeType,
    ProcessorCount,
    PhysicalMemoryMB,
    IsActive
FROM DBC.NodeInfoV
ORDER BY NodeID;
```

---

## 🔵 Sesiones

### Sesiones activas en este momento
```sql
SELECT
    UserName,
    SessionNo,
    ClientSystemName,
    ClientProgramName,
    LogonTime,
    DefaultDatabase
FROM DBC.SessionInfoV
ORDER BY LogonTime;
```

### Cuántas sesiones hay por usuario
```sql
SELECT
    UserName,
    COUNT(*) AS sesiones
FROM DBC.SessionInfoV
GROUP BY 1
ORDER BY sesiones DESC;
```

### Sesiones con más tiempo logoneadas
```sql
SELECT
    UserName,
    SessionNo,
    LogonTime,
    CAST(CURRENT_TIMESTAMP - LogonTime AS INTERVAL HOUR(4)) AS horas_conectado
FROM DBC.SessionInfoV
ORDER BY horas_conectado DESC;
```

### Usuarios conectados desde IPs o apps específicas
```sql
SELECT DISTINCT
    UserName,
    ClientSystemName,
    ClientProgramName
FROM DBC.SessionInfoV
ORDER BY UserName;
```

---

## 🟡 Queries Activas

### Queries que están corriendo ahora mismo
```sql
SELECT
    s.UserName,
    s.SessionNo,
    s.ClientProgramName,
    TRIM(q.QueryText) AS query_text,
    q.StartTime,
    CAST(CURRENT_TIMESTAMP - q.StartTime AS INTERVAL MINUTE(4)) AS minutos_corriendo
FROM DBC.SessionInfoV s
JOIN DBC.QryLogV q ON s.SessionNo = q.SessionNo
WHERE q.ErrorCode = 0
ORDER BY minutos_corriendo DESC;
```

### Queries con más de 30 minutos corriendo
```sql
SELECT
    UserName,
    SessionNo,
    StartTime,
    CAST(CURRENT_TIMESTAMP - StartTime AS INTERVAL MINUTE(4)) AS minutos,
    TRIM(SUBSTR(QueryText, 1, 300)) AS query_inicio
FROM DBC.QryLogV
WHERE StartTime < CURRENT_TIMESTAMP - INTERVAL '30' MINUTE
  AND ErrorCode = 0
ORDER BY minutos DESC;
```

### Las 10 queries más pesadas de hoy (CPU)
```sql
SELECT TOP 10
    UserName,
    CAST(StartTime AS TIME(0)) AS hora,
    AMPCPUTime,
    TotalIOCount,
    SpoolUsage / 1024 / 1024 AS spool_mb,
    TRIM(SUBSTR(QueryText, 1, 200)) AS query_inicio
FROM DBC.QryLogV
WHERE CAST(StartTime AS DATE) = CURRENT_DATE
ORDER BY AMPCPUTime DESC;
```

### Las 10 queries con más spool de hoy
```sql
SELECT TOP 10
    UserName,
    CAST(StartTime AS TIME(0)) AS hora,
    SpoolUsage / 1024 / 1024 / 1024 AS spool_gb,
    AMPCPUTime,
    TRIM(SUBSTR(QueryText, 1, 200)) AS query_inicio
FROM DBC.QryLogV
WHERE CAST(StartTime AS DATE) = CURRENT_DATE
  AND SpoolUsage > 0
ORDER BY SpoolUsage DESC;
```

### Queries que fallaron hoy
```sql
SELECT
    UserName,
    CAST(StartTime AS TIME(0)) AS hora,
    ErrorCode,
    ErrorText,
    TRIM(SUBSTR(QueryText, 1, 300)) AS query_inicio
FROM DBC.QryLogV
WHERE CAST(StartTime AS DATE) = CURRENT_DATE
  AND ErrorCode <> 0
ORDER BY StartTime DESC;
```

---

## 🔴 Locks

### Ver todos los locks activos
```sql
SELECT
    DatabaseName,
    TableName,
    LockType,
    LockStatus,
    SessionNo,
    BlockingSessionNo
FROM DBC.LockLogShredV
ORDER BY LockStatus, DatabaseName, TableName;
```

### Ver qué sesión está bloqueando a otras
```sql
SELECT
    BlockingSessionNo,
    COUNT(*) AS sesiones_bloqueadas
FROM DBC.LockLogShredV
WHERE BlockingSessionNo IS NOT NULL
GROUP BY 1
ORDER BY sesiones_bloqueadas DESC;
```

### Ver locks por tabla específica
```sql
SELECT
    LockType,
    LockStatus,
    SessionNo,
    BlockingSessionNo
FROM DBC.LockLogShredV
WHERE DatabaseName = 'nombre_base'
  AND TableName = 'nombre_tabla';
```

---

## 🟠 Espacio

### Uso de espacio por base de datos (resumen)
```sql
SELECT
    DatabaseName,
    SUM(CurrentPerm) / 1024 / 1024 / 1024 AS used_gb,
    SUM(MaxPerm) / 1024 / 1024 / 1024     AS max_gb,
    CAST(SUM(CurrentPerm) * 100.0 / NULLIFZERO(SUM(MaxPerm)) AS DECIMAL(5,2)) AS pct_used
FROM DBC.DiskSpaceV
GROUP BY 1
HAVING pct_used > 50
ORDER BY pct_used DESC;
```

### Top 10 tablas más grandes del sistema
```sql
SELECT TOP 10
    DatabaseName,
    TableName,
    SUM(CurrentPerm) / 1024 / 1024 / 1024 AS gb
FROM DBC.TableSizeV
GROUP BY 1, 2
ORDER BY gb DESC;
```

### Spool en uso ahora por sesión
```sql
SELECT
    UserName,
    SessionNo,
    SpoolUsage / 1024 / 1024 AS spool_mb
FROM DBC.SessionInfoV
WHERE SpoolUsage > 0
ORDER BY spool_mb DESC;
```

---

## 📊 Tendencias y Patrones (DBQL)

### Cuántas queries por hora del día (últimos 7 días)
```sql
SELECT
    EXTRACT(HOUR FROM StartTime) AS hora,
    COUNT(*) AS total_queries,
    AVG(AMPCPUTime) AS cpu_promedio
FROM DBC.QryLogV
WHERE CAST(StartTime AS DATE) >= CURRENT_DATE - 7
GROUP BY 1
ORDER BY 1;
```

### Top 10 usuarios más activos esta semana
```sql
SELECT TOP 10
    UserName,
    COUNT(*) AS total_queries,
    SUM(AMPCPUTime) AS cpu_total,
    AVG(AMPCPUTime) AS cpu_promedio
FROM DBC.QryLogV
WHERE CAST(StartTime AS DATE) >= CURRENT_DATE - 7
GROUP BY 1
ORDER BY total_queries DESC;
```

### Top 10 tablas más consultadas (últimos 7 días)
```sql
SELECT TOP 10
    ObjectDatabaseName,
    ObjectTableName,
    COUNT(*) AS accesos
FROM DBC.DBQLObjTbl
WHERE ObjectType = 'Tab'
  AND CAST(CollectTimeStamp AS DATE) >= CURRENT_DATE - 7
GROUP BY 1, 2
ORDER BY accesos DESC;
```

---

## 🔧 Diccionario de Datos

### Ver todas las tablas de una base de datos
```sql
SELECT
    TableName,
    TableKind,
    CreateTimeStamp,
    LastAlterTimeStamp
FROM DBC.TablesV
WHERE DatabaseName = 'nombre_base'
  AND TableKind IN ('T', 'V', 'O') -- T=tabla, V=vista, O=tabla sin log
ORDER BY TableName;
```

### Ver columnas de una tabla
```sql
SELECT
    ColumnName,
    ColumnType,
    ColumnLength,
    Nullable,
    DefaultValue,
    ColumnId
FROM DBC.ColumnsV
WHERE DatabaseName = 'nombre_base'
  AND TableName = 'nombre_tabla'
ORDER BY ColumnId;
```

### Ver el DDL de una tabla
```sql
SHOW TABLE nombre_base.nombre_tabla;
```

### Ver índices de una tabla
```sql
SELECT
    IndexNumber,
    IndexType,
    ColumnName,
    UniqueFlag,
    ColumnPosition
FROM DBC.IndicesV
WHERE DatabaseName = 'nombre_base'
  AND TableName = 'nombre_tabla'
ORDER BY IndexNumber, ColumnPosition;
```

---

## Siguiente módulo

→ [09 — Troubleshooting](../09_troubleshooting/README.md)
