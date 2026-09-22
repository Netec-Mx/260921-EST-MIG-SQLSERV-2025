# Optimización: Validación de configuración óptima del servidor y Ajustes básicos de performance

## Metadatos

| Duration | Complexity | Bloom level |
|---|---|---|
| 40 minutos | Medio | Aplicar |

## Descripción General

En esta práctica se evalúa la configuración de rendimiento de la instancia predeterminada `MSSQLSERVER` sobre Windows Server 2022 y se compara con una línea base controlada. Se revisan memoria, CPU, paralelismo, almacenamiento, TempDB, configuración de energía e Instant File Initialization (IFI), para después aplicar ajustes básicos y reversibles de SQL Server.

La práctica utiliza los valores iniciales definidos para el laboratorio: `max server memory = 16384 MB`, `MAXDOP = 4` y `cost threshold for parallelism = 50`. Windows Server 2025 se analiza únicamente como contexto tecnológico; todas las validaciones prácticas de este laboratorio se ejecutan en Windows Server 2022 Datacenter 21H2, build `20348.2402`. No se deben atribuir al sistema operativo instalado características exclusivas de Windows Server 2025, como mejoras específicas de Hyper-V, dMSA, SMB sobre QUIC u hotpatching.

## Objetivos de Aprendizaje

- [ ] Registrar una línea base de configuración, memoria, E/S y almacenamiento antes de realizar cambios.
- [ ] Aplicar valores controlados de memoria máxima, `MAXDOP` y `cost threshold for parallelism` en la instancia `MSSQLSERVER`.
- [ ] Validar que `tempdb` dispone de cuatro archivos de datos con tamaño y crecimiento homogéneos.
- [ ] Ejecutar y comparar las consultas de carga de la práctica 2 mediante indicadores de Query Store, tiempo, lecturas y E/S.
- [ ] Distinguir entre configuraciones aplicables al laboratorio Windows Server 2022 y capacidades que requerirían una evaluación separada en Windows Server 2025.

## Prerrequisitos

**Conocimientos requeridos**

- Haber completado la práctica 1, incluyendo la creación y configuración inicial de `Sql2025Lab`.
- Haber completado la práctica 2 y disponer de las consultas de carga o scripts de prueba utilizados en ella.
- Conocer los conceptos básicos de archivos de datos, archivos de log, `tempdb`, Query Store y planes de ejecución.
- Comprender que un cambio de configuración de instancia puede afectar a todas las bases de datos alojadas en SQL Server.

**Acceso y condiciones requeridas**

- Iniciar sesión en `SQLLAB-WS2022` con una cuenta con privilegios administrativos locales y permisos `sysadmin` en SQL Server.
- Conectarse a la instancia mediante `LOCALHOST,1433`.
- Disponer de una ventana de cambio aprobada para el laboratorio, ya que se cambiarán parámetros de instancia y se reiniciará el servicio SQL Server si es necesario mover archivos de `tempdb`.
- Confirmar que las rutas estándar existen o pueden ser creadas:
  - `C:\SQLLab2025\Scripts`
  - `C:\SQLLab2025\Reports`
  - `C:\SQLLab2025\Data`
  - `C:\SQLLab2025\Logs`
  - `C:\SQLLab2025\Backups`
- Usar la base de datos `Sql2025Lab`, configurada con nivel de compatibilidad `170`, modelo de recuperación `FULL` y Query Store en estado `READ_WRITE`.
- No modificar afinidad de CPU, trace flags globales, configuración de procesadores, configuración de red, políticas corporativas de energía ni parámetros avanzados no justificados.

> **Advertencia de seguridad:** las credenciales `lab_sa`, `lab_reader` y `lab_analyst` son exclusivas del laboratorio. No incluya contraseñas en capturas, repositorios, conversaciones con GitHub Copilot Chat ni documentación externa. Cambie las contraseñas temporales antes de reutilizar el entorno.

## Entorno de Laboratorio

| Componente | Valor del laboratorio |
|---|---|
| Servidor | `SQLLAB-WS2022` |
| Sistema operativo validado | Windows Server 2022 Datacenter 21H2, build `20348.2402` |
| Instancia SQL Server | Instancia predeterminada `MSSQLSERVER` |
| Conectividad | `LOCALHOST,1433` |
| Edición de SQL Server | SQL Server 2025 Developer Edition `17.0.1000.7` |
| Herramienta principal | SQL Server Management Studio `21.3.2` |
| Base de datos | `Sql2025Lab` |
| Compatibilidad | `170` |
| Query Store | `READ_WRITE` |
| Memoria máxima objetivo | `16384 MB` |
| MAXDOP objetivo | `4` |
| Umbral de costo objetivo | `50` |
| Archivos de datos TempDB | Cuatro archivos de datos de `1024 MB`, crecimiento de `256 MB` |
| Archivo de log TempDB | Un archivo de log con crecimiento fijo |
| Rutas de TempDB propuestas | Datos: `C:\SQLLab2025\Data`; log: `C:\SQLLab2025\Logs` |

Antes de iniciar, abra PowerShell como administrador y cree las rutas estándar. Si las rutas ya existen, el comando no genera error.

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

New-Item -ItemType Directory -Path $LabFolders -Force | Out-Null
```

Compruebe que el servicio SQL Server está iniciado y que el puerto esperado responde:

```powershell
Get-Service -Name MSSQLSERVER

Test-NetConnection -ComputerName localhost -Port 1433
```

Compruebe la versión del sistema operativo y el estado básico del almacenamiento. Estas comprobaciones son especialmente importantes porque las recomendaciones de E/S dependen del hardware y de la capa de virtualización subyacente.

```powershell
Get-ComputerInfo |
    Select-Object WindowsProductName, WindowsVersion, OsBuildNumber, CsNumberOfLogicalProcessors, CsTotalPhysicalMemory

Get-PhysicalDisk |
    Select-Object FriendlyName, MediaType, BusType, Size, HealthStatus, OperationalStatus |
    Format-Table -AutoSize

Get-Volume |
    Select-Object DriveLetter, FileSystem, FileSystemLabel, SizeRemaining, Size, HealthStatus |
    Format-Table -AutoSize
