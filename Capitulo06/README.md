# Simulación de migración: Simulación de migración y Validación post-migración

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 40 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción General

En esta práctica se simula la migración de la base de datos `MigracionLab_Source` desde una instancia de SQL Server 2017 (`SQL2017SRC`) hacia una instancia de SQL Server 2025 (`SQL2025TGT`). El proceso incluye inventario del origen, evaluación de compatibilidad con Data Migration Assistant (DMA), respaldo consistente, transferencia verificable, restauración controlada y validación técnica y funcional. La base de datos de origen no se modifica ni se elimina.

## Objetivos de Aprendizaje

- [ ] Inventariar la instancia y la base de datos de origen, incluyendo configuración, objetos, dependencias, usuarios y características relevantes.
- [ ] Ejecutar una evaluación de compatibilidad con DMA entre SQL Server 2017 y SQL Server 2025.
- [ ] Crear, verificar y transferir una copia de seguridad con `CHECKSUM` y hash SHA-256.
- [ ] Restaurar la base de datos en SQL Server 2025 conservando inicialmente el nivel de compatibilidad heredado.
- [ ] Validar integridad, datos, objetos, seguridad y rendimiento funcional antes y después de elevar la compatibilidad a nivel 170.

## Prerrequisitos

**Conocimientos requeridos**

- Conocimientos básicos de T-SQL, bases de datos SQL Server, copias de seguridad y restauración.
- Comprensión básica de niveles de compatibilidad, usuarios de base de datos y logins de instancia.
- Finalización de las prácticas previas que crearon y cargaron `MigracionLab_Source`.
- Conocimiento de los procedimientos almacenados, vistas y consultas representativas creados en las prácticas anteriores.

**Acceso requerido**

- Permisos equivalentes a `sysadmin` en `SQL2017SRC` y `SQL2025TGT`.
- Acceso a SQL Server Management Studio (SSMS) y Data Migration Assistant 5.6.6784.0.
- Permisos de lectura y escritura para la cuenta de servicio de SQL Server sobre:
  - `C:\LabSQL2025\Backup`
  - `C:\LabSQL2025\Reports`
  - `C:\SQLLab2025\Data`
  - `C:\SQLLab2025\Logs`
- Conectividad entre las instancias de origen y destino mediante los puertos configurados.
- La instancia de destino debe disponer de espacio libre suficiente para los archivos de datos, log y respaldo.

> **Nota de rutas:** esta práctica usa `C:\LabSQL2025\Backup` y `C:\LabSQL2025\Reports` para los artefactos de migración, tal como establece su especificación. Para los archivos restaurados se usan las rutas estándar del laboratorio `C:\SQLLab2025\Data` y `C:\SQLLab2025\Logs`.

## Entorno de Laboratorio

| Componente | Origen | Destino |
|---|---|---|
| Instancia | `SQL2017SRC` | `SQL2025TGT` |
| Motor | SQL Server 2017 Developer Edition | SQL Server 2025 Developer Edition |
| Base de datos | `MigracionLab_Source` | `MigracionLab_2025` |
| Propósito | Base de datos original; no debe modificarse | Base de datos restaurada para validación |
| Herramientas | SSMS 21.5.0, DMA 5.6.6784.0, PowerShell 7.4.6 | SSMS 21.5.0, PowerShell 7.4.6 |

Ejecute PowerShell **como administrador** en los equipos que alojan las instancias correspondientes para crear los directorios requeridos si aún no existen:

```powershell
New-Item -ItemType Directory -Force -Path `
  "C:\LabSQL2025\Backup", `
  "C:\LabSQL2025\Reports", `
  "C:\SQLLab2025\Data", `
  "C:\SQLLab2025\Logs"
```

Compruebe que la cuenta de servicio del motor SQL Server tenga permisos de modificación sobre las carpetas de respaldo, datos y log. El comando siguiente muestra la identidad de servicio configurada:

```sql
SELECT
    servicename,
    service_account,
    status_desc,
    startup_type_desc
FROM sys.dm_server_services;
```

> **Importante:** no incluya contraseñas en scripts, informes, capturas de pantalla ni conversaciones con asistentes de IA. Use autenticación de Windows para las tareas administrativas del laboratorio siempre que sea posible.

## Instrucciones Paso a Paso

### Paso 1: Confirmar conectividad y preparar directorios

**Objetivo:** comprobar que las instancias de origen y destino están disponibles, identificar sus versiones y preparar una ubicación controlada para los artefactos de migración.

**Instrucciones**

1. Abra SSMS y conéctese a `SQL2017SRC` mediante autenticación de Windows.
2. Abra una segunda ventana de SSMS y conéctese a `SQL2025TGT`.
3. En cada instancia, ejecute la siguiente consulta para registrar versión, edición, servidor y fecha de inicio:

```sql
SELECT
    @@SERVERNAME AS nombre_servidor,
    SERVERPROPERTY('MachineName') AS equipo,
    SERVERPROPERTY('InstanceName') AS instancia,
    SERVERPROPERTY('Edition') AS edicion,
    SERVERPROPERTY('ProductVersion') AS version_producto,
    SERVERPROPERTY('ProductLevel') AS nivel_producto,
    SERVERPROPERTY('ProductUpdateLevel') AS nivel_actualizacion,
    SERVERPROPERTY('ProductUpdateReference') AS referencia_actualizacion,
    sqlserver_start_time AS inicio_servicio
FROM sys.dm_os_sys_info;
```

4. En el equipo de origen, confirme que existe la carpeta de respaldos:

```powershell
Test-Path "C:\LabSQL2025\Backup"
```

