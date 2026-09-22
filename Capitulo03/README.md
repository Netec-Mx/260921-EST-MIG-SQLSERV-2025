# IA Generar queries con Copilot, Explicar un query complejo, Optimizar consultas usando IA y Documentación automática de objetos

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 95 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Crear |

## Descripción General

En esta práctica se utiliza GitHub Copilot Chat como asistente controlado para generar borradores T-SQL, explicar una consulta compleja, interpretar señales de rendimiento y producir documentación inicial de objetos de SQL Server. Las respuestas de IA no se implementan automáticamente: cada propuesta se clasifica como aceptada, modificada o rechazada según evidencia obtenida con Query Store, planes de ejecución, métricas y revisión humana.

El laboratorio utiliza la base de datos `Sql2025Lab`, los resultados de la práctica 2 y una conexión local desde Visual Studio Code. Al finalizar, se crea y valida la tabla `dbo.ObjectDocumentation`, se documentan al menos cuatro objetos y se genera el informe `C:\SQLLab2025\Reports\Lab03_AIReview.md`.

## Objetivos de Aprendizaje

- [ ] Generar y validar un borrador de consulta T-SQL de ventas por cliente y período mediante GitHub Copilot Chat.
- [ ] Explicar técnicamente `Reporting.usp_OrderSearch` y un fragmento de su plan de ejecución sin aceptar conclusiones de IA sin evidencia.
- [ ] Comparar una propuesta de optimización para filtros no sargables con métricas de Query Store, SSMS y los resultados de la práctica 2.
- [ ] Clasificar recomendaciones de IA como **aceptadas**, **modificadas** o **rechazadas**, justificando cada decisión.
- [ ] Crear documentación revisada de objetos de `Sql2025Lab` y exportar un informe de revisión técnica.

## Prerrequisitos

Conocimientos necesarios:

- Haber completado las prácticas 1 y 2.
- Comprender consultas `SELECT`, procedimientos almacenados, índices, estadísticas y planes de ejecución.
- Conocer el propósito de Query Store y la diferencia entre métricas observadas y recomendaciones.
- Saber ejecutar scripts en SQL Server Management Studio (SSMS) o Visual Studio Code con la extensión `mssql`.
- Comprender que GitHub Copilot es un asistente de productividad y no un sustituto de la validación técnica.

Acceso necesario:

- Cuenta local `.\SqlLabAdmin` con privilegios administrativos de laboratorio.
- Acceso autenticado a GitHub Copilot y GitHub Copilot Chat desde Visual Studio Code.
- Conectividad HTTPS saliente mediante el puerto TCP 443.
- Acceso local a SQL Server en `LOCALHOST,1433`.
- Base de datos `Sql2025Lab` disponible con nivel de compatibilidad 170, recovery model `FULL` y Query Store en modo `READ_WRITE`.
- Archivos de resultados de la práctica 2, si están disponibles:

```text
C:\SQLLab2025\Scripts\Lab02\QueryComparisonBeforeAfter.sql
C:\SQLLab2025\Reports\Lab02_PerformanceFindings.md
```

> **Regla de seguridad:** no envíe a Copilot credenciales, contraseñas, resultados reales de negocio, direcciones IP productivas, copias de archivos de auditoría, datos personales, tokens, claves, certificados ni planes XML completos que puedan contener literales sensibles.

## Entorno de Laboratorio

| Componente | Configuración de laboratorio |
|---|---|
| Servidor | `SQLLAB-WS2022` |
| Sistema operativo | Windows Server 2022 Datacenter 21H2 |
| Instancia SQL Server | Instancia predeterminada `MSSQLSERVER` |
| Punto de conexión | `LOCALHOST,1433` |
| Base de datos | `Sql2025Lab` |
| Compatibilidad | 170 |
| Recovery model | `FULL` |
| Query Store | `READ_WRITE` |
| SQL Server destino | SQL Server 2025 Developer Edition 17.0.1000.7 |
| Herramienta SQL | SSMS 21.3.2 o Visual Studio Code 1.96.4 |
| Extensión SQL para VS Code | `mssql` 1.28.0 |
| Asistente IA | GitHub Copilot 1.250.0 y Copilot Chat 0.23.2 |
| Memoria máxima inicial de SQL Server | 16384 MB |
| MAXDOP inicial | 4 |
| Cost threshold for parallelism | 50 |

Las rutas de trabajo requeridas son:

```text
C:\SQLLab2025\Scripts\Lab03
C:\SQLLab2025\Reports
```

Abra PowerShell como `.\SqlLabAdmin` y cree las carpetas de la práctica:

```powershell
New-Item -ItemType Directory -Force -Path `
    'C:\SQLLab2025\Scripts\Lab03', `
    'C:\SQLLab2025\Reports' | Out-Null
```

Ejecute la siguiente comprobación desde SSMS o desde una consulta de Visual Studio Code conectada a `Sql2025Lab`:

```sql
USE master;
GO

SELECT
    @@SERVERNAME AS ServerName,
    SERVERPROPERTY('ProductVersion') AS ProductVersion,
    SERVERPROPERTY('ProductLevel') AS ProductLevel,
    SERVERPROPERTY('Edition') AS Edition;
GO

SELECT
    name,
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
    max_storage_size_mb
FROM sys.database_query_store_options;
GO
```

Resultado esperado:

- SQL Server responde desde la instancia local.
- `Sql2025Lab` se encuentra en estado `ONLINE`.
- El nivel de compatibilidad es `170`.
- El modelo de recuperación es `FULL`.
- Query Store muestra `READ_WRITE` en `actual_state_desc`.

## Instrucciones Paso a Paso

### Paso 1: Preparar el espacio de trabajo y comprobar los artefactos de la práctica 2

**Objetivo:** confirmar que el entorno está preparado, localizar los resultados de rendimiento anteriores y crear el registro de decisiones de IA.

**Instrucciones:**

1. Abra Visual Studio Code con la cuenta `.\SqlLabAdmin`.

2. Seleccione **File > Open Folder** y abra:

   ```text
   C:\SQLLab2025\Scripts\Lab03
   ```

3. Cree los siguientes archivos vacíos en la carpeta abierta:

   ```text
   01_EnvironmentCheck.sql
   02_CopilotGeneratedSalesQuery.sql
   03_OrderSearchAnalysis.sql
   04_NonSargableFilterReview.sql
   05_ObjectDocumentation.sql
   ```

4. Cree el archivo de informe:

   ```text
   C:\SQLLab2025\Reports\Lab03_AIReview.md
   ```

5. En PowerShell, compruebe si los artefactos de la práctica 2 están disponibles:

   ```powershell
   Test-Path 'C:\SQLLab2025\Scripts\Lab02\QueryComparisonBeforeAfter.sql'
   Test-Path 'C:\SQLLab2025\Reports\Lab02_PerformanceFindings.md'
   ```