```

> **Nota sobre Windows Server 2025:** tecnologías como GPU-P, SMB sobre QUIC, dMSA, Azure Arc y hotpatching pueden formar parte de una evaluación futura de plataforma. No se habilitan ni se validan como si estuvieran disponibles en Windows Server 2022. La optimización de SQL Server realizada aquí se fundamenta en la configuración de la instancia, el almacenamiento disponible y las métricas observadas en este laboratorio.

## Instrucciones Paso a Paso

### Paso 1: Confirmar conectividad, versión y estado de Query Store

**Objetivo:** confirmar que se está trabajando en el servidor, instancia y base de datos correctos antes de recopilar la línea base.

**Instrucciones**

1. Abra SQL Server Management Studio.
2. Conéctese a `LOCALHOST,1433` mediante autenticación de Windows usando una cuenta con privilegios `sysadmin`, por ejemplo `.\SqlLabAdmin`.
3. Abra una nueva consulta y ejecute el siguiente script:

```sql
SELECT
    @@SERVERNAME AS ServerName,
    SERVERPROPERTY('MachineName') AS MachineName,
    SERVERPROPERTY('InstanceName') AS InstanceName,
    SERVERPROPERTY('Edition') AS Edition,
    SERVERPROPERTY('ProductVersion') AS ProductVersion,
    SERVERPROPERTY('ProductLevel') AS ProductLevel,
    SERVERPROPERTY('EngineEdition') AS EngineEdition,
    SYSDATETIME() AS ValidationDateTime;
GO

USE Sql2025Lab;
GO

SELECT
    name AS DatabaseName,
    compatibility_level,
    recovery_model_desc,
    state_desc
FROM sys.databases
WHERE name = N'Sql2025Lab';
GO

SELECT
    actual_state_desc,
    desired_state_desc,
    readonly_reason,
    current_storage_size_mb,
    max_storage_size_mb,
    flush_interval_seconds
FROM sys.database_query_store_options;
GO
```

4. Verifique también que los directorios estándar existen y que la cuenta de servicio SQL Server puede acceder a ellos. Identifique primero la cuenta que ejecuta el servicio:

```sql
SELECT
    servicename,
    startup_type_desc,
    status_desc,
    service_account,
    instant_file_initialization_enabled
FROM sys.dm_server_services
WHERE servicename LIKE N'SQL Server (%';
GO
```

5. Si la cuenta de servicio no dispone de permisos sobre `C:\SQLLab2025\Data` y `C:\SQLLab2025\Logs`, concédalos desde una consola PowerShell elevada. Sustituya la cuenta por el valor real mostrado en la consulta anterior.

```powershell
## Ejemplo: ajustar el nombre de cuenta al valor real de service_account.
icacls "C:\SQLLab2025\Data" /grant "NT SERVICE\MSSQLSERVER:(OI)(CI)M"
icacls "C:\SQLLab2025\Logs" /grant "NT SERVICE\MSSQLSERVER:(OI)(CI)M"
```

**Resultado esperado**

- El nombre del servidor corresponde a `SQLLAB-WS2022`.
- La instancia es la predeterminada; `InstanceName` puede mostrarse como `NULL`.
- La versión corresponde a SQL Server 2025 Developer Edition.
- `Sql2025Lab` está en línea, con compatibilidad `170`, recovery model `FULL` y Query Store en `READ_WRITE`.
- La consulta de servicios muestra la cuenta que ejecuta el motor SQL Server e indica si IFI está habilitado.

**Verificación**

Ejecute la siguiente consulta. El resultado debe devolver una fila con `actual_state_desc = READ_WRITE`.

```sql
USE Sql2025Lab;
GO

SELECT actual_state_desc
FROM sys.database_query_store_options;
GO
```

---

### Paso 2: Capturar la línea base de configuración, memoria, E/S y archivos

**Objetivo:** registrar métricas antes de aplicar cambios, de modo que la comparación posterior sea reproducible.

**Instrucciones**

1. Ejecute el siguiente script en `Sql2025Lab`. El script crea tablas de historial específicas para esta práctica si todavía no existen.
2. No elimine estas tablas durante la práctica; serán utilizadas para comparar el estado inicial y final.
3. El identificador `Baseline` representa el estado anterior a cualquier cambio.

```sql
USE Sql2025Lab;
GO

IF OBJECT_ID(N'dbo.Lab04_ConfigSnapshot', N'U') IS NULL
BEGIN
    CREATE TABLE dbo.Lab04_ConfigSnapshot
    (
        SnapshotLabel      varchar(20) NOT NULL,
        SnapshotDateTime   datetime2(0) NOT NULL,
        ConfigurationName  sysname NOT NULL,
        ConfiguredValue    sql_variant NULL,
        RunningValue       sql_variant NULL
    );
END;
GO

IF OBJECT_ID(N'dbo.Lab04_MemorySnapshot', N'U') IS NULL
BEGIN
    CREATE TABLE dbo.Lab04_MemorySnapshot
    (
        SnapshotLabel      varchar(20) NOT NULL,
        SnapshotDateTime   datetime2(0) NOT NULL,
        ClerkType          nvarchar(128) NOT NULL,
        PagesKB            bigint NOT NULL,
        VirtualMemoryKB    bigint NOT NULL
    );
END;
GO

IF OBJECT_ID(N'dbo.Lab04_IOSnapshot', N'U') IS NULL
BEGIN
    CREATE TABLE dbo.Lab04_IOSnapshot
    (
        SnapshotLabel          varchar(20) NOT NULL,
        SnapshotDateTime       datetime2(0) NOT NULL,
        DatabaseName           sysname NOT NULL,
        LogicalFileName        sysname NOT NULL,
        PhysicalName           nvarchar(260) NOT NULL,
        FileType               nvarchar(60) NOT NULL,
        NumOfReads             bigint NOT NULL,
        NumOfBytesRead         bigint NOT NULL,
        IoStallReadMS          bigint NOT NULL,
        NumOfWrites            bigint NOT NULL,
        NumOfBytesWritten      bigint NOT NULL,
        IoStallWriteMS         bigint NOT NULL,
        IoStallTotalMS         bigint NOT NULL
    );
END;
GO

DECLARE @SnapshotLabel varchar(20) = 'Baseline';
DECLARE @SnapshotDateTime datetime2(0) = SYSDATETIME();

INSERT INTO dbo.Lab04_ConfigSnapshot
(
    SnapshotLabel, SnapshotDateTime, ConfigurationName, ConfiguredValue, RunningValue
)
SELECT
    @SnapshotLabel,
    @SnapshotDateTime,
    name,
    value,
    value_in_use
FROM sys.configurations
WHERE name IN
(
    N'max server memory (MB)',
    N'min server memory (MB)',
    N'max degree of parallelism',
    N'cost threshold for parallelism',
    N'show advanced options'
);

