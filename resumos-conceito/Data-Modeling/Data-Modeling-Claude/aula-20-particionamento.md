# Aula 20 - Particionamento

## Objetivos da Aula
- Compreender estratégias de particionamento
- Implementar partition switching
- Desenvolver alinhamento de índices
- Criar manutenção de partições
- Implementar arquivamento de dados

## 1. Estratégias de Particionamento

### 1.1 Configuração Básica
```sql
-- Criar função de particionamento
CREATE PARTITION FUNCTION PF_Date (int)
AS RANGE RIGHT FOR VALUES 
(20240101, 20240201, 20240301, 20240401);

-- Criar esquema de particionamento
CREATE PARTITION SCHEME PS_Date
AS PARTITION PF_Date
ALL TO ([PRIMARY]);

-- Criar tabela particionada
CREATE TABLE fact.SalesPartitioned (
    Sale_SK bigint IDENTITY(1,1),
    Date_SK int NOT NULL,
    Product_SK bigint NOT NULL,
    Customer_SK bigint NOT NULL,
    Quantity int NOT NULL,
    Amount decimal(18,2) NOT NULL,
    CONSTRAINT PK_SalesPartitioned 
        PRIMARY KEY CLUSTERED (Date_SK, Sale_SK)
) ON PS_Date(Date_SK);
```

### 1.2 Metadados de Particionamento
```sql
-- View para informações de particionamento
CREATE VIEW meta.PartitionInfo
AS
SELECT 
    OBJECT_SCHEMA_NAME(p.object_id) as SchemaName,
    OBJECT_NAME(p.object_id) as TableName,
    p.partition_number,
    p.rows,
    au.total_pages,
    au.used_pages,
    CASE pf.boundary_value_on_right
        WHEN 1 THEN 'RIGHT'
        ELSE 'LEFT'
    END as RangeType,
    prv.value as BoundaryValue,
    fg.name as FileGroupName
FROM sys.partitions p
JOIN sys.allocation_units au 
    ON p.partition_id = au.container_id
JOIN sys.indexes i 
    ON p.object_id = i.object_id 
    AND p.index_id = i.index_id
JOIN sys.partition_schemes ps 
    ON i.data_space_id = ps.data_space_id
JOIN sys.partition_functions pf 
    ON ps.function_id = pf.function_id
LEFT JOIN sys.partition_range_values prv 
    ON pf.function_id = prv.function_id 
    AND p.partition_number = prv.boundary_id
JOIN sys.filegroups fg 
    ON au.data_space_id = fg.data_space_id
WHERE i.index_id <= 1;
```

## 2. Partition Switching

### 2.1 Preparação para Switching
```sql
-- Criar tabela staging com mesma estrutura
CREATE TABLE staging.Sales (
    Sale_SK bigint IDENTITY(1,1),
    Date_SK int NOT NULL,
    Product_SK bigint NOT NULL,
    Customer_SK bigint NOT NULL,
    Quantity int NOT NULL,
    Amount decimal(18,2) NOT NULL,
    CONSTRAINT PK_StagingSales 
        PRIMARY KEY CLUSTERED (Date_SK, Sale_SK)
) ON [PRIMARY];

-- Procedure para switching
CREATE PROCEDURE etl.usp_SwitchPartition
    @SourceTable nvarchar(128),
    @TargetTable nvarchar(128),
    @PartitionNumber int
AS
BEGIN
    DECLARE @SQL nvarchar(max);
    
    SET @SQL = '
    ALTER TABLE ' + @SourceTable + '
    SWITCH PARTITION ' + CAST(@PartitionNumber as varchar(10)) + '
    TO ' + @TargetTable + ' PARTITION ' + 
    CAST(@PartitionNumber as varchar(10)) + ';';
    
    EXEC sp_executesql @SQL;
END;
```