6. En `01_EnvironmentCheck.sql`, guarde y ejecute el siguiente script:

   ```sql
   USE Sql2025Lab;
   GO

   SELECT
       DB_NAME() AS CurrentDatabase,
       SUSER_SNAME() AS LoginName,
       USER_NAME() AS DatabaseUser,
       DATABASEPROPERTYEX(DB_NAME(), 'Status') AS DatabaseStatus;
   GO

   SELECT
       s.name AS SchemaName,
       o.name AS ObjectName,
       o.type_desc AS ObjectType
   FROM sys.objects AS o
   INNER JOIN sys.schemas AS s
       ON s.schema_id = o.schema_id
   WHERE (s.name = N'Reporting' AND o.name = N'usp_OrderSearch')
      OR (s.name = N'Sales' AND o.name IN (N'Orders', N'OrderDetails'))
   ORDER BY s.name, o.name;
   GO
   ```

7. Abra `Lab02_PerformanceFindings.md`, si existe, y anote las consultas, operadores, índices, lecturas lógicas o hallazgos relevantes que se hayan identificado en la práctica anterior.

8. Cree la siguiente tabla inicial en `Lab03_AIReview.md`:

   ```markdown
   | ID | Actividad | Recomendación o salida de IA | Decisión | Evidencia de validación |
   |---|---|---|---|---|
   | IA-01 | Consulta de ventas | Pendiente | Pendiente | Pendiente |
   | IA-02 | Explicación de procedimiento | Pendiente | Pendiente | Pendiente |
   | IA-03 | Filtro no sargable | Pendiente | Pendiente | Pendiente |
   | IA-04 | Documentación de objetos | Pendiente | Pendiente | Pendiente |
   ```

**Resultado esperado:**

- Existe la carpeta `C:\SQLLab2025\Scripts\Lab03`.
- Los cinco archivos SQL y el informe Markdown han sido creados.
- Se confirma la disponibilidad de `Reporting.usp_OrderSearch`, `Sales.Orders` y `Sales.OrderDetails`.
- Los resultados de la práctica 2 están localizados o se ha documentado su ausencia.

**Verificación:**

Ejecute la siguiente consulta. Debe devolver al menos tres objetos del laboratorio, incluyendo el procedimiento si las prácticas anteriores se completaron correctamente:

```sql
USE Sql2025Lab;
GO

SELECT
    QUOTENAME(s.name) + N'.' + QUOTENAME(o.name) AS ObjectName,
    o.type_desc
FROM sys.objects AS o
INNER JOIN sys.schemas AS s
    ON s.schema_id = o.schema_id
WHERE (s.name = N'Reporting' AND o.name = N'usp_OrderSearch')
   OR (s.name = N'Sales' AND o.name IN (N'Orders', N'OrderDetails'));
GO
```

---

### Paso 2: Configurar Visual Studio Code, la conexión segura y el uso controlado de Copilot

**Objetivo:** conectarse a SQL Server sin guardar secretos en el repositorio o en archivos compartidos, y establecer reglas de uso seguro para Copilot Chat.

**Instrucciones:**

1. En Visual Studio Code, abra la paleta de comandos con `Ctrl+Shift+P`.

2. Ejecute el comando:

   ```text
   MS SQL: Connect
   ```

3. Configure una conexión con los siguientes valores:

   | Propiedad | Valor |
   |---|---|
   | Server name | `LOCALHOST,1433` |
   | Authentication type | Windows Authentication |
   | Database | `Sql2025Lab` |
   | Trust server certificate | Solo si el laboratorio lo requiere |
   | Save password | No aplicable para autenticación Windows |

4. Si se solicita un perfil de conexión, use un nombre no sensible, por ejemplo:

   ```text
   SQL2025Lab-Local-WindowsAuth
   ```

5. No agregue la contraseña de `lab_sa` ni la contraseña de cuentas de prueba a archivos `.sql`, `.md`, configuraciones de VS Code ni repositorios.

6. Abra GitHub Copilot Chat. Antes de enviar un prompt, aplique estas reglas:

   - Use nombres de objetos del laboratorio, no nombres de producción.
   - Envíe únicamente estructura, requisitos funcionales y fragmentos técnicos mínimos.
   - No copie filas de resultados, archivos de auditoría, datos de clientes ni credenciales.
   - Solicite siempre que la respuesta indique supuestos y riesgos.
   - Trate toda respuesta como un borrador que debe validarse.

7. Agregue esta declaración al inicio de `Lab03_AIReview.md`:

   ```markdown
   ## Controles de uso de IA

   - La IA se utilizó únicamente para generar borradores, explicaciones e hipótesis técnicas.
   - No se compartieron credenciales, datos personales, resultados reales de negocio, secretos, archivos de auditoría ni direcciones IP productivas.
   - Toda recomendación fue validada mediante SQL Server, Query Store, planes de ejecución, métricas o revisión de metadatos.
   - La decisión final fue tomada por el estudiante y registrada como aceptada, modificada o rechazada.
   ```

8. Ejecute esta comprobación de contexto en una ventana SQL:

   ```sql
   USE Sql2025Lab;
   GO

   SELECT
       ORIGINAL_LOGIN() AS OriginalLogin,
       SUSER_SNAME() AS CurrentLogin,
       USER_NAME() AS CurrentDatabaseUser,
       HOST_NAME() AS HostName,
       APP_NAME() AS ApplicationName;
   GO
   ```

**Resultado esperado:**

- Visual Studio Code muestra una conexión activa a `LOCALHOST,1433`.
- La base de datos activa es `Sql2025Lab`.
- No se han almacenado secretos en la carpeta de scripts.
- El informe incluye los controles de uso de IA.

**Verificación:**

Revise la barra de estado de Visual Studio Code. Debe indicar una conexión activa a SQL Server y la base de datos `Sql2025Lab`. Confirme además que ningún archivo dentro de `C:\SQLLab2025\Scripts\Lab03` contiene las cadenas:

```text
SqlLab2025!ChangeMe
LabAccess2025!ChangeMe
```

Puede comprobarlo con PowerShell:

```powershell
Get-ChildItem 'C:\SQLLab2025\Scripts\Lab03' -Recurse -File |
    Select-String -Pattern 'SqlLab2025!ChangeMe|LabAccess2025!ChangeMe'
```

El resultado debe estar vacío.

---

### Paso 3: Generar y validar una consulta de ventas por cliente y período

**Objetivo:** usar Copilot Chat para producir un borrador de consulta parametrizada, comprobar su sintaxis, validar su semántica y medir su comportamiento básico.

**Instrucciones:**

1. Antes de solicitar código a Copilot, obtenga las columnas disponibles en las tablas del laboratorio:

   ```sql
   USE Sql2025Lab;
   GO

   SELECT
       SCHEMA_NAME(t.schema_id) AS SchemaName,
       t.name AS TableName,
       c.column_id,
       c.name AS ColumnName,
       TYPE_NAME(c.user_type_id) AS DataType,
       c.max_length,
       c.is_nullable
   FROM sys.tables AS t
   INNER JOIN sys.columns AS c
       ON c.object_id = t.object_id
   WHERE SCHEMA_NAME(t.schema_id) = N'Sales'
     AND t.name IN (N'Orders', N'OrderDetails')
   ORDER BY t.name, c.column_id;
   GO
   ```

