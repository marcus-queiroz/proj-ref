# Aula 22 - Data Marts e Agregações

## Objetivos da Aula
- Compreender design de data marts
- Implementar views materializadas
- Desenvolver tabelas de agregação
- Criar estratégias de refresh
- Otimizar para BI

## 1. Design de Data Marts

### 1.1 Estrutura Básica
```sql
-- Schema para Data Marts
CREATE SCHEMA mart;
GO

-- Views dependentes
CREATE VIEW mart.vw_SalesAnalysis
AS
WITH SalesSummary AS (
    SELECT 
        d.Year,
        d.Month,
        p.CategoryName,
        c.CustomerSegment,
        SUM(f.Amount) as TotalAmount,
        COUNT(DISTINCT f.Customer_SK) as CustomerCount,
        COUNT(DISTINCT f.Order_SK) as OrderCount
    FROM fact.Sales f
    JOIN dim.Date d ON f.Date_SK = d.Date_SK
    JOIN dim.Product p ON f.Product_SK = p.Product_SK
    JOIN dim.Customer c ON f.Customer_SK = c.Customer_SK
    GROUP BY
        d.Year,
        d.Month,
        p.CategoryName,
        c.CustomerSegment
)
SELECT 
    Year,
    Month,
    CategoryName,
    CustomerSegment,
    TotalAmount,
    CustomerCount,
    OrderCount,
    TotalAmount / NULLIF(CustomerCount, 0) as AvgCustomerAmount,
    TotalAmount / NULLIF(OrderCount, 0) as AvgOrderAmount
FROM SalesSummary;
```

### 1.2 Implementação de Data Marts
```sql
-- Tabela de controle de Data Marts
CREATE TABLE meta.DataMartControl (
    DataMart_SK bigint IDENTITY(1,1),
    DataMartName varchar(100) NOT NULL,
    SourceView varchar(100) NOT NULL,
    RefreshFrequency varchar(20) NOT NULL,
    LastRefresh datetime2 NULL,
    NextRefresh datetime2 NULL,
    Status varchar(20) NOT NULL,
    CONSTRAINT PK_DataMartControl PRIMARY KEY (DataMart_SK)
);

-- Procedure para criar Data Mart
CREATE PROCEDURE mart.usp_CreateDataMart
    @DataMartName varchar(100),
    @SourceView varchar(100),
    @RefreshFrequency varchar(20)
AS
BEGIN
    DECLARE @SQL nvarchar(max);

    -- Criar tabela do Data Mart
    SET @SQL = '
    CREATE TABLE mart.' + @DataMartName + '
    WITH (DISTRIBUTION = HASH(Date_SK))
    AS
    SELECT *
    FROM ' + @SourceView + '
    WHERE 1 = 2';

    EXEC sp_executesql @SQL;

    -- Registrar Data Mart
    INSERT INTO meta.DataMartControl (
        DataMartName,
        SourceView,
        RefreshFrequency,
        Status
    )
    VALUES (
        @DataMartName,
        @SourceView,
        @RefreshFrequency,
        'Created'
    );
END;
```

## 2. Views Materializadas

### 2.1 Criação e Manutenção
```sql
-- Procedure para criar view materializada
CREATE PROCEDURE mart.usp_CreateMaterializedView
    @ViewName varchar(100),
    @Query nvarchar(max)
AS
BEGIN
    DECLARE @SQL nvarchar(max);

    -- Criar tabela base
    SET @SQL = '
    CREATE TABLE mart.' + @ViewName + '_MV (
        MV_ID bigint IDENTITY(1,1),
        LastRefresh datetime2 NOT NULL,
        CONSTRAINT PK_' + @ViewName + '_MV PRIMARY KEY (MV_ID)
    );';

    EXEC sp_executesql @SQL;

    -- Adicionar colunas da query
    SET @SQL = '
    INSERT INTO mart.' + @ViewName + '_MV
    SELECT
        GETDATE() as LastRefresh,
        *
    FROM (' + @Query + ') q;';

    EXEC sp_executesql @SQL;

    -- Criar índices
    SET @SQL = '
    CREATE CLUSTERED COLUMNSTORE INDEX CCI_' + @ViewName + '_MV
    ON mart.' + @ViewName + '_MV;';

    EXEC sp_executesql @SQL;
END;
```