5. En el equipo de destino, confirme que existen las rutas de datos, logs y reportes:

```powershell
Test-Path "C:\SQLLab2025\Data"
Test-Path "C:\SQLLab2025\Logs"
Test-Path "C:\LabSQL2025\Reports"
```

6. Registre en su informe el nombre de cada instancia, versión exacta del motor, edición y ubicación física de cada carpeta.

**Resultado esperado**

- `SQL2017SRC` responde con información de SQL Server 2017.
- `SQL2025TGT` responde con información de SQL Server 2025.
- Todas las rutas requeridas existen y son accesibles.

**Verificación**

Ejecute en ambas instancias:

```sql
SELECT
    SERVERPROPERTY('ServerName') AS servidor,
    SERVERPROPERTY('ProductVersion') AS version,
    SERVERPROPERTY('Edition') AS edicion;
```

Registre la salida en `C:\LabSQL2025\Reports\06-00-01_RegistroEntorno.txt`.

---

### Paso 2: Inventariar el entorno de origen

**Objetivo:** recopilar una línea base técnica de `MigracionLab_Source` antes de iniciar la evaluación de compatibilidad y la migración.

**Instrucciones**

1. Conéctese a `SQL2017SRC`.
2. Ejecute la siguiente consulta para inventariar la base de datos de origen:

```sql
SELECT
    name AS nombre_base_datos,
    compatibility_level AS nivel_compatibilidad,
    recovery_model_desc AS modelo_recuperacion,
    state_desc AS estado,
    user_access_desc AS acceso_usuarios,
    page_verify_option_desc AS verificacion_paginas,
    collation_name AS intercalacion,
    create_date AS fecha_creacion
FROM sys.databases
WHERE name = N'MigracionLab_Source';
```

3. Cambie el contexto a la base de datos de origen y registre tamaño, ubicación y espacio utilizado de sus archivos:

```sql
USE [MigracionLab_Source];
GO

SELECT
    DB_NAME() AS base_datos,
    name AS nombre_archivo,
    type_desc AS tipo_archivo,
    physical_name AS ruta_fisica,
    CAST(size / 128.0 AS decimal(18,2)) AS tamano_mb,
    CAST(FILEPROPERTY(name, 'SpaceUsed') / 128.0 AS decimal(18,2)) AS espacio_usado_mb,
    CAST((size - FILEPROPERTY(name, 'SpaceUsed')) / 128.0 AS decimal(18,2)) AS espacio_libre_mb
FROM sys.database_files;
```

4. Obtenga la configuración relevante de la instancia de origen:

```sql
SELECT
    name,
    value_in_use AS valor_configurado,
    value AS valor_configurado_sp_configure,
    description
FROM sys.configurations
WHERE name IN
(
    'max server memory (MB)',
    'max degree of parallelism',
    'cost threshold for parallelism',
    'backup compression default'
)
ORDER BY name;
```

5. Inventaríe tablas, vistas, procedimientos almacenados, funciones y triggers:

```sql
USE [MigracionLab_Source];
GO

SELECT
    SCHEMA_NAME(o.schema_id) AS esquema,
    o.name AS objeto,
    o.type_desc AS tipo_objeto,
    o.create_date AS fecha_creacion,
    o.modify_date AS fecha_modificacion
FROM sys.objects AS o
WHERE o.is_ms_shipped = 0
ORDER BY
    o.type_desc,
    esquema,
    objeto;
```

6. Identifique módulos programables y registre una huella SHA-256 de sus definiciones para facilitar la comparación posterior:

```sql
SELECT
    SCHEMA_NAME(o.schema_id) AS esquema,
    o.name AS objeto,
    o.type_desc AS tipo_objeto,
    CONVERT(varchar(64), HASHBYTES(
        'SHA2_256',
        CONVERT(varbinary(max), sm.definition)
    ), 2) AS hash_definicion
FROM sys.objects AS o
INNER JOIN sys.sql_modules AS sm
    ON o.object_id = sm.object_id
WHERE o.is_ms_shipped = 0
ORDER BY
    esquema,
    objeto;
```

7. Inventaríe usuarios, roles y asociaciones con logins:

```sql
SELECT
    dp.name AS usuario_base_datos,
    dp.type_desc AS tipo_usuario,
    dp.authentication_type_desc AS tipo_autenticacion,
    sp.name AS login_asociado,
    dp.default_schema_name
FROM sys.database_principals AS dp
LEFT JOIN sys.server_principals AS sp
    ON dp.sid = sp.sid
WHERE dp.type NOT IN ('A', 'G', 'R', 'X')
  AND dp.name NOT IN ('dbo', 'guest', 'INFORMATION_SCHEMA', 'sys')
ORDER BY dp.name;
```

8. Revise dependencias de objetos dentro de la base de datos:

```sql
SELECT
    OBJECT_SCHEMA_NAME(d.referencing_id) AS esquema_origen,
    OBJECT_NAME(d.referencing_id) AS objeto_origen,
    d.referenced_schema_name AS esquema_referenciado,
    d.referenced_entity_name AS objeto_referenciado,
    d.referenced_database_name AS base_referenciada,
    d.referenced_server_name AS servidor_referenciado
FROM sys.sql_expression_dependencies AS d
WHERE d.referencing_id IS NOT NULL
ORDER BY
    esquema_origen,
    objeto_origen;
```

9. Inventaríe trabajos de SQL Server Agent. Recuerde que los trabajos de `msdb` no se transfieren mediante una restauración de base de datos:

```sql
USE [msdb];
GO

SELECT
    j.name AS nombre_trabajo,
    j.enabled AS habilitado,
    j.description AS descripcion,
    SUSER_SNAME(j.owner_sid) AS propietario,
    j.date_created AS fecha_creacion,
    j.date_modified AS fecha_modificacion
FROM dbo.sysjobs AS j
ORDER BY j.name;
```

10. Revise características persistentes de SQL Server utilizadas por la base:

```sql
USE [MigracionLab_Source];
GO

SELECT
    feature_name,
    feature_id
FROM sys.dm_db_persisted_sku_features
ORDER BY feature_name;
```

**Resultado esperado**

- Se dispone de un inventario de versión, compatibilidad, recuperación, archivos, objetos, usuarios, dependencias y trabajos.
- Se identifican componentes que no se trasladan mediante `BACKUP DATABASE` y `RESTORE DATABASE`, especialmente logins, trabajos de SQL Server Agent, linked servers y configuraciones de instancia.
- La base de datos de origen permanece sin modificaciones.

**Verificación**

Exporte los resultados de las consultas a archivos CSV o texto y guárdelos con el prefijo `06-00-01_Origen_` en:

```text
C:\LabSQL2025\Reports
```

Como mínimo, conserve:

```text
06-00-01_Origen_BaseDatos.csv
06-00-01_Origen_Archivos.csv
06-00-01_Origen_Objetos.csv
06-00-01_Origen_Usuarios.csv
06-00-01_Origen_Dependencias.csv
06-00-01_Origen_TrabajosAgent.csv
```

---

### Paso 3: Evaluar compatibilidad con Data Migration Assistant

**Objetivo:** identificar bloqueadores, cambios importantes y recomendaciones de compatibilidad antes de transferir la base de datos.

**Instrucciones**

1. Abra **Data Migration Assistant 5.6.6784.0**.
2. Seleccione **New** y cree un proyecto de tipo **Assessment**.
3. Asigne un nombre descriptivo, por ejemplo:

```text
DMA_MigracionLab_Source_2017_a_2025
```

4. Seleccione **SQL Server** como tipo de servidor de origen.
5. Configure la instancia de origen como `SQL2017SRC` y autentíquese con una cuenta administrativa del laboratorio.
6. Seleccione la base de datos `MigracionLab_Source`.
7. Seleccione **SQL Server 2025** como plataforma de destino, si está disponible en la versión instalada de DMA.
8. Incluya las categorías de evaluación disponibles, especialmente:
   - Problemas de compatibilidad.
   - Cambios importantes.
   - Características obsoletas.
   - Recomendaciones de rendimiento y comportamiento.
9. Ejecute la evaluación.
10. Revise cada hallazgo y clasifíquelo en una de las siguientes categorías:
    - **Bloqueador:** impide o desaconseja la migración hasta aplicar una corrección.
    - **Advertencia:** requiere prueba, revisión o mitigación.
    - **Información:** no impide la migración, pero debe documentarse.
11. Exporte el informe de DMA en formato HTML o JSON y guárdelo en:

```text
C:\LabSQL2025\Reports\06-00-01_DMA_MigracionLab_Source.html
```

12. Si DMA no ofrece explícitamente SQL Server 2025 como destino, documente esta limitación en el informe y use la versión de destino más reciente disponible únicamente como evaluación complementaria. No interprete la ausencia de hallazgos como garantía de compatibilidad.

**Resultado esperado**

- Se genera un informe de compatibilidad para `MigracionLab_Source`.
- Se identifican los elementos que requieren corrección o pruebas adicionales.
- Se dispone de evidencia exportable para la decisión técnica de migración.

**Verificación**

Complete una tabla de hallazgos en `C:\LabSQL2025\Reports\06-00-01_MatrizHallazgos.csv`:

| ID | Hallazgo | Severidad | Impacto potencial | Acción | Estado |
|---|---|---|---|---|---|
| DMA-01 | Ejemplo: característica obsoleta | Advertencia | Cambio de comportamiento | Probar consulta o reemplazar característica | Pendiente/Resuelto |
| DMA-02 | Ejemplo: dependencia externa | Información | Error de conectividad | Validar servidor y credenciales | Pendiente |

No continúe con una migración real si DMA identifica un bloqueador no evaluado. En este laboratorio, documente el bloqueador y continúe solo si el instructor autoriza la simulación.

---

### Paso 4: Crear y verificar el respaldo consistente del origen

**Objetivo:** crear una copia de seguridad consistente de `MigracionLab_Source`, verificar su legibilidad y calcular un hash para comprobar su integridad durante la transferencia.

**Instrucciones**

1. En `SQL2017SRC`, confirme que no se está ejecutando una operación de respaldo o restauración sobre la base:

```sql
SELECT
    r.session_id,
    r.command,
    r.status,
    r.percent_complete,
    r.start_time,
    r.estimated_completion_time / 1000.0 AS segundos_estimados
FROM sys.dm_exec_requests AS r
WHERE r.command IN ('BACKUP DATABASE', 'RESTORE DATABASE');
```

2. Cree una copia de seguridad completa de solo copia. Sustituya la fecha del nombre de archivo por la fecha real de ejecución si es necesario:

```sql
BACKUP DATABASE [MigracionLab_Source]
TO DISK = N'C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak'
WITH
    COPY_ONLY,
    CHECKSUM,
    COMPRESSION,
    STATS = 10,
    NAME = N'MigracionLab_Source - Respaldo de migración';
GO
```

3. Verifique el archivo sin restaurarlo:

```sql
RESTORE VERIFYONLY
FROM DISK = N'C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak'
WITH CHECKSUM;
GO
```