2. En Copilot Chat, use el siguiente prompt estandarizado. Adapte únicamente los nombres de columnas si el resultado del paso anterior demuestra una variación en el modelo:

   ```text
   Actúa como asistente de T-SQL para un entorno de laboratorio SQL Server 2025.
   Genera un borrador de consulta parametrizada de solo lectura para obtener ventas
   por cliente y por período. Usa Sales.Orders y Sales.OrderDetails, relacionándolas
   por OrderID. El período debe usar un límite inicial inclusivo y un límite final
   exclusivo para evitar problemas de hora. Devuelve CustomerID, cantidad de pedidos,
   unidades vendidas y total de ventas calculado como UnitPrice * Quantity.
   No uses SELECT *, SQL dinámico, sugerencias de índice ni cambios DDL.
   Incluye supuestos sobre nombres de columnas y explica cómo validar la consulta.
   No recibiste datos reales ni resultados de negocio.
   ```

3. Revise la respuesta. La propuesta debe cumplir, como mínimo, estos criterios:

   - Declarar parámetros de fecha tipados.
   - Usar `INNER JOIN` por `OrderID`.
   - Evitar `SELECT *`.
   - Usar fechas en formato ISO o parámetros `date`.
   - Aplicar un rango sargable:

     ```sql
     o.OrderDate >= @StartDate
     AND o.OrderDate < @EndDate
     ```

   - Agrupar correctamente por cliente.
   - No incluir cambios de esquema, sugerencias `NOLOCK` ni datos ficticios como si fueran datos reales.

4. Si la respuesta no cumple algún criterio, modifíquela manualmente. Guarde el resultado final en:

   ```text
   C:\SQLLab2025\Scripts\Lab03\02_CopilotGeneratedSalesQuery.sql
   ```

5. Use el siguiente patrón de consulta validado como referencia. Si Copilot generó una opción equivalente y correcta, puede conservarla; si no, use esta versión modificada:

   ```sql
   USE Sql2025Lab;
   GO

   DECLARE @StartDate date = '2026-01-01';
   DECLARE @EndDate   date = '2026-02-01';

   SET STATISTICS IO ON;
   SET STATISTICS TIME ON;
   GO

   SELECT
       o.CustomerID,
       COUNT_BIG(DISTINCT o.OrderID) AS OrderCount,
       SUM(CONVERT(bigint, od.Quantity)) AS UnitsSold,
       SUM(CONVERT(decimal(19,4), od.UnitPrice) * od.Quantity) AS TotalSales
   FROM Sales.Orders AS o
   INNER JOIN Sales.OrderDetails AS od
       ON od.OrderID = o.OrderID
   WHERE o.OrderDate >= @StartDate
     AND o.OrderDate < @EndDate
   GROUP BY o.CustomerID
   ORDER BY TotalSales DESC;
   GO

   SET STATISTICS IO OFF;
   SET STATISTICS TIME OFF;
   GO
   ```

6. Active el plan de ejecución real en SSMS con `Ctrl+M`, o use la opción equivalente de plan real en la herramienta disponible.

7. Ejecute la consulta al menos dos veces. Registre en el informe:

   - Si la consulta ejecutó correctamente.
   - Número de filas devueltas.
   - Lecturas lógicas observadas.
   - Tiempo de CPU y duración aproximada.
   - Operadores principales del plan.
   - Diferencias entre la respuesta original de Copilot y el script final.

8. Clasifique la salida de Copilot como:

   - **Aceptada:** se usó sin cambios funcionales significativos.
   - **Modificada:** se corrigieron tipos, filtros, agregaciones, convenciones o aspectos de rendimiento.
   - **Rechazada:** no cumplía la semántica, era insegura o no era compatible con el esquema.

**Resultado esperado:**

- Existe una consulta de ventas parametrizada y validada.
- La consulta usa un filtro de fecha sargable.
- Se han capturado métricas básicas de `STATISTICS IO` y `STATISTICS TIME`.
- La respuesta de IA ha sido clasificada y justificada.

**Verificación:**

Ejecute la siguiente consulta para comprobar que la instrucción se registró en Query Store después de su ejecución:

```sql
USE Sql2025Lab;
GO

SELECT TOP (10)
    q.query_id,
    p.plan_id,
    rs.count_executions,
    CAST(rs.avg_duration / 1000.0 AS decimal(18,2)) AS AvgDurationMs,
    CAST(rs.avg_cpu_time / 1000.0 AS decimal(18,2)) AS AvgCpuMs,
    qt.query_sql_text
FROM sys.query_store_query_text AS qt
INNER JOIN sys.query_store_query AS q
    ON q.query_text_id = qt.query_text_id
INNER JOIN sys.query_store_plan AS p
    ON p.query_id = q.query_id
INNER JOIN sys.query_store_runtime_stats AS rs
    ON rs.plan_id = p.plan_id
WHERE qt.query_sql_text LIKE N'%Sales.Orders%'
  AND qt.query_sql_text LIKE N'%Sales.OrderDetails%'
ORDER BY rs.last_execution_time DESC;
GO
```

---

### Paso 4: Explicar `Reporting.usp_OrderSearch` y evaluar un fragmento de plan de ejecución

**Objetivo:** usar IA para generar una explicación inicial del procedimiento y contrastarla con su definición, metadatos, plan de ejecución real y evidencia del motor.

**Instrucciones:**

1. Obtenga los parámetros y la definición del procedimiento:

   ```sql
   USE Sql2025Lab;
   GO

   SELECT
       p.parameter_id,
       p.name AS ParameterName,
       TYPE_NAME(p.user_type_id) AS DataType,
       p.max_length,
       p.is_output,
       p.has_default_value,
       p.default_value
   FROM sys.parameters AS p
   WHERE p.object_id = OBJECT_ID(N'Reporting.usp_OrderSearch')
   ORDER BY p.parameter_id;
   GO

   SELECT OBJECT_DEFINITION(OBJECT_ID(N'Reporting.usp_OrderSearch')) AS ProcedureDefinition;
   GO
   ```

2. Lea la definición antes de abrir Copilot. Identifique:

   - Tablas consultadas.
   - Parámetros de búsqueda.
   - Predicados `WHERE`.
   - Uniones.
   - Ordenamientos.
   - Posibles conversiones implícitas.
   - Funciones aplicadas sobre columnas filtradas.
   - Uso de SQL dinámico, si existe.

3. No copie resultados de negocio. Envíe a Copilot únicamente la estructura T-SQL necesaria y un resumen controlado. Use este prompt:

   ```text
   Analiza el siguiente procedimiento almacenado de un laboratorio SQL Server.
   Explica: propósito probable, parámetros, tablas referenciadas, predicados,
   uniones, posibles riesgos de rendimiento y supuestos que no puedes confirmar.
   No afirmes que un índice o una reescritura resolverá el problema sin métricas.
   Distingue entre hechos observables en el código e hipótesis.
   No se proporcionan datos reales.

   [PEGAR AQUÍ SOLO LA DEFINICIÓN O UN FRAGMENTO CONTROLADO DE Reporting.usp_OrderSearch]
   ```

