# Aula 19 - Indexação para DW

## Objetivos da Aula
- Compreender tipos de índices para DW
- Implementar índices columnstore
- Trabalhar com índices filtrados
- Criar índices cobrindo (covering indexes)
- Desenvolver estratégias de manutenção

## 1. Tipos de Índices

### 1.1 Índices Tradicionais
```sql
-- Índice clustered tradicional
CREATE CLUSTERED INDEX IX_Fact_Date
ON fact.Sales(Date_SK);

-- Índice non-clustered com include
CREATE NONCLUSTERED INDEX IX_Fact_Product
ON fact.Sales(Product_SK)
INCLUDE (Quantity, Amount);

-- Índice composto
CREATE NONCLUSTERED INDEX IX_Fact_DateProduct
ON fact.Sales(Date_SK, Product_SK)
INCLUDE (Quantity, Amount);
```

### 1.2 Índices Columnstore
```sql
-- Índice columnstore clustered
CREATE CLUSTERED COLUMNSTORE INDEX CCI_FactSales
ON fact.Sales;

-- Índice columnstore não clustered
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactSales
ON fact.Sales
(
    Date_SK,
    Product_SK,
    Customer_SK,
    Quantity,
    Amount
);

-- Índice columnstore filtrado
CREATE NONCLUSTERED COLUMNSTORE INDEX NCCI_FactSales_Current
ON fact.Sales
(
    Date_SK,
    Product_SK,
    Customer_SK,
    Quantity,
    Amount
)
WHERE Date_SK >= 20240101;
```

## 2. Estratégias de Indexação

### 2.1 Dimensões
```sql
-- Estratégia para dimensões pequenas
CREATE TABLE dim.SmallDimension (
    Dimension_SK bigint IDENTITY(1,1),
    Dimension_BK varchar(20) NOT NULL,
    Description nvarchar(100) NOT NULL,
    CONSTRAINT PK_SmallDimension PRIMARY KEY CLUSTERED (Dimension_SK)
);

CREATE NONCLUSTERED INDEX IX_SmallDimension_BK
ON dim.SmallDimension(Dimension_BK);

-- Estratégia para dimensões grandes
CREATE TABLE dim.LargeDimension (
    Dimension_SK bigint IDENTITY(1,1),
    Dimension_BK varchar(20) NOT NULL,
    Description nvarchar(100) NOT NULL,
    ValidFrom datetime2 NOT NULL,
    ValidTo datetime2 NULL,
    Current_FLG bit NOT NULL,
    CONSTRAINT PK_LargeDimension PRIMARY KEY CLUSTERED (Dimension_SK)
);

CREATE NONCLUSTERED INDEX IX_LargeDimension_BK
ON dim.LargeDimension(Dimension_BK, Current_FLG)
INCLUDE (Description);

CREATE NONCLUSTERED INDEX IX_LargeDimension_Date
ON dim.LargeDimension(ValidFrom, ValidTo)
WHERE Current_FLG = 1;
```

### 2.2 Fatos
```sql
-- Estratégia para fatos
CREATE TABLE fact.TransactionFact (
    Transaction_SK bigint IDENTITY(1,1),
    Date_SK int NOT NULL,
    Product_SK bigint NOT NULL,
    Customer_SK bigint NOT NULL,
    Quantity int NOT NULL,
    Amount decimal(18,2) NOT NULL,
    CONSTRAINT PK_TransactionFact 
        PRIMARY KEY NONCLUSTERED (Transaction_SK)
);

-- Índice columnstore principal
CREATE CLUSTERED COLUMNSTORE INDEX CCI_TransactionFact
ON fact.TransactionFact;

-- Índices auxiliares para queries específicas
CREATE NONCLUSTERED INDEX IX_TransactionFact_Date
ON fact.TransactionFact(Date_SK)
INCLUDE (Quantity, Amount);

CREATE NONCLUSTERED INDEX IX_TransactionFact_Product
ON fact.TransactionFact(Product_SK)
INCLUDE (Date_SK, Quantity, Amount);
```

## 3. Otimização de Índices

### 3.1 Análise de Fragmentação
```sql
-- Procedure para análise de fragmentação
CREATE PROCEDURE maintenance.usp_AnalyzeIndexFragmentation
AS
BEGIN
    SELECT 
        OBJECT_SCHEMA_NAME(i.object_id) as SchemaName,
        OBJECT_NAME(i.object_id) as TableName,
        i.name as IndexName,
        ips.avg_fragmentation_in_percent,
        ips.page_count,
        ips.record_count,
        CASE
            WHEN ips.avg_fragmentation_in_percent > 30 THEN 'Rebuild'
            WHEN ips.avg_fragmentation_in_percent > 10 THEN 'Reorganize'
            ELSE 'None'
        END as RecommendedAction
    FROM sys.dm_db_index_physical_stats(
        DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
    JOIN sys.indexes i ON ips.object_id = i.object_id 
        AND ips.index_id = i.index_id
    WHERE ips.page_count > 1000
    ORDER BY ips.avg_fragmentation_in_percent DESC;
END;
```