4. En PowerShell, calcule el hash SHA-256 del archivo generado:

```powershell
Get-FileHash `
  -Path "C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak" `
  -Algorithm SHA256 |
  Format-List
```

5. Guarde el resultado del hash en un archivo:

```powershell
Get-FileHash `
  -Path "C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak" `
  -Algorithm SHA256 |
  Out-File "C:\LabSQL2025\Reports\06-00-01_HashOrigen_SHA256.txt"
```

6. Registre el tamaño del archivo y la hora de finalización del respaldo:

```powershell
Get-Item "C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak" |
    Select-Object Name, Length, CreationTime, LastWriteTime
```

**Resultado esperado**

- El respaldo se completa sin errores.
- `RESTORE VERIFYONLY` confirma que el conjunto de respaldo es válido.
- Se obtiene una huella SHA-256 del archivo de respaldo.

**Verificación**

La salida de `RESTORE VERIFYONLY` debe indicar:

```text
The backup set on file 1 is valid.
```

El archivo `06-00-01_HashOrigen_SHA256.txt` debe contener el hash SHA-256 y la ruta del archivo de origen.

> **Nota:** `COPY_ONLY` evita alterar la secuencia normal de respaldos diferenciales de un entorno real. En este laboratorio también reduce el impacto sobre las prácticas previas.

---

### Paso 5: Transferir el respaldo y comprobar su integridad en el destino

**Objetivo:** transferir el respaldo al entorno destino sin alterar el archivo original y demostrar que el archivo recibido es idéntico al generado en origen.

**Instrucciones**

1. Transfiera el archivo `.bak` desde el origen hacia el directorio de respaldo del destino. Si ambos servicios están en equipos diferentes y existe una ruta compartida autorizada, use `robocopy`:

```powershell
robocopy `
  "\\SQL2017SRC\LabBackup" `
  "C:\LabSQL2025\Backup" `
  "MigracionLab_Source_2025-09-22.bak" `
  /Z /J /R:2 /W:5 /COPY:DAT
```

2. Si el archivo se transfiere mediante un recurso compartido diferente, una herramienta de transferencia aprobada o una copia local, documente el método utilizado y la ruta final.

3. En el servidor de destino, calcule el hash SHA-256 del archivo recibido:

```powershell
Get-FileHash `
  -Path "C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak" `
  -Algorithm SHA256 |
  Out-File "C:\LabSQL2025\Reports\06-00-01_HashDestino_SHA256.txt"
```

4. Compare visualmente ambos hashes o utilice PowerShell:

```powershell
$hashOrigen = (Get-Content "C:\LabSQL2025\Reports\06-00-01_HashOrigen_SHA256.txt" |
    Select-String -Pattern "^[A-Fa-f0-9]{64}" |
    Select-Object -First 1).ToString().Trim()

$hashDestino = (Get-Content "C:\LabSQL2025\Reports\06-00-01_HashDestino_SHA256.txt" |
    Select-String -Pattern "^[A-Fa-f0-9]{64}" |
    Select-Object -First 1).ToString().Trim()

$hashOrigen -eq $hashDestino
```

5. En `SQL2025TGT`, ejecute una segunda validación del respaldo:

```sql
RESTORE VERIFYONLY
FROM DISK = N'C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak'
WITH CHECKSUM;
GO
```

**Resultado esperado**

- El archivo de destino tiene el mismo hash SHA-256 que el archivo del origen.
- El archivo puede ser leído correctamente por SQL Server 2025.
- Se conserva el archivo original en el origen como evidencia y como elemento del plan de reversión.

**Verificación**

La comparación de hash debe devolver:

```text
True
```

No continúe si el hash difiere o si `RESTORE VERIFYONLY` devuelve errores. Elimine la copia incompleta del destino, repita la transferencia y vuelva a verificar.

---

### Paso 6: Restaurar la base de datos en SQL Server 2025

**Objetivo:** restaurar la base de datos con un nuevo nombre, manteniendo inicialmente el nivel de compatibilidad heredado.

**Instrucciones**

1. En `SQL2025TGT`, identifique los nombres lógicos de los archivos contenidos en el respaldo:

```sql
RESTORE FILELISTONLY
FROM DISK = N'C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak';
GO
```

2. Registre los valores de las columnas `LogicalName` y `Type`. Normalmente habrá al menos un archivo de datos (`D`) y un archivo de log (`L`).

3. Compruebe que no existe previamente la base `MigracionLab_2025`:

```sql
SELECT
    name,
    state_desc,
    compatibility_level
FROM sys.databases
WHERE name = N'MigracionLab_2025';
```

4. Restaure la base usando `MOVE`. Reemplace `MigracionLab_Source_Data` y `MigracionLab_Source_Log` por los nombres lógicos obtenidos en el paso anterior:

```sql
RESTORE DATABASE [MigracionLab_2025]
FROM DISK = N'C:\LabSQL2025\Backup\MigracionLab_Source_2025-09-22.bak'
WITH
    MOVE N'MigracionLab_Source_Data'
        TO N'C:\SQLLab2025\Data\MigracionLab_2025.mdf',
    MOVE N'MigracionLab_Source_Log'
        TO N'C:\SQLLab2025\Logs\MigracionLab_2025_log.ldf',
    STATS = 10,
    RECOVERY;
GO
```

5. Si el respaldo contiene archivos de datos secundarios, agregue una cláusula `MOVE` por cada archivo de tipo `D`:

```sql
MOVE N'NombreLogicoSecundario'
    TO N'C:\SQLLab2025\Data\MigracionLab_2025_Secondary.ndf',
```

6. Consulte la configuración de la base restaurada:

```sql
SELECT
    name,
    state_desc,
    compatibility_level,
    recovery_model_desc,
    collation_name,
    page_verify_option_desc
FROM sys.databases
WHERE name = N'MigracionLab_2025';
```

7. Habilite Query Store en modo de lectura y escritura para capturar la línea base de consultas. No cambie aún el nivel de compatibilidad:

```sql
ALTER DATABASE [MigracionLab_2025]
SET QUERY_STORE = ON;
GO

ALTER DATABASE [MigracionLab_2025]
SET QUERY_STORE
(
    OPERATION_MODE = READ_WRITE,
    QUERY_CAPTURE_MODE = AUTO
);
GO
```

**Resultado esperado**

- La base de datos se restaura como `MigracionLab_2025`.
- El nivel de compatibilidad inicialmente coincide con el valor heredado desde SQL Server 2017.
- Query Store queda habilitado y en modo `READ_WRITE`.

**Verificación**

Ejecute:

```sql
SELECT
    d.name,
    d.state_desc,
    d.compatibility_level,
    d.recovery_model_desc,
    q.actual_state_desc AS query_store_estado
FROM sys.databases AS d
LEFT JOIN sys.database_query_store_options AS q
    ON d.database_id = q.database_id
WHERE d.name = N'MigracionLab_2025';
```

La base debe estar en estado `ONLINE` y Query Store debe indicar `READ_WRITE`.

---

### Paso 7: Validar integridad, datos, objetos y seguridad antes del cambio de compatibilidad

**Objetivo:** demostrar que la base restaurada es coherente y funcional antes de cambiar a nivel de compatibilidad 170.

**Instrucciones**

1. Ejecute una comprobación de integridad completa en la base restaurada:

```sql
DBCC CHECKDB (N'MigracionLab_2025')
WITH NO_INFOMSGS, ALL_ERRORMSGS;
GO
```

2. Valide restricciones y referencias declaradas:

```sql
USE [MigracionLab_2025];
GO

DBCC CHECKCONSTRAINTS WITH ALL_CONSTRAINTS;
GO
```

3. Obtenga los conteos de filas de todas las tablas de usuario. Ejecute la misma consulta en origen y destino, exporte ambos resultados y compárelos:

```sql
SELECT
    SCHEMA_NAME(t.schema_id) AS esquema,
    t.name AS tabla,
    SUM(ps.row_count) AS filas
FROM sys.tables AS t
INNER JOIN sys.dm_db_partition_stats AS ps
    ON t.object_id = ps.object_id
WHERE t.is_ms_shipped = 0
  AND ps.index_id IN (0, 1)
GROUP BY
    t.schema_id,
    t.name
ORDER BY
    esquema,
    tabla;
```

4. Compare la definición de los objetos programables mediante la huella SHA-256. Ejecute la consulta siguiente en origen y en destino:

```sql
SELECT
    SCHEMA_NAME(o.schema_id) AS esquema,
    o.name AS objeto,
    o.type_desc AS tipo_objeto,
    CONVERT(varchar(64), HASHBYTES(
        'SHA2_256',
        CONVERT(varbinary(max), sm.definition)
    ), 2) AS hash_definicion
FROM sys.objects AS o
INNER JOIN sys.sql_modules AS sm
    ON o.object_id = sm.object_id
WHERE o.is_ms_shipped = 0
ORDER BY
    esquema,
    objeto;
```

5. Compruebe el propietario de la base restaurada:

```sql
SELECT
    name AS base_datos,
    SUSER_SNAME(owner_sid) AS propietario
FROM sys.databases
WHERE name = N'MigracionLab_2025';
```

6. Si el propietario no es el esperado para el entorno de laboratorio, asígnelo de manera controlada:

```sql
ALTER AUTHORIZATION ON DATABASE::[MigracionLab_2025] TO [sa];
GO
```

7. Detecte usuarios huérfanos. Un usuario SQL puede existir dentro de la base restaurada, pero su login asociado no se transfiere mediante el respaldo:

```sql
USE [MigracionLab_2025];
GO

SELECT
    dp.name AS usuario_huerfano,
    dp.type_desc AS tipo_usuario,
    dp.authentication_type_desc AS tipo_autenticacion
FROM sys.database_principals AS dp
LEFT JOIN sys.server_principals AS sp
    ON dp.sid = sp.sid
WHERE dp.type = 'S'
  AND dp.authentication_type = 1
  AND dp.name NOT IN ('dbo', 'guest', 'sys', 'INFORMATION_SCHEMA')
  AND sp.sid IS NULL
ORDER BY dp.name;
```

8. Para cada usuario detectado, confirme con el responsable del laboratorio que existe un login equivalente en destino. Después, corrija la asociación de forma explícita:

```sql
ALTER USER [NombreUsuario] WITH LOGIN = [NombreLogin];
GO
```

9. No cree logins ni copie contraseñas sin autorización. Si el login no existe en destino, documente el hallazgo y use el procedimiento autorizado de creación de logins del laboratorio.

10. Ejecute pruebas funcionales con procedimientos, funciones y vistas creados en prácticas anteriores. Use consultas representativas conocidas y no destructivas. Ejemplos de patrón:

```sql
USE [MigracionLab_2025];
GO

-- Sustituya por una vista creada en prácticas anteriores.
SELECT TOP (10) *
FROM dbo.NombreVistaExistente;
GO

-- Sustituya por un procedimiento de consulta conocido.
EXEC dbo.NombreProcedimientoExistente
    @ParametroEjemplo = 1;
GO
```

