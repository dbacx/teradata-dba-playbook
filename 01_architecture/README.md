# 01 — Arquitectura Teradata

## ¿Por qué importa entender la arquitectura?

Cada decisión de diseño en Teradata (PI, distribución, índices) tiene impacto directo en cómo los AMPs procesan los datos. Sin entender la arquitectura, el tuning es adivinanza.

---

## Componentes principales

### 🔵 Parsing Engine (PE)
- Recibe la query del cliente
- La parsea, optimiza y genera el execution plan
- **NO ejecuta datos** — solo coordina
- Una sesión = un PE asignado

### 🟡 AMP (Access Module Processor)
- Aquí vive y se procesa todo
- Cada AMP tiene su propio disco (vdisk)
- Los datos se distribuyen entre AMPs según el **Primary Index (PI)**
- Una query bien distribuida = todos los AMPs trabajan igual

### 🔗 BYNET
- Red interna que comunica PEs con AMPs
- Redesign en Vantage: InfiniBand o 100GbE

### 🗄️ Vdisk
- Disco virtual de cada AMP
- Contiene: datos de usuario, spool, temp, DBC

---

## Concepto clave: distribución de datos

```
fila → hash(Primary Index) → AMP destino
```

Si el PI tiene **pocos valores únicos** → muchas filas en pocos AMPs → **SKEW** → performance degradada.

---

## Queries de diagnóstico

### Ver cuántos AMPs tiene el sistema
```sql
SELECT COUNT(*) AS total_amps
FROM DBC.AMPUsage;
```

### Ver AMPs activos ahora mismo
```sql
SELECT NodeID, AMPNumber, CPUTime, IOCount
FROM DBC.AMPUsage
ORDER BY CPUTime DESC;
```

### Ver skew de una tabla
```sql
-- Cuántas filas tiene cada AMP para una tabla específica
SELECT
    Hashamp(Hashbucket(HashRow(columna_PI))) AS amp_number,
    COUNT(*) AS filas
FROM nombre_base.nombre_tabla
GROUP BY 1
ORDER BY 2 DESC;
```
> **Qué validar:** Si un AMP tiene 10x más filas que el promedio → PI mal elegido → evaluar cambio de PI o usar PPI.

### Ver distribución real de una tabla (más práctico)
```sql
SELECT
    TableName,
    SUM(CurrentPerm) / 1024 / 1024 AS perm_mb,
    MAX(CurrentPerm) / (SUM(CurrentPerm) / COUNT(*)) AS skew_factor
FROM DBC.TableSizeV
WHERE DatabaseName = 'nombre_base'
  AND TableName = 'nombre_tabla'
GROUP BY 1;
```
> **Qué validar:** `skew_factor` > 1.5 es señal de alerta. > 3 es problema grave.

### Ver información de nodos del sistema
```sql
SELECT
    NodeID,
    NodeType,
    ProcessorCount,
    PhysicalMemoryMB
FROM DBC.NodeInfoV
ORDER BY NodeID;
```

### Cuántas sesiones hay por PE ahora
```sql
SELECT
    PENumber,
    COUNT(*) AS sesiones_activas
FROM DBC.SessionInfoV
GROUP BY 1
ORDER BY 2 DESC;
```

---

## Cuándo usar estas queries

| Query | Cuándo |
|-------|--------|
| Ver AMPs | Baseline al asumir nuevo ambiente |
| Skew factor | Después de cargar una tabla grande o ante lentitud inexplicable |
| Nodos | Antes de hacer capacity planning |
| Sesiones por PE | Cuando un usuario reporta conexiones lentas o rechazadas |

---

## Checklist de arquitectura al asumir un nuevo ambiente

- [ ] ¿Cuántos nodos y AMPs tiene el sistema?
- [ ] ¿Cuál es la versión exacta de Teradata/Vantage?
- [ ] ¿Hay tablas grandes con skew conocido?
- [ ] ¿Cuántas sesiones máximas están configuradas?
- [ ] ¿Tiene TASM/TIWM configurado?
- [ ] ¿Está activo el Viewpoint para monitoreo?

---

## Siguiente módulo

→ [02 — Performance & EXPLAIN](../02_performance/README.md)