4. Compare la explicación de Copilot con los hechos obtenidos desde SQL Server. Registre discrepancias. Ejemplos de afirmaciones que deben revisarse:

   - “Esta tabla tiene un índice adecuado.”
   - “La consulta siempre hace un table scan.”
   - “El parámetro causa parameter sniffing.”
   - “La conversión implícita es la causa principal.”
   - “La sugerencia de índice debe implementarse.”

5. Ejecute el procedimiento con parámetros de laboratorio conocidos de la práctica 2. Si no se dispone de parámetros previos, consulte primero ejemplos válidos sin enviar datos a Copilot:

   ```sql
   SELECT TOP (10)
       name,
       system_type_name,
       suggested_value
   FROM sys.dm_exec_describe_first_result_set_for_object
   (
       OBJECT_ID(N'Reporting.usp_OrderSearch'),
       0
   );
   GO
   ```

   Si la vista no devuelve metadatos por la forma del procedimiento, utilice los parámetros identificados en `sys.parameters` y los valores de prueba definidos en la práctica 2.

6. Antes de ejecutar, active métricas y plan de ejecución real. Use este patrón, sustituyendo solo los parámetros reales del procedimiento:

   ```sql
   USE Sql2025Lab;
   GO

   SET STATISTICS IO ON;
   SET STATISTICS TIME ON;
   GO

   EXEC Reporting.usp_OrderSearch
       @ParameterName = @TestValue;
   GO

   SET STATISTICS IO OFF;
   SET STATISTICS TIME OFF;
   GO
   ```

7. En el plan de ejecución real, seleccione un operador relevante, por ejemplo un `Index Scan`, `Index Seek`, `Key Lookup`, `Sort`, `Hash Match` o `Nested Loops`. Copie solo un resumen técnico no sensible, por ejemplo:

   ```text
   Operador: Index Scan
   Objeto: Sales.Orders
   Filas estimadas: 500
   Filas reales: 25 000
   Predicado: función aplicada sobre OrderDate
   Advertencias: ninguna
   ```

8. En Copilot Chat, use el siguiente prompt para interpretar el fragmento:

   ```text
   Interpreta este fragmento textual y anonimizado de un plan de ejecución de SQL Server.
   Explica qué puede significar la diferencia entre filas estimadas y filas reales,
   qué evidencia adicional se necesita y qué acciones de bajo riesgo conviene evaluar.
   No recomiendes implementar índices ni cambios de producción de forma automática.

   Operador: [OPERADOR]
   Filas estimadas: [VALOR]
   Filas reales: [VALOR]
   Predicado: [RESUMEN]
   Advertencias: [RESUMEN]
   ```

9. Clasifique la explicación de IA. Acepte únicamente afirmaciones verificables; modifique conclusiones exageradas y rechace recomendaciones que no tengan evidencia.

10. Guarde las consultas de investigación y notas técnicas en:

   ```text
   C:\SQLLab2025\Scripts\Lab03\03_OrderSearchAnalysis.sql
   ```

**Resultado esperado:**

- Existe una explicación documentada del procedimiento con hechos, hipótesis y limitaciones.
- Se ha obtenido al menos un plan de ejecución real o una evidencia equivalente de ejecución.
- Se ha comparado la explicación de Copilot con la definición y métricas reales.
- No se ha implementado ningún cambio de índice basándose solo en una respuesta de IA.

**Verificación:**

Ejecute esta consulta para revisar los planes de Query Store asociados a texto que mencione el procedimiento o sus tablas principales:

```sql
USE Sql2025Lab;
GO

SELECT TOP (20)
    q.query_id,
    p.plan_id,
    p.is_forced_plan,
    rs.count_executions,
    CAST(rs.avg_duration / 1000.0 AS decimal(18,2)) AS AvgDurationMs,
    CAST(rs.avg_logical_io_reads AS decimal(18,2)) AS AvgLogicalReads,
    rs.last_execution_time,
    qt.query_sql_text
FROM sys.query_store_query_text AS qt
INNER JOIN sys.query_store_query AS q
    ON q.query_text_id = qt.query_text_id
INNER JOIN sys.query_store_plan AS p
    ON p.query_id = q.query_id
INNER JOIN sys.query_store_runtime_stats AS rs
    ON rs.plan_id = p.plan_id
WHERE qt.query_sql_text LIKE N'%OrderSearch%'
   OR qt.query_sql_text LIKE N'%Sales.Orders%'
ORDER BY rs.last_execution_time DESC;
GO
```

---

### Paso 5: Evaluar una optimización de IA para filtros no sargables

**Objetivo:** comparar una propuesta de reescritura generada por IA con la evidencia de Query Store, planes de ejecución y resultados de la práctica 2.

**Instrucciones:**

1. Revise `C:\SQLLab2025\Scripts\Lab02\QueryComparisonBeforeAfter.sql` y `C:\SQLLab2025\Reports\Lab02_PerformanceFindings.md`.

2. Identifique un patrón no sargable observado en la práctica 2. Ejemplos válidos:

   ```sql
   WHERE CONVERT(date, o.OrderDate) = @OrderDate
   ```

   ```sql
   WHERE YEAR(o.OrderDate) = @Year
   ```

   ```sql
   WHERE ISNULL(o.CustomerID, 0) = @CustomerID
   ```

   ```sql
   WHERE LEFT(o.SomeCode, 3) = @Prefix
   ```

3. Si la práctica 2 no contiene un patrón no sargable utilizable, use este ejemplo de comparación de solo lectura con fechas:

   ```sql
   USE Sql2025Lab;
   GO

   DECLARE @OrderDate date = '2026-01-15';
   DECLARE @NextDate  date = DATEADD(day, 1, @OrderDate);

   -- Patrón no sargable: referencia una función sobre la columna filtrada.
   SELECT
       COUNT_BIG(*) AS OrdersFound
   FROM Sales.Orders AS o
   WHERE CONVERT(date, o.OrderDate) = @OrderDate;

   -- Patrón sargable: rango sobre la columna original.
   SELECT
       COUNT_BIG(*) AS OrdersFound
   FROM Sales.Orders AS o
   WHERE o.OrderDate >= @OrderDate
     AND o.OrderDate < @NextDate;
   GO
   ```

4. Solicite a Copilot una hipótesis controlada mediante este prompt:

   ```text
   En SQL Server, evalúa conceptualmente una consulta que filtra una columna datetime
   usando CONVERT(date, ColumnaFecha) = @Fecha. Propón alternativas sargables,
   explica cómo preservar la semántica de fechas y enumera métricas que deben
   compararse antes de aceptar el cambio. No propongas cambios DDL ni índices
   sin revisar planes, estadísticas, Query Store y carga de escritura.
   ```

