# Revisión del entorno: Windows Server 2022, SQL Server 2025 y SSMS actualizado

## Metadatos

| Elemento | Valor |
|---|---|
| Duración | 45 minutos |
| Complejidad | Fácil |
| Nivel de Bloom | Aplicar |

## Descripción General

En esta práctica se verificará el estado operativo del servidor `SQLLAB-WS2022`, Windows Server 2022, SQL Server 2025 Developer Edition y SQL Server Management Studio (SSMS). Se validarán los servicios, la escucha TCP/IP en el puerto 1433, la autenticación, las rutas de archivos y los parámetros iniciales de la instancia.

Posteriormente, se creará la base de datos común `Sql2025Lab`, configurada con nivel de compatibilidad 170, modelo de recuperación FULL y Query Store en modo `READ_WRITE`. La práctica finaliza con la carga de datos de ejemplo reutilizables y la generación de una línea base documentada del entorno.

## Objetivos de Aprendizaje

- [ ] Verificar las versiones de Windows Server 2022, SQL Server 2025 Developer Edition y SSMS requeridas para el curso.
- [ ] Validar el estado de los servicios SQL Server, SQL Server Agent, TCP/IP y el puerto 1433.
- [ ] Conectarse a la instancia `LOCALHOST,1433` mediante autenticación de Windows y validar la autenticación SQL de laboratorio.
- [ ] Crear y configurar la base de datos `Sql2025Lab` con Query Store, recuperación FULL y compatibilidad 170.
- [ ] Crear una línea base de configuración y rendimiento inicial en `C:\SQLLab2025\Reports\Lab01_EnvironmentBaseline.md`.

## Prerrequisitos

**Conocimientos requeridos**

- Conocimientos básicos de Windows Server y administración de servicios.
- Capacidad para abrir SQL Server Management Studio y ejecutar scripts T-SQL.
- Comprensión básica de instancia, base de datos, archivo de datos, archivo de log y conectividad TCP/IP.
- Comprensión de que una configuración moderna de SQL Server depende del motor, Windows Server, red, almacenamiento, identidad y observabilidad.

**Acceso requerido**

- Acceso local interactivo al servidor `SQLLAB-WS2022`.
- Cuenta de Windows `.\SqlLabAdmin` con privilegios de administrador local.
- Acceso a SQL Server mediante autenticación de Windows.
- Login SQL de laboratorio `lab_sa` disponible para pruebas de conectividad. No se debe incluir la contraseña en scripts, archivos de resultados ni repositorios.
- Permisos `sysadmin` o permisos equivalentes de administración de instancia para crear la base de datos y revisar configuración.

## Entorno de Laboratorio

| Componente | Configuración esperada |
|---|---|
| Nombre del servidor | `SQLLAB-WS2022` |
| Sistema operativo | Windows Server 2022 Datacenter 21H2, build `20348.2402` |
| Instancia SQL Server | Instancia predeterminada `MSSQLSERVER` |
| Conexión principal | `LOCALHOST,1433` |
| SQL Server | SQL Server 2025 Developer Edition, versión `17.0.1000.7` |
| SSMS | SQL Server Management Studio `21.3.2` |
| Base de datos común | `Sql2025Lab` |
| Nivel de compatibilidad | `170` |
| Modelo de recuperación | `FULL` |
| Query Store | Habilitado y en `READ_WRITE` |
| Memoria máxima SQL Server | `16384 MB` |
| MAXDOP | `4` |
| Cost threshold for parallelism | `50` |

| Recurso | Valor mínimo de referencia |
|---|---|
| CPU | 8 vCPU; se recomiendan 12 vCPU |
| Memoria RAM | 24 GB; se recomiendan 32 GB |
| Almacenamiento libre | 120 GB SSD/NVMe |
| Red | TCP 1433 disponible y HTTPS saliente por 443 |
| Cuenta administrativa | `.\SqlLabAdmin` |

Las rutas estándar del laboratorio son las siguientes:

| Uso | Ruta |
|---|---|
| Scripts | `C:\SQLLab2025\Scripts` |
| Informes | `C:\SQLLab2025\Reports` |
| Datos secundarios | `C:\SQLLab2025\Data` |
| Logs | `C:\SQLLab2025\Logs` |
| Copias de seguridad | `C:\SQLLab2025\Backups` |
| Auditoría | `C:\SQLLab2025\Audit` |
| Certificados y claves | `C:\SQLLab2025\Keys` |
| Extended Events | `C:\SQLLab2025\XEvents` |

Ejecute PowerShell **como administrador** y cree la estructura de directorios si aún no existe:

```powershell
$LabFolders = @(
    'C:\SQLLab2025\Scripts',
    'C:\SQLLab2025\Reports',
    'C:\SQLLab2025\Data',
    'C:\SQLLab2025\Logs',
    'C:\SQLLab2025\Backups',
    'C:\SQLLab2025\Audit',
    'C:\SQLLab2025\Keys',
    'C:\SQLLab2025\XEvents'
)

$LabFolders | ForEach-Object {
    New-Item -Path $_ -ItemType Directory -Force | Out-Null
}

Get-ChildItem C:\SQLLab2025 -Directory
```

> **Nota operativa:** la cuenta de servicio del motor SQL Server debe disponer de permisos de modificación sobre las carpetas que almacenarán archivos administrados por SQL Server, especialmente `Data`, `Logs`, `Backups`, `Audit` y `XEvents`. No otorgue permisos amplios a cuentas no relacionadas con el servicio.

## Instrucciones Paso a Paso

### Paso 1: Verificar Windows Server y preparar las rutas del laboratorio

**Objetivo**

Confirmar que el servidor corresponde al entorno de laboratorio definido y que las rutas estándar están disponibles antes de crear archivos administrados por SQL Server.

**Instrucciones**

1. Inicie sesión en `SQLLAB-WS2022` con la cuenta `.\SqlLabAdmin`.

2. Abra el cuadro **Ejecutar** con `Win + R`, escriba el siguiente comando y presione **Enter**:

   ```text
   winver
   ```

3. Confirme que se muestra Windows Server 2022 Datacenter, versión 21H2, con compilación `20348.2402`.

