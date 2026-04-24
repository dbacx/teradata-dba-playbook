# 07 — Utilidades

## Resumen rápido

| Utilidad | Dirección | Volumen | Velocidad | Cuándo usarla |
|----------|-----------|---------|-----------|---------------|
| **BTEQ** | Bidireccional | Bajo/medio | Lenta | Scripts SQL, DDL, reportes simples |
| **FastLoad** | Entrada | Alto | Muy rápida | Carga inicial de tablas vacías |
| **FastExport** | Salida | Alto | Muy rápida | Extraer datos masivos a archivo |
| **MultiLoad** | Entrada | Alto | Rápida | INSERT/UPDATE/DELETE en tablas activas |
| **TPT** | Bidireccional | Alto | Muy rápida | Reemplaza FastLoad/FastExport/MultiLoad |

---

## BTEQ

La navaja suiza del DBA. Conecta, ejecuta SQL, genera reportes, automatiza scripts.

### Estructura básica de un script BTEQ
```bteq
.LOGTABLE dbc.log_bteq;
.LOGON servidor/usuario,contraseña;

.SET ERROROUT STDOUT
.SET WIDTH 200
.SET SEPARATOR '|'

-- Tu SQL aquí
SELECT * FROM nombre_base.nombre_tabla SAMPLE 10;

.IF ERRORCODE <> 0 THEN .QUIT 12;

.LOGOFF;
.QUIT 0;
```

### Exportar resultado a archivo
```bteq
.EXPORT REPORT FILE=salida.txt;
SELECT col1, col2 FROM nombre_base.tabla;
.EXPORT RESET;
```

### Ejecutar script desde shell
```bash
bteq < script.bteq > log_ejecucion.log 2>&1
echo "Exit code: $?"
```

### Validar exit code en shell
```bash
bteq < carga.bteq
if [ $? -ne 0 ]; then
    echo "ERROR en BTEQ"
    exit 1
fi
```

---

## FastLoad

Carga masiva a tablas **vacías**. No permite duplicados durante la carga. No soporta índices secundarios ni triggers.

### Script básico FastLoad
```fastload
.LOGTABLE nombre_base.log_fastload_tabla;
.LOGON servidor/usuario,contraseña;

DATABASE nombre_base;
DROP TABLE ET_nombre_tabla;
DROP TABLE UV_nombre_tabla;

BEGIN LOADING nombre_base.nombre_tabla
    ERRORFILES nombre_base.ET_nombre_tabla,
               nombre_base.UV_nombre_tabla
    CHECKPOINT 50000;

DEFINE
    col1 (CHAR(10)),
    col2 (VARCHAR(50)),
    col3 (INTEGER),
    col4 (DATE)
FILE=archivo_entrada.txt;

INSERT INTO nombre_base.nombre_tabla
VALUES (
    :col1,
    :col2,
    :col3,
    :col4
);

END LOADING;
LOGOFF;
```
> **Tablas de error:**
> - `ET_` = errores de conversión/constraint
> - `UV_` = violaciones de unicidad

### Verificar errores post-carga
```sql
SELECT COUNT(*), ErrorCode, ErrorText
FROM nombre_base.ET_nombre_tabla
GROUP BY 2, 3;

SELECT COUNT(*)
FROM nombre_base.UV_nombre_tabla;
```

---

## FastExport

Extrae datos masivos de Teradata a archivo. Más rápido que SELECT en BTEQ para volúmenes grandes.

### Script básico FastExport
```fastexport
.LOGTABLE nombre_base.log_fexp;
.LOGON servidor/usuario,contraseña;

.BEGIN EXPORT SESSIONS 4
    FILE=salida.txt;

.EXPORT OUTFILE salida.txt
    FORMAT TEXT;

SELECT
    CAST(col1 AS CHAR(10)),
    CAST(col2 AS VARCHAR(50)),
    CAST(col3 AS CHAR(10))
FROM nombre_base.nombre_tabla
WHERE fecha_proceso = DATE '2024-01-01';

.END EXPORT;
.LOGOFF;
```

---

## MultiLoad