5. Evalúe la respuesta contra estos criterios:

   - Debe proponer un rango con límite inicial inclusivo y final exclusivo.
   - Debe reconocer que el tipo de datos de la columna importa.
   - Debe pedir comparación de resultados para preservar semántica.
   - Debe mencionar planes, lecturas lógicas, CPU, duración y estimaciones.
   - No debe asegurar una mejora universal.
   - No debe recomendar `NOLOCK` como solución de rendimiento.

6. Guarde la prueba en `04_NonSargableFilterReview.sql`. Ejecute ambas variantes con métricas activadas:

   ```sql
   USE Sql2025Lab;
   GO

   DECLARE @OrderDate date = '2026-01-15';
   DECLARE @NextDate  date = DATEADD(day, 1, @OrderDate);

   SET STATISTICS IO ON;
   SET STATISTICS TIME ON;
   GO

   BEGIN TRANSACTION;
   GO

   SELECT
       COUNT_BIG(*) AS NonSargableCount
   FROM Sales.Orders AS o
   WHERE CONVERT(date, o.OrderDate) = @OrderDate;
   GO

   SELECT
       COUNT_BIG(*) AS SargableCount
   FROM Sales.Orders AS o
   WHERE o.OrderDate >= @OrderDate
     AND o.OrderDate < @NextDate;
   GO

   ROLLBACK TRANSACTION;
   GO

   SET STATISTICS IO OFF;
   SET STATISTICS TIME OFF;
   GO
   ```

7. Compruebe que ambas consultas devuelven el mismo conteo. Si los resultados difieren, no acepte la reescritura hasta investigar el tipo de datos, zonas horarias, valores `NULL` y semántica requerida.

8. Compare las métricas observadas con:

   - Los datos registrados durante la práctica 2.
   - Los planes de ejecución reales.
   - La información disponible en Query Store.
   - Los índices ya existentes.
   - El impacto potencial sobre inserciones y actualizaciones si Copilot sugiere un índice.

9. Consulte los índices actuales antes de considerar cualquier propuesta DDL:

   ```sql
   USE Sql2025Lab;
   GO

   SELECT
       s.name AS SchemaName,
       t.name AS TableName,
       i.name AS IndexName,
       i.type_desc,
       i.is_primary_key,
       i.is_unique,
       STRING_AGG
       (
           CASE WHEN ic.is_included_column = 0
                THEN c.name
           END,
           N', '
       ) WITHIN GROUP (ORDER BY ic.key_ordinal) AS KeyColumns
   FROM sys.tables AS t
   INNER JOIN sys.schemas AS s
       ON s.schema_id = t.schema_id
   INNER JOIN sys.indexes AS i
       ON i.object_id = t.object_id
   INNER JOIN sys.index_columns AS ic
       ON ic.object_id = i.object_id
      AND ic.index_id = i.index_id
   INNER JOIN sys.columns AS c
       ON c.object_id = ic.object_id
      AND c.column_id = ic.column_id
   WHERE s.name = N'Sales'
     AND t.name = N'Orders'
     AND i.index_id > 0
   GROUP BY
       s.name, t.name, i.name, i.type_desc, i.is_primary_key, i.is_unique
   ORDER BY i.index_id;
   GO
   ```

10. Registre la decisión en `Lab03_AIReview.md`. Un ejemplo de decisión adecuada es:

   ```markdown
   | ID | Actividad | Recomendación o salida de IA | Decisión | Evidencia de validación |
   |---|---|---|---|---|
   | IA-03 | Filtro no sargable | Reemplazar CONVERT(date, OrderDate) = @Fecha por un rango de fechas. | Aceptada o Modificada | Conteos equivalentes; revisión de plan real; comparación de lecturas lógicas, CPU y duración; revisión de Query Store. |
   ```

**Resultado esperado:**

- Se ha evaluado una reescritura sargable sin aplicar cambios permanentes innecesarios.
- Se ha comprobado la equivalencia funcional entre ambas variantes.
- La decisión se apoya en métricas y no exclusivamente en una recomendación de IA.
- Se ha revisado el inventario de índices existente antes de contemplar cambios DDL.

**Verificación:**

Ejecute la siguiente consulta para revisar señales de estadísticas en las columnas de fecha relevantes:

```sql
USE Sql2025Lab;
GO

SELECT
    s.name AS SchemaName,
    t.name AS TableName,
    st.name AS StatisticsName,
    STATS_DATE(st.object_id, st.stats_id) AS LastUpdated,
    sp.rows AS SampledRows,
    sp.rows_sampled AS RowsSampled,
    sp.modification_counter
FROM sys.stats AS st
INNER JOIN sys.tables AS t
    ON t.object_id = st.object_id
INNER JOIN sys.schemas AS s
    ON s.schema_id = t.schema_id
CROSS APPLY sys.dm_db_stats_properties(st.object_id, st.stats_id) AS sp
WHERE s.name = N'Sales'
  AND t.name = N'Orders'
ORDER BY LastUpdated DESC;
GO
```

---

### Paso 6: Crear y validar documentación automática de objetos

**Objetivo:** crear `dbo.ObjectDocumentation`, registrar información revisada de al menos cuatro objetos y validar que la documentación coincide con los metadatos técnicos.

**Instrucciones:**

1. Identifique cuatro objetos que documentará. Deben incluir, como mínimo:

   - `Sales.Orders`
   - `Sales.OrderDetails`
   - `Reporting.usp_OrderSearch`
   - Un índice existente de `Sales.Orders`

2. Obtenga el nombre de un índice existente de `Sales.Orders`:

   ```sql
   USE Sql2025Lab;
   GO

   SELECT TOP (1)
       i.name AS IndexName,
       i.type_desc,
       i.is_primary_key,
       i.is_unique
   FROM sys.indexes AS i
   WHERE i.object_id = OBJECT_ID(N'Sales.Orders')
     AND i.index_id > 0
     AND i.name IS NOT NULL
   ORDER BY
       CASE WHEN i.is_primary_key = 1 THEN 0 ELSE 1 END,
       i.index_id;
   GO
   ```

3. Solicite a Copilot un borrador de documentación, sin pedirle que invente dependencias. Use este prompt:

   ```text
   Genera un borrador de documentación técnica para objetos de SQL Server.
   Para cada objeto, usa las columnas: nombre, tipo, propósito, dependencias,
   riesgo de cambio, fecha de revisión y revisor. Distingue entre hechos
   verificados por metadatos e hipótesis que requieren revisión humana.
   No inventes columnas, índices, dependencias ni reglas de negocio.
   Objetos a documentar:
   - Sales.Orders (tabla)
   - Sales.OrderDetails (tabla)
   - Reporting.usp_OrderSearch (procedimiento almacenado)
   - [NOMBRE DEL ÍNDICE] sobre Sales.Orders
   ```

4. Revise el borrador. Corrija manualmente cualquier propósito, dependencia o riesgo no respaldado por metadatos o por la definición del procedimiento.