4. Abra PowerShell como administrador y ejecute:

   ```powershell
   $OS = Get-ComputerInfo

   [PSCustomObject]@{
       ComputerName      = $env:COMPUTERNAME
       WindowsProduct    = $OS.WindowsProductName
       WindowsVersion    = $OS.WindowsVersion
       OsBuildNumber     = $OS.OsBuildNumber
       CsManufacturer    = $OS.CsManufacturer
       CsModel           = $OS.CsModel
       LogicalProcessors = $OS.CsNumberOfLogicalProcessors
       TotalMemoryGB     = [math]::Round($OS.CsTotalPhysicalMemory / 1GB, 2)
   } | Format-List
   ```

5. Cree o valide las rutas estándar mediante el script presentado en la sección anterior.

6. Compruebe el espacio libre de las unidades locales:

   ```powershell
   Get-Volume |
       Where-Object { $_.DriveLetter } |
       Select-Object DriveLetter,
                     FileSystemLabel,
                     @{Name='SizeGB';Expression={[math]::Round($_.Size / 1GB, 2)}},
                     @{Name='FreeGB';Expression={[math]::Round($_.SizeRemaining / 1GB, 2)}} |
       Format-Table -AutoSize
   ```

7. Abra **Administrador del servidor** y revise que no existan alertas críticas de servicios, discos o eventos del sistema que puedan afectar a la práctica.

**Resultado esperado**

- El nombre del equipo es `SQLLAB-WS2022`.
- El sistema operativo corresponde a Windows Server 2022 Datacenter 21H2.
- La compilación del sistema operativo es `20348.2402`.
- Existen las carpetas bajo `C:\SQLLab2025`.
- Hay espacio suficiente para archivos de datos, logs, copias de seguridad e informes.
- El servidor dispone de al menos 8 procesadores lógicos y de la memoria definida para el laboratorio.

**Verificación**

Ejecute:

```powershell
Test-Path C:\SQLLab2025\Scripts
Test-Path C:\SQLLab2025\Reports
Test-Path C:\SQLLab2025\Data
Test-Path C:\SQLLab2025\Logs
```

Las cuatro instrucciones deben devolver `True`.

---

### Paso 2: Validar servicios SQL Server, TCP/IP y puerto 1433

**Objetivo**

Comprobar que el motor de SQL Server y SQL Server Agent están en ejecución, que el protocolo TCP/IP está habilitado y que la instancia predeterminada escucha en el puerto TCP 1433.

**Instrucciones**

1. Abra `services.msc` desde el cuadro **Ejecutar**.

2. Localice los servicios siguientes:

   | Servicio esperado | Estado esperado | Tipo de inicio recomendado para el laboratorio |
   |---|---:|---|
   | SQL Server (`MSSQLSERVER`) | En ejecución | Automático |
   | SQL Server Agent (`MSSQLSERVER`) | En ejecución | Automático o Manual, según política del laboratorio |
   | SQL Server Browser | No requerido | Detenido o deshabilitado |

3. No inicie SQL Server Browser como requisito de esta práctica. La instancia es predeterminada y se accede explícitamente mediante `LOCALHOST,1433`.

4. Abra **SQL Server Configuration Manager**. Puede intentar abrirlo desde PowerShell:

   ```powershell
   Start-Process "SQLServerManager17.msc"
   ```

   Si el comando no encuentra el archivo, abra manualmente SQL Server Configuration Manager desde el menú Inicio o revise la consola correspondiente a la instalación.

5. En **SQL Server Network Configuration** > **Protocols for MSSQLSERVER**, confirme que **TCP/IP** está habilitado.

6. Abra las propiedades de **TCP/IP**, vaya a la pestaña **IP Addresses** y confirme, en la sección **IPAll**, que el puerto TCP configurado es `1433` y que no se utiliza un puerto dinámico para esta práctica.

7. Si realizó cambios de protocolo o puerto, reinicie únicamente el servicio **SQL Server (`MSSQLSERVER`)** y espere a que vuelva al estado `Running`.

8. En PowerShell, valide los servicios y el puerto:

   ```powershell
   Get-Service -Name MSSQLSERVER, SQLSERVERAGENT |
       Select-Object Name, Status, StartType

   Test-NetConnection -ComputerName LOCALHOST -Port 1433

   Get-NetTCPConnection -LocalPort 1433 -State Listen -ErrorAction SilentlyContinue |
       Select-Object LocalAddress, LocalPort, State, OwningProcess
   ```

9. Si `Get-NetTCPConnection` no produce resultados, ejecute:

   ```powershell
   netstat -ano | findstr :1433
   ```

**Resultado esperado**

- `MSSQLSERVER` está en estado `Running`.
- `SQLSERVERAGENT` está disponible y, preferentemente, en estado `Running`.
- TCP/IP está habilitado para la instancia.
- `Test-NetConnection` informa `TcpTestSucceeded : True`.
- Se observa una escucha local en TCP 1433.

**Verificación**

El siguiente comando debe mostrar una conexión TCP correcta:

```powershell
Test-NetConnection LOCALHOST -Port 1433 |
    Select-Object ComputerName, RemotePort, TcpTestSucceeded
```

El valor de `TcpTestSucceeded` debe ser `True`.

---

### Paso 3: Verificar SSMS y conectarse a la instancia SQL Server

**Objetivo**

Confirmar la versión de SSMS requerida y establecer conectividad con la instancia mediante autenticación de Windows y autenticación SQL de laboratorio.

**Instrucciones**

1. En PowerShell, compruebe la versión del ejecutable de SSMS:

   ```powershell
   $SsmsPath = 'C:\Program Files (x86)\Microsoft SQL Server Management Studio 21\Common7\IDE\Ssms.exe'

   if (Test-Path $SsmsPath) {
       (Get-Item $SsmsPath).VersionInfo |
           Select-Object ProductName, ProductVersion, FileVersion
   }
   else {
       Write-Warning "No se encontró SSMS en la ruta esperada. Ubique Ssms.exe y consulte su versión."
   }
   ```

2. Confirme que la versión instalada corresponde a SSMS `21.3.2`.

3. Abra SQL Server Management Studio.

