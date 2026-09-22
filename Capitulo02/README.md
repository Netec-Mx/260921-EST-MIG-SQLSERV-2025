# Comparación: Comparar ejecución de queries, Antes vs optimizadas automáticamente y Uso de herramientas de diagnóstico en SSMS

## Metadatos

| Duration | Complexity | Bloom level |
|---|---|---|
| 70 minutos | Media | Aplicar |

## Descripción General

En esta práctica se capturará una línea base de rendimiento para procedimientos almacenados deliberadamente ineficientes en la base de datos `Sql2025Lab`. Se utilizarán `STATISTICS IO`, `STATISTICS TIME`, planes de ejecución reales y Query Store para identificar lecturas excesivas, uso de CPU, scans, conversiones implícitas, estimaciones incorrectas y regresiones de planes.

Después de aplicar índices, actualizar estadísticas, reescribir una consulta no sargable y corregir una regresión mediante Query Store, se compararán las métricas antes y después. La práctica demuestra que las capacidades inteligentes del motor ayudan al diagnóstico y a la corrección, pero no sustituyen el diseño adecuado de consultas, índices y estadísticas.

> **Nota sobre automatización:** la corrección automática de planes puede variar entre SQL Server local, Azure SQL Database y Azure SQL Managed Instance. El resultado obligatorio de esta práctica es aplicar y verificar una corrección controlada mediante **plan forcing** o **Query Store hints**, no asumir que Automatic Plan Correction actuará de forma autónoma.

## Objetivos de Aprendizaje

- [ ] Capturar duración, CPU, lecturas lógicas y planes de ejecución de consultas con rendimiento deficiente.
- [ ] Identificar consultas costosas y regresiones de plan mediante Query Store, `STATISTICS IO`, `STATISTICS TIME` y Actual Execution Plan.
- [ ] Mejorar consultas mediante índices no agrupados, actualización de estadísticas y reescritura sargable.
- [ ] Aplicar una corrección reproducible de regresión mediante Query Store plan forcing o Query Store hints.
- [ ] Documentar evidencia técnica antes y después en archivos de informe reutilizables.

## Prerrequisitos

**Conocimientos requeridos**

- Sintaxis básica de Transact-SQL: `SELECT`, `JOIN`, `WHERE`, `GROUP BY`, procedimientos almacenados e índices.
- Interpretación básica de operadores de planes de ejecución, especialmente `Table Scan`, `Index Scan`, `Index Seek`, `Key Lookup`, `Sort` y `Hash Match`.
- Conceptos de estadísticas, selectividad, parameter sniffing y consultas sargables.
- Uso básico de SQL Server Management Studio (SSMS).

**Acceso y estado requerido**

- La práctica 1 debe estar completada.
- Debe existir la base de datos `Sql2025Lab` con nivel de compatibilidad `170`, recovery model `FULL` y Query Store habilitado en modo `READ_WRITE`.
- Se requiere acceso con `.\SqlLabAdmin` o una cuenta con privilegios equivalentes a `sysadmin`.
- Debe poder conectarse a `LOCALHOST,1433`.
- La cuenta SQL `lab_sa` puede utilizarse solamente si está habilitada y su contraseña de laboratorio ha sido cambiada conforme a la política del curso.
- Debe haber espacio libre suficiente en `C:\SQLLab2025` para scripts e informes.

> **Seguridad:** no reutilice las contraseñas de laboratorio fuera de este entorno. No copie datos sensibles, resultados reales de producción ni cadenas de conexión a herramientas de IA.

## Entorno de Laboratorio

| Componente | Configuración de laboratorio |
|---|---|
| Servidor | `SQLLAB-WS2022` |
| Sistema operativo | Windows Server 2022 Datacenter |
| Instancia SQL Server | Instancia predeterminada `MSSQLSERVER` |
| Conexión | `LOCALHOST,1433` |
| Base de datos | `Sql2025Lab` |
| SQL Server | SQL Server 2025 Developer Edition, compilación de laboratorio `17.0.1000.7` |
| Herramienta principal | SSMS 21.3.2 |
| Compatibilidad | `170` |
| Recovery model | `FULL` |
| Query Store | `READ_WRITE` |
| Memoria máxima inicial | `16384 MB` |
| MAXDOP inicial | `4` |
| Cost threshold for parallelism | `50` |

Ejecute PowerShell **como administrador** para crear las rutas requeridas:

```powershell
New-Item -ItemType Directory -Force `
  -Path C:\SQLLab2025\Scripts,
        C:\SQLLab2025\Reports,
        C:\SQLLab2025\Data,
        C:\SQLLab2025\Logs,
        C:\SQLLab2025\Backups,
        C:\SQLLab2025\Audit,
        C:\SQLLab2025\Keys,
        C:\SQLLab2025\XEvents
```

Compruebe la versión, conectividad, compatibilidad y Query Store desde una ventana de consulta de SSMS conectada a `LOCALHOST,1433`:

```sql
SELECT
    @@SERVERNAME AS Servidor,
    SERVERPROPERTY('ProductVersion') AS VersionProducto,
    SERVERPROPERTY('ProductLevel') AS NivelProducto,
    SERVERPROPERTY('Edition') AS Edicion;

SELECT
    name,
    compatibility_level,
    recovery_model_desc,
    is_query_store_on
FROM sys.databases
WHERE name = N'Sql2025Lab';
```

Resultado esperado: una fila para `Sql2025Lab`, con `compatibility_level = 170`, `recovery_model_desc = FULL` e `is_query_store_on = 1`.

## Instrucciones Paso a Paso

### Paso 1: Verificar la configuración inicial del motor y Query Store

**Objetivo:** confirmar que el entorno está listo para medir consultas y conservar su historial de planes y métricas.

1. Abra SSMS con una cuenta administrativa y conecte a `LOCALHOST,1433`.
2. Cree una nueva ventana de consulta.
3. Ejecute el siguiente script en contexto de `master`:

```sql
USE master;
GO