5. Guarde y ejecute el siguiente script en `05_ObjectDocumentation.sql`. Sustituya el valor de `@IndexName` por el índice identificado en la instrucción 2.

   ```sql
   USE Sql2025Lab;
   GO

   IF OBJECT_ID(N'dbo.ObjectDocumentation', N'U') IS NULL
   BEGIN
       CREATE TABLE dbo.ObjectDocumentation
       (
           ObjectDocumentationID int IDENTITY(1,1) NOT NULL
               CONSTRAINT PK_ObjectDocumentation PRIMARY KEY,
           ObjectName nvarchar(512) NOT NULL,
           ObjectType nvarchar(128) NOT NULL,
           Purpose nvarchar(2000) NOT NULL,
           Dependencies nvarchar(2000) NULL,
           ChangeRisk nvarchar(1000) NOT NULL,
           ReviewedAt datetime2(0) NOT NULL,
           Reviewer sysname NOT NULL,
           CONSTRAINT UQ_ObjectDocumentation_ObjectName UNIQUE (ObjectName)
       );
   END;
   GO

   DECLARE @Reviewer sysname = SUSER_SNAME();
   DECLARE @IndexName sysname =
   (
       SELECT TOP (1) i.name
       FROM sys.indexes AS i
       WHERE i.object_id = OBJECT_ID(N'Sales.Orders')
         AND i.index_id > 0
         AND i.name IS NOT NULL
       ORDER BY
           CASE WHEN i.is_primary_key = 1 THEN 0 ELSE 1 END,
           i.index_id
   );

   IF @IndexName IS NULL
   BEGIN
       THROW 51000, 'No se encontró un índice documentable en Sales.Orders.', 1;
   END;
   GO

   DECLARE @Reviewer sysname = SUSER_SNAME();
   DECLARE @IndexName sysname =
   (
       SELECT TOP (1) i.name
       FROM sys.indexes AS i
       WHERE i.object_id = OBJECT_ID(N'Sales.Orders')
         AND i.index_id > 0
         AND i.name IS NOT NULL
       ORDER BY
           CASE WHEN i.is_primary_key = 1 THEN 0 ELSE 1 END,
           i.index_id
   );

   DECLARE @Documentation TABLE
   (
       ObjectName nvarchar(512) NOT NULL,
       ObjectType nvarchar(128) NOT NULL,
       Purpose nvarchar(2000) NOT NULL,
       Dependencies nvarchar(2000) NULL,
       ChangeRisk nvarchar(1000) NOT NULL
   );

   INSERT INTO @Documentation
   (
       ObjectName,
       ObjectType,
       Purpose,
       Dependencies,
       ChangeRisk
   )
   VALUES
   (
       N'Sales.Orders',
       N'TABLE',
       N'Almacena pedidos del laboratorio. El propósito exacto debe ser confirmado con las columnas y reglas de negocio disponibles.',
       N'Referenciado por Sales.OrderDetails y por procedimientos o consultas que recuperan pedidos.',
       N'Alto: modificar columnas, claves o tipos puede afectar uniones, procedimientos, informes, índices y cargas.'
   ),
   (
       N'Sales.OrderDetails',
       N'TABLE',
       N'Almacena líneas o detalles asociados a pedidos del laboratorio.',
       N'Depende funcionalmente de Sales.Orders mediante OrderID; confirmar restricciones y relaciones en metadatos.',
       N'Alto: cambios en cantidades, precios o claves pueden afectar agregados de ventas y resultados históricos.'
   ),
   (
       N'Reporting.usp_OrderSearch',
       N'SQL_STORED_PROCEDURE',
       N'Procedimiento de consulta para búsqueda o recuperación de pedidos. El comportamiento exacto se valida mediante su definición.',
       N'Dependencias obtenidas desde sys.sql_expression_dependencies y revisión de la definición.',
       N'Medio/alto: cambios de parámetros, filtros o tipos pueden alterar resultados, planes de ejecución y consumidores de informes.'
   ),
   (
       N'Sales.Orders.' + @IndexName,
       N'INDEX',
       N'Índice existente sobre Sales.Orders documentado a partir de sys.indexes y sys.index_columns.',
       N'Sales.Orders; columnas de clave e incluidas verificadas en metadatos.',
       N'Medio/alto: modificar o eliminar el índice puede afectar planes, lecturas, escrituras, espacio y mantenimiento.'
   );

   MERGE dbo.ObjectDocumentation AS target
   USING @Documentation AS source
       ON target.ObjectName = source.ObjectName
   WHEN MATCHED THEN
       UPDATE SET
           ObjectType = source.ObjectType,
           Purpose = source.Purpose,
           Dependencies = source.Dependencies,
           ChangeRisk = source.ChangeRisk,
           ReviewedAt = SYSDATETIME(),
           Reviewer = @Reviewer
   WHEN NOT MATCHED THEN
       INSERT
       (
           ObjectName,
           ObjectType,
           Purpose,
           Dependencies,
           ChangeRisk,
           ReviewedAt,
           Reviewer
       )
       VALUES
       (
           source.ObjectName,
           source.ObjectType,
           source.Purpose,
           source.Dependencies,
           source.ChangeRisk,
           SYSDATETIME(),
           @Reviewer
       );
   GO
   ```

6. Obtenga dependencias reales del procedimiento y compárelas con el texto documentado:

   ```sql
   USE Sql2025Lab;
   GO

   SELECT
       OBJECT_SCHEMA_NAME(d.referencing_id) AS ReferencingSchema,
       OBJECT_NAME(d.referencing_id) AS ReferencingObject,
       d.referenced_schema_name AS ReferencedSchema,
       d.referenced_entity_name AS ReferencedEntity,
       d.referenced_database_name AS ReferencedDatabase,
       d.is_ambiguous,
       d.is_caller_dependent
   FROM sys.sql_expression_dependencies AS d
   WHERE d.referencing_id = OBJECT_ID(N'Reporting.usp_OrderSearch')
   ORDER BY
       d.referenced_schema_name,
       d.referenced_entity_name;
   GO
   ```

7. Corrija el campo `Dependencies` si el procedimiento referencia objetos que no estaban documentados. No invente dependencias dinámicas: si el procedimiento usa SQL dinámico y una dependencia no se detecta en `sys.sql_expression_dependencies`, anote explícitamente que requiere revisión manual.