4. En la ventana **Connect to Server**, configure:

   | Campo | Valor |
   |---|---|
   | Server type | Database Engine |
   | Server name | `LOCALHOST,1433` |
   | Authentication | Windows Authentication |
   | User name | `SQLLAB-WS2022\SqlLabAdmin` o `.\SqlLabAdmin` |

5. Seleccione **Connect**.

6. Abra una ventana de consulta y ejecute:

   ```sql
   SELECT
       ORIGINAL_LOGIN() AS login_original,
       SUSER_SNAME() AS login_actual,
       HOST_NAME() AS nombre_cliente,
       APP_NAME() AS aplicacion,
       @@SERVERNAME AS nombre_instancia;
   ```

7. Pruebe la autenticación SQL sin registrar la contraseña en scripts:

   - Abra una nueva conexión en SSMS.
   - Use `LOCALHOST,1433`.
   - Seleccione **SQL Server Authentication**.
   - Escriba el login `lab_sa`.
   - Introduzca manualmente la contraseña temporal de laboratorio.
   - Después de una conexión satisfactoria, cierre esta sesión de prueba.

8. No guarde contraseñas en conexiones registradas, scripts `.sql`, historial de terminal, archivos Markdown ni capturas compartidas.

**Resultado esperado**

- SSMS informa versión `21.3.2`.
- La conexión de Windows a `LOCALHOST,1433` se realiza correctamente.
- La consulta identifica la cuenta de Windows del laboratorio.
- La autenticación SQL con `lab_sa` funciona exclusivamente como prueba controlada.

**Verificación**

En la conexión mediante autenticación de Windows, ejecute:

```sql
SELECT
    SERVERPROPERTY('ServerName') AS nombre_servidor,
    SERVERPROPERTY('InstanceName') AS nombre_instancia,
    CONNECTIONPROPERTY('net_transport') AS transporte_red,
    CONNECTIONPROPERTY('local_tcp_port') AS puerto_local;
```

El transporte debe ser `TCP` y el puerto local debe ser `1433`.

---

### Paso 4: Ejecutar el inventario técnico de la instancia

**Objetivo**

Documentar versión, edición, configuración, capacidad de CPU y memoria, rutas predeterminadas y estado de escucha de SQL Server. Esta información constituye la línea base técnica antes de crear la base de datos de laboratorio.

**Instrucciones**

1. En SSMS, conéctese mediante autenticación de Windows a `LOCALHOST,1433`.

2. Cree el archivo `C:\SQLLab2025\Scripts\Lab01_Inventory.sql`.

3. Copie y ejecute el siguiente script:

   ```sql
   USE master;
   GO

   SET NOCOUNT ON;
   GO

   SELECT
       SERVERPROPERTY('ServerName') AS nombre_servidor,
       SERVERPROPERTY('MachineName') AS nombre_equipo,
       SERVERPROPERTY('InstanceName') AS nombre_instancia,
       SERVERPROPERTY('Edition') AS edicion_sql_server,
       SERVERPROPERTY('ProductVersion') AS version_producto,
       SERVERPROPERTY('ProductLevel') AS nivel_actualizacion,
       SERVERPROPERTY('ProductMajorVersion') AS version_principal,
       SERVERPROPERTY('EngineEdition') AS tipo_motor,
       SERVERPROPERTY('IsClustered') AS es_instancia_en_cluster,
       @@VERSION AS version_completa;
   GO

   SELECT
       cpu_count AS cpu_logicas_sql_server,
       scheduler_count AS programadores_sql,
       numa_node_count AS nodos_numa,
       physical_memory_kb / 1024 AS memoria_fisica_mb,
       committed_kb / 1024 AS memoria_comprometida_sql_mb,
       committed_target_kb / 1024 AS memoria_objetivo_sql_mb,
       sqlserver_start_time AS inicio_servicio_sql
   FROM sys.dm_os_sys_info;
   GO

   SELECT
       name AS parametro,
       value AS valor_configurado,
       value_in_use AS valor_en_uso,
       description AS descripcion
   FROM sys.configurations
   WHERE name IN
   (
       N'max server memory (MB)',
       N'max degree of parallelism',
       N'cost threshold for parallelism'
   )
   ORDER BY name;
   GO

   SELECT
       SERVERPROPERTY('InstanceDefaultDataPath') AS ruta_predeterminada_datos,
       SERVERPROPERTY('InstanceDefaultLogPath') AS ruta_predeterminada_logs,
       SERVERPROPERTY('InstanceDefaultBackupPath') AS ruta_predeterminada_backups;
   GO

   SELECT
       ip_address AS direccion_ip,
       port AS puerto,
       type_desc AS tipo_escucha,
       state_desc AS estado
   FROM sys.dm_tcp_listener_states
   ORDER BY port, ip_address;
   GO

   SELECT
       name AS base_de_datos,
       compatibility_level AS nivel_compatibilidad,
       recovery_model_desc AS modelo_recuperacion,
       state_desc AS estado,
       is_query_store_on AS query_store_habilitado
   FROM sys.databases
   ORDER BY name;
   GO
   ```

4. Revise el conjunto de resultados de versión y confirme que la edición es **Developer Edition** y la versión del producto es `17.0.1000.7`.

5. Revise el resultado de configuración y confirme los valores iniciales del laboratorio:

   - `max server memory (MB) = 16384`
   - `max degree of parallelism = 4`
   - `cost threshold for parallelism = 50`

6. Si alguno de estos valores difiere, ejecútelos únicamente en el entorno de laboratorio:

   ```sql
   USE master;
   GO

   EXEC sys.sp_configure N'show advanced options', 1;
   RECONFIGURE;
   GO

   EXEC sys.sp_configure N'max server memory (MB)', 16384;
   RECONFIGURE;
   GO

   EXEC sys.sp_configure N'max degree of parallelism', 4;
   RECONFIGURE;
   GO

   EXEC sys.sp_configure N'cost threshold for parallelism', 50;
   RECONFIGURE;
   GO
   ```

7. Vuelva a ejecutar la sección de `sys.configurations` para confirmar que `value_in_use` coincide con la configuración esperada.

**Resultado esperado**

