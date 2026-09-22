# Configuración: Roles, Auditoría y Encriptación

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 60 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción General

En esta práctica se implementan controles de seguridad complementarios para la base de datos `Sql2025Lab`: acceso basado en roles de mínimo privilegio, auditoría de eventos de seguridad y cifrado transparente de datos mediante Transparent Data Encryption (TDE). Se validarán los permisos usando cuentas SQL de laboratorio, se revisarán los eventos generados en archivos de auditoría y se realizará un respaldo de la base de datos protegida por TDE.

La práctica aplica una estrategia de defensa en profundidad: autenticar identidades, limitar autorizaciones, registrar acciones relevantes y proteger los archivos físicos de la base de datos y sus respaldos.

## Objetivos de Aprendizaje

- [ ] Implementar logins, usuarios y roles de base de datos aplicando el principio de mínimo privilegio.
- [ ] Configurar permisos diferenciados para lectores, analistas y ejecutores de procedimientos autorizados.
- [ ] Crear y validar una auditoría de SQL Server para eventos de inicio de sesión y acciones sobre objetos críticos.
- [ ] Habilitar TDE en `Sql2025Lab`, respaldar el certificado requerido y generar un respaldo recuperable.
- [ ] Verificar que los controles de acceso, auditoría y cifrado funcionan según lo esperado.

## Prerrequisitos

**Conocimientos requeridos**

- Diferenciar autenticación, autorización y elevación de privilegios.
- Conocer los conceptos de login, usuario de base de datos, esquema, rol y permiso en SQL Server.
- Comprender que TDE protege archivos de datos, archivos de log y respaldos en reposo, pero no reemplaza los permisos de acceso.
- Conocer los riesgos de perder certificados y claves criptográficas.

**Acceso y condiciones requeridas**

- La base de datos `Sql2025Lab` existe, tiene nivel de compatibilidad `170`, modelo de recuperación `FULL` y Query Store habilitado.
- La instancia predeterminada `MSSQLSERVER` está disponible en `LOCALHOST,1433`.
- La cuenta que ejecuta la práctica pertenece al rol fijo de servidor `sysadmin`.
- Existen las carpetas:
  - `C:\SQLLab2025\Audit`
  - `C:\SQLLab2025\Keys`
  - `C:\SQLLab2025\Backups`
- La cuenta de servicio de SQL Server tiene permisos de modificación sobre dichas carpetas.
- La tabla `Sales.Orders`, el esquema `Reporting` y la tabla de inventario `dbo.ObjectDocumentation` están disponibles en `Sql2025Lab`.
- Se recomienda realizar la práctica en una instancia aislada de laboratorio. No reutilice las contraseñas indicadas fuera de este entorno.

## Entorno de Laboratorio

| Componente | Configuración de laboratorio |
|---|---|
| Servidor | `SQLLAB-WS2022` |
| Sistema operativo | Windows Server 2022 Datacenter |
| Instancia | `MSSQLSERVER` |
| Conexión | `LOCALHOST,1433` |
| Base de datos | `Sql2025Lab` |
| SQL Server destino | SQL Server 2025 Developer Edition |
| Herramienta principal | SQL Server Management Studio 21.3.2 |
| Memoria máxima de SQL Server | 16384 MB |
| MAXDOP | 4 |
| Cost threshold for parallelism | 50 |

Abra SSMS con la cuenta Windows `.\SqlLabAdmin` o con otra cuenta autorizada como `sysadmin`. Conéctese a:

```text
Servidor: LOCALHOST,1433
Autenticación: Windows Authentication
```

Ejecute las siguientes comprobaciones iniciales en una nueva ventana de consulta:

```sql
USE master;
GO

SELECT
    @@SERVERNAME AS server_name,
    SERVERPROPERTY('ProductVersion') AS product_version,
    SERVERPROPERTY('Edition') AS edition,
    SUSER_SNAME() AS current_login,
    IS_SRVROLEMEMBER(N'sysadmin') AS is_sysadmin;
GO

SELECT
    name,
    compatibility_level,
    recovery_model_desc,
    state_desc,
    is_encrypted
FROM sys.databases
WHERE name = N'Sql2025Lab';
GO

USE Sql2025Lab;
GO

SELECT TOP (20) *
FROM dbo.ObjectDocumentation;
GO

SELECT
    SCHEMA_NAME(schema_id) AS schema_name,
    name AS object_name,
    type_desc
FROM sys.objects
WHERE schema_id IN (SCHEMA_ID(N'Sales'), SCHEMA_ID(N'Reporting'))
ORDER BY schema_name, type_desc, object_name;
GO
```

## Instrucciones Paso a Paso

### Paso 1: Verificar rutas, permisos y activos de seguridad

**Objetivo:** Confirmar que las rutas requeridas existen, que SQL Server puede escribir en ellas y que los objetos de laboratorio necesarios están disponibles.

**Instrucciones**

1. Abra PowerShell como administrador en `SQLLAB-WS2022`.

2. Compruebe la identidad de la cuenta de servicio de SQL Server:

   ```powershell
   Get-CimInstance Win32_Service -Filter "Name='MSSQLSERVER'" |
       Select-Object Name, StartName, State
   ```