11. Antes de cada consulta representativa, active estadísticas de tiempo e I/O y habilite el plan de ejecución real en SSMS con **Ctrl+M**:

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
GO

-- Ejecute aquí una consulta representativa no destructiva.
SELECT TOP (100) *
FROM dbo.NombreTablaExistente;
GO

SET STATISTICS IO OFF;
SET STATISTICS TIME OFF;
GO
```

**Resultado esperado**

- `DBCC CHECKDB` no informa errores de asignación, consistencia ni corrupción.
- Los conteos de filas de origen y destino coinciden.
- Los objetos programables tienen la misma huella, salvo cambios documentados.
- Los usuarios válidos están asociados a logins correctos en destino.
- Las consultas y módulos representativos producen los resultados esperados.

**Verificación**

Conserve como evidencia:

```text
06-00-01_Destino_CHECKDB.txt
06-00-01_Conteos_Origen.csv
06-00-01_Conteos_Destino.csv
06-00-01_ComparacionObjetos.csv
06-00-01_UsuariosHuerfanos.csv
06-00-01_PruebasFuncionales_Pre170.txt
```

Si existen diferencias de datos, definiciones, permisos o resultados funcionales, clasifique el estado como **Corrección requerida** y no eleve aún el nivel de compatibilidad.

---

### Paso 8: Elevar la compatibilidad a 170 y repetir pruebas representativas

**Objetivo:** cambiar de forma controlada el nivel de compatibilidad de la base restaurada a 170 y evaluar posibles cambios de comportamiento o rendimiento.

**Instrucciones**

1. Registre el nivel de compatibilidad actual:

```sql
SELECT
    name,
    compatibility_level
FROM sys.databases
WHERE name = N'MigracionLab_2025';
```

2. Revise las conexiones activas a la base. Programe el cambio durante una ventana de mantenimiento del laboratorio:

```sql
SELECT
    session_id,
    login_name,
    host_name,
    program_name,
    status
FROM sys.dm_exec_sessions
WHERE database_id = DB_ID(N'MigracionLab_2025')
  AND session_id <> @@SPID;
```

3. Cambie el nivel de compatibilidad:

```sql
ALTER DATABASE [MigracionLab_2025]
SET COMPATIBILITY_LEVEL = 170;
GO
```

4. Compruebe el cambio:

```sql
SELECT
    name,
    compatibility_level
FROM sys.databases
WHERE name = N'MigracionLab_2025';
```

5. Ejecute nuevamente las mismas consultas representativas utilizadas antes del cambio. Mantenga constantes, parámetros, número de ejecuciones y condiciones de prueba.

6. Active el plan de ejecución real en SSMS y capture:
   - Duración total.
   - Lecturas lógicas.
   - Uso de CPU, si se informa en `STATISTICS TIME`.
   - Número estimado y real de filas.
   - Operadores nuevos o cambios relevantes del plan.
   - Resultado funcional de la consulta o procedimiento.

7. Revise Query Store para identificar consultas ejecutadas y sus métricas:

```sql
USE [MigracionLab_2025];
GO

SELECT TOP (20)
    q.query_id,
    p.plan_id,
    rs.count_executions,
    CAST(rs.avg_duration / 1000.0 AS decimal(18,2)) AS duracion_promedio_ms,
    CAST(rs.avg_cpu_time / 1000.0 AS decimal(18,2)) AS cpu_promedio_ms,
    rs.avg_logical_io_reads,
    qt.query_sql_text
FROM sys.query_store_query AS q
INNER JOIN sys.query_store_query_text AS qt
    ON q.query_text_id = qt.query_text_id
INNER JOIN sys.query_store_plan AS p
    ON q.query_id = p.query_id
INNER JOIN sys.query_store_runtime_stats AS rs
    ON p.plan_id = rs.plan_id
ORDER BY rs.avg_duration DESC;
```

8. Revise los planes almacenados para una consulta específica, sustituyendo el identificador correspondiente:

```sql
SELECT
    p.query_id,
    p.plan_id,
    p.is_forced_plan,
    p.last_execution_time,
    p.query_plan
FROM sys.query_store_plan AS p
WHERE p.query_id = 1;
```

9. Documente los resultados en una tabla comparativa.

| Consulta o módulo | Resultado pre-170 | Resultado post-170 | Duración pre-170 | Duración post-170 | Plan cambió | Decisión |
|---|---|---|---:|---:|---|---|
| Consulta representativa 1 | Correcto | Correcto | Registrar | Registrar | Sí/No | Aceptar/Investigar |
| Procedimiento representativo | Correcto | Correcto | Registrar | Registrar | Sí/No | Aceptar/Investigar |

**Resultado esperado**

- `MigracionLab_2025` queda en nivel de compatibilidad `170`.
- Las consultas funcionales devuelven resultados equivalentes a los obtenidos antes del cambio.
- Query Store registra consultas y planes para análisis posterior.
- Cualquier regresión de rendimiento o cambio funcional queda documentado.

**Verificación**

Ejecute:

```sql
SELECT
    d.name,
    d.compatibility_level,
    q.actual_state_desc AS query_store_estado,
    q.desired_state_desc AS query_store_estado_deseado
FROM sys.databases AS d
INNER JOIN sys.database_query_store_options AS q
    ON d.database_id = q.database_id