INSERT INTO dbo.Lab04_MemorySnapshot
(
    SnapshotLabel, SnapshotDateTime, ClerkType, PagesKB, VirtualMemoryKB
)
SELECT TOP (20)
    @SnapshotLabel,
    @SnapshotDateTime,
    type,
    pages_kb,
    virtual_memory_committed_kb
FROM sys.dm_os_memory_clerks
ORDER BY pages_kb DESC;

INSERT INTO dbo.Lab04_IOSnapshot
(
    SnapshotLabel,
    SnapshotDateTime,
    DatabaseName,
    LogicalFileName,
    PhysicalName,
    FileType,
    NumOfReads,
    NumOfBytesRead,
    IoStallReadMS,
    NumOfWrites,
    NumOfBytesWritten,
    IoStallWriteMS,
    IoStallTotalMS
)
SELECT
    @SnapshotLabel,
    @SnapshotDateTime,
    DB_NAME(mf.database_id),
    mf.name,
    mf.physical_name,
    mf.type_desc,
    vfs.num_of_reads,
    vfs.num_of_bytes_read,
    vfs.io_stall_read_ms,
    vfs.num_of_writes,
    vfs.num_of_bytes_written,
    vfs.io_stall_write_ms,
    vfs.io_stall
FROM sys.master_files AS mf
CROSS APPLY sys.dm_io_virtual_file_stats(mf.database_id, mf.file_id) AS vfs;
GO

SELECT *
FROM dbo.Lab04_ConfigSnapshot
WHERE SnapshotLabel = 'Baseline'
ORDER BY ConfigurationName;

SELECT *
FROM dbo.Lab04_MemorySnapshot
WHERE SnapshotLabel = 'Baseline'
ORDER BY PagesKB DESC;

SELECT *
FROM dbo.Lab04_IOSnapshot
WHERE SnapshotLabel = 'Baseline'
ORDER BY DatabaseName, LogicalFileName;
GO
```

4. Revise los archivos de todas las bases de datos y, en especial, de `tempdb`.

```sql
SELECT
    DB_NAME(database_id) AS DatabaseName,
    name AS LogicalFileName,
    type_desc,
    physical_name,
    CAST(size / 128.0 AS decimal(12,2)) AS CurrentSizeMB,
    CASE
        WHEN is_percent_growth = 1 THEN CONCAT(growth, N'%')
        ELSE CONCAT(CAST(growth / 128.0 AS decimal(12,2)), N' MB')
    END AS FileGrowth
FROM sys.master_files
ORDER BY DatabaseName, type_desc, file_id;
GO

USE tempdb;
GO

SELECT
    file_id,
    name AS LogicalFileName,
    type_desc,
    physical_name,
    CAST(size / 128.0 AS decimal(12,2)) AS CurrentSizeMB,
    CASE
        WHEN is_percent_growth = 1 THEN CONCAT(growth, N'%')
        ELSE CONCAT(CAST(growth / 128.0 AS decimal(12,2)), N' MB')
    END AS FileGrowth
FROM sys.database_files
ORDER BY type_desc, file_id;
GO
```

5. Desde PowerShell, capture los indicadores básicos del sistema operativo. Ejecute el comando desde una consola con privilegios administrativos:

```powershell
$Counters = @(
    '\Memory\Available MBytes',
    '\Processor Information(_Total)\% Processor Time',
    '\PhysicalDisk(_Total)\Avg. Disk sec/Read',
    '\PhysicalDisk(_Total)\Avg. Disk sec/Write',
    '\PhysicalDisk(_Total)\Disk Transfers/sec'
)

Get-Counter -Counter $Counters -SampleInterval 1 -MaxSamples 5 |
    Select-Object -ExpandProperty CounterSamples |
    Select-Object Path, CookedValue, TimeStamp |
    Export-Csv -Path 'C:\SQLLab2025\Reports\Lab04_WindowsCounters_Baseline.csv' -NoTypeInformation -Encoding UTF8

Get-Content 'C:\SQLLab2025\Reports\Lab04_WindowsCounters_Baseline.csv'
```

6. Compruebe el plan de energía activo:

```powershell
powercfg /GETACTIVESCHEME
```

**Resultado esperado**

- Las tablas de snapshot contienen registros con la etiqueta `Baseline`.
- Se muestran los parámetros actuales de memoria y paralelismo.
- La salida de `sys.master_files` permite identificar rutas, tamaños y crecimiento de archivos.
- Se crea el archivo `Lab04_WindowsCounters_Baseline.csv`.
- El plan de energía activo queda documentado, sin modificarlo.

**Verificación**

Ejecute la siguiente consulta. Debe devolver al menos cinco filas de configuración para la etiqueta `Baseline`.

```sql
USE Sql2025Lab;
GO

SELECT
    SnapshotLabel,
    COUNT(*) AS ConfigurationRows,
    MIN(SnapshotDateTime) AS SnapshotDateTime
FROM dbo.Lab04_ConfigSnapshot
GROUP BY SnapshotLabel;
GO
```

---

### Paso 3: Analizar la línea base y validar criterios de ajuste

**Objetivo:** interpretar las métricas recopiladas antes de cambiar la configuración, evitando aplicar ajustes de forma automática o sin justificación.

**Instrucciones**

1. Revise la configuración inicial de memoria y paralelismo:

```sql
SELECT
    name,
    value AS ConfiguredValue,
    value_in_use AS RunningValue,
    description
FROM sys.configurations
WHERE name IN
(
    N'max server memory (MB)',
    N'min server memory (MB)',
    N'max degree of parallelism',
    N'cost threshold for parallelism'
)
ORDER BY name;
GO
```

2. Revise el consumo de memoria de los principales clerks:

```sql
SELECT TOP (15)
    type,
    name,
    pages_kb / 1024.0 AS PagesMB,
    virtual_memory_committed_kb / 1024.0 AS VirtualMemoryCommittedMB
FROM sys.dm_os_memory_clerks
ORDER BY pages_kb DESC;
GO
```

3. Revise el estado de memoria general del host SQL Server:

```sql
SELECT
    total_physical_memory_kb / 1024.0 AS TotalPhysicalMemoryMB,
    available_physical_memory_kb / 1024.0 AS AvailablePhysicalMemoryMB,
    total_page_file_kb / 1024.0 AS TotalPageFileMB,
    available_page_file_kb / 1024.0 AS AvailablePageFileMB,
    system_memory_state_desc
FROM sys.dm_os_sys_memory;
GO