### 2.2 Processo de Carga com Switching
```sql
-- Procedure para carga incremental com switching
CREATE PROCEDURE etl.usp_LoadSalesWithSwitching
    @Date_SK int
AS
BEGIN
    DECLARE @PartitionNumber int;
    
    -- Identificar partição
    SELECT @PartitionNumber = partition_number
    FROM meta.PartitionInfo
    WHERE TableName = 'SalesPartitioned'
    AND @Date_SK >= BoundaryValue;

    -- Truncar staging
    TRUNCATE TABLE staging.Sales;

    -- Carregar dados novos
    INSERT INTO staging.Sales (
        Date_SK,
        Product_SK,
        Customer_SK,
        Quantity,
        Amount
    )
    SELECT 
        Date_SK,
        Product_SK,
        Customer_SK,
        Quantity,
        Amount
    FROM source.Sales
    WHERE Date_SK = @Date_SK;

    -- Executar switch
    EXEC etl.usp_SwitchPartition
        @SourceTable = 'staging.Sales',
        @TargetTable = 'fact.SalesPartitioned',
        @PartitionNumber = @PartitionNumber;
END;
```

## 3. Alinhamento de Índices

### 3.1 Índices Alinhados
```sql
-- Criar índice alinhado
CREATE NONCLUSTERED INDEX IX_SalesPartitioned_Product
ON fact.SalesPartitioned(Product_SK, Date_SK)
INCLUDE (Quantity, Amount)
ON PS_Date(Date_SK);

-- Criar índice columnstore alinhado
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_SalesPartitioned
ON fact.SalesPartitioned
(
    Date_SK,
    Product_SK,
    Customer_SK,
    Quantity,
    Amount
)
ON PS_Date(Date_SK);
```

### 3.2 Manutenção de Índices Particionados
```sql
-- Procedure para manutenção de índices por partição
CREATE PROCEDURE maintenance.usp_MaintainPartitionedIndexes
    @TableName nvarchar(128),
    @PartitionNumber int = NULL
AS
BEGIN
    DECLARE @SQL nvarchar(max);
    DECLARE @IndexName nvarchar(128);
    
    -- Cursor para índices particionados
    DECLARE index_cursor CURSOR FOR
    SELECT i.name
    FROM sys.indexes i
    JOIN sys.objects o ON i.object_id = o.object_id
    WHERE o.name = @TableName
    AND i.data_space_id IN (
        SELECT data_space_id
        FROM sys.partition_schemes
    );

    OPEN index_cursor;
    FETCH NEXT FROM index_cursor INTO @IndexName;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Rebuild específico da partição
        SET @SQL = 'ALTER INDEX ' + QUOTENAME(@IndexName) + 
                  ' ON ' + @TableName + 
                  ' REBUILD PARTITION = ' + 
                  CAST(ISNULL(@PartitionNumber, 'ALL') as varchar(10)) +
                  ' WITH (ONLINE = ON)';
        
        EXEC sp_executesql @SQL;

        FETCH NEXT FROM index_cursor INTO @IndexName;
    END;

    CLOSE index_cursor;
    DEALLOCATE index_cursor;
END;
```

## 4. Manutenção de Partições

### 4.1 Split e Merge
```sql
-- Procedure para split de partição
CREATE PROCEDURE maintenance.usp_SplitPartition
    @Value sql_variant,
    @PartitionFunctionName nvarchar(128)
AS
BEGIN
    DECLARE @SQL nvarchar(max);
    
    SET @SQL = 'ALTER PARTITION FUNCTION ' + 
               QUOTENAME(@PartitionFunctionName) + 
               '() SPLIT RANGE (@Value)';
    
    EXEC sp_executesql @SQL, 
        N'@Value sql_variant', 
        @Value;
END;

-- Procedure para merge de partição
CREATE PROCEDURE maintenance.usp_MergePartition
    @Value sql_variant,
    @PartitionFunctionName nvarchar(128)
AS
BEGIN
    DECLARE @SQL nvarchar(max);
    
    SET @SQL = 'ALTER PARTITION FUNCTION ' + 
               QUOTENAME(@PartitionFunctionName) + 
               '() MERGE RANGE (@Value)';
    
    EXEC sp_executesql @SQL, 
        N'@Value sql_variant', 
        @Value;
END;
```