SELECT
    name AS BaseDeDatos,
    compatibility_level AS NivelCompatibilidad,
    recovery_model_desc AS ModeloRecuperacion,
    is_query_store_on AS QueryStoreActivo
FROM sys.databases
WHERE name = N'Sql2025Lab';
GO

EXEC sys.sp_configure N'show advanced options', 1;
RECONFIGURE;
GO

SELECT
    name,
    value_in_use
FROM sys.configurations
WHERE name IN
(
    N'max server memory (MB)',
    N'max degree of parallelism',
    N'cost threshold for parallelism'
);
GO
```

4. Si Query Store no está activo o no está en modo lectura-escritura, ejecute:

```sql
ALTER DATABASE Sql2025Lab
SET QUERY_STORE = ON;
GO

ALTER DATABASE Sql2025Lab
SET QUERY_STORE
(
    OPERATION_MODE = READ_WRITE,
    QUERY_CAPTURE_MODE = ALL,
    CLEANUP_POLICY = (STALE_QUERY_THRESHOLD_DAYS = 30),
    DATA_FLUSH_INTERVAL_SECONDS = 60,
    INTERVAL_LENGTH_MINUTES = 1,
    MAX_STORAGE_SIZE_MB = 256
);
GO
```

5. Verifique el estado operativo de Query Store:

```sql
USE Sql2025Lab;
GO

SELECT
    actual_state_desc,
    desired_state_desc,
    current_storage_size_mb,
    max_storage_size_mb,
    query_capture_mode_desc
FROM sys.database_query_store_options;
GO
```

**Resultado esperado:** Query Store debe mostrar `actual_state_desc = READ_WRITE`. Los valores de instancia deben reflejar aproximadamente `16384 MB` para memoria máxima, `4` para MAXDOP y `50` para el umbral de paralelismo.

**Verificación:** documente en sus notas que la base permanece en compatibilidad `170`. No reduzca el nivel de compatibilidad para esta práctica: el objetivo es observar y validar el comportamiento del motor SQL Server 2025 en el nivel configurado.

---

### Paso 2: Crear la carga adicional, procedimientos ineficientes y registro de métricas

**Objetivo:** crear un conjunto controlado de datos y consultas que permita observar scans, conversiones implícitas, consultas no sargables y parameter sniffing.

1. En SSMS, abra una nueva consulta y seleccione la base de datos `Sql2025Lab`.
2. Guarde el archivo como:

```text
C:\SQLLab2025\Scripts\Lab02_SetupAndWorkload.sql
```

3. Ejecute el siguiente script completo. La carga es intencionalmente sintética y exclusiva del laboratorio.

```sql
USE Sql2025Lab;
GO

SET NOCOUNT ON;
GO

IF SCHEMA_ID(N'Reporting') IS NULL
    EXEC(N'CREATE SCHEMA Reporting AUTHORIZATION dbo;');
GO

DROP PROCEDURE IF EXISTS Reporting.usp_SalesByCustomerRange;
DROP PROCEDURE IF EXISTS Reporting.usp_OrderSearch;
DROP PROCEDURE IF EXISTS Reporting.usp_MonthlySalesSummary;
GO

DROP TABLE IF EXISTS dbo.LabExecutionLog;
DROP TABLE IF EXISTS dbo.LabSalesOrder;
DROP TABLE IF EXISTS dbo.LabCustomer;
GO

CREATE TABLE dbo.LabCustomer
(
    CustomerID      int           NOT NULL CONSTRAINT PK_LabCustomer PRIMARY KEY,
    CustomerCode    varchar(12)   NOT NULL,
    CustomerName    nvarchar(120) NOT NULL,
    RegionCode      char(2)       NOT NULL,
    CreatedDate     date          NOT NULL,
    CONSTRAINT UQ_LabCustomer_CustomerCode UNIQUE (CustomerCode)
);
GO

CREATE TABLE dbo.LabSalesOrder
(
    SalesOrderID    bigint         NOT NULL IDENTITY(1,1),
    CustomerID      int            NOT NULL,
    OrderDate       datetime2(0)   NOT NULL,
    OrderStatus     char(1)        NOT NULL,
    TotalAmount     decimal(12,2)  NOT NULL,
    Notes           nvarchar(200)  NULL,
    CONSTRAINT PK_LabSalesOrder PRIMARY KEY CLUSTERED (SalesOrderID),
    CONSTRAINT FK_LabSalesOrder_LabCustomer
        FOREIGN KEY (CustomerID) REFERENCES dbo.LabCustomer(CustomerID)
);
GO

CREATE TABLE dbo.LabExecutionLog
(
    ExecutionLogID      bigint IDENTITY(1,1) NOT NULL
        CONSTRAINT PK_LabExecutionLog PRIMARY KEY,
    Phase               varchar(20) NOT NULL,
    ProcedureName       sysname NOT NULL,
    CapturedAt          datetime2(0) NOT NULL
        CONSTRAINT DF_LabExecutionLog_CapturedAt DEFAULT SYSUTCDATETIME(),
    QueryID             bigint NULL,
    PlanID              bigint NULL,
    AvgDurationMs       decimal(18,2) NULL,
    AvgCpuMs            decimal(18,2) NULL,
    AvgLogicalReads     decimal(18,2) NULL,
    ExecutionCount      bigint NULL,
    Notes               nvarchar(1000) NULL
);
GO

;WITH N AS
(
    SELECT TOP (10000)
        ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
    FROM sys.all_objects AS a
    CROSS JOIN sys.all_objects AS b
)
INSERT dbo.LabCustomer
(
    CustomerID, CustomerCode, CustomerName, RegionCode, CreatedDate
)
SELECT
    n,
    CONCAT('C', RIGHT(CONCAT('00000000000', n), 11)),
    CONCAT(N'Cliente de laboratorio ', n),
    CHOOSE(((n - 1) % 5) + 1, 'NO', 'NE', 'SO', 'SE', 'CE'),
    DATEADD(day, -(n % 1825), CONVERT(date, '2025-01-01'))