- SQL Server informa edición Developer Edition.
- La versión del producto es `17.0.1000.7`.
- La instancia no requiere SQL Server Browser.
- Existe un listener TCP en el puerto 1433.
- Las rutas predeterminadas de datos, logs y backups se muestran correctamente.
- Los parámetros iniciales de rendimiento coinciden con la configuración del laboratorio.

**Verificación**

Ejecute esta consulta resumida:

```sql
SELECT
    SERVERPROPERTY('Edition') AS edicion,
    SERVERPROPERTY('ProductVersion') AS version_producto,
    SERVERPROPERTY('InstanceDefaultDataPath') AS ruta_datos,
    SERVERPROPERTY('InstanceDefaultLogPath') AS ruta_logs;
```

La edición debe contener `Developer Edition` y la versión debe comenzar por `17.0.1000.7`.

---

### Paso 5: Crear Sql2025Lab, esquemas y datos de ejemplo

**Objetivo**

Crear la base de datos común de las prácticas, habilitar Query Store y cargar un conjunto de datos suficiente para generar planes de ejecución, métricas y consultas reproducibles en prácticas posteriores.

**Instrucciones**

1. Confirme que las carpetas `C:\SQLLab2025\Data` y `C:\SQLLab2025\Logs` existen y que la cuenta de servicio SQL Server puede escribir en ellas.

2. En SSMS, cree el archivo `C:\SQLLab2025\Scripts\Lab01_CreateSql2025Lab.sql`.