SELECT
    physical_memory_in_use_kb / 1024.0 AS SqlServerMemoryInUseMB,
    large_page_allocations_kb / 1024.0 AS LargePageAllocationsMB,
    locked_page_allocations_kb / 1024.0 AS LockedPageAllocationsMB,
    memory_utilization_percentage,
    available_commit_limit_kb / 1024.0 AS AvailableCommitLimitMB,
    process_physical_memory_low,
    process_virtual_memory_low
FROM sys.dm_os_process_memory;
GO
```

4. Calcule una referencia de latencia acumulada de E/S. Estos valores son acumulados desde el arranque del servicio y no equivalen por sí mismos a una medición de carga puntual.

```sql
SELECT
    DB_NAME(mf.database_id) AS DatabaseName,
    mf.name AS LogicalFileName,
    mf.type_desc,
    mf.physical_name,
    vfs.num_of_reads,
    CASE
        WHEN vfs.num_of_reads = 0 THEN 0
        ELSE CAST(vfs.io_stall_read_ms * 1.0 / vfs.num_of_reads AS decimal(12,2))
    END AS AvgReadLatencyMS,
    vfs.num_of_writes,
    CASE
        WHEN vfs.num_of_writes = 0 THEN 0
        ELSE CAST(vfs.io_stall_write_ms * 1.0 / vfs.num_of_writes AS decimal(12,2))
    END AS AvgWriteLatencyMS
FROM sys.master_files AS mf
CROSS APPLY sys.dm_io_virtual_file_stats(mf.database_id, mf.file_id) AS vfs
ORDER BY AvgWriteLatencyMS DESC, AvgReadLatencyMS DESC;
GO
```

5. Revise la configuración de `tempdb` y confirme si el estado actual ya cumple el objetivo de cuatro archivos de datos homogéneos. No elimine archivos adicionales sin una evaluación previa.

```sql
USE tempdb;
GO

SELECT
    type_desc,
    COUNT(*) AS FileCount,
    MIN(size) / 128.0 AS MinSizeMB,
    MAX(size) / 128.0 AS MaxSizeMB,
    MIN(growth) / 128.0 AS MinGrowthMB,
    MAX(growth) / 128.0 AS MaxGrowthMB
FROM sys.database_files
GROUP BY type_desc;
GO
```

6. Documente brevemente en `C:\SQLLab2025\Reports\Lab04_Observaciones.txt`:
   - Memoria física y memoria disponible.
   - Valor actual de memoria máxima SQL Server.
   - Plan de energía.
   - Número de archivos de datos de `tempdb`.
   - Ubicación de los archivos de datos y log.
   - Si IFI aparece habilitado.
   - Cualquier valor claramente distinto de la línea base objetivo.

7. Considere las siguientes pautas de interpretación:
   - `max server memory` debe dejar memoria suficiente para Windows, SSMS, antivirus, servicios de monitoreo y otros procesos del laboratorio.
   - `min server memory` no obliga a SQL Server a reservar memoria al inicio; representa un mínimo después de que SQL Server haya crecido hasta dicho valor. En este laboratorio se conserva en `0`.
   - `MAXDOP = 4` es un punto inicial controlado para un host de ocho vCPU; no es un valor universal.
   - `cost threshold for parallelism = 50` reduce el uso de paralelismo para consultas de costo bajo respecto al valor histórico predeterminado de `5`.
   - Las latencias deben evaluarse durante una carga representativa. Una cifra acumulada alta requiere investigación, no una conclusión automática.
   - IFI acelera la inicialización de archivos de datos, pero no acelera la inicialización de archivos de log, ya que estos deben inicializarse en cero por razones de recuperación.

**Resultado esperado**

- El estudiante dispone de un diagnóstico inicial y puede identificar diferencias entre la configuración actual y el objetivo del laboratorio.
- Se confirma que no existe presión de memoria crítica antes de ejecutar pruebas.
- Se identifica la distribución actual de archivos de `tempdb`.

**Verificación**

La siguiente consulta debe devolver los parámetros que se modificarán o validarán durante el siguiente paso:

```sql
SELECT
    name,
    value,
    value_in_use
FROM sys.configurations
WHERE name IN
(
    N'max server memory (MB)',
    N'min server memory (MB)',
    N'max degree of parallelism',
    N'cost threshold for parallelism'
);
GO
```

---

### Paso 4: Aplicar configuración básica de memoria y paralelismo

**Objetivo:** establecer los valores iniciales controlados de rendimiento definidos para este laboratorio.

**Instrucciones**

1. Confirme que la ventana de cambio sigue activa.
2. Abra una nueva ventana de consulta con permisos `sysadmin`.
3. Ejecute el siguiente script completo:

```sql
USE master;
GO

EXEC sys.sp_configure N'show advanced options', 1;
RECONFIGURE;
GO

EXEC sys.sp_configure N'max server memory (MB)', 16384;
RECONFIGURE;
GO

EXEC sys.sp_configure N'min server memory (MB)', 0;
RECONFIGURE;
GO

EXEC sys.sp_configure N'max degree of parallelism', 4;
RECONFIGURE;
GO

EXEC sys.sp_configure N'cost threshold for parallelism', 50;
RECONFIGURE;
GO

SELECT
    name,
    value AS ConfiguredValue,
    value_in_use AS RunningValue
FROM sys.configurations
WHERE name IN
(
    N'max server memory (MB)',
    N'min server memory (MB)',
    N'max degree of parallelism',
    N'cost threshold for parallelism'
)
ORDER BY name;
GO
```

4. No configure `min server memory` con un valor elevado para “forzar” rendimiento. Manténgalo en `0` para el laboratorio.
5. No cambie afinidad de CPU, trace flags globales, opciones de inicio ni configuraciones de scheduler.
6. No altere el plan de energía del sistema operativo durante esta práctica salvo que exista una política explícita de infraestructura y un cambio aprobado por el responsable del servidor.

**Resultado esperado**

La consulta final debe reflejar los siguientes valores configurados y en uso:

| Parámetro | Valor esperado |
|---|---:|
| `max server memory (MB)` | `16384` |
| `min server memory (MB)` | `0` |
| `max degree of parallelism` | `4` |
| `cost threshold for parallelism` | `50` |

**Verificación**

Ejecute:

```sql
SELECT
    name,
    CAST(value AS int) AS ConfiguredValue,
    CAST(value_in_use AS int) AS RunningValue