FROM N;
GO

;WITH N AS
(
    SELECT TOP (150000)
        ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS n
    FROM sys.all_objects AS a
    CROSS JOIN sys.all_objects AS b
)
INSERT dbo.LabSalesOrder
(
    CustomerID, OrderDate, OrderStatus, TotalAmount, Notes
)
SELECT
    CASE
        WHEN n <= 50000 THEN ((n - 1) % 25) + 1
        ELSE ((n - 1) % 9975) + 26
    END,
    DATEADD(minute, -(n * 13 % 1051200), SYSUTCDATETIME()),
    CHOOSE(((n - 1) % 4) + 1, 'C', 'P', 'S', 'R'),
    CONVERT(decimal(12,2), 10 + ((n * 37) % 50000) / 10.0),
    CASE WHEN n % 20 = 0 THEN N'Pedido con revisión manual' ELSE N'Pedido estándar' END
FROM N;
GO

UPDATE STATISTICS dbo.LabCustomer WITH FULLSCAN;
UPDATE STATISTICS dbo.LabSalesOrder WITH FULLSCAN;
GO

CREATE OR ALTER PROCEDURE Reporting.usp_SalesByCustomerRange
    @CustomerIDFrom int,
    @CustomerIDTo   int
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        o.SalesOrderID,
        c.CustomerCode,
        c.CustomerName,
        o.OrderDate,
        o.OrderStatus,
        o.TotalAmount,
        o.Notes
    FROM dbo.LabSalesOrder AS o
    INNER JOIN dbo.LabCustomer AS c
        ON c.CustomerID = o.CustomerID
    WHERE o.CustomerID BETWEEN @CustomerIDFrom AND @CustomerIDTo
    ORDER BY o.OrderDate DESC;
END;
GO

CREATE OR ALTER PROCEDURE Reporting.usp_OrderSearch
    @OrderDate date,
    @CustomerCode nvarchar(12)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        o.SalesOrderID,
        o.OrderDate,
        o.OrderStatus,
        o.TotalAmount,
        c.CustomerCode,
        c.CustomerName
    FROM dbo.LabSalesOrder AS o
    INNER JOIN dbo.LabCustomer AS c
        ON c.CustomerID = o.CustomerID
    WHERE CONVERT(date, o.OrderDate) = @OrderDate
      AND c.CustomerCode = @CustomerCode;
END;
GO

CREATE OR ALTER PROCEDURE Reporting.usp_MonthlySalesSummary
    @Year int
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        DATENAME(month, o.OrderDate) AS MonthName,
        DATEPART(month, o.OrderDate) AS MonthNumber,
        SUM(o.TotalAmount) AS TotalSales,
        COUNT_BIG(*) AS OrderCount
    FROM dbo.LabSalesOrder AS o
    WHERE YEAR(o.OrderDate) = @Year
    GROUP BY
        DATENAME(month, o.OrderDate),
        DATEPART(month, o.OrderDate)
    ORDER BY MonthNumber;
END;
GO

ALTER DATABASE CURRENT SET QUERY_STORE CLEAR ALL;
GO
```

4. Espere a que finalice la carga. No interrumpa el proceso.
5. Compruebe que se cargaron las filas esperadas:

```sql
SELECT
    (SELECT COUNT(*) FROM dbo.LabCustomer) AS Clientes,
    (SELECT COUNT(*) FROM dbo.LabSalesOrder) AS Pedidos;
GO
```

**Resultado esperado:** se deben mostrar aproximadamente `10,000` clientes y `150,000` pedidos. En esta etapa no existen índices no agrupados sobre `LabSalesOrder`.

**Verificación:** ejecute el siguiente comando y confirme que inicialmente sólo existe el índice clustered de la clave primaria sobre pedidos:

```sql
EXEC sys.sp_helpindex N'dbo.LabSalesOrder';
GO
```

---

### Paso 3: Capturar la línea base con STATISTICS y planes reales

**Objetivo:** medir el comportamiento inicial y registrar evidencia de duración, CPU, lecturas lógicas y operadores costosos.

1. En SSMS, active **Include Actual Execution Plan** mediante:
   - Menú **Query** > **Include Actual Execution Plan**, o
   - Tecla `Ctrl+M`.
2. Active la pestaña **Messages** y ejecute el siguiente script:

```sql
USE Sql2025Lab;
GO

SET STATISTICS IO ON;
SET STATISTICS TIME ON;
GO

EXEC Reporting.usp_SalesByCustomerRange
    @CustomerIDFrom = 1,
    @CustomerIDTo = 25;
GO

EXEC Reporting.usp_OrderSearch
    @OrderDate = '2024-06-15',
    @CustomerCode = N'C00000000001';
GO

EXEC Reporting.usp_MonthlySalesSummary
    @Year = 2024;
GO

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
GO
```

3. Revise la pestaña **Messages**. Para cada procedimiento, anote:
   - `logical reads`;
   - `CPU time`;
   - `elapsed time`;
   - tablas o índices leídos.
4. Revise la pestaña **Execution Plan**. Identifique, como mínimo:
   - Un `Clustered Index Scan` o `Table Scan` sobre `LabSalesOrder`.
   - La advertencia de conversión implícita en la consulta de `usp_OrderSearch`, si aparece en la compilación.
   - Operadores `Sort`, `Hash Match` o lecturas de alto costo, si están presentes.
   - Diferencias entre filas estimadas y filas reales.
5. Consulte Query Store para identificar las sentencias de los procedimientos:

```sql
USE Sql2025Lab;
GO