### 4.2 Archiving
```sql
-- Tabela de arquivo
CREATE TABLE archive.Sales (
    Sale_SK bigint NOT NULL,
    Date_SK int NOT NULL,
    Product_SK bigint NOT NULL,
    Customer_SK bigint NOT NULL,
    Quantity int NOT NULL,
    Amount decimal(18,2) NOT NULL,
    ArchiveDate datetime2 NOT NULL DEFAULT(GETDATE()),
    CONSTRAINT PK_ArchiveSales 
        PRIMARY KEY CLUSTERED (Date_SK, Sale_SK)
);

-- Procedure para arquivamento
CREATE PROCEDURE maintenance.usp_ArchivePartition
    @PartitionNumber int
AS
BEGIN
    -- Criar tabela temporária para switching
    CREATE TABLE staging.SalesArchive (
        Sale_SK bigint NOT NULL,
        Date_SK int NOT NULL,
        Product_SK bigint NOT NULL,
        Customer_SK bigint NOT NULL,
        Quantity int NOT NULL,
        Amount decimal(18,2) NOT NULL,
        CONSTRAINT PK_StagingSalesArchive 
            PRIMARY KEY CLUSTERED (Date_SK, Sale_SK)
    );

    -- Switch para staging
    ALTER TABLE fact.SalesPartitioned
    SWITCH PARTITION @PartitionNumber 
    TO staging.SalesArchive;

    -- Mover para arquivo
    INSERT INTO archive.Sales (
        Sale_SK,
        Date_SK,
        Product_SK,
        Customer_SK,
        Quantity,
        Amount
    )
    SELECT 
        Sale_SK,
        Date_SK,
        Product_SK,
        Customer_SK,
        Quantity,
        Amount
    FROM staging.SalesArchive;

    -- Limpeza
    DROP TABLE staging.SalesArchive;
END;
```

## 5. Monitoramento

### 5.1 Análise de Partições
```sql
-- View para análise de partições
CREATE VIEW monitor.PartitionStats
AS
SELECT 
    OBJECT_SCHEMA_NAME(p.object_id) as SchemaName,
    OBJECT_NAME(p.object_id) as TableName,
    p.partition_number,
    p.rows as RowCount,
    au.total_pages * 8 as TotalSpaceKB,
    au.used_pages * 8 as UsedSpaceKB,
    CASE 
        WHEN p.rows > 0 
        THEN CAST(au.used_pages * 8.0 / p.rows as decimal(10,2))
        ELSE 0 
    END as AvgRowSizeKB
FROM sys.partitions p
JOIN sys.allocation_units au 
    ON p.partition_id = au.container_id
WHERE p.index_id <= 1;

-- Procedure para análise de crescimento
CREATE PROCEDURE monitor.usp_AnalyzePartitionGrowth
AS
BEGIN
    WITH PartitionTrend AS (
        SELECT 
            SchemaName,
            TableName,
            partition_number,
            RowCount,
            TotalSpaceKB,
            LAG(RowCount) OVER (
                PARTITION BY SchemaName, TableName 
                ORDER BY partition_number
            ) as PreviousRowCount,
            LAG(TotalSpaceKB) OVER (
                PARTITION BY SchemaName, TableName 
                ORDER BY partition_number
            ) as PreviousSpaceKB
        FROM monitor.PartitionStats
    )
    SELECT 
        SchemaName,
        TableName,
        partition_number,
        RowCount,
        TotalSpaceKB,
        CASE 
            WHEN PreviousRowCount > 0 
            THEN (RowCount - PreviousRowCount) * 100.0 / PreviousRowCount
            ELSE 0 
        END as RowGrowthPercent,
        CASE 
            WHEN PreviousSpaceKB > 0 
            THEN (TotalSpaceKB - PreviousSpaceKB) * 100.0 / PreviousSpaceKB
            ELSE 0 
        END as SpaceGrowthPercent
    FROM PartitionTrend;
END;
```

## Exercícios Práticos

### Exercício 1: Implementação
1. Criar esquema de particionamento
2. Implementar sliding window
3. Configurar processo de carga

### Exercício 2: Manutenção
1. Desenvolver rotinas de arquivamento
2. Implementar processo de split/merge
3. Criar jobs de manutenção

### Exercício 3: Monitoramento
1. Implementar monitoramento
2. Analisar crescimento
3. Otimizar performance

## Material Complementar

### Documentação
- Partition Design Guidelines
- Maintenance Best Practices
- Performance Optimization

### Scripts Úteis
- Scripts de análise
- Procedures de manutenção
- Queries de monitoramento

## Próxima Aula
Na próxima aula, abordaremos Otimização de Consultas, incluindo análise de planos de execução e otimização de performance.

## Avaliação
1. Quiz sobre particionamento
2. Projeto prático
3. Análise de implementação