3. Cree las rutas requeridas si no existen:

   ```powershell
   New-Item -ItemType Directory -Force -Path `
     'C:\SQLLab2025\Audit', `
     'C:\SQLLab2025\Keys', `
     'C:\SQLLab2025\Backups'
   ```

4. Si la instancia utiliza la cuenta virtual predeterminada, otorgue permisos de modificación a la cuenta de servicio:

   ```powershell
   icacls C:\SQLLab2025\Audit /grant "NT SERVICE\MSSQLSERVER:(OI)(CI)M"
   icacls C:\SQLLab2025\Keys /grant "NT SERVICE\MSSQLSERVER:(OI)(CI)M"
   icacls C:\SQLLab2025\Backups /grant "NT SERVICE\MSSQLSERVER:(OI)(CI)M"
   ```

   Si el servicio usa una cuenta de dominio o una cuenta administrada distinta, sustituya `NT SERVICE\MSSQLSERVER` por la identidad devuelta en `StartName`.

5. En SSMS, ejecute la comprobación de objetos críticos:

   ```sql
   USE Sql2025Lab;
   GO

   SELECT
       OBJECT_ID(N'Sales.Orders', N'U') AS sales_orders_object_id,
       SCHEMA_ID(N'Reporting') AS reporting_schema_id,
       OBJECT_ID(N'dbo.ObjectDocumentation', N'U') AS documentation_object_id;
   GO
   ```

6. Revise los procedimientos existentes que puedan ser autorizados para el rol ejecutor. Seleccione uno que esté aprobado para consultas de laboratorio y anote su nombre completo.

   ```sql
   USE Sql2025Lab;
   GO

   SELECT
       SCHEMA_NAME(p.schema_id) AS schema_name,
       p.name AS procedure_name,
       p.create_date,
       p.modify_date
   FROM sys.procedures AS p
   ORDER BY SCHEMA_NAME(p.schema_id), p.name;
   GO
   ```

**Resultado esperado**

- Las tres carpetas existen.
- La cuenta de servicio de SQL Server dispone de permiso de escritura en las rutas.
- `Sales.Orders`, `Reporting` y `dbo.ObjectDocumentation` devuelven identificadores distintos de `NULL`.
- Existe al menos un procedimiento almacenado aprobado para otorgar permisos de ejecución.

**Verificación**

Ejecute el siguiente script. Si devuelve filas, no continúe hasta resolver los elementos faltantes:

```sql
USE Sql2025Lab;
GO

SELECT N'Sales.Orders no existe' AS validation_error
WHERE OBJECT_ID(N'Sales.Orders', N'U') IS NULL

UNION ALL

SELECT N'El esquema Reporting no existe'
WHERE SCHEMA_ID(N'Reporting') IS NULL

UNION ALL

SELECT N'dbo.ObjectDocumentation no existe'
WHERE OBJECT_ID(N'dbo.ObjectDocumentation', N'U') IS NULL;
GO
```

---

### Paso 2: Crear logins, usuarios y roles de mínimo privilegio

**Objetivo:** Implementar identidades SQL de laboratorio y roles personalizados con permisos separados según su función.

**Instrucciones**

1. Cree los logins SQL de laboratorio en `master`. Las cuentas `lab_reader` y `lab_analyst` tendrán una contraseña temporal definida para esta práctica.

   ```sql
   USE master;
   GO

   IF NOT EXISTS (SELECT 1 FROM sys.sql_logins WHERE name = N'lab_reader')
   BEGIN
       CREATE LOGIN lab_reader
       WITH PASSWORD = 'LabAccess2025!ChangeMe',
            CHECK_POLICY = ON,
            CHECK_EXPIRATION = OFF,
            DEFAULT_DATABASE = Sql2025Lab;
   END;
   GO

   IF NOT EXISTS (SELECT 1 FROM sys.sql_logins WHERE name = N'lab_analyst')
   BEGIN
       CREATE LOGIN lab_analyst
       WITH PASSWORD = 'LabAccess2025!ChangeMe',
            CHECK_POLICY = ON,
            CHECK_EXPIRATION = OFF,
            DEFAULT_DATABASE = Sql2025Lab;
   END;
   GO
   ```

2. Cree los usuarios correspondientes dentro de `Sql2025Lab`.

   ```sql
   USE Sql2025Lab;
   GO

   IF NOT EXISTS (SELECT 1 FROM sys.database_principals WHERE name = N'lab_reader')
       CREATE USER lab_reader FOR LOGIN lab_reader;
   GO

   IF NOT EXISTS (SELECT 1 FROM sys.database_principals WHERE name = N'lab_analyst')
       CREATE USER lab_analyst FOR LOGIN lab_analyst;
   GO
   ```

3. Cree los tres roles personalizados requeridos.

   ```sql
   USE Sql2025Lab;
   GO

   IF NOT EXISTS (SELECT 1 FROM sys.database_principals WHERE name = N'db_lab_reader')
       CREATE ROLE db_lab_reader AUTHORIZATION dbo;
   GO

   IF NOT EXISTS (SELECT 1 FROM sys.database_principals WHERE name = N'db_lab_analyst')
       CREATE ROLE db_lab_analyst AUTHORIZATION dbo;
   GO

   IF NOT EXISTS (SELECT 1 FROM sys.database_principals WHERE name = N'db_lab_executor')
       CREATE ROLE db_lab_executor AUTHORIZATION dbo;
   GO
   ```

4. Asigne los usuarios a los roles. En esta práctica, `lab_reader` representa la función lectora y `lab_analyst` representa la función analista. El rol `db_lab_executor` se crea para demostrar una función separada de ejecución controlada.

   ```sql
   ALTER ROLE db_lab_reader ADD MEMBER lab_reader;
   GO

   ALTER ROLE db_lab_analyst ADD MEMBER lab_analyst;
   GO
   ```

5. Compruebe que ningún usuario de laboratorio pertenece a roles con privilegios excesivos como `db_owner`, `db_securityadmin` o `db_accessadmin`.

   ```sql
   USE Sql2025Lab;
   GO

   SELECT
       role_name = roles.name,
       member_name = members.name
   FROM sys.database_role_members AS drm
   INNER JOIN sys.database_principals AS roles
       ON drm.role_principal_id = roles.principal_id
   INNER JOIN sys.database_principals AS members
       ON drm.member_principal_id = members.principal_id
   WHERE members.name IN (N'lab_reader', N'lab_analyst')
   ORDER BY members.name, roles.name;
   GO
   ```

**Resultado esperado**

- Los logins `lab_reader` y `lab_analyst` existen en la instancia.
- Los usuarios con el mismo nombre existen en `Sql2025Lab`.
- Existen los roles `db_lab_reader`, `db_lab_analyst` y `db_lab_executor`.
- Los usuarios no son miembros de roles administrativos de base de datos.

**Verificación**

```sql
USE Sql2025Lab;
GO