FROM sys.configurations
WHERE name IN
(
    N'max server memory (MB)',
    N'min server memory (MB)',
    N'max degree of parallelism',
    N'cost threshold for parallelism'
)
ORDER BY name;
GO
```

Los valores de `ConfiguredValue` y `RunningValue` deben coincidir con los objetivos de la práctica.

---

### Paso 5: Configurar y validar archivos de TempDB

**Objetivo:** dejar `tempdb` con cuatro archivos de datos homogéneos de `1024 MB`, crecimiento fijo de `256 MB` y un archivo de log con crecimiento fijo.

**Instrucciones**

1. Inspeccione nuevamente los archivos actuales de `tempdb`:

```sql
USE tempdb;
GO

SELECT
    file_id,
    name,
    type_desc,
    physical_name,
    CAST(size / 128.0 AS decimal(12,2)) AS SizeMB,
    CASE
        WHEN is_percent_growth = 1 THEN CONCAT(growth, N'%')
        ELSE CONCAT(CAST(growth / 128.0 AS decimal(12,2)), N' MB')
    END AS GrowthSetting
FROM sys.database_files
ORDER BY type_desc, file_id;
GO
```

2. Antes de ejecutar el cambio, confirme que `tempdb` tiene únicamente un archivo de datos y un archivo de log. El siguiente control detiene el proceso si existen archivos adicionales, porque eliminarlos o reorganizarlos requiere análisis específico.

```sql
USE master;
GO

IF
(
    SELECT COUNT(*)
    FROM sys.master_files
    WHERE database_id = DB_ID(N'tempdb')
      AND type_desc = N'ROWS'
) > 1
OR
(
    SELECT COUNT(*)
    FROM sys.master_files
    WHERE database_id = DB_ID(N'tempdb')
      AND type_desc = N'LOG'
) > 1
BEGIN
    THROW 50001, 'TempDB tiene archivos adicionales. No ejecute la configuración automática; revise la distribución actual antes de continuar.', 1;
END;
GO
```

3. Si el control anterior no genera error, ejecute el siguiente script. Este script:
   - Configura el archivo de datos principal `tempdev`.
   - Configura el archivo de log `templog`.
   - Agrega tres archivos de datos adicionales.
   - Deja preparados los nuevos nombres físicos para el próximo inicio del servicio.

```sql
USE master;
GO

ALTER DATABASE tempdb
MODIFY FILE
(
    NAME = N'tempdev',
    FILENAME = N'C:\SQLLab2025\Data\tempdb.mdf',
    SIZE = 1024MB,
    FILEGROWTH = 256MB
);
GO

ALTER DATABASE tempdb
MODIFY FILE
(
    NAME = N'templog',
    FILENAME = N'C:\SQLLab2025\Logs\templog.ldf',
    SIZE = 1024MB,
    FILEGROWTH = 256MB
);
GO

ALTER DATABASE tempdb
ADD FILE
(
    NAME = N'tempdev2',
    FILENAME = N'C:\SQLLab2025\Data\tempdb2.ndf',
    SIZE = 1024MB,
    FILEGROWTH = 256MB
);
GO

ALTER DATABASE tempdb
ADD FILE
(
    NAME = N'tempdev3',
    FILENAME = N'C:\SQLLab2025\Data\tempdb3.ndf',
    SIZE = 1024MB,
    FILEGROWTH = 256MB
);
GO

ALTER DATABASE tempdb
ADD FILE
(
    NAME = N'tempdev4',
    FILENAME = N'C:\SQLLab2025\Data\tempdb4.ndf',
    SIZE = 1024MB,
    FILEGROWTH = 256MB
);
GO
```

4. Reinicie el servicio SQL Server para que el cambio de ruta del archivo principal y del archivo de log de `tempdb` se materialice. Hágalo desde PowerShell elevado:

```powershell
Restart-Service -Name MSSQLSERVER -Force

Get-Service -Name MSSQLSERVER
Test-NetConnection -ComputerName localhost -Port 1433
```

5. Espere a que el servicio alcance el estado `Running`.
6. Reconéctese con SSMS y valide los archivos efectivos:

```sql
USE tempdb;
GO

SELECT
    file_id,
    name AS LogicalFileName,
    type_desc,
    physical_name,
    CAST(size / 128.0 AS decimal(12,2)) AS CurrentSizeMB,
    CASE
        WHEN is_percent_growth = 1 THEN CONCAT(growth, N'%')
        ELSE CONCAT(CAST(growth / 128.0 AS decimal(12,2)), N' MB')
    END AS FileGrowth
FROM sys.database_files
ORDER BY type_desc, file_id;
GO
```

7. Compruebe que los parámetros de memoria y paralelismo siguen en uso tras reiniciar el servicio:

```sql
SELECT
    name,
    value,
    value_in_use
FROM sys.configurations
WHERE name IN
(
    N'max server memory (MB)',
    N'min server memory (MB)',
    N'max degree of parallelism',
    N'cost threshold for parallelism'
);
GO
```

**Resultado esperado**

- `tempdb` contiene cuatro archivos de tipo `ROWS`: `tempdev`, `tempdev2`, `tempdev3` y `tempdev4`.
- Cada archivo de datos tiene un tamaño configurado de `1024 MB` y crecimiento fijo de `256 MB`.
- Existe un archivo de log `templog` con crecimiento fijo de `256 MB`.
- Los archivos de datos se encuentran en `C:\SQLLab2025\Data`.
- El archivo de log se encuentra en `C:\SQLLab2025\Logs`.
- El servicio `MSSQLSERVER` se encuentra en estado `Running`.

**Verificación**

Ejecute la siguiente consulta. Debe devolver cuatro archivos de datos y un archivo de log.

```sql
USE tempdb;
GO

SELECT
    type_desc,
    COUNT(*) AS FileCount,
    MIN(size) / 128.0 AS MinSizeMB,
    MAX(size) / 128.0 AS MaxSizeMB,
    MIN(growth) / 128.0 AS MinGrowthMB,
    MAX(growth) / 128.0 AS MaxGrowthMB
FROM sys.database_files
GROUP BY type_desc;
GO
```

La fila `ROWS` debe mostrar `FileCount = 4`, tamaños homogéneos de aproximadamente `1024 MB` y crecimiento de `256 MB`.

---

### Paso 6: Ejecutar la carga de prueba y medir el comportamiento posterior

**Objetivo:** repetir las consultas de la práctica 2 utilizando la configuración ajustada y recolectar evidencia de rendimiento.

**Instrucciones**

1. Abra los scripts de carga utilizados en la práctica 2 desde `C:\SQLLab2025\Scripts`.
2. Antes de ejecutar cada consulta representativa, agregue un comentario identificador al inicio. Esto permite localizar la ejecución en Query Store sin exponer datos sensibles.

```sql
/* LAB04_WORKLOAD */
```

3. Active estadísticas de tiempo y E/S para la sesión de prueba:

```sql
USE Sql2025Lab;
GO