3. Ejecute el siguiente script. El script está diseñado para crear la base de datos solo si no existe; no elimina una base creada previamente.

   ```sql
   USE master;
   GO

   SET NOCOUNT ON;
   GO

   IF DB_ID(N'Sql2025Lab') IS NULL
   BEGIN
       CREATE DATABASE Sql2025Lab
       ON PRIMARY
       (
           NAME = N'Sql2025Lab_Primary',
           FILENAME = N'C:\SQLLab2025\Data\Sql2025Lab_Primary.mdf',
           SIZE = 256MB,
           FILEGROWTH = 64MB
       )
       LOG ON
       (
           NAME = N'Sql2025Lab_Log',
           FILENAME = N'C:\SQLLab2025\Logs\Sql2025Lab_Log.ldf',
           SIZE = 128MB,
           FILEGROWTH = 64MB
       );
   END;
   GO

   ALTER DATABASE Sql2025Lab SET COMPATIBILITY_LEVEL = 170;
   ALTER DATABASE Sql2025Lab SET RECOVERY FULL;
   ALTER DATABASE Sql2025Lab SET AUTO_CLOSE OFF;
   ALTER DATABASE Sql2025Lab SET AUTO_SHRINK OFF;
   ALTER DATABASE Sql2025Lab SET PAGE_VERIFY CHECKSUM;
   GO

   ALTER DATABASE Sql2025Lab SET QUERY_STORE = ON;
   GO

   ALTER DATABASE Sql2025Lab SET QUERY_STORE
   (
       OPERATION_MODE = READ_WRITE,
       QUERY_CAPTURE_MODE = AUTO,
       CLEANUP_POLICY = (STALE_QUERY_THRESHOLD_DAYS = 30),
       DATA_FLUSH_INTERVAL_SECONDS = 900,
       INTERVAL_LENGTH_MINUTES = 60,
       MAX_STORAGE_SIZE_MB = 256
   );
   GO

   USE Sql2025Lab;
   GO

   IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'Sales')
       EXEC(N'CREATE SCHEMA Sales AUTHORIZATION dbo;');
   GO

   IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'Reporting')
       EXEC(N'CREATE SCHEMA Reporting AUTHORIZATION dbo;');
   GO

   IF NOT EXISTS (SELECT 1 FROM sys.schemas WHERE name = N'Security')
       EXEC(N'CREATE SCHEMA Security AUTHORIZATION dbo;');
   GO

   IF OBJECT_ID(N'dbo.LabExecutionLog', N'U') IS NULL
   BEGIN
       CREATE TABLE dbo.LabExecutionLog
       (
           ExecutionLogId bigint IDENTITY(1,1) NOT NULL
               CONSTRAINT PK_LabExecutionLog PRIMARY KEY,
           LabCode varchar(20) NOT NULL,
           StepName nvarchar(200) NOT NULL,
           ExecutedAt datetime2(0) NOT NULL
               CONSTRAINT DF_LabExecutionLog_ExecutedAt DEFAULT SYSUTCDATETIME(),
           ExecutedBy sysname NOT NULL
               CONSTRAINT DF_LabExecutionLog_ExecutedBy DEFAULT ORIGINAL_LOGIN(),
           Status varchar(20) NOT NULL
               CONSTRAINT DF_LabExecutionLog_Status DEFAULT ('Completed'),
           Notes nvarchar(1000) NULL
       );
   END;
   GO

   IF OBJECT_ID(N'Sales.Customers', N'U') IS NULL
   BEGIN
       CREATE TABLE Sales.Customers
       (
           CustomerId int IDENTITY(1,1) NOT NULL
               CONSTRAINT PK_Customers PRIMARY KEY,
           CustomerCode varchar(20) NOT NULL
               CONSTRAINT UQ_Customers_CustomerCode UNIQUE,
           CustomerName nvarchar(150) NOT NULL,
           Email nvarchar(254) NOT NULL,
           CountryCode char(2) NOT NULL,
           CreatedDate date NOT NULL,
           IsActive bit NOT NULL
               CONSTRAINT DF_Customers_IsActive DEFAULT (1)
       );
   END;
   GO

   IF OBJECT_ID(N'Sales.Orders', N'U') IS NULL
   BEGIN
       CREATE TABLE Sales.Orders
       (
           OrderId bigint IDENTITY(1,1) NOT NULL
               CONSTRAINT PK_Orders PRIMARY KEY,
           CustomerId int NOT NULL,
           OrderDate datetime2(0) NOT NULL,
           OrderStatus varchar(20) NOT NULL,
           SalesChannel varchar(20) NOT NULL,
           TotalAmount decimal(12,2) NOT NULL,
           CreatedAt datetime2(0) NOT NULL
               CONSTRAINT DF_Orders_CreatedAt DEFAULT SYSUTCDATETIME(),
           CONSTRAINT FK_Orders_Customers
               FOREIGN KEY (CustomerId)
               REFERENCES Sales.Customers(CustomerId)
       );
   END;
   GO

   IF OBJECT_ID(N'Sales.OrderLines', N'U') IS NULL
   BEGIN
       CREATE TABLE Sales.OrderLines
       (
           OrderLineId bigint IDENTITY(1,1) NOT NULL
               CONSTRAINT PK_OrderLines PRIMARY KEY,
           OrderId bigint NOT NULL,
           LineNumber tinyint NOT NULL,
           ProductCode varchar(30) NOT NULL,
           Quantity smallint NOT NULL,
           UnitPrice decimal(12,2) NOT NULL,
           DiscountAmount decimal(12,2) NOT NULL
               CONSTRAINT DF_OrderLines_DiscountAmount DEFAULT (0),
           LineAmount AS
               CONVERT(decimal(12,2), (Quantity * UnitPrice) - DiscountAmount) PERSISTED,
           CONSTRAINT UQ_OrderLines_Order_Line UNIQUE (OrderId, LineNumber),
           CONSTRAINT FK_OrderLines_Orders
               FOREIGN KEY (OrderId)
               REFERENCES Sales.Orders(OrderId)
       );
   END;
   GO

   IF NOT EXISTS
   (
       SELECT 1
       FROM sys.indexes
       WHERE object_id = OBJECT_ID(N'Sales.Orders')
         AND name = N'IX_Orders_CustomerId_OrderDate'
   )
   BEGIN
       CREATE INDEX IX_Orders_CustomerId_OrderDate
       ON Sales.Orders (CustomerId, OrderDate DESC)
       INCLUDE (OrderStatus, SalesChannel, TotalAmount);
   END;
   GO

   IF NOT EXISTS
   (
       SELECT 1
       FROM sys.indexes
       WHERE object_id = OBJECT_ID(N'Sales.Orders')
         AND name = N'IX_Orders_OrderDate'
   )
   BEGIN
       CREATE INDEX IX_Orders_OrderDate
       ON Sales.Orders (OrderDate DESC)
       INCLUDE (CustomerId, OrderStatus, SalesChannel, TotalAmount);
   END;
   GO

   IF NOT EXISTS
   (
       SELECT 1
       FROM sys.indexes
       WHERE object_id = OBJECT_ID(N'Sales.OrderLines')
         AND name = N'IX_OrderLines_ProductCode'
   )
   BEGIN
       CREATE INDEX IX_OrderLines_ProductCode
       ON Sales.OrderLines (ProductCode)
       INCLUDE (OrderId, Quantity, UnitPrice, DiscountAmount, LineAmount);
   END;
   GO

   IF NOT EXISTS (SELECT 1 FROM Sales.Customers)
   BEGIN
       ;WITH Numbers AS
       (
           SELECT TOP (10000)
               ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS NumberValue
           FROM sys.all_objects AS a
           CROSS JOIN sys.all_objects AS b
       )
       INSERT INTO Sales.Customers
       (
           CustomerCode,
           CustomerName,
           Email,
           CountryCode,
           CreatedDate,
           IsActive
       )
       SELECT
           CONCAT('CUST', RIGHT(CONCAT('000000', NumberValue), 6)),
           CONCAT(N'Cliente de laboratorio ', NumberValue),
           CONCAT(N'cliente', NumberValue, N'@sqllab.example'),
           CASE NumberValue % 5
               WHEN 0 THEN 'ES'
               WHEN 1 THEN 'MX'
               WHEN 2 THEN 'CO'
               WHEN 3 THEN 'AR'
               ELSE 'CL'
           END,
           DATEADD(DAY, -(NumberValue % 1825), CONVERT(date, '2025-01-01')),
           CASE WHEN NumberValue % 20 = 0 THEN 0 ELSE 1 END
       FROM Numbers;
   END;
   GO

   IF NOT EXISTS (SELECT 1 FROM Sales.Orders)
   BEGIN
       ;WITH Numbers AS
       (
           SELECT TOP (100000)
               ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS NumberValue
           FROM sys.all_objects AS a
           CROSS JOIN sys.all_objects AS b
       )
       INSERT INTO Sales.Orders
       (
           CustomerId,
           OrderDate,
           OrderStatus,
           SalesChannel,
           TotalAmount
       )
       SELECT
           ((NumberValue - 1) % 10000) + 1,
           DATEADD(MINUTE, -(NumberValue * 11), CONVERT(datetime2(0), '2025-12-31 23:59:00')),
           CASE
               WHEN NumberValue % 25 = 0 THEN 'Cancelled'
               WHEN NumberValue % 10 = 0 THEN 'Pending'
               ELSE 'Completed'
           END,
           CASE NumberValue % 3
               WHEN 0 THEN 'Web'
               WHEN 1 THEN 'Store'
               ELSE 'Partner'
           END,
           CONVERT(decimal(12,2), 50 + ((NumberValue % 200) * 7.35))
       FROM Numbers;
   END;
   GO

   IF NOT EXISTS (SELECT 1 FROM Sales.OrderLines)
   BEGIN
       INSERT INTO Sales.OrderLines
       (
           OrderId,
           LineNumber,
           ProductCode,
           Quantity,
           UnitPrice,
           DiscountAmount
       )
       SELECT
           o.OrderId,
           n.LineNumber,
           CONCAT('PRD-', RIGHT(CONCAT('0000', ((o.OrderId + n.LineNumber) % 750) + 1), 4)),
           CONVERT(smallint, ((o.OrderId + n.LineNumber) % 5) + 1),
           CONVERT(decimal(12,2), 10 + (((o.OrderId + n.LineNumber) % 100) * 2.75)),
           CASE
               WHEN (o.OrderId + n.LineNumber) % 10 = 0
                   THEN CONVERT(decimal(12,2), 5.00)
               ELSE CONVERT(decimal(12,2), 0.00)
           END
       FROM Sales.Orders AS o
       CROSS JOIN
       (
           VALUES (CONVERT(tinyint, 1)),
                  (CONVERT(tinyint, 2)),
                  (CONVERT(tinyint, 3))
       ) AS n(LineNumber);
   END;
   GO

   UPDATE STATISTICS Sales.Customers WITH FULLSCAN;
   UPDATE STATISTICS Sales.Orders WITH FULLSCAN;
   UPDATE STATISTICS Sales.OrderLines WITH FULLSCAN;
   GO

   INSERT INTO dbo.LabExecutionLog
   (
       LabCode,
       StepName,
       Status,
       Notes
   )
   VALUES
   (
       '01-00-01',
       N'Creación de Sql2025Lab y carga inicial',
       'Completed',
       N'Base de datos configurada con compatibilidad 170, FULL y Query Store READ_WRITE.'
   );
   GO
   ```

