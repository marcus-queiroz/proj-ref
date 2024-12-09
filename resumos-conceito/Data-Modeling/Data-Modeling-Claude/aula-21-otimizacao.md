# Aula 21 - Otimização de Consultas

## Objetivos da Aula
- Compreender planos de execução
- Trabalhar com estatísticas
- Utilizar hints apropriadamente
- Identificar problemas comuns
- Implementar técnicas de otimização

## 1. Planos de Execução

### 1.1 Análise de Planos
```sql
-- Habilitar plano de execução
SET SHOWPLAN_XML ON;
GO

-- Query para análise
SELECT 
    d.YearMonth,
    p.CategoryName,
    SUM(f.Amount) as TotalAmount,
    COUNT(DISTINCT f.Customer_SK) as CustomerCount
FROM fact.Sales f
JOIN dim.Date d ON f.Date_SK = d.Date_SK
JOIN dim.Product p ON f.Product_SK = p.Product_SK
WHERE d.Year = 2024
GROUP BY d.YearMonth, p.CategoryName;

-- Procedure para capturar planos
CREATE PROCEDURE monitor.usp_CaptureQueryPlan
    @QueryText nvarchar(max)
AS
BEGIN
    DECLARE @PlanXML xml;

    -- Capturar plano estimado
    SET @PlanXML = (
        SELECT query_plan
        FROM sys.dm_exec_query_plan(
            (SELECT TOP 1 plan_handle
             FROM sys.dm_exec_cached_plans cp
             CROSS APPLY sys.dm_exec_sql_text(cp.plan_handle) st
             WHERE st.text LIKE @QueryText)
        )
    );

    -- Salvar plano
    INSERT INTO monitor.QueryPlans (
        QueryText,
        QueryPlan,
        CaptureDate
    )
    VALUES (
        @QueryText,
        @PlanXML,
        GETDATE()
    );
END;
```

### 1.2 Análise de Operadores
```sql
-- View para análise de operadores
CREATE VIEW monitor.QueryOperators
AS
WITH XMLNAMESPACES (
    DEFAULT 'http://schemas.microsoft.com/sqlserver/2004/07/showplan'
)
SELECT 
    qp.QueryPlanID,
    op.value('@PhysicalOp', 'nvarchar(100)') as PhysicalOperator,
    op.value('@LogicalOp', 'nvarchar(100)') as LogicalOperator,
    op.value('@EstimateRows', 'float') as EstimatedRows,
    op.value('@EstimateIO', 'float') as EstimatedIO,
    op.value('@EstimateCPU', 'float') as EstimatedCPU
FROM monitor.QueryPlans qp
CROSS APPLY qp.QueryPlan.nodes('//RelOp') as t(op);
```

## 2. Estatísticas

### 2.1 Manutenção de Estatísticas
```sql
-- Procedure para atualização de estatísticas
CREATE PROCEDURE maintenance.usp_UpdateStatistics
    @TableName nvarchar(128),
    @SamplePercent decimal(5,2) = NULL
AS
BEGIN
    DECLARE @SQL nvarchar(max);
    DECLARE @StatName nvarchar(128);
    
    -- Cursor para estatísticas
    DECLARE stats_cursor CURSOR FOR
    SELECT name
    FROM sys.stats
    WHERE object_id = OBJECT_ID(@TableName);

    OPEN stats_cursor;
    FETCH NEXT FROM stats_cursor INTO @StatName;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @SQL = 'UPDATE STATISTICS ' + @TableName + 
                  ' (' + @StatName + ')';

        IF @SamplePercent IS NOT NULL
            SET @SQL = @SQL + ' WITH SAMPLE ' + 
                      CAST(@SamplePercent as varchar(10)) + ' PERCENT';

        EXEC sp_executesql @SQL;

        FETCH NEXT FROM stats_cursor INTO @StatName;
    END;

    CLOSE stats_cursor;
    DEALLOCATE stats_cursor;
END;
```