SET NOCOUNT ON;
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
GO
```

4. Ejecute las mismas consultas de prueba de la práctica 2 con los mismos parámetros, orden y número de repeticiones utilizados para el baseline.

5. Registre para cada ejecución:
   - Duración total.
   - CPU time mostrado por `SET STATISTICS TIME`.
   - Logical reads mostradas por `SET STATISTICS IO`.
   - Número de filas devueltas.
   - Plan de ejecución real, si fue utilizado en la práctica 2.
   - Observaciones sobre operadores paralelos, spills, scans o advertencias.

6. Desactive las estadísticas al finalizar:

```sql
SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
GO
```

7. Consulte Query Store para revisar las consultas etiquetadas. Ajuste el intervalo de tiempo si fuera necesario.

```sql
USE Sql2025Lab;
GO

SELECT TOP (20)
    q.query_id,
    p.plan_id,
    rs.count_executions,
    CAST(rs.avg_duration / 1000.0 AS decimal(18,2)) AS AvgDurationMS,
    CAST(rs.avg_cpu_time / 1000.0 AS decimal(18,2)) AS AvgCpuMS,
    rs.avg_logical_io_reads,
    rs.avg_logical_io_writes,
    rs.last_execution_time,
    qt.query_sql_text
FROM sys.query_store_query_text AS qt
INNER JOIN sys.query_store_query AS q
    ON qt.query_text_id = q.query_text_id
INNER JOIN sys.query_store_plan AS p
    ON q.query_id = p.query_id
INNER JOIN sys.query_store_runtime_stats AS rs
    ON p.plan_id = rs.plan_id
WHERE qt.query_sql_text LIKE N'%LAB04_WORKLOAD%'
ORDER BY rs.last_execution_time DESC, rs.avg_duration DESC;
GO
```

8. Revise si las consultas generan planes paralelos. El siguiente script identifica operadores paralelos en planes capturados por Query Store:

```sql
USE Sql2025Lab;
GO

SELECT TOP (20)
    q.query_id,
    p.plan_id,
    p.last_execution_time,
    p.avg_duration / 1000.0 AS AvgDurationMS,
    qt.query_sql_text
FROM sys.query_store_plan AS p
INNER JOIN sys.query_store_query AS q
    ON p.query_id = q.query_id
INNER JOIN sys.query_store_query_text AS qt
    ON q.query_text_id = qt.query_text_id
WHERE qt.query_sql_text LIKE N'%LAB04_WORKLOAD%'
  AND CAST(p.query_plan AS nvarchar(max)) LIKE N'%Parallelism%'
ORDER BY p.last_execution_time DESC;
GO
```

9. No fuerce un plan en Query Store durante esta práctica. El objetivo es observar el efecto de los ajustes de instancia, no reemplazar el proceso de análisis de planes, estadísticas e índices realizado en prácticas posteriores.

**Resultado esperado**

- Las consultas de la práctica 2 se ejecutan correctamente después del ajuste.
- Query Store registra ejecuciones asociadas al comentario `LAB04_WORKLOAD`.
- Se obtiene evidencia comparable respecto de duración, CPU, lecturas lógicas y comportamiento de planes.
- La reducción o aumento de duración se interpreta con cautela, ya que un entorno de laboratorio puede variar por caché, concurrencia, arranque reciente del servicio y carga del sistema operativo.

**Verificación**

Ejecute la siguiente consulta. Debe retornar al menos una ejecución de carga si las consultas fueron etiquetadas correctamente.

```sql
USE Sql2025Lab;
GO

SELECT
    COUNT(*) AS RuntimeStatsRows,
    MAX(rs.last_execution_time) AS LastExecutionTime
FROM sys.query_store_query_text AS qt
INNER JOIN sys.query_store_query AS q
    ON qt.query_text_id = q.query_text_id
INNER JOIN sys.query_store_plan AS p
    ON q.query_id = p.query_id
INNER JOIN sys.query_store_runtime_stats AS rs
    ON p.plan_id = rs.plan_id
WHERE qt.query_sql_text LIKE N'%LAB04_WORKLOAD%';
GO
```

---

### Paso 7: Capturar el estado posterior y comparar con la línea base

**Objetivo:** registrar el estado final de configuración y comparar métricas acumuladas de E/S con la línea base.

**Instrucciones**

1. Ejecute el siguiente script para guardar un snapshot posterior con la etiqueta `PostChange`.

```sql
USE Sql2025Lab;
GO

DECLARE @SnapshotLabel varchar(20) = 'PostChange';
DECLARE @SnapshotDateTime datetime2(0) = SYSDATETIME();

INSERT INTO dbo.Lab04_ConfigSnapshot
(
    SnapshotLabel, SnapshotDateTime, ConfigurationName, ConfiguredValue, RunningValue
)
SELECT
    @SnapshotLabel,
    @SnapshotDateTime,
    name,
    value,
    value_in_use
FROM sys.configurations
WHERE name IN
(
    N'max server memory (MB)',
    N'min server memory (MB)',
    N'max degree of parallelism',
    N'cost threshold for parallelism',
    N'show advanced options'
);

INSERT INTO dbo.Lab04_MemorySnapshot
(
    SnapshotLabel, SnapshotDateTime, ClerkType, PagesKB, VirtualMemoryKB
)
SELECT TOP (20)
    @SnapshotLabel,
    @SnapshotDateTime,
    type,
    pages_kb,
    virtual_memory_committed_kb
FROM sys.dm_os_memory_clerks
ORDER BY pages_kb DESC;

INSERT INTO dbo.Lab04_IOSnapshot
(
    SnapshotLabel,
    SnapshotDateTime,
    DatabaseName,
    LogicalFileName,
    PhysicalName,
    FileType,
    NumOfReads,
    NumOfBytesRead,
    IoStallReadMS,
    NumOfWrites,
    NumOfBytesWritten,
    IoStallWriteMS,
    IoStallTotalMS
)
SELECT
    @SnapshotLabel,
    @SnapshotDateTime,
    DB_NAME(mf.database_id),
    mf.name,
    mf.physical_name,
    mf.type_desc,
    vfs.num_of_reads,
    vfs.num_of_bytes_read,
    vfs.io_stall_read_ms,
    vfs.num_of_writes,
    vfs.num_of_bytes_written,
    vfs.io_stall_write_ms,
    vfs.io_stall