4. Espere a que termine la carga de datos. No cancele el proceso, ya que las tablas y estadísticas deben quedar en un estado coherente para las prácticas posteriores.

5. Ejecute la siguiente consulta para comprobar el volumen cargado:

   ```sql
   USE Sql2025Lab;
   GO

   SELECT
       N'Sales.Customers' AS tabla,
       COUNT_BIG(*) AS filas
   FROM Sales.Customers

   UNION ALL

   SELECT
       N'Sales.Orders',
       COUNT_BIG(*)
   FROM Sales.Orders

   UNION ALL

   SELECT
       N'Sales.OrderLines',
       COUNT_BIG(*)
   FROM Sales.OrderLines

   UNION ALL

   SELECT
       N'dbo.LabExecutionLog',
       COUNT_BIG(*)
   FROM dbo.LabExecutionLog;
   GO
   ```

**Resultado esperado**

- La base de datos `Sql2025Lab` existe.
- El nivel de compatibilidad es `170`.
- El modelo de recuperación es `FULL`.
- Query Store está habilitado en modo `READ_WRITE`.
- Existen los esquemas `Sales`, `Reporting`, `Security` y `dbo`.
- Las tablas `Sales.Customers`, `Sales.Orders`, `Sales.OrderLines` y `dbo.LabExecutionLog` existen.
- La carga inicial contiene aproximadamente:
  - `10,000` clientes.
  - `100,000` pedidos.
  - `300,000` líneas de pedido.

**Verificación**

Ejecute:

```sql
USE Sql2025Lab;
GO

SELECT
    d.name AS base_de_datos,
    d.compatibility_level,
    d.recovery_model_desc,
    d.is_query_store_on,
    q.actual_state_desc AS estado_query_store
FROM sys.databases AS d
LEFT JOIN sys.database_query_store_options AS q
    ON d.database_id = q.database_id
WHERE d.name = N'Sql2025Lab';
GO
```

El resultado debe indicar:

- `compatibility_level = 170`
- `recovery_model_desc = FULL`
- `is_query_store_on = 1`
- `estado_query_store = READ_WRITE`

---

### Paso 6: Generar consultas de línea base y documentar el entorno

**Objetivo**

Ejecutar una carga de consulta controlada para inicializar métricas en Query Store y crear el informe `Lab01_EnvironmentBaseline.md`.

**Instrucciones**

1. En SSMS, active el plan de ejecución real con **Include Actual Execution Plan** o mediante `Ctrl + M`.

2. Ejecute el siguiente bloque dos veces en la base de datos `Sql2025Lab`. La repetición permite que Query Store capture la actividad de consulta.

   ```sql
   USE Sql2025Lab;
   GO

   SET NOCOUNT ON;
   SET STATISTICS IO ON;
   SET STATISTICS TIME ON;
   GO

   DECLARE @StartDate datetime2(0) = '2025-01-01 00:00:00';
   DECLARE @EndDate datetime2(0) = '2025-12-31 23:59:59';
   DECLARE @CustomerId int = 500;

   SELECT
       c.CustomerCode,
       c.CustomerName,
       COUNT_BIG(o.OrderId) AS total_pedidos,
       SUM(o.TotalAmount) AS importe_total,
       MAX(o.OrderDate) AS ultimo_pedido
   FROM Sales.Customers AS c
   INNER JOIN Sales.Orders AS o
       ON o.CustomerId = c.CustomerId
   WHERE c.CustomerId = @CustomerId
     AND o.OrderDate >= @StartDate
     AND o.OrderDate <= @EndDate
   GROUP BY
       c.CustomerCode,
       c.CustomerName;
   GO

   SELECT TOP (20)
       ol.ProductCode,
       SUM(ol.Quantity) AS unidades_vendidas,
       SUM(ol.LineAmount) AS ingresos_linea
   FROM Sales.OrderLines AS ol
   INNER JOIN Sales.Orders AS o
       ON o.OrderId = ol.OrderId
   WHERE o.OrderStatus = 'Completed'
     AND o.OrderDate >= @StartDate
     AND o.OrderDate <= @EndDate
   GROUP BY ol.ProductCode
   ORDER BY ingresos_linea DESC;
   GO

   SET STATISTICS IO OFF;
   SET STATISTICS TIME OFF;
   GO
   ```

3. Revise el panel de mensajes. Registre de forma general los valores de lecturas lógicas y tiempo de CPU; no es necesario que sean idénticos entre equipos.

4. Compruebe que Query Store tiene consultas capturadas:

   ```sql
   USE Sql2025Lab;
   GO

   SELECT TOP (10)
       q.query_id,
       p.plan_id,
       rs.count_executions,
       CAST(rs.avg_duration / 1000.0 AS decimal(18,2)) AS duracion_promedio_ms,
       CAST(rs.avg_cpu_time / 1000.0 AS decimal(18,2)) AS cpu_promedio_ms,
       qt.query_sql_text
   FROM sys.query_store_query AS q
   INNER JOIN sys.query_store_query_text AS qt
       ON q.query_text_id = qt.query_text_id
   INNER JOIN sys.query_store_plan AS p
       ON q.query_id = p.query_id
   INNER JOIN sys.query_store_runtime_stats AS rs
       ON p.plan_id = rs.plan_id
   ORDER BY rs.last_execution_time DESC;
   GO
   ```

5. Cree el archivo `C:\SQLLab2025\Reports\Lab01_EnvironmentBaseline.md`.