SELECT
    dp.name,
    dp.type_desc,
    dp.authentication_type_desc
FROM sys.database_principals AS dp
WHERE dp.name IN
(
    N'lab_reader',
    N'lab_analyst',
    N'db_lab_reader',
    N'db_lab_analyst',
    N'db_lab_executor'
)
ORDER BY dp.type_desc, dp.name;
GO
```

---

### Paso 3: Otorgar permisos mínimos y autorizar procedimientos aprobados

**Objetivo:** Aplicar permisos específicos por rol, evitando permisos amplios como `db_datareader`, `db_datawriter`, `db_owner` o `CONTROL DATABASE`.

**Instrucciones**

1. Otorgue al rol lector únicamente permiso `SELECT` sobre el esquema `Reporting`.

   ```sql
   USE Sql2025Lab;
   GO

   GRANT SELECT ON SCHEMA::Reporting TO db_lab_reader;
   GO
   ```

2. Otorgue al rol analista permisos de lectura sobre el esquema `Reporting` y sobre el objeto de negocio autorizado `Sales.Orders`.

   ```sql
   USE Sql2025Lab;
   GO

   GRANT SELECT ON SCHEMA::Reporting TO db_lab_analyst;
   GRANT SELECT ON OBJECT::Sales.Orders TO db_lab_analyst;
   GO
   ```

3. Seleccione un procedimiento aprobado de la lista obtenida en el paso 1. Para este ejemplo se utiliza `Reporting.usp_LabOrderSummary`. Si el nombre no existe en su base de datos, sustitúyalo por un procedimiento existente y aprobado.

   ```sql
   USE Sql2025Lab;
   GO

   -- Sustituya Reporting.usp_LabOrderSummary por el procedimiento aprobado de su entorno.
   GRANT EXECUTE ON OBJECT::Reporting.usp_LabOrderSummary TO db_lab_analyst;
   GRANT EXECUTE ON OBJECT::Reporting.usp_LabOrderSummary TO db_lab_executor;
   GO
   ```

4. Si el procedimiento seleccionado requiere parámetros, identifique su definición antes de realizar las pruebas.

   ```sql
   USE Sql2025Lab;
   GO

   EXEC sys.sp_helptext N'Reporting.usp_LabOrderSummary';
   GO
   ```

5. Revise los permisos explícitos otorgados a los roles de laboratorio.

   ```sql
   USE Sql2025Lab;
   GO

   SELECT
       grantee.name AS grantee_name,
       permission_name = dp.permission_name,
       state_desc = dp.state_desc,
       class_desc = dp.class_desc,
       object_name = OBJECT_SCHEMA_NAME(dp.major_id) + N'.' + OBJECT_NAME(dp.major_id)
   FROM sys.database_permissions AS dp
   INNER JOIN sys.database_principals AS grantee
       ON dp.grantee_principal_id = grantee.principal_id
   WHERE grantee.name IN
   (
       N'db_lab_reader',
       N'db_lab_analyst',
       N'db_lab_executor'
   )
   ORDER BY grantee.name, dp.permission_name, object_name;
   GO
   ```

**Resultado esperado**

- `db_lab_reader` posee solamente `SELECT` sobre el esquema `Reporting`.
- `db_lab_analyst` posee `SELECT` sobre `Reporting` y `Sales.Orders`, además de `EXECUTE` sobre el procedimiento autorizado.
- `db_lab_executor` posee solamente `EXECUTE` sobre el procedimiento autorizado.
- No se asignan permisos de modificación sobre `Sales.Orders` a los usuarios de prueba.

**Verificación**

```sql
USE Sql2025Lab;
GO

SELECT
    HAS_PERMS_BY_NAME(N'Reporting', N'SCHEMA', N'SELECT') AS current_user_reporting_select,
    HAS_PERMS_BY_NAME(N'Sales.Orders', N'OBJECT', N'UPDATE') AS current_user_orders_update;
