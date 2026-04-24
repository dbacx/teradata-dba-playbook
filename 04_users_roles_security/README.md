# 04 — Usuarios, Roles y Seguridad

## Modelo de seguridad en Teradata

```
Usuario DBC (superusuario)
    └── Bases de datos / Usuarios
            └── Roles
                    └── Privilegios (GRANT)
```

- Un **usuario** puede contener objetos (tablas, vistas, macros)
- Un **rol** agrupa privilegios y se asigna a usuarios
- Los privilegios se pueden otorgar directamente al usuario o a través de roles

---

## Queries de diagnóstico

### Ver todos los usuarios del sistema
```sql
SELECT
    UserName,
    CreatorName,
    CreateTimeStamp,
    LastAccessTimeStamp,
    PermSpace / 1024 / 1024 / 1024 AS perm_gb,
    SpoolSpace / 1024 / 1024 / 1024 AS spool_gb
FROM DBC.UsersV
ORDER BY UserName;
```

### Ver roles existentes
```sql
SELECT RoleName, CreateTimeStamp
FROM DBC.RolesV
ORDER BY RoleName;
```

### Ver qué roles tiene asignados un usuario
```sql
SELECT
    UserName,
    RoleName,
    WhenGranted,
    DefaultRole,
    WithAdminOption
FROM DBC.RoleMembers
WHERE UserName = 'nombre_usuario'
ORDER BY RoleName;
```

### Ver todos los privilegios de un usuario (directo + por rol)
```sql
-- Privilegios directos
SELECT
    UserName,
    DatabaseName,
    TableName,
    AccessRight,
    GrantAuthority
FROM DBC.AllRightsV
WHERE UserName = 'nombre_usuario'
ORDER BY DatabaseName, TableName;
```

### Ver privilegios de un rol
```sql
SELECT
    RoleName,
    DatabaseName,
    TableName,
    AccessRight
FROM DBC.RoleRightsV
WHERE RoleName = 'nombre_rol'
ORDER BY DatabaseName, TableName;
```

### Ver quién tiene acceso a una tabla específica
```sql
SELECT
    UserName,
    AccessRight,
    GrantAuthority
FROM DBC.AllRightsV
WHERE DatabaseName = 'nombre_base'
  AND TableName = 'nombre_tabla'
ORDER BY UserName;
```

### Ver usuarios que no han entrado en los últimos 90 días
```sql
SELECT
    UserName,
    LastAccessTimeStamp,
    CAST(CURRENT_DATE - CAST(LastAccessTimeStamp AS DATE) AS INTEGER) AS dias_sin_acceso
FROM DBC.UsersV
WHERE LastAccessTimeStamp < CURRENT_TIMESTAMP - INTERVAL '90' DAY
  AND UserName NOT IN ('DBC', 'TDPUSER', 'TDWMAdmin')
ORDER BY dias_sin_acceso DESC;
```
> **Qué validar:** Usuarios inactivos 90+ días son candidatos a deshabilitar o eliminar (política de seguridad).

### Ver el perfil asignado a un usuario
```sql
SELECT
    UserName,
    ProfileName
FROM DBC.UsersV
WHERE ProfileName IS NOT NULL
ORDER BY ProfileName, UserName;
```

### Ver configuración de perfiles (intentos fallidos, expiración, etc.)
```sql
SELECT
    ProfileName,
    PasswordAttributes
FROM DBC.ProfilesV
ORDER BY ProfileName;
```

---

## Gestión de usuarios y roles

### Crear usuario
```sql
CREATE USER nombre_usuario
    AS PERM = 1000000000        -- 1 GB
       SPOOL = 5000000000       -- 5 GB
       TEMP = 1000000000        -- 1 GB
       PASSWORD = 'Password123!'
       DEFAULT DATABASE = nombre_base;
```

### Crear rol
```sql
CREATE ROLE nombre_rol;
```

### Asignar rol a usuario
```sql
GRANT nombre_rol TO nombre_usuario;
```

### Otorgar privilegios a un rol
```sql
-- SELECT sobre una base de datos completa
GRANT SELECT ON nombre_base TO nombre_rol;

-- INSERT/UPDATE sobre una tabla específica
GRANT INSERT, UPDATE ON nombre_base.nombre_tabla TO nombre_rol;

-- SELECT + INSERT + UPDATE + DELETE
GRANT SELECT, INSERT, UPDATE, DELETE ON nombre_base TO nombre_rol;

-- Privilegios de ejecución en macros/procedimientos
GRANT EXECUTE ON nombre_base.nombre_macro TO nombre_rol;
```

### Revocar privilegio
```sql
REVOKE SELECT ON nombre_base FROM nombre_usuario;
```

### Cambiar contraseña de un usuario
```sql
MODIFY USER nombre_usuario AS PASSWORD = 'NuevaClave123!';
```

### Eliminar usuario (debe estar vacío)
```sql
DROP USER nombre_usuario;
```
> ⚠️ Si el usuario tiene objetos, primero borrarlos o moverlos. Si tiene espacio asignado, primero modificar PERM = 0.

---

## Buenas prácticas

- Nunca otorgar privilegios directamente a usuarios — siempre usar roles
- Crear roles por función: `rol_lectura_ventas`, `rol_carga_etl`, `rol_dba_soporte`
- Auditar usuarios inactivos cada 90 días
- Usar perfiles para controlar intentos fallidos y expiración de contraseñas
- El usuario `DBC` no debe usarse en procesos automáticos o ETL

---

## Checklist de seguridad

- [ ] Usuarios con acceso directo (sin rol) identificados
- [ ] Usuarios inactivos 90+ días revisados
- [ ] Roles documentados con su función
- [ ] Accesos a tablas sensibles auditados
- [ ] Perfiles configurados con política de contraseñas

---

## Siguiente módulo

→ [05 — Backup & Recovery](../05_backup_recovery/README.md)