SELECT
    q.query_id,
    p.plan_id,
    OBJECT_SCHEMA_NAME(q.object_id) AS Esquema,
    OBJECT_NAME(q.object_id) AS Objeto,
    rs.count_executions,
    CAST(rs.avg_duration / 1000.0 AS decimal(18,2)) AS DuracionPromedioMs,
    CAST(rs.avg_cpu_time / 1000.0 AS decimal(18,2)) AS CpuPromedioMs,
    CAST(rs.avg_logical_io_reads AS decimal(18,2)) AS LecturasLogicasPromedio,
    qt.query_sql_text
FROM sys.query_store_query AS q
INNER JOIN sys.query_store_query_text AS qt
    ON q.query_text_id = qt.query_text_id
INNER JOIN sys.query_store_plan AS p
    ON q.query_id = p.query_id
INNER JOIN sys.query_store_runtime_stats AS rs
    ON p.plan_id = rs.plan_id
WHERE q.object_id IN
(
    OBJECT_ID(N'Reporting.usp_SalesByCustomerRange'),
    OBJECT_ID(N'Reporting.usp_OrderSearch'),
    OBJECT_ID(N'Reporting.usp_MonthlySalesSummary')
)
ORDER BY DuracionPromedioMs DESC;
GO
```

6. Registre una captura inicial resumida en `dbo.LabExecutionLog`:

```sql
;WITH Metric AS
(
    SELECT
        q.query_id,
        p.plan_id,
        q.object_id,
        SUM(rs.count_executions) AS ExecutionCount,
        SUM(rs.avg_duration * rs.count_executions)
            / NULLIF(SUM(rs.count_executions), 0) / 1000.0 AS AvgDurationMs,
        SUM(rs.avg_cpu_time * rs.count_executions)
            / NULLIF(SUM(rs.count_executions), 0) / 1000.0 AS AvgCpuMs,
        SUM(rs.avg_logical_io_reads * rs.count_executions)
            / NULLIF(SUM(rs.count_executions), 0) AS AvgLogicalReads
    FROM sys.query_store_query AS q
    INNER JOIN sys.query_store_plan AS p
        ON q.query_id = p.query_id
    INNER JOIN sys.query_store_runtime_stats AS rs
        ON p.plan_id = rs.plan_id
    WHERE q.object_id IN
    (
        OBJECT_ID(N'Reporting.usp_SalesByCustomerRange'),
        OBJECT_ID(N'Reporting.usp_OrderSearch'),
        OBJECT_ID(N'Reporting.usp_MonthlySalesSummary')
    )
    GROUP BY q.query_id, p.plan_id, q.object_id
)
INSERT dbo.LabExecutionLog
(
    Phase, ProcedureName, QueryID, PlanID,
    AvgDurationMs, AvgCpuMs, AvgLogicalReads, ExecutionCount, Notes
)
SELECT
    'BEFORE',
    OBJECT_SCHEMA_NAME(object_id) + N'.' + OBJECT_NAME(object_id),
    query_id,
    plan_id,
    CAST(AvgDurationMs AS decimal(18,2)),
    CAST(AvgCpuMs AS decimal(18,2)),
    CAST(AvgLogicalReads AS decimal(18,2)),
    ExecutionCount,
    N'Línea base previa a índices, estadísticas y reescritura.'
FROM Metric;
GO
```

**Resultado esperado:** las consultas deben mostrar lecturas elevadas respecto a la selectividad de sus filtros. `usp_OrderSearch` puede mostrar un scan y una conversión implícita debido a la comparación entre `nvarchar(12)` y `varchar(12)`.

**Verificación:** confirme que existen filas con `Phase = 'BEFORE'`:

```sql
SELECT *
FROM dbo.LabExecutionLog
WHERE Phase = 'BEFORE'
ORDER BY ExecutionLogID;
GO
```

---

### Paso 4: Analizar consultas mediante los informes de Query Store en SSMS

**Objetivo:** utilizar la observabilidad histórica de Query Store para localizar consultas costosas y comparar planes.

1. En Object Explorer, expanda:

```text
Databases
  > Sql2025Lab
    > Query Store
      > Top Resource Consuming Queries
```

2. Abra **Top Resource Consuming Queries**.
3. Cambie el intervalo de tiempo para incluir las ejecuciones del laboratorio.
4. Ordene o filtre por:
   - duración promedio;
   - CPU promedio;
   - lecturas lógicas promedio, si el informe disponible lo expone;
   - número de ejecuciones.
5. Seleccione una consulta perteneciente a `Reporting.usp_OrderSearch`.
6. Abra el detalle de plan y compare:
   - filas estimadas frente a filas reales;
   - presencia de scan;
   - advertencias de conversión;
   - costo de los operadores.
7. Seleccione una consulta perteneciente a `Reporting.usp_SalesByCustomerRange`.
8. Documente la hipótesis inicial: la ausencia de un índice por `CustomerID` obliga a leer un volumen innecesario de pedidos para rangos selectivos.

**Resultado esperado:** Query Store debe conservar las consultas ejecutadas y al menos un plan por sentencia. Los valores concretos dependen del equipo, caché, carga del sistema y hora de ejecución.

**Verificación:** ejecute esta consulta para confirmar que Query Store contiene los tres objetos:

```sql
SELECT
    OBJECT_SCHEMA_NAME(object_id) AS Esquema,
    OBJECT_NAME(object_id) AS Procedimiento,
    COUNT(*) AS ConsultasCapturadas,
    COUNT(DISTINCT query_id) AS QueryIDs
FROM sys.query_store_query
WHERE object_id IN
(
    OBJECT_ID(N'Reporting.usp_SalesByCustomerRange'),
    OBJECT_ID(N'Reporting.usp_OrderSearch'),
    OBJECT_ID(N'Reporting.usp_MonthlySalesSummary')
)
GROUP BY object_id;
GO
```

---

### Paso 5: Aplicar índices, actualizar estadísticas y reescribir una consulta no sargable

**Objetivo:** realizar mejoras controladas basadas en la evidencia obtenida, no en recomendaciones genéricas.

1. Cree índices no agrupados alineados con los predicados y columnas recuperadas:

```sql
USE Sql2025Lab;
GO