GO
```

> El resultado anterior corresponde al usuario administrador actual. La validación efectiva con las cuentas de laboratorio se realiza en el siguiente paso.

---

### Paso 4: Validar accesos permitidos y denegados

**Objetivo:** Comprobar que las cuentas de laboratorio pueden realizar únicamente las acciones autorizadas.

**Instrucciones**

1. Abra una nueva ventana de consulta en SSMS y conéctese mediante autenticación de SQL Server con estas credenciales:

   ```text
   Servidor: LOCALHOST,1433
   Autenticación: SQL Server Authentication
   Login: lab_reader
   Contraseña: LabAccess2025!ChangeMe
   Base de datos: Sql2025Lab
   ```

2. Ejecute una consulta sobre un objeto del esquema `Reporting`:

   ```sql
   SELECT TOP (5) *
   FROM Reporting.Orders;
   GO
   ```

   Si `Reporting.Orders` no existe, seleccione otra vista o tabla disponible en el esquema `Reporting`.

3. Intente leer la tabla de negocio directamente. El acceso debe ser denegado para `lab_reader`.

   ```sql
   SELECT TOP (5) *
   FROM Sales.Orders;
   GO
   ```

4. Intente modificar la tabla. El acceso debe ser denegado.

   ```sql
   UPDATE Sales.Orders
   SET OrderDate = OrderDate
   WHERE 1 = 0;
   GO
   ```

5. Abra otra conexión SQL con el usuario `lab_analyst`.

   ```text
   Servidor: LOCALHOST,1433
   Autenticación: SQL Server Authentication
   Login: lab_analyst
   Contraseña: LabAccess2025!ChangeMe
   Base de datos: Sql2025Lab
   ```

6. Compruebe que el analista puede leer `Sales.Orders`:

   ```sql
   SELECT TOP (5) *
   FROM Sales.Orders;
   GO
   ```

7. Ejecute el procedimiento autorizado, ajustando parámetros si corresponde:

   ```sql
   EXEC Reporting.usp_LabOrderSummary;
   GO
   ```

8. Compruebe que `lab_analyst` no puede modificar datos:

   ```sql
   UPDATE Sales.Orders
   SET OrderDate = OrderDate
   WHERE 1 = 0;
   GO
   ```

**Resultado esperado**

- `lab_reader` puede consultar objetos de `Reporting`.
- `lab_reader` recibe un error de permiso al consultar o modificar `Sales.Orders`.
- `lab_analyst` puede consultar `Sales.Orders` y ejecutar el procedimiento autorizado.
- `lab_analyst` recibe un error de permiso al intentar ejecutar `UPDATE` sobre `Sales.Orders`.

**Verificación**

En una conexión administrativa, revise los permisos efectivos declarados:

```sql
USE Sql2025Lab;
GO

EXECUTE AS USER = N'lab_reader';
SELECT
    USER_NAME() AS execution_context,
    HAS_PERMS_BY_NAME(N'Reporting', N'SCHEMA', N'SELECT') AS can_select_reporting,
    HAS_PERMS_BY_NAME(N'Sales.Orders', N'OBJECT', N'SELECT') AS can_select_orders,
    HAS_PERMS_BY_NAME(N'Sales.Orders', N'OBJECT', N'UPDATE') AS can_update_orders;
REVERT;
GO

EXECUTE AS USER = N'lab_analyst';
SELECT
    USER_NAME() AS execution_context,
    HAS_PERMS_BY_NAME(N'Reporting', N'SCHEMA', N'SELECT') AS can_select_reporting,
    HAS_PERMS_BY_NAME(N'Sales.Orders', N'OBJECT', N'SELECT') AS can_select_orders,
    HAS_PERMS_BY_NAME(N'Sales.Orders', N'OBJECT', N'UPDATE') AS can_update_orders;