8. Compruebe la existencia y tipo de los objetos documentados:

   ```sql
   USE Sql2025Lab;
   GO

   SELECT
       d.ObjectName,
       d.ObjectType,
       d.ReviewedAt,
       d.Reviewer,
       CASE
           WHEN d.ObjectName = N'Sales.Orders'
                AND OBJECT_ID(N'Sales.Orders', N'U') IS NOT NULL THEN N'Válido'
           WHEN d.ObjectName = N'Sales.OrderDetails'
                AND OBJECT_ID(N'Sales.OrderDetails', N'U') IS NOT NULL THEN N'Válido'
           WHEN d.ObjectName = N'Reporting.usp_OrderSearch'
                AND OBJECT_ID(N'Reporting.usp_OrderSearch', N'P') IS NOT NULL THEN N'Válido'
           WHEN d.ObjectType = N'INDEX'
                AND EXISTS
                (
                    SELECT 1
                    FROM sys.indexes AS i
                    WHERE d.ObjectName =
                        N'Sales.Orders.' + i.name
                      AND i.object_id = OBJECT_ID(N'Sales.Orders')
                ) THEN N'Válido'
           ELSE N'Requiere revisión'
       END AS MetadataValidation
   FROM dbo.ObjectDocumentation AS d
   ORDER BY d.ObjectDocumentationID;
   GO
   ```

**Resultado esperado:**

- Existe la tabla `dbo.ObjectDocumentation`.
- Hay al menos cuatro objetos documentados.
- Cada fila incluye nombre, tipo, propósito, dependencias, riesgo de cambio, fecha de revisión y revisor.
- Las dependencias del procedimiento se han contrastado con metadatos.
- La documentación distingue hechos técnicos de elementos que requieren revisión humana.

**Verificación:**

Ejecute:

```sql
USE Sql2025Lab;
GO

SELECT
    COUNT(*) AS DocumentedObjects,
    MIN(ReviewedAt) AS FirstReview,
    MAX(ReviewedAt) AS LastReview
FROM dbo.ObjectDocumentation;
GO
```

El valor de `DocumentedObjects` debe ser igual o mayor que `4`.

---

### Paso 7: Elaborar el informe final de revisión de IA

**Objetivo:** consolidar las decisiones, evidencia y límites de uso de Copilot en un informe reproducible.

**Instrucciones:**

1. Abra:

   ```text
   C:\SQLLab2025\Reports\Lab03_AIReview.md
   ```

2. Complete el informe con la siguiente estructura:

   ```markdown
   # Informe de revisión de IA — Lab 03

   ## Controles de uso de IA
   [Incluir los controles definidos en el paso 2.]

   ## Entorno validado
   - Servidor:
   - Instancia:
   - Base de datos:
   - Compatibilidad:
   - Estado de Query Store:
   - Herramienta de ejecución:

   ## IA-01: Consulta de ventas por cliente y período
   - Prompt utilizado:
   - Clasificación: Aceptada / Modificada / Rechazada
   - Cambios manuales aplicados:
   - Validación sintáctica:
   - Validación funcional:
   - Métricas observadas:
   - Plan de ejecución revisado:
   - Conclusión:

   ## IA-02: Explicación de Reporting.usp_OrderSearch
   - Hechos validados en la definición:
   - Hipótesis de IA:
   - Elementos confirmados:
   - Elementos no confirmados o rechazados:
   - Evidencia de Query Store y plan:
   - Conclusión:

   ## IA-03: Filtro no sargable
   - Patrón original:
   - Alternativa propuesta:
   - Equivalencia de resultados:
   - Comparación de lecturas, CPU y duración:
   - Comparación de planes:
   - Decisión:
   - Riesgos o limitaciones:

   ## IA-04: Documentación de objetos
   - Objetos documentados:
   - Metadatos validados:
   - Dependencias confirmadas:
   - Dependencias que requieren revisión manual:
   - Decisión:

   ## Conclusión general
   Copilot aceleró la generación de borradores y la interpretación inicial,
   pero Query Store, planes de ejecución, estadísticas, pruebas controladas
   y revisión humana determinaron las decisiones técnicas finales.
   ```

3. Incluya una tabla de decisiones final:

   ```markdown
   | ID | Decisión | Justificación breve |
   |---|---|---|
   | IA-01 | Aceptada / Modificada / Rechazada | Basada en sintaxis, resultados y métricas. |
   | IA-02 | Aceptada / Modificada / Rechazada | Basada en definición, plan y Query Store. |
   | IA-03 | Aceptada / Modificada / Rechazada | Basada en equivalencia y rendimiento observado. |
   | IA-04 | Aceptada / Modificada / Rechazada | Basada en metadatos y revisión humana. |
   ```

4. Añada una conclusión explícita que responda a estas preguntas:

   - ¿Qué sugerencia de IA fue útil como hipótesis inicial?
   - ¿Qué sugerencia requirió modificación?
   - ¿Qué afirmación no pudo confirmarse con evidencia?
   - ¿Qué métrica o artefacto de SQL Server fue más importante para la decisión?
   - ¿Por qué no se debe desplegar una sugerencia de IA sin validación?

5. Guarde el informe y compruebe su existencia:

   ```powershell
   Get-Item 'C:\SQLLab2025\Reports\Lab03_AIReview.md' |
       Select-Object FullName, Length, LastWriteTime
   ```

**Resultado esperado:**

- El informe final existe en la ruta requerida.
- El informe registra prompts, decisiones, evidencia y límites.
- Las cuatro actividades de IA tienen una clasificación.
- La conclusión establece que la IA asistió el análisis, pero no sustituyó la validación técnica.

**Verificación:**

Compruebe que los scripts requeridos y el informe existen:

```powershell
$requiredFiles = @(
    'C:\SQLLab2025\Scripts\Lab03\01_EnvironmentCheck.sql',
    'C:\SQLLab2025\Scripts\Lab03\02_CopilotGeneratedSalesQuery.sql',
    'C:\SQLLab2025\Scripts\Lab03\03_OrderSearchAnalysis.sql',
    'C:\SQLLab2025\Scripts\Lab03\04_NonSargableFilterReview.sql',
    'C:\SQLLab2025\Scripts\Lab03\05_ObjectDocumentation.sql',
    'C:\SQLLab2025\Reports\Lab03_AIReview.md'
)

$requiredFiles | ForEach-Object {
    [PSCustomObject]@{
        Path = $_
        Exists = Test-Path $_
    }
}
```

Todos los valores de `Exists` deben ser `True`.

## Validación y Pruebas

Complete las siguientes validaciones antes de dar por finalizada la práctica.

| Validación | Método | Criterio de aceptación |
|---|---|---|
| Conectividad SQL | Consulta de contexto y versión | Conexión a `LOCALHOST,1433` y base `Sql2025Lab` |
| Query Store | Consulta a `sys.database_query_store_options` | Estado `READ_WRITE` |
| Consulta generada | Ejecución con parámetros de laboratorio | Sintaxis correcta, resultados válidos y filtro de fechas sargable |
| Evidencia de rendimiento | `STATISTICS IO`, `STATISTICS TIME` y plan real | Métricas registradas en el informe |
| Explicación del procedimiento | Comparación con definición y plan real | Hechos separados de hipótesis |
| Filtro no sargable | Comparación de conteos y planes | Equivalencia funcional confirmada antes de aceptar la reescritura |
| Índices | Consulta a `sys.indexes` y `sys.index_columns` | No se propone DDL sin revisar índices existentes |
| Documentación | Consulta a `dbo.ObjectDocumentation` | Cuatro o más objetos documentados |
| Dependencias | `sys.sql_expression_dependencies` | Dependencias verificadas o marcadas para revisión manual |
| Seguridad de IA | Revisión de scripts e informe | No hay contraseñas ni datos sensibles almacenados |

