# 05 — Backup & Recovery

## Herramientas de backup en Teradata

| Herramienta | Tipo | Uso |
|-------------|------|-----|
| **ARC (Archive/Recovery)** | Nativo Teradata | Backup a cinta o disco, online/offline |
| **BAR (Backup and Recovery)** | Solución Teradata | Orquesta ARC con interfaces gráficas |
| **DSA (Data Stream Architecture)** | Moderno | Backup en paralelo, más rápido que ARC |
| **TARA (Teradata Active System Management)** | Cloud | Para entornos Vantage Cloud |

---

## Conceptos clave

- **Full backup:** copia todos los datos de la base o tabla
- **Incremental backup:** solo los cambios desde el último full (solo con DSA)
- **Online backup:** mientras el sistema está en producción (puede haber locks breves)
- **Offline backup:** sistema detenido — mayor consistencia
- **Restore:** recuperación completa
- **Recovery:** recuperación parcial (tabla específica, punto en el tiempo)

---

## ARC — Comandos básicos

### Backup de una base de datos completa
```sql
.LOGTABLE dbc.arc_log;

.LOGON tdserver/dba_user,password;

ARCHIVE DATA TABLES IN nombre_base,
    RELEASE LOCK,
    FILE=backup_nombre_base;

.LOGOFF;
```

### Backup de una tabla específica
```sql
ARCHIVE DATA TABLE nombre_base.nombre_tabla,
    RELEASE LOCK,
    FILE=backup_tabla;
```

### Restore de una base de datos
```sql
RESTORE DATA TABLES IN nombre_base,
    FILE=backup_nombre_base;
```

### Restore de una tabla específica
```sql
RESTORE DATA TABLE nombre_base.nombre_tabla,
    FILE=backup_tabla;
```

> ⚠️ El restore sobreescribe los datos actuales. Si necesitas recuperar sin perder lo que hay, usa un ambiente alternativo primero.

---

## Queries de diagnóstico

### Ver historial de backups (requiere tabla de log configurada)
```sql
SELECT
    LogDate,
    LogTime,
    ActivityType,
    DatabaseName,
    TableName,
    ErrorCode,
    ErrorText
FROM DBC.ArcLog
WHERE LogDate >= CURRENT_DATE - 7
ORDER BY LogDate DESC, LogTime DESC;
```

### Ver objetos en un archivo de backup (ARC)
```sql
.LOGTABLE dbc.arc_log;
.LOGON tdserver/dba_user,password;

SHOW ARCHIVE FILE=backup_nombre_base;

.LOGOFF;
```

### Ver sesiones de backup activas ahora
```sql
SELECT
    UserName,
    SessionNo,
    LogonTime,
    ClientProgramName
FROM DBC.SessionInfoV
WHERE ClientProgramName LIKE '%ARC%'
   OR ClientProgramName LIKE '%DSA%'
ORDER BY LogonTime;
```

### Ver locks activos (relevante antes de backup)
```sql
SELECT
    VProc,
    DatabaseName,
    TableName,
    LockType,
    LockStatus,
    BlockingSessionNo
FROM DBC.LockLogShredV
WHERE LockStatus = 'L'
ORDER BY DatabaseName, TableName;
```
> **Qué validar:** Locks tipo `Exclusive` sobre tablas que vas a respaldar pueden bloquear el backup. Identifica la sesión con `BlockingSessionNo` y coordina.

---

## DSA — Conceptos básicos

DSA es más moderno que ARC. Trabaja con **jobs** que se definen en XML o por interfaz.

### Componentes DSA
- **DSC (Data Stream Controller):** orquesta los jobs
- **DSA Agent:** corre en cada nodo Teradata
- **Media Server:** destino del backup (NFS, cloud, cinta)

### Verificar jobs DSA activos (desde DSC)
```sql
-- Ejecutar en la base de datos DSC
SELECT
    JobName,
    JobStatus,
    StartTime,
    EndTime,
    ErrorCode
FROM DSA.Jobs
ORDER BY StartTime DESC;
```

---

## Checklist de backup

- [ ] ¿Hay un schedule de backup full semanal definido?
- [ ] ¿Los backups incrementales corren diariamente?
- [ ] ¿Se verifica que los archivos de backup son válidos (SHOW ARCHIVE)?
- [ ] ¿Cuánto tiempo tarda un restore? ¿Está documentado?
- [ ] ¿Hay un ambiente de prueba para validar restores?
- [ ] ¿Los logs de backup se revisan diariamente?
- [ ] ¿El destino del backup tiene espacio suficiente?

---

## Buenas prácticas

- Nunca hacer el primer restore de producción en producción — prueba primero
- Documentar el tiempo promedio de backup y restore por base de datos
- Separar los backups de tablas críticas de los backups generales
- Mantener al menos 2 copias del backup en ubicaciones distintas
- Probar el restore al menos una vez al mes en ambiente de prueba

---

## Siguiente módulo

→ [06 — Workload Management](../06_workload_management/README.md)