REVERT;
GO
```

---

### Paso 5: Crear y habilitar la auditoría de SQL Server

**Objetivo:** Configurar una auditoría basada en archivos para registrar inicios de sesión y acciones relevantes sobre `Sales.Orders`.

**Instrucciones**

1. Cree la auditoría de servidor en la ruta de laboratorio. Ejecute el script desde `master`.

   ```sql
   USE master;
   GO

   IF NOT EXISTS
   (
       SELECT 1
       FROM sys.server_file_audits
       WHERE name = N'SQL2025Lab_Audit'
   )
   BEGIN
       CREATE SERVER AUDIT SQL2025Lab_Audit
       TO FILE
       (
           FILEPATH = N'C:\SQLLab2025\Audit\',
           MAXSIZE = 128 MB,
           MAX_ROLLOVER_FILES = 10,
           RESERVE_DISK_SPACE = OFF
       )
       WITH
       (
           QUEUE_DELAY = 1000,
           ON_FAILURE = CONTINUE
       );
   END;
   GO

   ALTER SERVER AUDIT SQL2025Lab_Audit
   WITH (STATE = ON);
   GO
   ```

2. Cree la especificación de auditoría de servidor para registrar inicios de sesión exitosos y fallidos.

   ```sql
   USE master;
   GO

   IF NOT EXISTS
   (
       SELECT 1
       FROM sys.server_audit_specifications
       WHERE name = N'SQL2025Lab_ServerAuditSpec'
   )
   BEGIN
       CREATE SERVER AUDIT SPECIFICATION SQL2025Lab_ServerAuditSpec
       FOR SERVER AUDIT SQL2025Lab_Audit
       ADD (SUCCESSFUL_LOGIN_GROUP),
       ADD (FAILED_LOGIN_GROUP);
   END;
   GO

   ALTER SERVER AUDIT SPECIFICATION SQL2025Lab_ServerAuditSpec
   WITH (STATE = ON);
   GO
   ```

3. Cree la especificación de auditoría de base de datos para capturar operaciones de datos sobre `Sales.Orders` y cambios de esquema.

   ```sql
   USE Sql2025Lab;
   GO

   IF NOT EXISTS
   (
       SELECT 1
       FROM sys.database_audit_specifications
       WHERE name = N'SQL2025Lab_DatabaseAuditSpec'
   )
   BEGIN
       CREATE DATABASE AUDIT SPECIFICATION SQL2025Lab_DatabaseAuditSpec
       FOR SERVER AUDIT SQL2025Lab_Audit
       ADD (SELECT ON OBJECT::Sales.Orders BY public),
       ADD (INSERT ON OBJECT::Sales.Orders BY public),
       ADD (UPDATE ON OBJECT::Sales.Orders BY public),
       ADD (DELETE ON OBJECT::Sales.Orders BY public),
       ADD (SCHEMA_OBJECT_CHANGE_GROUP);
   END;
   GO

   ALTER DATABASE AUDIT SPECIFICATION SQL2025Lab_DatabaseAuditSpec
   WITH (STATE = ON);
   GO
   ```

4. Compruebe el estado de los componentes de auditoría.

   ```sql
   USE master;
   GO

   SELECT
       name,
       type_desc,
       is_state_enabled,
       log_file_path,
       max_file_size,
       max_rollover_files
   FROM sys.server_file_audits
   WHERE name = N'SQL2025Lab_Audit';
   GO

   SELECT
       name,
       is_state_enabled
   FROM sys.server_audit_specifications
   WHERE name = N'SQL2025Lab_ServerAuditSpec';
   GO

   USE Sql2025Lab;
   GO

   SELECT
       name,
       is_state_enabled
   FROM sys.database_audit_specifications
   WHERE name = N'SQL2025Lab_DatabaseAuditSpec';
   GO
   ```

**Resultado esperado**

- La auditoría `SQL2025Lab_Audit` está habilitada.
- La especificación de servidor registra logins exitosos y fallidos.
- La especificación de base de datos registra operaciones `SELECT`, `INSERT`, `UPDATE` y `DELETE` sobre `Sales.Orders`, además de cambios de objetos de esquema.
- Se crean archivos con extensión `.sqlaudit` en `C:\SQLLab2025\Audit`.

**Verificación**

```sql
SELECT
    name,
    status_desc,
    audit_file_path,
    audit_file_size
FROM sys.dm_server_audit_status
WHERE name = N'SQL2025Lab_Audit';
GO
```

---

### Paso 6: Generar y consultar eventos auditados

**Objetivo:** Producir eventos controlados y revisar la evidencia generada por SQL Server Audit.

**Instrucciones**

1. Desde la conexión de `lab_analyst`, ejecute una consulta permitida:

   ```sql
   USE Sql2025Lab;
   GO

   SELECT TOP (3) *
   FROM Sales.Orders;
   GO
   ```

2. Desde una conexión administrativa, genere eventos de modificación sin alterar filas. Las sentencias se auditan aunque el predicado no encuentre registros.

   ```sql
   USE Sql2025Lab;
   GO

   UPDATE Sales.Orders
   SET OrderDate = OrderDate
   WHERE 1 = 0;
   GO

   DELETE FROM Sales.Orders
   WHERE 1 = 0;
   GO
   ```

3. Genere un cambio de esquema controlado creando y eliminando una tabla temporal de laboratorio:

   ```sql
   USE Sql2025Lab;
   GO

   CREATE TABLE dbo.AuditSchemaChangeLab
   (
       AuditSchemaChangeLabId int NOT NULL PRIMARY KEY,
       CreatedAt datetime2(0) NOT NULL DEFAULT SYSUTCDATETIME()
   );
   GO

   DROP TABLE dbo.AuditSchemaChangeLab;
   GO
   ```

4. Genere un inicio de sesión fallido. Puede hacerlo desde una ventana de terminal o PowerShell:

   ```powershell
   sqlcmd -S LOCALHOST,1433 -U lab_reader -P "ClaveIncorrecta!2025" -Q "SELECT 1"
   ```

5. Genere un inicio de sesión exitoso:

   ```powershell
   sqlcmd -S LOCALHOST,1433 -U lab_reader -P "LabAccess2025!ChangeMe" -d Sql2025Lab -Q "SELECT SUSER_SNAME() AS login_name;"
   ```

6. Espere unos segundos para que SQL Server procese la cola de auditoría y consulte los archivos:

   ```sql
   SELECT TOP (100)
       event_time,
       action_id,
       succeeded,
       server_principal_name,
       database_name,
       schema_name,
       object_name,
       statement,
       additional_information
   FROM sys.fn_get_audit_file
   (
       N'C:\SQLLab2025\Audit\SQL2025Lab_Audit_*.sqlaudit',
       DEFAULT,
       DEFAULT
   )
   WHERE database_name = N'Sql2025Lab'
      OR server_principal_name IN (N'lab_reader', N'lab_analyst')
   ORDER BY event_time DESC;
   GO
   ```

**Resultado esperado**

- Se registran eventos de consulta y de intentos de modificación sobre `Sales.Orders`.
- Se registra el cambio de esquema asociado a la creación o eliminación de `dbo.AuditSchemaChangeLab`.
- Se registran al menos un inicio de sesión exitoso y uno fallido.
- Las columnas `server_principal_name`, `succeeded`, `object_name` y `statement` ayudan a identificar la acción realizada.

**Verificación**

Ejecute una consulta centrada en la tabla protegida:

```sql
SELECT TOP (50)
    event_time,
    succeeded,
    server_principal_name,
    database_name,
    schema_name,
    object_name,
    statement