### 3.2 Manutenção Automática
```sql
-- Procedure para manutenção de índices
CREATE PROCEDURE maintenance.usp_MaintainIndexes
    @TableName nvarchar(128),
    @FragmentationLimit decimal(5,2) = 30.0,
    @PageCountLimit int = 1000
AS
BEGIN
    DECLARE @SQL nvarchar(max);
    DECLARE @IndexName nvarchar(128);
    DECLARE @Fragmentation decimal(5,2);
    
    -- Cursor para índices fragmentados
    DECLARE index_cursor CURSOR FOR
    SELECT 
        i.name,
        ips.avg_fragmentation_in_percent
    FROM sys.dm_db_index_physical_stats(
        DB_ID(), OBJECT_ID(@TableName), NULL, NULL, 'LIMITED') ips
    JOIN sys.indexes i ON ips.object_id = i.object_id 
        AND ips.index_id = i.index_id
    WHERE ips.page_count > @PageCountLimit;

    OPEN index_cursor;
    FETCH NEXT FROM index_cursor 
    INTO @IndexName, @Fragmentation;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        IF @Fragmentation > @FragmentationLimit
            SET @SQL = 'ALTER INDEX ' + QUOTENAME(@IndexName) + 
                      ' ON ' + @TableName + ' REBUILD;'
        ELSE IF @Fragmentation > 10.0
            SET @SQL = 'ALTER INDEX ' + QUOTENAME(@IndexName) + 
                      ' ON ' + @TableName + ' REORGANIZE;'
        
        IF @SQL IS NOT NULL
            EXEC sp_executesql @SQL;

        FETCH NEXT FROM index_cursor 
        INTO @IndexName, @Fragmentation;
    END;

    CLOSE index_cursor;
    DEALLOCATE index_cursor;
END;
```

## 4. Monitoramento de Índices

### 4.1 Análise de Uso
```sql
-- View para análise de uso de índices
CREATE VIEW monitor.IndexUsage
AS
SELECT 
    OBJECT_SCHEMA_NAME(i.object_id) as SchemaName,
    OBJECT_NAME(i.object_id) as TableName,
    i.name as IndexName,
    i.type_desc as IndexType,
    ius.user_seeks,
    ius.user_scans,
    ius.user_lookups,
    ius.user_updates,
    ius.last_user_seek,
    ius.last_user_scan,
    ius.last_user_lookup,
    ius.last_user_update
FROM sys.indexes i
LEFT JOIN sys.dm_db_index_usage_stats ius 
    ON i.object_id = ius.object_id 
    AND i.index_id = ius.index_id
WHERE OBJECTPROPERTY(i.object_id, 'IsMSShipped') = 0;

-- Procedure para identificar índices não utilizados
CREATE PROCEDURE monitor.usp_UnusedIndexes
AS
BEGIN
    SELECT 
        SchemaName,
        TableName,
        IndexName,
        IndexType
    FROM monitor.IndexUsage
    WHERE user_seeks = 0 
        AND user_scans = 0 
        AND user_lookups = 0
        AND user_updates > 0;
END;
```

### 4.2 Recomendações
```sql
-- Procedure para recomendações de índices
CREATE PROCEDURE monitor.usp_IndexRecommendations
AS
BEGIN
    -- Missing indexes
    SELECT 
        OBJECT_SCHEMA_NAME(d.object_id) as SchemaName,
        OBJECT_NAME(d.object_id) as TableName,
        'CREATE INDEX [IX_' + 
        OBJECT_NAME(d.object_id) + '_' +
        REPLACE(REPLACE(REPLACE(
            d.equality_columns + '_' + d.inequality_columns,
            '[', ''), ']', ''), ', ', '_') + '] ON ' +
        d.statement + ' (' + 
        ISNULL(d.equality_columns, '') +
        CASE 
            WHEN d.equality_columns IS NOT NULL 
                AND d.inequality_columns IS NOT NULL 
            THEN ',' 
            ELSE '' 
        END +
        ISNULL(d.inequality_columns, '') + 
        ') ' +
        'INCLUDE (' + ISNULL(d.included_columns, '') + ')' 
        as CreateIndexStatement,
        s.user_seeks,
        s.user_scans,
        s.avg_total_user_cost,
        s.avg_user_impact
    FROM sys.dm_db_missing_index_details d
    JOIN sys.dm_db_missing_index_groups g 
        ON d.index_handle = g.index_handle
    JOIN sys.dm_db_missing_index_group_stats s 
        ON g.index_group_handle = s.group_handle
    WHERE d.database_id = DB_ID()
    ORDER BY (s.user_seeks + s.user_scans) * s.avg_user_impact DESC;
END;
```

## Exercícios Práticos

### Exercício 1: Implementação
1. Criar esquema de indexação para:
   - Dimensões de diferentes tamanhos
   - Fatos com diferentes características
   - Tabelas de staging

### Exercício 2: Otimização
1. Analisar fragmentação
2. Implementar manutenção
3. Monitorar performance

### Exercício 3: Análise
1. Identificar índices não utilizados
2. Avaliar missing indexes
3. Criar recomendações

## Material Complementar

### Documentação
- Index Design Guidelines
- Columnstore Index Best Practices
- Maintenance Strategies

### Scripts Úteis
- Scripts de análise
- Procedures de manutenção
- Queries de monitoramento

## Próxima Aula
Na próxima aula, abordaremos Particionamento, incluindo estratégias e implementação.

## Avaliação
1. Quiz sobre indexação
2. Projeto prático
3. Análise de performance