WHERE d.name = N'MigracionLab_2025';
```

El resultado debe indicar compatibilidad `170` y Query Store en `READ_WRITE`.

---

### Paso 9: Elaborar la decisión técnica y el plan de reversión

**Objetivo:** consolidar la evidencia recolectada y decidir si la simulación de migración es aceptable, requiere corrección o debe revertirse.

**Instrucciones**

1. Cree el archivo:

```text
C:\LabSQL2025\Reports\06-00-01_InformeDecisionMigracion.md
```

2. Incluya los siguientes apartados:
   - Identificación de origen y destino.
   - Fecha y hora de respaldo, transferencia y restauración.
   - Hash SHA-256 en origen y destino.
   - Resultado de DMA.
   - Resultado de `RESTORE VERIFYONLY`.
   - Resultado de `DBCC CHECKDB`.
   - Comparación de conteos por tabla.
   - Comparación de objetos programables.
   - Estado de usuarios, logins, roles y permisos.
   - Resultado de pruebas funcionales.
   - Resultado de pruebas previas y posteriores a compatibilidad 170.
   - Riesgos y acciones de mitigación.
   - Decisión final.

3. Registre riesgos comunes de migración y su tratamiento:

| Riesgo | Evidencia o señal | Mitigación |
|---|---|---|
| Usuarios huérfanos | Usuario de base sin login asociado | Crear o asociar login autorizado con `ALTER USER ... WITH LOGIN` |
| Trabajos no migrados | Trabajos presentes en `msdb` de origen | Script y recreación controlada de SQL Server Agent Jobs |
| Dependencias externas | Linked servers, referencias entre bases o servidores | Validar red, nombres, credenciales y permisos |
| Cambio de planes | Diferencias en plan o duración tras compatibilidad 170 | Analizar Query Store, estadísticas, índices y consultas |
| Archivo transferido incorrectamente | Hash SHA-256 diferente | Repetir transferencia y verificar nuevamente |
| Problema de integridad | Errores de `DBCC CHECKDB` | Detener aceptación, investigar y restaurar una copia válida |

4. Use los siguientes criterios de aceptación:
   - DMA no contiene bloqueadores sin resolver o aceptar formalmente.
   - El hash del respaldo es idéntico en origen y destino.
   - `RESTORE VERIFYONLY` y `DBCC CHECKDB` finalizan correctamente.
   - Los conteos de datos y objetos coinciden o las diferencias están justificadas.
   - No existen usuarios huérfanos sin tratamiento.
   - Las pruebas funcionales críticas son correctas.
   - Las consultas representativas no presentan regresiones inaceptables.
   - El plan de reversión está documentado y es viable.

5. Registre una de las decisiones siguientes:
   - **Aceptación:** todos los criterios esenciales se cumplen.
   - **Corrección requerida:** existen hallazgos reparables antes de una migración real.
   - **Reversión:** se detectó un problema crítico de integridad, funcionalidad, seguridad o rendimiento.

6. Documente el plan de reversión:
   - Mantener `MigracionLab_Source` como base original y sin modificaciones.
   - Conservar el archivo `.bak` original y su hash.
   - No redirigir aplicaciones de producción al destino hasta aprobar los criterios de aceptación.
   - Si se hubiese realizado un cambio de aplicación, redirigir la conexión nuevamente a la instancia origen.
   - Investigar el hallazgo en una nueva copia restaurada, sin modificar la evidencia original.

**Resultado esperado**

- Se genera una decisión técnica basada en evidencias.
- Los riesgos, acciones y responsables quedan documentados.
- Existe un plan claro de retorno al origen.

**Verificación**

El informe debe concluir con una declaración similar a una de las siguientes:

```text
Decisión: Aceptación condicionada.
Justificación: la integridad, conteos, objetos y pruebas funcionales fueron correctos.
Acción pendiente: recrear y validar los trabajos de SQL Server Agent antes de una migración productiva.
```

```text
Decisión: Corrección requerida.
Justificación: se detectaron usuarios huérfanos y una regresión de duración en una consulta crítica tras elevar la compatibilidad a 170.
Acción: corregir asociación de logins y analizar el plan en Query Store antes de aceptar.
```

## Validación y Pruebas

Utilice la siguiente lista de comprobación final:

| Validación | Comando o evidencia | Criterio de aprobación |
|---|---|---|
| Inventario de origen | Informes de consultas de `sys.databases`, objetos, usuarios y dependencias | Archivos exportados y revisados |
| Evaluación DMA | Informe HTML o JSON | Bloqueadores resueltos, aceptados o documentados |
| Respaldo | `BACKUP DATABASE ... WITH CHECKSUM` | Finaliza sin error |
| Verificación del respaldo | `RESTORE VERIFYONLY ... WITH CHECKSUM` | Respaldo válido en origen y destino |
| Integridad de transferencia | `Get-FileHash -Algorithm SHA256` | Hashes idénticos |
| Restauración | `RESTORE DATABASE ... WITH MOVE` | Base `ONLINE` en destino |
| Integridad lógica y física | `DBCC CHECKDB` | Sin errores |
| Integridad referencial | `DBCC CHECKCONSTRAINTS` | Sin errores de restricciones |
| Datos | Conteos por tabla | Coinciden con origen |
| Objetos | Hash SHA-256 de módulos | Coinciden con origen o hay justificación |
| Seguridad | Consulta de usuarios huérfanos | Sin usuarios sin tratamiento |
| Compatibilidad | `compatibility_level = 170` | Cambio registrado |
| Query Store | `actual_state_desc = READ_WRITE` | Captura habilitada |
| Rendimiento funcional | Estadísticas, planes y Query Store | Sin regresiones no aceptadas |
| Decisión | Informe final | Aceptación, corrección o reversión documentada |

Ejecute esta consulta final en `SQL2025TGT`:

```sql
SELECT
    d.name AS base_datos,
    d.state_desc AS estado,
    d.compatibility_level AS nivel_compatibilidad,
    d.recovery_model_desc AS modelo_recuperacion,
    q.actual_state_desc AS query_store