FROM sys.fn_get_audit_file
(
    N'C:\SQLLab2025\Audit\SQL2025Lab_Audit_*.sqlaudit',
    DEFAULT,
    DEFAULT
)
WHERE object_name = N'Orders'
ORDER BY event_time DESC;
GO
```

Documente en `C:\SQLLab2025\Reports` los siguientes datos: fecha y hora, identidad, acción, objeto, resultado y observaciones de cada evento relevante.

---

### Paso 7: Crear la jerarquía criptográfica y habilitar TDE

**Objetivo:** Crear una Database Master Key en `master`, crear y respaldar un certificado de servidor, crear la Database Encryption Key y activar TDE para `Sql2025Lab`.

**Instrucciones**

1. Determine si `master` ya tiene una Database Master Key. No cree una segunda clave si ya existe.

   ```sql
   USE master;
   GO

   SELECT
       name,
       key_length,
       algorithm_desc,
       create_date,
       modify_date
   FROM sys.symmetric_keys
   WHERE name = N'##MS_DatabaseMasterKey##';
   GO
   ```

2. Si la consulta no devuelve filas, cree la Database Master Key. Sustituya el texto de ejemplo por una contraseña robusta y custodiada fuera del script.

   ```sql
   USE master;
   GO

   CREATE MASTER KEY ENCRYPTION BY PASSWORD =
       'Reemplace_Esta_Clave_Maestra_2025!';
   GO
   ```

3. Cree el certificado de servidor que protegerá la clave de cifrado de `Sql2025Lab`.

   ```sql
   USE master;
   GO

   IF NOT EXISTS
   (
       SELECT 1
       FROM sys.certificates
       WHERE name = N'SQL2025Lab_TDECert'
   )
   BEGIN
       CREATE CERTIFICATE SQL2025Lab_TDECert
       WITH SUBJECT = N'Certificado TDE para Sql2025Lab',
            EXPIRY_DATE = '20351231';
   END;
   GO
   ```

4. Respalde inmediatamente el certificado y su clave privada. Sustituya la contraseña de ejemplo por una contraseña distinta de la usada para la Database Master Key.

   ```sql
   USE master;
   GO

   BACKUP CERTIFICATE SQL2025Lab_TDECert
   TO FILE = N'C:\SQLLab2025\Keys\SQL2025Lab_TDECert.cer'
   WITH PRIVATE KEY
   (
       FILE = N'C:\SQLLab2025\Keys\SQL2025Lab_TDECert_PrivateKey.pvk',
       ENCRYPTION BY PASSWORD = 'Reemplace_Clave_Respaldo_Certificado_2025!'
   );
   GO
   ```

5. Compruebe que los archivos del certificado existen. En PowerShell:

   ```powershell
   Get-ChildItem C:\SQLLab2025\Keys\SQL2025Lab_TDECert*
   ```

6. Cree la Database Encryption Key en `Sql2025Lab`. Este script debe ejecutarse una sola vez.

   ```sql
   USE Sql2025Lab;
   GO

   IF NOT EXISTS
   (
       SELECT 1
       FROM sys.symmetric_keys
       WHERE name = N'##MS_DatabaseEncryptionKey##'
   )
   BEGIN
       CREATE DATABASE ENCRYPTION KEY
       WITH ALGORITHM = AES_256
       ENCRYPTION BY SERVER CERTIFICATE SQL2025Lab_TDECert;
   END;
   GO
   ```

7. Habilite el cifrado de la base de datos:

   ```sql
   USE master;
   GO

   ALTER DATABASE Sql2025Lab
   SET ENCRYPTION ON;
   GO
   ```

8. Consulte el progreso y el estado de cifrado:

   ```sql
   SELECT
       DB_NAME(dek.database_id) AS database_name,
       dek.encryption_state,
       CASE dek.encryption_state
           WHEN 0 THEN N'Sin clave de cifrado'
           WHEN 1 THEN N'Sin cifrar'
           WHEN 2 THEN N'Cifrado en curso'
           WHEN 3 THEN N'Cifrada'
           WHEN 4 THEN N'Cambio de clave en curso'
           WHEN 5 THEN N'Descifrado en curso'
           WHEN 6 THEN N'Cambio de protección en curso'
           ELSE N'Estado desconocido'
       END AS encryption_state_desc,
       dek.percent_complete,
       dek.key_algorithm,
       dek.key_length,
       c.name AS certificate_name
   FROM sys.dm_database_encryption_keys AS dek
   INNER JOIN master.sys.certificates AS c
       ON dek.encryptor_thumbprint = c.thumbprint
   WHERE DB_NAME(dek.database_id) = N'Sql2025Lab';
   GO
   ```

**Resultado esperado**

- Existe una Database Master Key en `master`.
- Existe el certificado `SQL2025Lab_TDECert`.
- Existen los archivos `.cer` y `.pvk` en `C:\SQLLab2025\Keys`.
- `Sql2025Lab` alcanza el estado `3 - Cifrada`.
- El algoritmo de la Database Encryption Key es `AES_256`.

**Verificación**

```sql
SELECT
    name,
    is_encrypted
FROM sys.databases
WHERE name = N'Sql2025Lab';
GO

USE master;
GO

SELECT
    name,
    subject,
    expiry_date,
    thumbprint