Ejecute esta comprobación final consolidada:

```sql
USE Sql2025Lab;
GO

SELECT
    DB_NAME() AS DatabaseName,
    DATABASEPROPERTYEX(DB_NAME(), 'Status') AS DatabaseStatus,
    (SELECT actual_state_desc FROM sys.database_query_store_options) AS QueryStoreState,
    OBJECT_ID(N'dbo.ObjectDocumentation', N'U') AS ObjectDocumentationTableId,
    (SELECT COUNT(*) FROM dbo.ObjectDocumentation) AS DocumentedObjectCount,
    OBJECT_ID(N'Reporting.usp_OrderSearch', N'P') AS OrderSearchProcedureId;
GO
```

Resultado mínimo esperado:

- `DatabaseStatus = ONLINE`
- `QueryStoreState = READ_WRITE`
- `ObjectDocumentationTableId` no es `NULL`
- `DocumentedObjectCount >= 4`
- `OrderSearchProcedureId` no es `NULL`

## Solución de Problemas

### Incidencia 1: GitHub Copilot Chat no responde, solicita autenticación o no está disponible

**Síntomas:**

- Copilot Chat muestra un mensaje de autenticación pendiente.
- La ventana de chat no genera respuestas.
- Visual Studio Code indica que la extensión está deshabilitada o no autorizada.
- Se produce un error relacionado con conectividad de red.

**Causa probable:**

La cuenta de GitHub no está autorizada para Copilot, la extensión no está instalada o habilitada, o la conectividad HTTPS saliente por el puerto 443 está bloqueada por una política de red.

**Corrección:**

1. En Visual Studio Code, confirme que las extensiones **GitHub Copilot** y **GitHub Copilot Chat** están instaladas y habilitadas.
2. Ejecute `GitHub: Sign In` desde la paleta de comandos.
3. Complete la autenticación con una cuenta autorizada para GitHub Copilot.
4. Compruebe la conectividad HTTPS:

   ```powershell
   Test-NetConnection github.com -Port 443
   ```

5. Si la conexión falla, revise la configuración de proxy, firewall o DNS del laboratorio con el administrador responsable.
6. Mientras se corrige el acceso, complete las validaciones SQL manualmente y documente que la generación de borradores por IA no estuvo disponible. No sustituya la evidencia técnica por supuestas respuestas de IA.

### Incidencia 2: La consulta de Query Store no devuelve ejecuciones recientes o Query Store aparece como READ_ONLY

**Síntomas:**

- Las consultas sobre `sys.query_store_runtime_stats` no muestran la ejecución realizada.
- `actual_state_desc` muestra `READ_ONLY`.
- Query Store no registra nuevas consultas pese a que la base de datos está disponible.

**Causa probable:**

Query Store alcanzó su cuota de almacenamiento, está configurado como solo lectura, la consulta se ejecutó en otra base de datos o aún no se ha consolidado la información de intervalos de ejecución.

**Corrección:**

1. Confirme la base de datos actual:

   ```sql
   SELECT DB_NAME() AS CurrentDatabase;
   ```

2. Revise el estado y el espacio de Query Store:

   ```sql
   USE Sql2025Lab;
   GO

   SELECT
       actual_state_desc,
       desired_state_desc,
       current_storage_size_mb,
       max_storage_size_mb,
       readonly_reason
   FROM sys.database_query_store_options;
   GO
   ```

3. Si el estado es `READ_ONLY`, no elimine datos ni cambie parámetros sin autorización del instructor. Registre la condición en el informe.
4. Si el instructor autoriza el ajuste en este entorno de laboratorio, revise la cuota configurada y habilite Query Store en modo de lectura y escritura conforme a la política del curso.
5. Ejecute nuevamente la consulta de prueba, espere unos minutos y vuelva a consultar Query Store.
6. Como evidencia complementaria, use el plan de ejecución real y `SET STATISTICS IO/TIME ON` mientras Query Store vuelve a estar disponible.

## Limpieza

La práctica conserva scripts, informe y documentación para revisión en prácticas posteriores. No elimine los archivos requeridos ni la tabla `dbo.ObjectDocumentation`, ya que será revisada desde perspectivas de configuración y seguridad en las prácticas 4 y 5.

Realice únicamente estas acciones de cierre:

1. Compruebe que no quedó ninguna transacción abierta:

   ```sql
   SELECT
       @@TRANCOUNT AS OpenTransactionCount,
       XACT_STATE() AS TransactionState;
   GO
   ```

   El resultado esperado es:

   ```text
   OpenTransactionCount = 0
   TransactionState = 0
   ```

2. Desactive cualquier opción de métricas que haya quedado activada en una ventana de consulta:

   ```sql
   SET STATISTICS IO OFF;
   SET STATISTICS TIME OFF;
   GO
   ```

3. Cierre las ventanas de consulta que contengan valores de prueba o fragmentos de definición no necesarios.

4. Cierre la conexión de Visual Studio Code si el equipo será utilizado por otro estudiante.

5. Verifique que no se guardaron contraseñas en scripts o informes:

   ```powershell
   Get-ChildItem 'C:\SQLLab2025\Scripts\Lab03','C:\SQLLab2025\Reports' -Recurse -File |
       Select-String -Pattern 'SqlLab2025!ChangeMe|LabAccess2025!ChangeMe'
   ```

   El resultado debe estar vacío.

## Resumen

En esta práctica se empleó GitHub Copilot Chat como asistente para generar código, explicar procedimientos, formular hipótesis de rendimiento y crear documentación inicial. Las propuestas se validaron mediante ejecución controlada, planes de ejecución reales, `STATISTICS IO`, `STATISTICS TIME`, Query Store, metadatos e inspección humana.

El resultado principal no es solamente una consulta generada por IA, sino un proceso repetible de evaluación: **solicitar un borrador, identificar supuestos, recopilar evidencia, probar de forma controlada, clasificar la recomendación y documentar la decisión**. La tabla `dbo.ObjectDocumentation`, los scripts de `C:\SQLLab2025\Scripts\Lab03` y el informe `C:\SQLLab2025\Reports\Lab03_AIReview.md` constituyen los entregables de esta práctica.

Recursos recomendados:

- [Supervisar el rendimiento mediante Query Store](https://learn.microsoft.com/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store)
- [Planes de ejecución de consultas en SQL Server](https://learn.microsoft.com/sql/relational-databases/performance/execution-plans)
- [Procesamiento inteligente de consultas](https://learn.microsoft.com/sql/relational-databases/performance/intelligent-query-processing-details)
- [Catálogos del sistema de SQL Server](https://learn.microsoft.com/sql/relational-databases/system-catalog-views/catalog-views-transact-sql)
- [Documentación de GitHub Copilot](https://docs.github.com/copilot)