### 2.2 Análise de Estatísticas
```sql
-- View para análise de estatísticas
CREATE VIEW monitor.StatisticsInfo
AS
SELECT 
    OBJECT_SCHEMA_NAME(s.object_id) as SchemaName,
    OBJECT_NAME(s.object_id) as TableName,
    s.name as StatisticsName,
    sp.last_updated,
    sp.rows,
    sp.rows_sampled,
    sp.steps as Histogram_Steps,
    sp.unfiltered_rows,
    sp.modification_counter
FROM sys.stats s
CROSS APPLY sys.dm_db_stats_properties(s.object_id, s.stats_id) sp
WHERE s.object_id > 100;

-- Procedure para análise detalhada
CREATE PROCEDURE monitor.usp_AnalyzeStatistics
    @TableName nvarchar(128)
AS
BEGIN
    -- Estatísticas desatualizadas
    SELECT 
        StatisticsName,
        last_updated,
        rows,
        modification_counter,
        CAST(modification_counter * 100.0 / rows as decimal(5,2)) 
            as ModificationPercent
    FROM monitor.StatisticsInfo
    WHERE TableName = @TableName
    AND modification_counter > 0;

    -- Histograma
    DBCC SHOW_STATISTICS(@TableName, NULL) WITH HISTOGRAM;
END;
```

## 3. Query Hints

### 3.1 Uso Apropriado de Hints
```sql
-- Exemplos de hints
-- FORCE ORDER
SELECT /*+ FORCE ORDER */
    d.YearMonth,
    p.CategoryName,
    SUM(f.Amount) as TotalAmount
FROM fact.Sales f
JOIN dim.Date d ON f.Date_SK = d.Date_SK
JOIN dim.Product p ON f.Product_SK = p.Product_SK
GROUP BY d.YearMonth, p.CategoryName;

-- INDEX
SELECT /*+ INDEX(f IX_Sales_Date) */
    f.Date_SK,
    SUM(f.Amount) as TotalAmount
FROM fact.Sales f
GROUP BY f.Date_SK;

-- HASH JOIN
SELECT /*+ HASH JOIN */
    d.YearMonth,
    SUM(f.Amount) as TotalAmount
FROM fact.Sales f
JOIN dim.Date d ON f.Date_SK = d.Date_SK
GROUP BY d.YearMonth;
```

### 3.2 Monitoramento de Hints
```sql
-- Procedure para análise de hints
CREATE PROCEDURE monitor.usp_AnalyzeQueryHints
AS
BEGIN
    WITH XMLNAMESPACES (
        DEFAULT 'http://schemas.microsoft.com/sqlserver/2004/07/showplan'
    )
    SELECT 
        qp.QueryText,
        op.value('@StatementOptmLevel', 'nvarchar(100)') 
            as OptimizationLevel,
        op.value('@StatementOptmEarlyAbortReason', 'nvarchar(100)') 
            as EarlyAbortReason,
        op.value('@StatementSubTreeCost', 'float') as SubTreeCost,
        qp.CaptureDate
    FROM monitor.QueryPlans qp
    CROSS APPLY qp.QueryPlan.nodes('//StmtSimple') as t(op)
    WHERE qp.QueryText LIKE '%/*+%';
END;
```

## 4. Problemas Comuns

### 4.1 Identificação de Problemas
```sql
-- View para identificar problemas comuns
CREATE VIEW monitor.QueryProblems
AS
WITH XMLNAMESPACES (
    DEFAULT 'http://schemas.microsoft.com/sqlserver/2004/07/showplan'
)
SELECT 
    qp.QueryPlanID,
    qp.QueryText,
    CASE 
        WHEN qp.QueryPlan.exist('//RelOp[@PhysicalOp="Table Scan"]') = 1 
        THEN 'Table Scan Detected'
        WHEN qp.QueryPlan.exist('//RelOp[@PhysicalOp="Nested Loops"]') = 1 
        THEN 'Nested Loops with High Row Count'
        WHEN qp.QueryPlan.exist('//MemoryGrantWarning') = 1 
        THEN 'Memory Grant Warning'
        WHEN qp.QueryPlan.exist('//MissingIndexGroup') = 1 
        THEN 'Missing Index'
        ELSE NULL
    END as ProblemType,
    qp.CaptureDate
FROM monitor.QueryPlans qp;

-- Procedure para análise detalhada
CREATE PROCEDURE monitor.usp_AnalyzeQueryProblems
    @QueryText nvarchar(max)
AS
BEGIN
    -- Análise de espera
    SELECT TOP 10
        wait_type,
        waiting_tasks_count,
        wait_time_ms,
        max_wait_time_ms,
        signal_wait_time_ms
    FROM sys.dm_os_wait_stats
    WHERE waiting_tasks_count > 0
    ORDER BY wait_time_ms DESC;

    -- Análise de IO
    SELECT TOP 10
        DB_NAME(database_id) as DatabaseName,
        file_id,
        num_of_reads,
        num_of_writes,
        io_stall_read_ms,
        io_stall_write_ms
    FROM sys.dm_io_virtual_file_stats(NULL, NULL)
    ORDER BY (io_stall_read_ms + io_stall_write_ms) DESC;
END;
```