FROM sys.certificates
WHERE name = N'SQL2025Lab_TDECert';
GO
```

> En un entorno real, copie el archivo `.cer`, el archivo `.pvk` y la contraseña de la clave privada a una ubicación externa, segura y con control de acceso. Conservarlos únicamente en el mismo servidor no protege ante pérdida total del host.

---

### Paso 8: Crear y validar el respaldo protegido por TDE

**Objetivo:** Generar un respaldo de `Sql2025Lab` después de habilitar TDE y confirmar que el certificado es un requisito de restauración.

**Instrucciones**

1. Realice un respaldo completo de la base de datos.

   ```sql
   USE master;
   GO

   BACKUP DATABASE Sql2025Lab
   TO DISK = N'C:\SQLLab2025\Backups\Sql2025Lab_TDE_FULL.bak'
   WITH
   (
       INIT,
       COMPRESSION,
       CHECKSUM,
       STATS = 10,
       DESCRIPTION = N'Respaldo completo de Sql2025Lab protegido mediante TDE'
   );
   GO
   ```

2. Valide la estructura lógica del respaldo sin restaurarlo:

   ```sql
   RESTORE VERIFYONLY
   FROM DISK = N'C:\SQLLab2025\Backups\Sql2025Lab_TDE_FULL.bak'
   WITH CHECKSUM;
   GO
   ```

3. Revise la cabecera del respaldo:

   ```sql
   RESTORE HEADERONLY
   FROM DISK = N'C:\SQLLab2025\Backups\Sql2025Lab_TDE_FULL.bak';
   GO
   ```

4. Compruebe la presencia de los archivos de seguridad y del respaldo:

   ```powershell
   Get-ChildItem `
     C:\SQLLab2025\Keys\SQL2025Lab_TDECert.cer,
     C:\SQLLab2025\Keys\SQL2025Lab_TDECert_PrivateKey.pvk,
     C:\SQLLab2025\Backups\Sql2025Lab_TDE_FULL.bak |
     Select-Object Name, Length, LastWriteTime
   ```

5. Documente la dependencia criptográfica en el informe de la práctica:

   - El respaldo se generó después de habilitar TDE.
   - La restauración en otra instancia requiere el certificado `SQL2025Lab_TDECert`.
   - También requiere la clave privada asociada y su contraseña.
   - Sin estos elementos, SQL Server no puede abrir la Database Encryption Key ni restaurar correctamente una base protegida por TDE.

**Resultado esperado**

- El respaldo `Sql2025Lab_TDE_FULL.bak` existe en `C:\SQLLab2025\Backups`.
- `RESTORE VERIFYONLY` finaliza correctamente.
- Se conservaron el certificado, la clave privada y sus contraseñas de protección en una ubicación segura.
- El estudiante puede explicar por qué copiar solamente el archivo `.bak` no es suficiente para recuperar una base con TDE en otra instancia.

**Verificación**

```sql
RESTORE FILELISTONLY
FROM DISK = N'C:\SQLLab2025\Backups\Sql2025Lab_TDE_FULL.bak';
GO
```

## Validación y Pruebas

Complete la siguiente lista de validación antes de dar por terminada la práctica.

| Control | Prueba | Resultado esperado |
|---|---|---|
| Autenticación | Conectar con `lab_reader` | Inicio de sesión exitoso |
| Rol lector | Consultar un objeto de `Reporting` | Operación permitida |
| Rol lector | Consultar `Sales.Orders` | Permiso denegado |
| Rol lector | Ejecutar `UPDATE` sobre `Sales.Orders` | Permiso denegado |
| Rol analista | Consultar `Sales.Orders` | Operación permitida |
| Rol analista | Ejecutar procedimiento autorizado | Operación permitida |
| Rol analista | Ejecutar `UPDATE` sobre `Sales.Orders` | Permiso denegado |
| Auditoría de login | Intentar conexión con contraseña incorrecta | Evento `FAILED_LOGIN_GROUP` registrado |
| Auditoría de objetos | Consultar `Sales.Orders` | Evento con objeto `Orders` registrado |
| Auditoría de esquema | Crear y eliminar tabla de prueba | Evento de cambio de esquema registrado |
| TDE | Consultar `sys.dm_database_encryption_keys` | Estado `3 - Cifrada` |
| Certificado | Verificar archivos `.cer` y `.pvk` | Archivos presentes y protegidos |
| Respaldo | Ejecutar `RESTORE VERIFYONLY` | Verificación correcta |

Ejecute esta consulta consolidada para conservar evidencia del estado final:

```sql
USE master;
GO

SELECT
    d.name,
    d.recovery_model_desc,
    d.is_encrypted,
    dek.encryption_state,
    dek.percent_complete,
    c.name AS tde_certificate
FROM sys.databases AS d
LEFT JOIN sys.dm_database_encryption_keys AS dek
    ON d.database_id = dek.database_id
LEFT JOIN sys.certificates AS c
    ON dek.encryptor_thumbprint = c.thumbprint
WHERE d.name = N'Sql2025Lab';
GO

SELECT
    name,
    status_desc,
    audit_file_path
FROM sys.dm_server_audit_status
WHERE name = N'SQL2025Lab_Audit';
GO
```

## Solución de Problemas

**Problema 1: La creación de la auditoría o el respaldo del certificado falla con “Access is denied” o error del sistema operativo**