FROM sys.master_files AS mf
CROSS APPLY sys.dm_io_virtual_file_stats(mf.database_id, mf.file_id) AS vfs;
GO
```

2. Capture los contadores de Windows posteriores:

```powershell
$Counters = @(
    '\Memory\Available MBytes',
    '\Processor Information(_Total)\% Processor Time',
    '\PhysicalDisk(_Total)\Avg. Disk sec/Read',
    '\PhysicalDisk(_Total)\Avg. Disk sec/Write',
    '\PhysicalDisk(_Total)\Disk Transfers/sec'
)

Get-Counter -Counter $Counters -SampleInterval 1 -MaxSamples 5 |
    Select-Object -ExpandProperty CounterSamples |
    Select-Object Path, CookedValue, TimeStamp |
    Export-Csv -Path 'C:\SQLLab2025\Reports\Lab04_WindowsCounters_PostChange.csv' -NoTypeInformation -Encoding UTF8
```

3. Compare la configuración antes y después:

```sql
USE Sql2025Lab;
GO

WITH Baseline AS
(
    SELECT
        ConfigurationName,
        ConfiguredValue,
        RunningValue
    FROM dbo.Lab04_ConfigSnapshot
    WHERE SnapshotLabel = 'Baseline'
),
PostChange AS
(
    SELECT
        ConfigurationName,
        ConfiguredValue,
        RunningValue
    FROM dbo.Lab04_ConfigSnapshot
    WHERE SnapshotLabel = 'PostChange'
)
SELECT
    COALESCE(b.ConfigurationName, p.ConfigurationName) AS ConfigurationName,
    b.ConfiguredValue AS BaselineConfiguredValue,
    b.RunningValue AS BaselineRunningValue,
    p.ConfiguredValue AS PostChangeConfiguredValue,
    p.RunningValue AS PostChangeRunningValue
FROM Baseline AS b
FULL OUTER JOIN PostChange AS p
    ON b.ConfigurationName = p.ConfigurationName
ORDER BY ConfigurationName;
GO
```

4. Compare las diferencias acumuladas de E/S. Como el servicio fue reiniciado al configurar `tempdb`, los contadores de E/S pueden reiniciarse. Por esta razón, use esta consulta principalmente para documentar el estado posterior y no para inferir diferencias matemáticas si los contadores fueron reinicializados.

```sql
USE Sql2025Lab;
GO

SELECT
    DatabaseName,
    LogicalFileName,
    FileType,
    PhysicalName,
    NumOfReads,
    NumOfWrites,
    CASE
        WHEN NumOfReads = 0 THEN 0
        ELSE CAST(IoStallReadMS * 1.0 / NumOfReads AS decimal(12,2))
    END AS AvgReadLatencyMS,
    CASE
        WHEN NumOfWrites = 0 THEN 0
        ELSE CAST(IoStallWriteMS * 1.0 / NumOfWrites AS decimal(12,2))
    END AS AvgWriteLatencyMS
FROM dbo.Lab04_IOSnapshot
WHERE SnapshotLabel = 'PostChange'
ORDER BY AvgWriteLatencyMS DESC, AvgReadLatencyMS DESC;
GO
```

5. Redacte un informe breve en `C:\SQLLab2025\Reports\Lab04_Resultados.md` que incluya:
   - Fecha y hora de la práctica.
   - Configuración anterior y posterior.
   - Estado de `tempdb`.
   - Estado de IFI.
   - Plan de energía observado.
   - Resultados de las consultas de la práctica 2.
   - Hallazgos de Query Store.
   - Interpretación de la latencia observada.
   - Una conclusión explícita que indique que los resultados se validaron sobre Windows Server 2022, no sobre Windows Server 2025.

**Resultado esperado**

- Las tablas de historial contienen snapshots `Baseline` y `PostChange`.
- La configuración final coincide con los valores definidos para el laboratorio.
- Existen dos archivos CSV con métricas de Windows.
- El informe final identifica qué cambios fueron aplicados y qué resultados fueron observados.

**Verificación**

Ejecute esta validación integral:

```sql
USE Sql2025Lab;
GO

SELECT
    CASE
        WHEN MAX(CASE WHEN name = N'max server memory (MB)' AND value_in_use = 16384 THEN 1 ELSE 0 END) = 1
         AND MAX(CASE WHEN name = N'max degree of parallelism' AND value_in_use = 4 THEN 1 ELSE 0 END) = 1
         AND MAX(CASE WHEN name = N'cost threshold for parallelism' AND value_in_use = 50 THEN 1 ELSE 0 END) = 1
        THEN N'Configuración de instancia validada'
        ELSE N'Revisar configuración de instancia'
    END AS InstanceConfigurationValidation
FROM sys.configurations
WHERE name IN
(
    N'max server memory (MB)',
    N'max degree of parallelism',
    N'cost threshold for parallelism'
);
GO

USE tempdb;
GO

SELECT
    CASE
        WHEN COUNT(CASE WHEN type_desc = N'ROWS' THEN 1 END) = 4
         AND COUNT(CASE WHEN type_desc = N'LOG' THEN 1 END) = 1
        THEN N'Configuración de TempDB validada'
        ELSE N'Revisar archivos de TempDB'
    END AS TempDBValidation
FROM sys.database_files;
GO
```

## Validación y Pruebas

La práctica se considera completada cuando se cumplen los siguientes criterios:

| Área validada | Criterio de aceptación |
|---|---|
| Conectividad | La instancia responde en `LOCALHOST,1433`. |
| Base de datos | `Sql2025Lab` está en línea, con compatibilidad `170`, recovery model `FULL` y Query Store en `READ_WRITE`. |
| Memoria SQL Server | `max server memory (MB) = 16384` y `min server memory (MB) = 0`. |
| Paralelismo | `max degree of parallelism = 4` y `cost threshold for parallelism = 50`. |
| TempDB | Cuatro archivos de datos de `1024 MB` con crecimiento fijo de `256 MB`; un archivo de log con crecimiento fijo. |
| Rutas | Archivos de datos TempDB en `C:\SQLLab2025\Data` y archivo de log en `C:\SQLLab2025\Logs`. |
| Evidencia | Existen snapshots `Baseline` y `PostChange` en las tablas `dbo.Lab04_*`. |
| Carga | Las consultas de la práctica 2 fueron ejecutadas y registradas en Query Store mediante `LAB04_WORKLOAD`. |
| Sistema operativo | Se documentó el plan de energía, memoria disponible y contadores básicos de disco y CPU. |
| Contexto tecnológico | El informe indica que las pruebas se realizaron sobre Windows Server 2022 build `20348.2402`. |

Use la siguiente consulta como revisión final de archivos y rutas:

```sql
SELECT
    DB_NAME(database_id) AS DatabaseName,
    name AS LogicalFileName,
    type_desc,
    physical_name,
    CAST(size / 128.0 AS decimal(12,2)) AS SizeMB,
    CASE
        WHEN is_percent_growth = 1 THEN CONCAT(growth, N'%')
        ELSE CONCAT(CAST(growth / 128.0 AS decimal(12,2)), N' MB')
    END AS FileGrowth
