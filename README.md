# 🟡 Teradata DBA Playbook

Guía práctica para DBAs de Teradata: conceptos, queries listas para usar, validaciones y troubleshooting desde la experiencia real.

> Escrito en español. Orientado a DBAs que trabajan en entornos productivos con Teradata 14+.

---

## 📁 Contenido

| # | Módulo | Descripción |
|---|--------|-------------|
| 01 | [Arquitectura](./01_architecture/README.md) | AMPs, PEs, Parsing Engine, nodos |
| 02 | [Performance & EXPLAIN](./02_performance/README.md) | Skew, estadísticas, PI, EXPLAIN |
| 03 | [Space Management](./03_space_management/README.md) | Perm, Spool, Temp space |
| 04 | [Usuarios, Roles y Seguridad](./04_users_roles_security/README.md) | GRANT, profiles, LDAP |
| 05 | [Backup & Recovery](./05_backup_recovery/README.md) | ARC, BAR, DSA |
| 06 | [Workload Management](./06_workload_management/README.md) | TASM, TIWM, priorities |
| 07 | [Utilidades](./07_utilities/README.md) | FastLoad, BTEQ, TPT, Fastexport |
| 08 | [Queries de Monitoreo](./08_monitoring_queries/README.md) | ← listas para copiar/pegar |
| 09 | [Troubleshooting](./09_troubleshooting/README.md) | Casos reales y soluciones |

---

## 🚀 Cómo usar este repo

Cada módulo tiene:
- **Concepto** — qué es y para qué sirve
- **Query** — lista para ejecutar en tu ambiente
- **Qué validar** — cómo interpretar el resultado
- **Cuándo usarlo** — contexto real de uso

---

## 🛠️ Versiones cubiertas

- Teradata Database 14.x, 15.x, 16.x, 17.x
- Vantage 1.x, 2.x
- TD Studio, BTEQ, Viewpoint

---

## 🤝 Contribuciones

Pull requests bienvenidos. Si tienes queries probadas en producción, ábrelas como PR con el template del módulo correspondiente.

---

## 👤 Autor

Ricardo Enciso — DBA Senior Teradata | Colombia  
[LinkedIn](https://www.linkedin.com/in/raeb/) · [GitHub](https://github.com/dbacx)

---

> ⚠️ Las queries están probadas en ambientes reales pero siempre valida antes de ejecutar en producción.