CREATE INDEX IX_LabSalesOrder_CustomerID_OrderDate
ON dbo.LabSalesOrder (CustomerID, OrderDate DESC)
INCLUDE (OrderStatus, TotalAmount, Notes);
GO

CREATE INDEX IX_LabSalesOrder_OrderDate_CustomerID
ON dbo.LabSalesOrder (OrderDate, CustomerID)
INCLUDE (OrderStatus, TotalAmount);
GO
```

2. Actualice las estadísticas para que el optimizador disponga de información actualizada:

```sql
UPDATE STATISTICS dbo.LabCustomer WITH FULLSCAN;
UPDATE STATISTICS dbo.LabSalesOrder WITH FULLSCAN;
GO
```

3. Reescriba `usp_OrderSearch` para:
   - usar un tipo de dato coherente con `CustomerCode`;
   - eliminar `CONVERT(date, o.OrderDate)` del lado de la columna;
   - emplear un rango de fechas sargable.

```sql
CREATE OR ALTER PROCEDURE Reporting.usp_OrderSearch
    @OrderDate date,
    @CustomerCode varchar(12)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT
        o.SalesOrderID,
        o.OrderDate,
        o.OrderStatus,
        o.TotalAmount,
        c.CustomerCode,
        c.CustomerName
    FROM dbo.LabSalesOrder AS o
    INNER JOIN dbo.LabCustomer AS c
        ON c.CustomerID = o.CustomerID
    WHERE o.OrderDate >= @OrderDate
      AND o.OrderDate < DATEADD(day, 1, @OrderDate)
      AND c.CustomerCode = @CustomerCode;
END;
GO
```

4. Reescriba `usp_MonthlySalesSummary` para reemplazar el predicado no sargable `YEAR(OrderDate) = @Year` por un rango:

```sql
CREATE OR ALTER PROCEDURE Reporting.usp_MonthlySalesSummary
    @Year int
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @StartDate date = DATEFROMPARTS(@Year, 1, 1);
    DECLARE @EndDate   date = DATEFROMPARTS(@Year + 1, 1, 1);

    SELECT
        DATENAME(month, o.OrderDate) AS MonthName,
        DATEPART(month, o.OrderDate) AS MonthNumber,
        SUM(o.TotalAmount) AS TotalSales,
        COUNT_BIG(*) AS OrderCount
    FROM dbo.LabSalesOrder AS o
    WHERE o.OrderDate >= @StartDate
      AND o.OrderDate < @EndDate
    GROUP BY
        DATENAME(month, o.OrderDate),
        DATEPART(month, o.OrderDate)
    ORDER BY MonthNumber;
END;
GO
```

5. Revise los índices creados:

```sql
EXEC sys.sp_helpindex N'dbo.LabSalesOrder';
GO
```

**Resultado esperado:** existen dos índices no agrupados. Las consultas reescritas ya no aplican una función sobre `OrderDate` dentro del predicado de filtrado.

**Verificación:** consulte las estadísticas de los índices:

```sql
SELECT
    i.name AS Indice,
    s.name AS Estadistica,
    STATS_DATE(s.object_id, s.stats_id) AS FechaUltimaActualizacion
FROM sys.stats AS s
INNER JOIN sys.indexes AS i
    ON s.object_id = i.object_id
   AND s.stats_id = i.index_id
WHERE s.object_id = OBJECT_ID(N'dbo.LabSalesOrder');
GO
```

---

### Paso 6: Inducir y corregir una regresión de plan con Query Store

**Objetivo:** observar el efecto de parameter sniffing y aplicar una corrección controlada y verificable.

1. Ejecute primero un rango amplio para compilar un plan adecuado para un volumen alto de filas:

```sql
USE Sql2025Lab;
GO

EXEC sys.sp_recompile N'Reporting.usp_SalesByCustomerRange';
GO

EXEC Reporting.usp_SalesByCustomerRange
    @CustomerIDFrom = 1,
    @CustomerIDTo = 10000;
GO
```

2. Fuerce una recompilación del procedimiento y ejecute un rango muy selectivo:

```sql
EXEC sys.sp_recompile N'Reporting.usp_SalesByCustomerRange';
GO

EXEC Reporting.usp_SalesByCustomerRange
    @CustomerIDFrom = 1,
    @CustomerIDTo = 1;
GO
```

3. Sin recompilar de nuevo, ejecute otra vez el rango amplio. El plan compilado para el rango selectivo puede reutilizarse para un conjunto mucho mayor de filas:

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
GO

EXEC Reporting.usp_SalesByCustomerRange
    @CustomerIDFrom = 1,
    @CustomerIDTo = 10000;
GO

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
GO
```

4. Localice los planes históricos para la sentencia del procedimiento:

```sql
SELECT
    q.query_id,
    p.plan_id,
    p.is_forced_plan,
    p.force_failure_count,
    p.last_force_failure_reason_desc,
    rs.count_executions,
    CAST(rs.avg_duration / 1000.0 AS decimal(18,2)) AS AvgDurationMs,
    CAST(rs.avg_cpu_time / 1000.0 AS decimal(18,2)) AS AvgCpuMs,
    CAST(rs.avg_logical_io_reads AS decimal(18,2)) AS AvgLogicalReads,
    p.query_plan
FROM sys.query_store_query AS q
INNER JOIN sys.query_store_plan AS p
    ON q.query_id = p.query_id
INNER JOIN sys.query_store_runtime_stats AS rs
    ON p.plan_id = rs.plan_id
WHERE q.object_id = OBJECT_ID(N'Reporting.usp_SalesByCustomerRange')
ORDER BY AvgDurationMs DESC;
GO
```

5. En Query Store o en el resultado anterior, identifique:
   - el `query_id` de la sentencia principal;
   - un plan razonable para el rango amplio;
   - un plan regresado para el rango amplio reutilizado después de la compilación selectiva.