FROM sys.master_files
WHERE database_id = DB_ID(N'tempdb')
ORDER BY type_desc, file_id;
GO
```

## Solución de Problemas

### Incidencia 1: El servicio SQL Server no inicia después de modificar TempDB

**Síntomas**

- `Get-Service MSSQLSERVER` muestra estado `Stopped`.
- SSMS no puede conectarse a `LOCALHOST,1433`.
- El log de errores de SQL Server indica que no se puede crear o abrir un archivo de `tempdb`.

**Causa probable**

La cuenta de servicio SQL Server no dispone de permisos sobre `C:\SQLLab2025\Data` o `C:\SQLLab2025\Logs`, la unidad no está disponible, o existe un error en la ruta configurada para alguno de los archivos.

**Corrección**

1. Revise la cuenta de servicio desde SQL Server Configuration Manager o mediante `sys.dm_server_services` cuando el servicio esté disponible.
2. Confirme que las rutas existen:

```powershell
Test-Path 'C:\SQLLab2025\Data'
Test-Path 'C:\SQLLab2025\Logs'
```

3. Conceda permisos Modify a la cuenta de servicio correcta:

```powershell
icacls "C:\SQLLab2025\Data" /grant "NT SERVICE\MSSQLSERVER:(OI)(CI)M"
icacls "C:\SQLLab2025\Logs" /grant "NT SERVICE\MSSQLSERVER:(OI)(CI)M"
```

4. Revise el SQL Server Error Log para identificar el archivo específico.
5. Si la ruta configurada es incorrecta, inicie SQL Server con los parámetros de recuperación apropiados según el procedimiento institucional, corrija la definición de archivo y reinicie el servicio. No borre manualmente archivos de `tempdb` mientras el servicio esté iniciado.

### Incidencia 2: Las consultas de carga no aparecen en Query Store

**Síntomas**

- La consulta filtrada por `LAB04_WORKLOAD` devuelve cero filas.
- Las consultas se ejecutaron, pero no se observan estadísticas de ejecución en Query Store.

**Causa probable**

La consulta se ejecutó sin el comentario identificador, Query Store no está en `READ_WRITE`, la consulta se ejecutó en otra base de datos o la captura automática de Query Store aún no ha registrado la actividad esperada.

**Corrección**

1. Confirme el estado de Query Store:

```sql
USE Sql2025Lab;
GO

SELECT
    actual_state_desc,
    desired_state_desc,
    readonly_reason
FROM sys.database_query_store_options;
GO
```

2. Ejecute la consulta de carga dentro de `Sql2025Lab` y agregue el comentario en el texto enviado a SQL Server:

```sql
USE Sql2025Lab;
GO

/* LAB04_WORKLOAD */
-- Pegue aquí una consulta de la práctica 2.
```

3. Espere unos segundos y ejecute nuevamente la consulta de Query Store.
4. Si Query Store está en modo `READ_ONLY`, investigue la causa antes de modificar su configuración; por ejemplo, espacio máximo alcanzado o condición de solo lectura de la base de datos.

## Limpieza

Esta práctica establece una configuración objetivo que debe permanecer disponible para las siguientes actividades del curso. Por ello, **no revierta** los siguientes ajustes al finalizar:

- `max server memory (MB) = 16384`
- `min server memory (MB) = 0`
- `MAXDOP = 4`
- `cost threshold for parallelism = 50`
- Cuatro archivos de datos de `tempdb` configurados para el laboratorio

Realice únicamente las siguientes acciones de limpieza:

1. Guarde los scripts ejecutados en `C:\SQLLab2025\Scripts`.
2. Mantenga los archivos de evidencia en `C:\SQLLab2025\Reports`.
3. Cierre sesiones de SSMS que no sean necesarias.
4. Desactive cualquier opción de estadísticas que haya quedado activa en una ventana de consulta:

```sql
SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
GO
```

5. Verifique que no quedaron operaciones administrativas en ejecución:

```sql
SELECT
    session_id,
    command,
    status,
    percent_complete,
    estimated_completion_time / 1000.0 AS EstimatedCompletionSeconds
FROM sys.dm_exec_requests
WHERE session_id <> @@SPID
  AND command IN
(
    N'DB STARTUP',
    N'BACKUP DATABASE',
    N'BACKUP LOG',
    N'RESTORE DATABASE',
    N'RESTORE LOG',
    N'ALTER DATABASE'
);
GO
```

6. Si el entorno se reutilizará fuera del curso, cambie las contraseñas temporales de laboratorio y elimine cualquier archivo de informe que contenga información operativa sensible.

## Resumen

En esta práctica se estableció una línea base de configuración y rendimiento para SQL Server 2025 sobre Windows Server 2022. Se validaron memoria, paralelismo, rutas de archivos, estado de IFI, energía del sistema, almacenamiento y métricas de E/S.

Los ajustes aplicados fueron deliberadamente limitados y reversibles: memoria máxima de SQL Server de `16384 MB`, `MAXDOP = 4` y `cost threshold for parallelism = 50`. También se configuró `tempdb` con cuatro archivos de datos homogéneos, reduciendo el riesgo de crecimiento desigual y contención asociada a una configuración inicial insuficiente.

Los resultados de las consultas de carga deben interpretarse junto con Query Store, planes de ejecución, estadísticas, índices y latencias de almacenamiento. Windows Server 2025 ofrece capacidades relevantes para futuras plataformas SQL Server —por ejemplo, mejoras de virtualización, almacenamiento, identidad y administración híbrida—, pero esta práctica valida exclusivamente el comportamiento observado en Windows Server 2022 build `20348.2402`.

Como continuación, use los datos recopilados para justificar optimizaciones específicas de consultas, índices o estadísticas en lugar de aplicar cambios globales adicionales sin evidencia.