6. Copie la siguiente plantilla en el archivo y complete los campos entre corchetes con los resultados obtenidos. No incluya contraseñas, cadenas de conexión con credenciales ni datos no autorizados.

   ```markdown
   # Línea base del entorno — Lab 01-00-01

   ## Identificación

   | Elemento | Valor observado |
   |---|---|
   | Fecha y hora de validación | [AAAA-MM-DD HH:MM] |
   | Servidor | SQLLAB-WS2022 |
   | Usuario ejecutor | [cuenta Windows] |
   | Windows Server | [edición, versión y build] |
   | CPU lógica | [valor] |
   | Memoria física | [valor GB] |

   ## SQL Server

   | Elemento | Valor observado |
   |---|---|
   | Instancia | MSSQLSERVER |
   | Punto de conexión | LOCALHOST,1433 |
   | Edición | [Developer Edition] |
   | Versión | [17.0.1000.7] |
   | SSMS | [21.3.2] |
   | TCP/IP | Habilitado |
   | Puerto TCP | 1433 |
   | SQL Server Agent | [estado] |

   ## Parámetros de instancia

   | Parámetro | Valor esperado | Valor observado |
   |---|---:|---:|
   | max server memory (MB) | 16384 | [valor] |
   | max degree of parallelism | 4 | [valor] |
   | cost threshold for parallelism | 50 | [valor] |

   ## Base de datos Sql2025Lab

   | Elemento | Valor observado |
   |---|---|
   | Nivel de compatibilidad | 170 |
   | Modelo de recuperación | FULL |
   | Query Store | READ_WRITE |
   | Clientes | [recuento] |
   | Pedidos | [recuento] |
   | Líneas de pedido | [recuento] |

   ## Observaciones operativas

   - Separación lógica de aplicación y motor SQL Server: [confirmada o pendiente].
   - Almacenamiento de datos y logs: [rutas observadas].
   - Riesgos detectados: [ninguno o descripción].
   - Validación de conectividad TCP/IP: [correcta o descripción].
   - Próxima revisión recomendada: [fecha o práctica siguiente].
   ```

7. Agregue una entrada final al registro de ejecución:

   ```sql
   USE Sql2025Lab;
   GO

   INSERT INTO dbo.LabExecutionLog
   (
       LabCode,
       StepName,
       Status,
       Notes
   )
   VALUES
   (
       '01-00-01',
       N'Línea base de entorno documentada',
       'Completed',
       N'Informe generado en C:\SQLLab2025\Reports\Lab01_EnvironmentBaseline.md.'
   );
   GO
   ```

**Resultado esperado**

- Query Store registra consultas y planes de ejecución.
- Las consultas de referencia producen resultados sobre pedidos y líneas de pedido.
- El archivo `Lab01_EnvironmentBaseline.md` existe y contiene la línea base del entorno.
- La tabla `dbo.LabExecutionLog` contiene al menos dos entradas correspondientes a esta práctica.

**Verificación**

Ejecute:

```sql
USE Sql2025Lab;
GO

SELECT
    ExecutionLogId,
    LabCode,
    StepName,
    ExecutedAt,
    ExecutedBy,
    Status,
    Notes
FROM dbo.LabExecutionLog
WHERE LabCode = '01-00-01'
ORDER BY ExecutionLogId;
GO
```

En PowerShell, valide la existencia del informe:

```powershell
Get-Item C:\SQLLab2025\Reports\Lab01_EnvironmentBaseline.md |
    Select-Object FullName, Length, LastWriteTime
```

## Validación y Pruebas

Ejecute el siguiente script integral en SSMS. La práctica se considera completada cuando todas las validaciones presentan el valor esperado.

```sql
USE master;
GO

SELECT
    CASE
        WHEN SERVERPROPERTY('Edition') LIKE '%Developer%'
            THEN 'CORRECTO'
        ELSE 'REVISAR'
    END AS validacion_edicion,
    SERVERPROPERTY('Edition') AS edicion,
    SERVERPROPERTY('ProductVersion') AS version_producto;
GO

SELECT
    name AS parametro,
    value_in_use AS valor_en_uso,
    CASE
        WHEN name = N'max server memory (MB)' AND value_in_use = 16384 THEN 'CORRECTO'
        WHEN name = N'max degree of parallelism' AND value_in_use = 4 THEN 'CORRECTO'
        WHEN name = N'cost threshold for parallelism' AND value_in_use = 50 THEN 'CORRECTO'
        ELSE 'REVISAR'
    END AS estado
FROM sys.configurations
WHERE name IN
(
    N'max server memory (MB)',
    N'max degree of parallelism',
    N'cost threshold for parallelism'
)
ORDER BY name;
GO

SELECT
    ip_address,
    port,
    state_desc,
    CASE
        WHEN port = 1433 AND state_desc = 'ONLINE' THEN 'CORRECTO'
        ELSE 'REVISAR'
    END AS estado
FROM sys.dm_tcp_listener_states;
GO

USE Sql2025Lab;
GO

SELECT
    d.name,
    d.compatibility_level,
    d.recovery_model_desc,
    q.actual_state_desc AS query_store_estado,
    CASE
        WHEN d.compatibility_level = 170
         AND d.recovery_model_desc = 'FULL'
         AND q.actual_state_desc = 'READ_WRITE'
            THEN 'CORRECTO'
        ELSE 'REVISAR'
    END AS estado
FROM sys.databases AS d
INNER JOIN sys.database_query_store_options AS q
    ON d.database_id = q.database_id
WHERE d.name = N'Sql2025Lab';
GO

SELECT
    s.name AS esquema,
    t.name AS tabla,
    SUM(p.rows) AS filas
FROM sys.tables AS t
INNER JOIN sys.schemas AS s
    ON t.schema_id = s.schema_id
INNER JOIN sys.partitions AS p
    ON t.object_id = p.object_id
WHERE p.index_id IN (0, 1)
  AND
  (
      (s.name = N'Sales' AND t.name IN (N'Customers', N'Orders', N'OrderLines'))
      OR
      (s.name = N'dbo' AND t.name = N'LabExecutionLog')
  )
GROUP BY s.name, t.name
ORDER BY s.name, t.name;
GO

SELECT
    COUNT_BIG(*) AS planes_en_query_store
FROM sys.query_store_plan;
GO
```