6. Aplique **una sola** de las siguientes alternativas.

   **Alternativa A: forzar el plan histórico estable**

```sql
-- Sustituya 123 y 456 por los valores observados en su instancia.
EXEC sys.sp_query_store_force_plan
    @query_id = 123,
    @plan_id = 456;
GO
```

   **Alternativa B: aplicar un Query Store hint**, si desea favorecer una compilación representativa:

```sql
-- Sustituya 123 por el query_id observado.
EXEC sys.sp_query_store_set_hints
    @query_id = 123,
    @query_hints = N'OPTION (OPTIMIZE FOR (@CustomerIDFrom = 1, @CustomerIDTo = 10000))';
GO
```

7. Ejecute de nuevo el rango amplio y capture plan real, lecturas y tiempos:

```sql
SET STATISTICS IO ON;
SET STATISTICS TIME ON;
GO

EXEC Reporting.usp_SalesByCustomerRange
    @CustomerIDFrom = 1,
    @CustomerIDTo = 10000;
GO

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
GO
```

8. Compruebe el estado de la corrección:

```sql
SELECT
    q.query_id,
    p.plan_id,
    p.is_forced_plan,
    p.force_failure_count,
    p.last_force_failure_reason_desc
FROM sys.query_store_query AS q
INNER JOIN sys.query_store_plan AS p
    ON q.query_id = p.query_id
WHERE q.object_id = OBJECT_ID(N'Reporting.usp_SalesByCustomerRange')
ORDER BY p.is_forced_plan DESC, p.plan_id;
GO
```

**Resultado esperado:** Query Store debe mostrar múltiples planes para la sentencia o, como mínimo, evidencia de recompilación y plan seleccionado. Si se usa plan forcing, `is_forced_plan` debe ser `1` para el plan elegido. Si se usa Query Store hint, la consulta debe mostrar el comportamiento asociado al hint aplicado.

**Verificación:** compare los mensajes de `STATISTICS IO` y `STATISTICS TIME` de la ejecución amplia antes y después de la corrección. La evidencia válida es una mejora reproducible o una estabilización del plan para el patrón de carga seleccionado.

> **Importante:** no fuerce planes sin revisar su aplicabilidad. Un plan óptimo para una carga amplia puede ser ineficiente para una búsqueda selectiva. Plan forcing y Query Store hints son mecanismos de estabilización controlada, no sustitutos permanentes de una revisión del diseño y del patrón de parámetros.

---

### Paso 7: Ejecutar la comparación posterior y generar los informes requeridos

**Objetivo:** capturar las métricas posteriores y producir los dos entregables técnicos que se usarán en la práctica de IA.

1. Active **Include Actual Execution Plan** si aún no está activo.
2. Ejecute nuevamente la carga de prueba:

```sql
USE Sql2025Lab;
GO

SET STATISTICS IO ON;
SET STATISTICS TIME ON;
GO

EXEC Reporting.usp_SalesByCustomerRange
    @CustomerIDFrom = 1,
    @CustomerIDTo = 25;
GO

EXEC Reporting.usp_OrderSearch
    @OrderDate = '2024-06-15',
    @CustomerCode = 'C00000000001';
GO

EXEC Reporting.usp_MonthlySalesSummary
    @Year = 2024;
GO

SET STATISTICS TIME OFF;
SET STATISTICS IO OFF;
GO
```

3. Registre las métricas posteriores desde Query Store:

```sql
;WITH Metric AS
(
    SELECT
        q.query_id,
        p.plan_id,
        q.object_id,
        SUM(rs.count_executions) AS ExecutionCount,
        SUM(rs.avg_duration * rs.count_executions)
            / NULLIF(SUM(rs.count_executions), 0) / 1000.0 AS AvgDurationMs,
        SUM(rs.avg_cpu_time * rs.count_executions)
            / NULLIF(SUM(rs.count_executions), 0) / 1000.0 AS AvgCpuMs,
        SUM(rs.avg_logical_io_reads * rs.count_executions)
            / NULLIF(SUM(rs.count_executions), 0) AS AvgLogicalReads
    FROM sys.query_store_query AS q
    INNER JOIN sys.query_store_plan AS p
        ON q.query_id = p.query_id
    INNER JOIN sys.query_store_runtime_stats AS rs
        ON p.plan_id = rs.plan_id
    WHERE q.object_id IN
    (
        OBJECT_ID(N'Reporting.usp_SalesByCustomerRange'),
        OBJECT_ID(N'Reporting.usp_OrderSearch'),
        OBJECT_ID(N'Reporting.usp_MonthlySalesSummary')
    )
    GROUP BY q.query_id, p.plan_id, q.object_id
)
INSERT dbo.LabExecutionLog
(
    Phase, ProcedureName, QueryID, PlanID,
    AvgDurationMs, AvgCpuMs, AvgLogicalReads, ExecutionCount, Notes
)
SELECT
    'AFTER',
    OBJECT_SCHEMA_NAME(object_id) + N'.' + OBJECT_NAME(object_id),
    query_id,
    plan_id,
    CAST(AvgDurationMs AS decimal(18,2)),
    CAST(AvgCpuMs AS decimal(18,2)),
    CAST(AvgLogicalReads AS decimal(18,2)),
    ExecutionCount,
    N'Métricas posteriores a índices, estadísticas, reescritura y corrección controlada.'
FROM Metric;
GO
```

4. Cree un nuevo archivo en SSMS con el siguiente nombre exacto:

```text
C:\SQLLab2025\Reports\Lab02_QueryComparisonBeforeAfter.sql
```

5. Guarde en ese archivo la siguiente consulta de comparación:

```sql
USE Sql2025Lab;
GO

SELECT
    b.ProcedureName,
    b.QueryID,
    b.PlanID AS PlanBefore,
    a.PlanID AS PlanAfter,
    b.AvgDurationMs AS DurationBeforeMs,
    a.AvgDurationMs AS DurationAfterMs,
    b.AvgCpuMs AS CpuBeforeMs,
    a.AvgCpuMs AS CpuAfterMs,
    b.AvgLogicalReads AS ReadsBefore,
    a.AvgLogicalReads AS ReadsAfter,
    CAST
    (
        100.0 * (b.AvgDurationMs - a.AvgDurationMs)
        / NULLIF(b.AvgDurationMs, 0)
        AS decimal(10,2)
    ) AS DurationImprovementPercent
FROM dbo.LabExecutionLog AS b
INNER JOIN dbo.LabExecutionLog AS a
    ON a.ProcedureName = b.ProcedureName
WHERE b.Phase = 'BEFORE'
  AND a.Phase = 'AFTER'
ORDER BY b.ProcedureName, DurationImprovementPercent DESC;
GO

SELECT
    Phase,
    ProcedureName,
    QueryID,
    PlanID,
    AvgDurationMs,
    AvgCpuMs,
    AvgLogicalReads,
    ExecutionCount,
    CapturedAt,
    Notes
FROM dbo.LabExecutionLog
ORDER BY ProcedureName, Phase, CapturedAt;
GO
```

6. Ejecute el archivo y guarde también una copia de los resultados en formato `.csv` o `.txt` si su instructor lo solicita.
7. Cree el archivo Markdown:

```text
C:\SQLLab2025\Reports\Lab02_PerformanceFindings.md
```

8. Incluya, como mínimo, el siguiente contenido. Sustituya los valores entre corchetes por los datos reales de su ejecución:

```markdown
### Hallazgos de rendimiento — Lab 02

- Fecha de prueba: [fecha y hora].
- Servidor e instancia: SQLLAB-WS2022 / MSSQLSERVER.
- Base de datos y compatibilidad: Sql2025Lab / 170.
- Query Store: READ_WRITE.

#### Línea base

| Procedimiento | Duración | CPU | Lecturas lógicas | Observación |
|---|---:|---:|---:|---|
| usp_SalesByCustomerRange | [valor] | [valor] | [valor] | [scan, lookup o estimación] |
| usp_OrderSearch | [valor] | [valor] | [valor] | [función no sargable o conversión] |
| usp_MonthlySalesSummary | [valor] | [valor] | [valor] | [filtro YEAR no sargable] |

#### Cambios aplicados

1. Se creó `IX_LabSalesOrder_CustomerID_OrderDate`.
2. Se creó `IX_LabSalesOrder_OrderDate_CustomerID`.
3. Se actualizaron estadísticas con `FULLSCAN`.
4. Se reescribió `usp_OrderSearch` para usar un rango de fechas y tipos compatibles.
5. Se reescribió `usp_MonthlySalesSummary` para filtrar mediante rango de fechas.
6. Se aplicó [plan forcing / Query Store hint] al query_id [valor], si correspondió.

#### Resultado posterior

| Procedimiento | Duración antes | Duración después | Lecturas antes | Lecturas después | Conclusión |
|---|---:|---:|---:|---:|---|
| usp_SalesByCustomerRange | [valor] | [valor] | [valor] | [valor] | [conclusión] |
| usp_OrderSearch | [valor] | [valor] | [valor] | [valor] | [conclusión] |
| usp_MonthlySalesSummary | [valor] | [valor] | [valor] | [valor] | [conclusión] |

#### Decisión operativa

La corrección aplicada se considera [aceptada / pendiente de revisión] porque [evidencia].
No se asume que Automatic Plan Correction esté disponible o habilitado en SQL Server local.
```

**Resultado esperado:** deben existir los archivos `Lab02_QueryComparisonBeforeAfter.sql` y `Lab02_PerformanceFindings.md` en `C:\SQLLab2025\Reports`.

**Verificación:** ejecute PowerShell:

```powershell
Get-Item `
  C:\SQLLab2025\Reports\Lab02_QueryComparisonBeforeAfter.sql,
  C:\SQLLab2025\Reports\Lab02_PerformanceFindings.md |
Select-Object FullName, Length, LastWriteTime
```

## Validación y Pruebas

Complete las siguientes validaciones antes de marcar la práctica como finalizada.

1. **Validación de Query Store**

```sql
USE Sql2025Lab;
GO

SELECT
    actual_state_desc,
    query_capture_mode_desc,
    current_storage_size_mb,
    max_storage_size_mb
FROM sys.database_query_store_options;
GO
```

Criterio de aceptación: `actual_state_desc` debe ser `READ_WRITE`.

2. **Validación de índices**

```sql
SELECT
    i.name,
    i.type_desc,
    i.is_disabled,
    i.fill_factor
FROM sys.indexes AS i
WHERE i.object_id = OBJECT_ID(N'dbo.LabSalesOrder')
ORDER BY i.index_id;
GO
```

Criterio de aceptación: deben existir `IX_LabSalesOrder_CustomerID_OrderDate` e `IX_LabSalesOrder_OrderDate_CustomerID`, ambos habilitados.

3. **Validación de reescritura sargable**

```sql
SELECT
    OBJECT_SCHEMA_NAME(object_id) AS Esquema,
    OBJECT_NAME(object_id) AS Objeto,
    definition
FROM sys.sql_modules
WHERE object_id IN
(
    OBJECT_ID(N'Reporting.usp_OrderSearch'),
    OBJECT_ID(N'Reporting.usp_MonthlySalesSummary')
);
GO
```

Criterio de aceptación: `usp_OrderSearch` debe usar `OrderDate >= @OrderDate` y `OrderDate < DATEADD(day, 1, @OrderDate)`. `usp_MonthlySalesSummary` debe usar límites de fecha, no `YEAR(o.OrderDate)` en el filtro.

4. **Validación de evidencia antes y después**

```sql
SELECT
    ProcedureName,
    Phase,
    COUNT(*) AS Registros