### 2.2 Refresh Strategy
```sql
-- Procedure para refresh incremental
CREATE PROCEDURE mart.usp_RefreshMaterializedView
    @ViewName varchar(100),
    @IncrementalColumn varchar(100)
AS
BEGIN
    DECLARE @SQL nvarchar(max);
    DECLARE @LastRefresh datetime2;

    -- Obter última atualização
    SELECT @LastRefresh = MAX(LastRefresh)
    FROM mart.DataMartControl
    WHERE DataMartName = @ViewName;

    -- Refresh incremental
    SET @SQL = '
    INSERT INTO mart.' + @ViewName + '_MV
    SELECT
        GETDATE() as LastRefresh,
        *
    FROM (' + @Query + ') q
    WHERE ' + @IncrementalColumn + ' > @LastRefresh;';

    EXEC sp_executesql @SQL, 
        N'@LastRefresh datetime2',
        @LastRefresh;

    -- Atualizar controle
    UPDATE meta.DataMartControl
    SET 
        LastRefresh = GETDATE(),
        NextRefresh = DATEADD(DAY, 
            CASE RefreshFrequency
                WHEN 'Daily' THEN 1
                WHEN 'Weekly' THEN 7
                WHEN 'Monthly' THEN 30
            END,
            GETDATE()
        )
    WHERE DataMartName = @ViewName;
END;
```

## 3. Tabelas de Agregação

### 3.1 Design de Agregações
```sql
-- Criar tabela de agregação
CREATE PROCEDURE mart.usp_CreateAggregation
    @AggregateName varchar(100),
    @GrainColumns varchar(max),
    @MeasureColumns varchar(max)
AS
BEGIN
    DECLARE @SQL nvarchar(max);

    -- Criar tabela de agregação
    SET @SQL = '
    CREATE TABLE mart.' + @AggregateName + ' (
        Aggregate_SK bigint IDENTITY(1,1),
        ' + @GrainColumns + ',
        ' + @MeasureColumns + ',
        LastRefresh datetime2 NOT NULL,
        CONSTRAINT PK_' + @AggregateName + 
        ' PRIMARY KEY (Aggregate_SK)
    );';

    EXEC sp_executesql @SQL;

    -- Criar índices apropriados
    SET @SQL = '
    CREATE CLUSTERED COLUMNSTORE INDEX CCI_' + 
        @AggregateName + '
    ON mart.' + @AggregateName + ';';

    EXEC sp_executesql @SQL;
END;
```

### 3.2 Processo de Agregação
```sql
-- Procedure para processo de agregação
CREATE PROCEDURE mart.usp_ProcessAggregation
    @AggregateName varchar(100),
    @SourceTable varchar(100),
    @GrainColumns varchar(max),
    @AggregateColumns varchar(max)
AS
BEGIN
    DECLARE @SQL nvarchar(max);

    -- Truncar dados existentes
    SET @SQL = '
    TRUNCATE TABLE mart.' + @AggregateName;
    
    EXEC sp_executesql @SQL;

    -- Inserir dados agregados
    SET @SQL = '
    INSERT INTO mart.' + @AggregateName + '
    SELECT
        ' + @GrainColumns + ',
        ' + @AggregateColumns + ',
        GETDATE()
    FROM ' + @SourceTable + '
    GROUP BY ' + @GrainColumns;

    EXEC sp_executesql @SQL;
END;
```

## 4. Otimização para BI

### 4.1 Índices para BI
```sql
-- Procedure para otimizar para BI
CREATE PROCEDURE mart.usp_OptimizeForBI
    @TableName varchar(100)
AS
BEGIN
    DECLARE @SQL nvarchar(max);

    -- Criar índice columnstore
    SET @SQL = '
    CREATE CLUSTERED COLUMNSTORE INDEX CCI_' + 
        @TableName + '
    ON mart.' + @TableName + ';';

    EXEC sp_executesql @SQL;

    -- Criar índices complementares
    SET @SQL = '
    CREATE NONCLUSTERED INDEX IX_' + @TableName + '_Date
    ON mart.' + @TableName + '(Date_SK)
    INCLUDE (Amount);';

    EXEC sp_executesql @SQL;

    -- Atualizar estatísticas
    SET @SQL = '
    UPDATE STATISTICS mart.' + @TableName + '
    WITH FULLSCAN;';

    EXEC sp_executesql @SQL;
END;
```