- **Síntoma:** `CREATE SERVER AUDIT`, `BACKUP CERTIFICATE` o `BACKUP DATABASE` devuelve un error de acceso a `C:\SQLLab2025\Audit`, `C:\SQLLab2025\Keys` o `C:\SQLLab2025\Backups`.
- **Causa:** La cuenta que ejecuta el servicio `MSSQLSERVER` no posee permisos NTFS de escritura o modificación sobre la carpeta indicada. El permiso del usuario administrador conectado a SSMS no sustituye el permiso de la cuenta de servicio de SQL Server.
- **Corrección:** Identifique la cuenta de servicio con `Get-CimInstance Win32_Service -Filter "Name='MSSQLSERVER'"`. Conceda permiso de modificación a esa identidad mediante `icacls`, valide la herencia de permisos y repita la operación. No otorgue permisos amplios a `Everyone` o `Users`.

**Problema 2: No se pueden consultar eventos con `sys.fn_get_audit_file` o no aparecen los eventos esperados**

- **Síntoma:** La función no devuelve filas, muestra un error de ruta o los eventos de login y acceso a objetos no aparecen.
- **Causa:** La auditoría o una especificación está deshabilitada, la ruta comodín no coincide con los archivos creados, el evento se generó antes de habilitar la especificación o la cola de auditoría todavía no se ha vaciado.
- **Corrección:** Compruebe `sys.dm_server_audit_status`, `sys.server_audit_specifications` y `sys.database_audit_specifications`; confirme que todos tienen `is_state_enabled = 1`. Revise físicamente `C:\SQLLab2025\Audit`, espere unos segundos y ajuste la ruta de `sys.fn_get_audit_file` al nombre real de los archivos `.sqlaudit`. Para eventos de login, use conexiones SQL reales mediante `sqlcmd` o una nueva conexión de SSMS, no solamente `EXECUTE AS`.

## Limpieza

> Realice esta sección solamente si el instructor solicita revertir la práctica. En un entorno real, no elimine certificados, claves, respaldos ni archivos de auditoría sin cumplir los requisitos de retención, respaldo y recuperación.

1. Conserve una copia externa de los siguientes archivos antes de cualquier limpieza:

   ```text
   C:\SQLLab2025\Keys\SQL2025Lab_TDECert.cer
   C:\SQLLab2025\Keys\SQL2025Lab_TDECert_PrivateKey.pvk
   C:\SQLLab2025\Backups\Sql2025Lab_TDE_FULL.bak
   ```

2. Deshabilite y elimine las especificaciones de auditoría y la auditoría:

   ```sql
   USE Sql2025Lab;
   GO

   ALTER DATABASE AUDIT SPECIFICATION SQL2025Lab_DatabaseAuditSpec
   WITH (STATE = OFF);
   GO

   DROP DATABASE AUDIT SPECIFICATION SQL2025Lab_DatabaseAuditSpec;
   GO

   USE master;
   GO

   ALTER SERVER AUDIT SPECIFICATION SQL2025Lab_ServerAuditSpec
   WITH (STATE = OFF);
   GO

   DROP SERVER AUDIT SPECIFICATION SQL2025Lab_ServerAuditSpec;
   GO

   ALTER SERVER AUDIT SQL2025Lab_Audit
   WITH (STATE = OFF);
   GO

   DROP SERVER AUDIT SQL2025Lab_Audit;
   GO
   ```

3. Elimine los usuarios, roles y logins de prueba:

   ```sql
   USE Sql2025Lab;
   GO

   DROP USER IF EXISTS lab_reader;
   DROP USER IF EXISTS lab_analyst;
   GO

   DROP ROLE IF EXISTS db_lab_reader;
   DROP ROLE IF EXISTS db_lab_analyst;
   DROP ROLE IF EXISTS db_lab_executor;
   GO

   USE master;
   GO

   DROP LOGIN IF EXISTS lab_reader;
   DROP LOGIN IF EXISTS lab_analyst;
   GO
   ```

4. No elimine el certificado ni la Database Master Key si `Sql2025Lab` continúa cifrada. Para retirar TDE se requiere una operación planificada, validada y potencialmente prolongada:

   ```sql
   USE master;
   GO

   ALTER DATABASE Sql2025Lab
   SET ENCRYPTION OFF;
   GO
   ```

5. Espere hasta que `sys.dm_database_encryption_keys` indique que la base ya no está cifrada antes de considerar la eliminación de la Database Encryption Key o del certificado. No elimine la Database Master Key de `master` si puede ser utilizada por otros certificados, credenciales o componentes de la instancia.

6. Elimine manualmente archivos de auditoría únicamente después de confirmar que la retención definida para el laboratorio ha concluido.

## Resumen

En esta práctica se aplicaron controles complementarios de seguridad para `Sql2025Lab`:

- Se crearon logins SQL, usuarios y roles personalizados con permisos mínimos.
- Se separaron las capacidades de lectura, análisis y ejecución de procedimientos aprobados.
- Se configuró SQL Server Audit para registrar autenticaciones y acciones relevantes sobre `Sales.Orders`.
- Se revisaron los eventos mediante `sys.fn_get_audit_file`.
- Se habilitó TDE usando una Database Master Key, un certificado de servidor y una Database Encryption Key con `AES_256`.
- Se respaldó el certificado con su clave privada y se generó un respaldo de la base de datos.
- Se verificó que el certificado y la clave privada son indispensables para restaurar una base protegida por TDE en otra instancia.

La seguridad efectiva no depende de un único mecanismo. Los roles reducen el riesgo de privilegios excesivos, la auditoría genera evidencia para detección e investigación, y TDE protege los archivos físicos frente a accesos no autorizados al almacenamiento. Estos controles deben revisarse periódicamente y complementarse con seguridad de red, protección de identidades, copias externas de claves y pruebas de restauración.