Complete además estas validaciones operativas:

| Prueba | Resultado esperado |
|---|---|
| `Test-NetConnection LOCALHOST -Port 1433` | `TcpTestSucceeded = True` |
| Servicio `MSSQLSERVER` | `Running` |
| Servicio `SQLSERVERAGENT` | Disponible y normalmente `Running` |
| Autenticación Windows | Conexión satisfactoria a `LOCALHOST,1433` |
| Autenticación SQL con `lab_sa` | Conexión satisfactoria sin exponer la contraseña |
| Informe de línea base | Existe en `C:\SQLLab2025\Reports\Lab01_EnvironmentBaseline.md` |
| Query Store | Estado `READ_WRITE` y al menos un plan registrado |

## Solución de Problemas

**Incidencia 1: No es posible conectarse a `LOCALHOST,1433` y `Test-NetConnection` devuelve `TcpTestSucceeded : False`.**

- **Síntomas:** SSMS muestra errores de red o de instancia; `sqlcmd` no conecta; no aparece una escucha en `netstat -ano | findstr :1433`.
- **Causa probable:** el servicio `MSSQLSERVER` está detenido, TCP/IP está deshabilitado, el puerto configurado no es 1433 o no se reinició el servicio después de modificar la configuración de red.
- **Corrección:**
  1. Compruebe el estado del servicio con `Get-Service MSSQLSERVER`.
  2. Inicie el servicio si está detenido:
     ```powershell
     Start-Service MSSQLSERVER
     ```
  3. En SQL Server Configuration Manager, habilite TCP/IP para `MSSQLSERVER`.
  4. Confirme que `IPAll` usa el puerto TCP `1433` y que no existe un valor conflictivo de puerto dinámico.
  5. Reinicie `MSSQLSERVER`.
  6. Repita `Test-NetConnection LOCALHOST -Port 1433`.

**Incidencia 2: La creación de `Sql2025Lab` falla con “Access is denied” o error del sistema operativo al crear archivos `.mdf` o `.ldf`.**

- **Síntomas:** el comando `CREATE DATABASE` devuelve un error de acceso sobre `C:\SQLLab2025\Data` o `C:\SQLLab2025\Logs`.
- **Causa probable:** la cuenta de servicio del motor SQL Server no tiene permisos de escritura en las carpetas de datos o logs, o las rutas no existen.
- **Corrección:**
  1. Compruebe que existen las carpetas:
     ```powershell
     Test-Path C:\SQLLab2025\Data
     Test-Path C:\SQLLab2025\Logs
     ```
  2. Identifique la cuenta de servicio en SQL Server Configuration Manager, sección **SQL Server Services**.
  3. Otorgue permisos de modificación únicamente a esa cuenta de servicio sobre las rutas requeridas:
     ```powershell
     icacls C:\SQLLab2025\Data /grant "NT SERVICE\MSSQLSERVER:(OI)(CI)M"
     icacls C:\SQLLab2025\Logs /grant "NT SERVICE\MSSQLSERVER:(OI)(CI)M"
     ```
  4. Si la instancia utiliza una cuenta de dominio o una cuenta administrada distinta, sustituya `NT SERVICE\MSSQLSERVER` por la identidad real del servicio.
  5. Ejecute de nuevo el script de creación. No elimine archivos parcialmente creados sin confirmar previamente que no pertenecen a una base de datos activa.

## Limpieza

Esta práctica crea componentes que son prerrequisitos obligatorios de las prácticas posteriores. Por tanto, **no elimine** la base de datos `Sql2025Lab`, sus esquemas, tablas, índices, datos, Query Store ni el informe de línea base.

Realice únicamente las siguientes acciones de limpieza:

1. Cierre las sesiones de SSMS que hayan usado autenticación SQL con `lab_sa`.
2. Guarde los scripts ejecutados en:
   - `C:\SQLLab2025\Scripts\Lab01_Inventory.sql`
   - `C:\SQLLab2025\Scripts\Lab01_CreateSql2025Lab.sql`
3. Verifique que el informe permanece disponible:

   ```powershell
   Test-Path C:\SQLLab2025\Reports\Lab01_EnvironmentBaseline.md
   ```

4. Si creó archivos temporales de resultados o capturas que incluyan información sensible, elimínelos o almacénelos conforme a la política del laboratorio.
5. No reduzca manualmente el archivo de log ni ejecute operaciones de mantenimiento no solicitadas. La base queda preparada para las prácticas de rendimiento, seguridad, auditoría y migración.

## Resumen

En esta práctica se validó un entorno local moderno de SQL Server sobre Windows Server 2022, considerando componentes que influyen directamente en la operación de una plataforma on-premise: sistema operativo, memoria, CPU, servicios, conectividad TCP/IP, identidad y almacenamiento.

También se creó `Sql2025Lab` con configuración orientada al curso:

- Compatibilidad `170`.
- Modelo de recuperación `FULL`.
- Query Store en modo `READ_WRITE`.
- Esquemas `Sales`, `Reporting`, `Security` y `dbo`.
- Datos de ejemplo para consultas, planes de ejecución, índices y estadísticas.
- Registro operativo en `dbo.LabExecutionLog`.
- Línea base documentada en `C:\SQLLab2025\Reports\Lab01_EnvironmentBaseline.md`.

Esta base permitirá analizar rendimiento, seguridad, control de acceso, auditoría, cifrado y escenarios de migración desde SQL Server 2017 hacia SQL Server 2025 en las prácticas siguientes.

**Recursos recomendados**

- [Documentación de SQL Server en Microsoft Learn](https://learn.microsoft.com/es-es/sql/sql-server/)
- [Query Store en SQL Server](https://learn.microsoft.com/es-es/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store)
- [Configuración de red de SQL Server](https://learn.microsoft.com/es-es/sql/database-engine/configure-windows/configure-the-windows-firewall-to-allow-sql-server-access)
- [Información general de grupos de disponibilidad Always On](https://learn.microsoft.com/es-es/sql/database-engine/availability-groups/windows/overview-of-always-on-availability-groups-sql-server)