### 4.2 Particionamento para BI
```sql
-- Procedure para particionar Data Mart
CREATE PROCEDURE mart.usp_PartitionDataMart
    @DataMartName varchar(100),
    @PartitionColumn varchar(100),
    @PartitionFunction varchar(100)
AS
BEGIN
    DECLARE @SQL nvarchar(max);

    -- Criar função de particionamento
    SET @SQL = '
    CREATE PARTITION FUNCTION ' + @PartitionFunction + 
    ' (int)
    AS RANGE RIGHT FOR VALUES (
        20240101, 20240201, 20240301, 20240401
    );';

    EXEC sp_executesql @SQL;

    -- Criar esquema de particionamento
    SET @SQL = '
    CREATE PARTITION SCHEME PS_' + @DataMartName + '
    AS PARTITION ' + @PartitionFunction + '
    ALL TO ([PRIMARY]);';

    EXEC sp_executesql @SQL;

    -- Aplicar particionamento
    SET @SQL = '
    CREATE CLUSTERED COLUMNSTORE INDEX CCI_' + 
        @DataMartName + '
    ON mart.' + @DataMartName + '
    WITH (DROP_EXISTING = ON)
    ON PS_' + @DataMartName + '(' + @PartitionColumn + ');';

    EXEC sp_executesql @SQL;
END;
```

## 5. Manutenção e Monitoramento

### 5.1 Processos de Manutenção
```sql
-- Procedure para manutenção de Data Marts
CREATE PROCEDURE mart.usp_MaintainDataMarts
AS
BEGIN
    -- Atualizar Data Marts que precisam de refresh
    DECLARE @DataMartName varchar(100);
    DECLARE @SourceView varchar(100);
    
    DECLARE mart_cursor CURSOR FOR
    SELECT 
        DataMartName,
        SourceView
    FROM meta.DataMartControl
    WHERE NextRefresh <= GETDATE()
    AND Status = 'Active';

    OPEN mart_cursor;
    FETCH NEXT FROM mart_cursor 
    INTO @DataMartName, @SourceView;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Refresh Data Mart
        EXEC mart.usp_RefreshMaterializedView
            @ViewName = @DataMartName,
            @IncrementalColumn = 'LastModifiedDate';

        FETCH NEXT FROM mart_cursor 
        INTO @DataMartName, @SourceView;
    END;

    CLOSE mart_cursor;
    DEALLOCATE mart_cursor;

    -- Manutenção de índices
    EXEC mart.usp_MaintainIndexes;

    -- Atualizar estatísticas
    EXEC mart.usp_UpdateStatistics;
END;
```

### 5.2 Monitoramento
```sql
-- View para monitoramento
CREATE VIEW monitor.DataMartStatus
AS
SELECT 
    dm.DataMartName,
    dm.LastRefresh,
    dm.NextRefresh,
    dm.Status,
    p.rows as RowCount,
    p.total_pages * 8 as TotalSpaceKB,
    CASE 
        WHEN dm.NextRefresh <= GETDATE() THEN 'Needs Refresh'
        ELSE 'Up to Date'
    END as RefreshStatus
FROM meta.DataMartControl dm
JOIN sys.partitions p 
    ON OBJECT_ID('mart.' + dm.DataMartName) = p.object_id
WHERE p.index_id <= 1;
```

## Exercícios Práticos

### Exercício 1: Data Marts
1. Criar Data Mart de vendas
2. Implementar agregações
3. Otimizar performance

### Exercício 2: Views Materializadas
1. Criar views materializadas
2. Implementar refresh
3. Monitorar performance

### Exercício 3: Manutenção
1. Desenvolver rotinas de manutenção
2. Implementar monitoramento
3. Otimizar processos

## Material Complementar

### Documentação
- Data Mart Design Patterns
- Materialized View Best Practices
- Optimization Guidelines

### Scripts Úteis
- Scripts de criação
- Procedures de manutenção
- Queries de monitoramento

## Próxima Aula
Na próxima aula, abordaremos Arquitetura e Escalabilidade, incluindo configurações físicas e otimizações de hardware.

## Avaliação
1. Quiz sobre Data Marts
2. Projeto prático
3. Análise de performance