Permite INSERT, UPDATE, DELETE en tablas que ya tienen datos. Más lento que FastLoad pero más flexible.

### Script básico MultiLoad
```multiload
.LOGTABLE nombre_base.log_mload;
.LOGON servidor/usuario,contraseña;

.BEGIN MLOAD TABLES nombre_base.nombre_tabla;

.LAYOUT entrada;
.FIELD col1 * CHAR(10);
.FIELD col2 * VARCHAR(50);
.FIELD col3 * INTEGER;

.DML LABEL ins_upd;
INSERT INTO nombre_base.nombre_tabla
    (col1, col2, col3)
VALUES
    (:col1, :col2, :col3);

.IMPORT INFILE archivo.txt
    FORMAT TEXT
    APPLY ins_upd;

.END MLOAD;
.LOGOFF;
```

---

## TPT (Teradata Parallel Transporter)

Reemplaza a todas las utilidades anteriores. Más moderno, paralelo, flexible.

### Script TPT básico — carga desde CSV
```tpt
DEFINE JOB carga_tabla
DESCRIPTION 'Carga tabla desde CSV'
(
    DEFINE OPERATOR LOAD_OPERATOR
    TYPE LOAD
    SCHEMA *
    ATTRIBUTES
    (
        VARCHAR PrivateLogName = 'load_log',
        VARCHAR TargetTable = 'nombre_base.nombre_tabla',
        VARCHAR LogTable = 'nombre_base.log_tpt_tabla',
        VARCHAR ErrorTable1 = 'nombre_base.ET_nombre_tabla',
        VARCHAR ErrorTable2 = 'nombre_base.UV_nombre_tabla',
        VARCHAR TdpId = 'servidor',
        VARCHAR UserName = 'usuario',
        VARCHAR UserPassword = 'contraseña'
    );

    DEFINE SCHEMA file_schema
    (
        col1 VARCHAR(10),
        col2 VARCHAR(50),
        col3 VARCHAR(10)
    );

    DEFINE OPERATOR FILE_READER
    TYPE DATACONNECTOR PRODUCER
    SCHEMA file_schema
    ATTRIBUTES
    (
        VARCHAR DirectoryPath = '/ruta/archivos/',
        VARCHAR FileName = 'archivo.csv',
        VARCHAR Format = 'Delimited',
        VARCHAR OpenMode = 'Read',
        VARCHAR TextDelimiter = ','
    );

    STEP LOAD_STEP
    (
        APPLY
        (
            'INSERT INTO nombre_base.nombre_tabla VALUES(:col1, :col2, CAST(:col3 AS INTEGER));'
        )
        TO OPERATOR (LOAD_OPERATOR)
        SELECT * FROM OPERATOR (FILE_READER);
    );
);
```

### Ejecutar TPT desde shell
```bash
tbuild -f script.tpt -j nombre_job > log_tpt.log 2>&1
echo "Exit code: $?"
```

---

## Queries de diagnóstico de utilidades

### Ver sesiones de utilidades activas
```sql
SELECT
    UserName,
    SessionNo,
    ClientProgramName,
    LogonTime
FROM DBC.SessionInfoV
WHERE ClientProgramName IN ('FastLoad', 'MultiLoad', 'FastExport', 'BTEQ', 'tbuild')
   OR ClientProgramName LIKE '%TPT%'
ORDER BY LogonTime;
```

### Ver locks generados por utilidades
```sql
SELECT
    DatabaseName,
    TableName,
    LockType,
    LockStatus,
    SessionNo
FROM DBC.LockLogShredV
WHERE LockStatus = 'L'
ORDER BY DatabaseName, TableName;
```

---

## Checklist de utilidades

- [ ] ¿Los scripts BTEQ validan el exit code?
- [ ] ¿Las tablas ET_ y UV_ se revisan post-carga?
- [ ] ¿FastLoad se usa solo en tablas vacías?
- [ ] ¿Los jobs TPT tienen log configurado?
- [ ] ¿Los procesos de carga corren fuera de horario pico?

---

## Siguiente módulo

→ [08 — Queries de Monitoreo](../08_monitoring_queries/README.md)