FROM sys.databases AS d
LEFT JOIN sys.database_query_store_options AS q
    ON d.database_id = q.database_id
WHERE d.name = N'MigracionLab_2025';
```

El resultado esperado es:

- Base de datos: `MigracionLab_2025`.
- Estado: `ONLINE`.
- Nivel de compatibilidad: `170`.
- Query Store: `READ_WRITE`.

## Solución de Problemas

### Problema 1: La restauración falla porque no se pueden crear los archivos en la ruta de destino

**Síntomas**

- Error similar a `Operating system error 3` o `Access is denied`.
- Error que indica que el archivo físico no puede crearse.
- La restauración falla al ejecutar `RESTORE DATABASE ... WITH MOVE`.

**Causa**

La ruta de datos o logs no existe en el servidor de destino, o la cuenta de servicio de SQL Server no tiene permisos de escritura. También es posible que los nombres lógicos usados en `MOVE` no coincidan con los nombres devueltos por `RESTORE FILELISTONLY`.

**Corrección**

1. Ejecute `RESTORE FILELISTONLY` y copie exactamente los valores de `LogicalName`.
2. Cree las carpetas requeridas en el servidor destino.
3. Compruebe la cuenta de servicio con `sys.dm_server_services`.
4. Conceda permisos de modificación a esa cuenta sobre `C:\SQLLab2025\Data` y `C:\SQLLab2025\Logs`.
5. Ejecute nuevamente la restauración con una cláusula `MOVE` para cada archivo del respaldo.

### Problema 2: Se detectan usuarios huérfanos después de la restauración

**Síntomas**

- La consulta de usuarios huérfanos devuelve filas.
- Un usuario puede existir en `MigracionLab_2025`, pero no puede iniciar sesión o acceder a la base.
- Una aplicación recibe errores de autenticación o de usuario no asociado a un login.

**Causa**

Los usuarios de base de datos se incluyen en el respaldo, pero los logins son objetos de instancia y no se restauran automáticamente desde SQL Server 2017 al servidor SQL Server 2025.

**Corrección**

1. Identifique el login equivalente autorizado en `SQL2025TGT`.
2. Cree o migre el login mediante el procedimiento de seguridad aprobado, preservando el SID cuando sea necesario.
3. Asocie el usuario al login:

```sql
USE [MigracionLab_2025];
GO
ALTER USER [NombreUsuario] WITH LOGIN = [NombreLogin];
GO
```

4. Vuelva a ejecutar la consulta de detección de usuarios huérfanos.
5. Pruebe los permisos con una cuenta de prueba autorizada; no otorgue `sysadmin` como solución rápida.

## Limpieza

1. No elimine `MigracionLab_Source`; es la referencia de origen y forma parte del plan de reversión.
2. No elimine el respaldo original ni los archivos de hash mientras la práctica esté en revisión.
3. Mantenga `MigracionLab_2025` para análisis posterior, salvo que el instructor solicite su eliminación.
4. Si se requiere retirar la base restaurada al finalizar el laboratorio, confirme primero que los informes estén guardados y ejecute en `SQL2025TGT`:

```sql
USE [master];
GO

ALTER DATABASE [MigracionLab_2025]
SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO

DROP DATABASE [MigracionLab_2025];
GO
```

5. Si se elimina la base, conserve como mínimo:
   - El informe DMA.
   - Los hashes SHA-256.
   - El informe de decisión.
   - Los resultados de `DBCC CHECKDB`.
   - Las comparaciones de conteos, objetos y seguridad.
6. Elimine copias temporales incompletas del respaldo únicamente después de comprobar que la copia validada y su evidencia permanecen disponibles.

## Resumen

En esta práctica se aplicó un proceso controlado de simulación de migración desde SQL Server 2017 hacia SQL Server 2025. Se inventarió el entorno de origen, se evaluó la compatibilidad con DMA, se creó un respaldo verificable con `CHECKSUM` y SHA-256, se restauró la base en un destino aislado y se validaron integridad, datos, objetos, permisos y funcionamiento.

La elevación del nivel de compatibilidad a 170 se realizó únicamente después de una validación inicial satisfactoria. Query Store, estadísticas de ejecución y planes reales permitieron comparar el comportamiento funcional y de rendimiento antes y después del cambio. La migración no debe considerarse aceptada solo porque la restauración finalizó correctamente: la decisión técnica debe basarse en evidencia de integridad, seguridad, dependencias, funcionalidad, rendimiento y reversibilidad.

**Recursos recomendados**

- [Documentación de Microsoft sobre actualización de SQL Server](https://learn.microsoft.com/sql/database-engine/install-windows/upgrade-sql-server)
- [Documentación de BACKUP DATABASE](https://learn.microsoft.com/sql/t-sql/statements/backup-transact-sql)
- [Documentación de RESTORE VERIFYONLY](https://learn.microsoft.com/sql/t-sql/statements/restore-statements-verifyonly-transact-sql)
- [Documentación de DBCC CHECKDB](https://learn.microsoft.com/sql/t-sql/database-console-commands/dbcc-checkdb-transact-sql)
- [Documentación de Query Store](https://learn.microsoft.com/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store)
- [Documentación de Data Migration Assistant](https://learn.microsoft.com/sql/dma/dma-overview)