FROM dbo.LabExecutionLog
GROUP BY ProcedureName, Phase
ORDER BY ProcedureName, Phase;
GO
```

Criterio de aceptación: debe existir al menos un registro `BEFORE` y uno `AFTER` para los procedimientos analizados.

5. **Validación del plan forzado o hint, si se aplicó**

```sql
SELECT
    q.query_id,
    p.plan_id,
    p.is_forced_plan,
    p.force_failure_count,
    p.last_force_failure_reason_desc
FROM sys.query_store_query AS q
INNER JOIN sys.query_store_plan AS p
    ON p.query_id = q.query_id
WHERE q.object_id = OBJECT_ID(N'Reporting.usp_SalesByCustomerRange');
GO
```

Criterio de aceptación: si se utilizó plan forcing, el plan seleccionado debe mostrar `is_forced_plan = 1` y no debe presentar errores de forzado. Si se utilizó Query Store hint, documente el `query_id`, el hint exacto y su efecto medido.

6. **Interpretación final**

La práctica se considera aprobada cuando el estudiante puede explicar, con datos observables:

- qué consulta presentaba un problema de sargabilidad;
- qué consulta tenía una conversión implícita potencialmente perjudicial;
- qué índice se creó y qué predicado respalda;
- cuál fue la evidencia de una regresión o inestabilidad de plan;
- por qué se eligió plan forcing o Query Store hint;
- por qué una mejora del motor no elimina la necesidad de medir, diseñar y validar.

## Solución de Problemas

### Problema 1: Query Store aparece como READ_ONLY o no captura las consultas

**Síntomas:** `sys.database_query_store_options` muestra `READ_ONLY`, el informe de Query Store no contiene las consultas recientes o la tabla de Query Store no se actualiza.

**Causa:** Query Store alcanzó su límite de almacenamiento, fue configurado con captura restrictiva o la base no está en modo `READ_WRITE`.

**Corrección:** revise el espacio usado y aumente el límite sólo para el laboratorio. Después establezca el modo de operación y captura adecuados:

```sql
ALTER DATABASE Sql2025Lab
SET QUERY_STORE
(
    OPERATION_MODE = READ_WRITE,
    QUERY_CAPTURE_MODE = ALL,
    MAX_STORAGE_SIZE_MB = 512
);
GO
```

Espere al menos un intervalo de recolección o ejecute nuevamente las consultas. Verifique el estado con `sys.database_query_store_options`.

### Problema 2: No se observan dos planes o no se aprecia una regresión de parameter sniffing

**Síntomas:** `usp_SalesByCustomerRange` muestra un único plan en Query Store, los tiempos son similares o no resulta evidente qué plan debe corregirse.

**Causa:** el optimizador puede elegir el mismo plan para ambos parámetros debido al hardware, distribución de datos, caché, estadísticas o costo estimado. La regresión inducida es didáctica y no está garantizada con la misma intensidad en todos los equipos.

**Corrección:** ejecute de nuevo la secuencia de compilación con `sp_recompile`, primero para el rango amplio y después para el rango selectivo. Confirme que está activado Actual Execution Plan. Si continúan existiendo pocos cambios, no fuerce un plan arbitrariamente: documente que no se reprodujo una regresión significativa y aplique la comparación de planes únicamente como ejercicio de observabilidad. Mantenga las mejoras verificables de índices, estadísticas y sargabilidad.

## Limpieza

> Realice esta sección únicamente cuando el instructor confirme que ya no necesita los objetos ni los informes para la siguiente práctica. Los archivos de informe son entrada técnica para la práctica de IA y deben conservarse.

1. Si aplicó plan forcing, elimínelo antes de borrar objetos:

```sql
-- Sustituya 123 por el query_id utilizado.
EXEC sys.sp_query_store_unforce_plan
    @query_id = 123,
    @plan_id = 456;
GO
```

2. Si aplicó un Query Store hint, elimínelo:

```sql
-- Sustituya 123 por el query_id utilizado.
EXEC sys.sp_query_store_clear_hints
    @query_id = 123;
GO
```

3. Para eliminar solamente la carga creada por esta práctica, ejecute:

```sql
USE Sql2025Lab;
GO

DROP PROCEDURE IF EXISTS Reporting.usp_SalesByCustomerRange;
DROP PROCEDURE IF EXISTS Reporting.usp_OrderSearch;
DROP PROCEDURE IF EXISTS Reporting.usp_MonthlySalesSummary;
GO

DROP TABLE IF EXISTS dbo.LabExecutionLog;
DROP TABLE IF EXISTS dbo.LabSalesOrder;
DROP TABLE IF EXISTS dbo.LabCustomer;
GO
```

4. No elimine estos archivos si continuará con la práctica de IA:

```text
C:\SQLLab2025\Reports\Lab02_QueryComparisonBeforeAfter.sql
C:\SQLLab2025\Reports\Lab02_PerformanceFindings.md
```

## Resumen

En esta práctica se aplicó un proceso de diagnóstico basado en evidencia para consultas SQL Server: captura de métricas, análisis de planes reales, consulta de historial mediante Query Store y validación posterior a los cambios.

Se comprobó que las mejoras del motor y las capacidades de observabilidad de SQL Server 2025 facilitan la detección y estabilización de problemas, pero no reemplazan prácticas fundamentales: índices adecuados, estadísticas actualizadas, predicados sargables, tipos de datos compatibles y validación controlada de cambios. Los archivos generados servirán como fuente técnica para analizar y documentar resultados con asistencia de IA en un entorno controlado.

**Recursos opcionales**

- [Supervisar el rendimiento mediante Query Store](https://learn.microsoft.com/es-es/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store)
- [Forzar planes mediante Query Store](https://learn.microsoft.com/es-es/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store)
- [Estadísticas para la optimización de consultas](https://learn.microsoft.com/es-es/sql/relational-databases/statistics/statistics)
- [Guía de arquitectura y diseño de índices](https://learn.microsoft.com/es-es/sql/relational-databases/sql-server-index-design-guide)