### 4.2 Soluções Comuns
```sql
-- Procedure para aplicar correções comuns
CREATE PROCEDURE maintenance.usp_ApplyQueryFixes
    @TableName nvarchar(128)
AS
BEGIN
    -- Atualizar estatísticas
    EXEC maintenance.usp_UpdateStatistics 
        @TableName = @TableName;

    -- Rebuild índices fragmentados
    ALTER INDEX ALL ON @TableName REBUILD;

    -- Criar índices missing
    SELECT 
        'CREATE INDEX ' + 
        'IX_' + OBJECT_NAME(d.object_id) + '_' +
        REPLACE(REPLACE(d.equality_columns, '[', ''), ']', '') +
        ' ON ' + d.statement +
        ' (' + d.equality_columns + 
        CASE 
            WHEN d.inequality_columns IS NOT NULL 
            THEN ',' + d.inequality_columns 
            ELSE '' 
        END + ')'
    FROM sys.dm_db_missing_index_details d
    WHERE d.statement = @TableName;
END;
```

## 5. Técnicas de Otimização

### 5.1 Reescrita de Queries
```sql
-- Exemplos de reescrita
-- Original
SELECT 
    c.CustomerName,
    COUNT(DISTINCT o.OrderID) as OrderCount
FROM dim.Customer c
LEFT JOIN fact.Orders o ON c.Customer_SK = o.Customer_SK
GROUP BY c.CustomerName;

-- Otimizado
WITH CustomerOrders AS (
    SELECT 
        Customer_SK,
        COUNT(DISTINCT OrderID) as OrderCount
    FROM fact.Orders
    GROUP BY Customer_SK
)
SELECT 
    c.CustomerName,
    ISNULL(co.OrderCount, 0) as OrderCount
FROM dim.Customer c
LEFT JOIN CustomerOrders co ON c.Customer_SK = co.Customer_SK;
```

### 5.2 Materialização de Dados
```sql
-- Criar view materializada
CREATE PROCEDURE etl.usp_MaterializeData
    @ViewName nvarchar(128)
AS
BEGIN
    DECLARE @SQL nvarchar(max);
    
    -- Criar tabela materializada
    SET @SQL = '
    CREATE TABLE ' + @ViewName + '_Materialized
    WITH (DISTRIBUTION = HASH(Date_SK))
    AS
    SELECT *
    FROM ' + @ViewName + '
    ';
    
    EXEC sp_executesql @SQL;

    -- Criar índices na tabela materializada
    SET @SQL = '
    CREATE CLUSTERED COLUMNSTORE INDEX CCI_' + 
        @ViewName + '_Materialized
    ON ' + @ViewName + '_Materialized
    ';
    
    EXEC sp_executesql @SQL;
END;
```

## Exercícios Práticos

### Exercício 1: Análise de Planos
1. Analisar planos de execução
2. Identificar gargalos
3. Propor melhorias

### Exercício 2: Otimização
1. Reescrever queries
2. Implementar índices
3. Atualizar estatísticas

### Exercício 3: Monitoramento
1. Capturar métricas
2. Analisar performance
3. Documentar melhorias

## Material Complementar

### Documentação
- Query Performance Guidelines
- Statistics Management
- Optimization Techniques

### Scripts Úteis
- Scripts de análise
- Templates de otimização
- Queries de diagnóstico

## Próxima Aula
Na próxima aula, abordaremos Data Marts e Agregações, incluindo design e otimização.

## Avaliação
1. Quiz sobre otimização
2. Projeto prático
3. Análise de performance