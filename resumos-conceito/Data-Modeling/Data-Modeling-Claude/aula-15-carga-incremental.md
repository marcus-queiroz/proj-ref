# Aula 15 - Carga Incremental

## Objetivos da Aula
- Compreender estratégias de carga incremental
- Implementar Change Data Capture (CDC)
- Trabalhar com Change Tracking
- Desenvolver processos de Merge
- Criar logging de mudanças

## 1. Estratégias de Carga Incremental

### 1.1 Controle por Data/Timestamp
```sql
-- Tabela de controle de carga
CREATE TABLE meta.LoadControl (
    TableName varchar(100) NOT NULL,
    LastLoadDate datetime2 NOT NULL,
    Status varchar(20) NOT NULL,
    RowsProcessed int NULL,
    CONSTRAINT PK_LoadControl PRIMARY KEY (TableName)
);

-- Procedure para carga incremental por data
CREATE PROCEDURE etl.usp_LoadIncrementalByDate
    @TableName varchar(100)
AS
BEGIN
    DECLARE @LastLoadDate datetime2;
    DECLARE @CurrentLoadDate datetime2 = GETDATE();

    -- Obter última data de carga
    SELECT @LastLoadDate = LastLoadDate
    FROM meta.LoadControl
    WHERE TableName = @TableName;

    -- Processar dados novos/alterados
    INSERT INTO stg.Customer (
        Customer_BK,
        CustomerName,
        Email,
        Phone,
        ModifiedDate
    )
    SELECT 
        CustomerID,
        CustomerName,
        Email,
        Phone,
        ModifiedDate
    FROM source.Customer
    WHERE ModifiedDate > @LastLoadDate;

    -- Atualizar controle
    UPDATE meta.LoadControl
    SET 
        LastLoadDate = @CurrentLoadDate,
        Status = 'Completed',
        RowsProcessed = @@ROWCOUNT
    WHERE TableName = @TableName;
END;
```

### 1.2 Controle por Versão
```sql
-- Tabela de controle de versão
CREATE TABLE meta.VersionControl (
    TableName varchar(100) NOT NULL,
    CurrentVersion bigint NOT NULL,
    LastProcessedVersion bigint NOT NULL,
    LastUpdateDate datetime2 NOT NULL,
    CONSTRAINT PK_VersionControl PRIMARY KEY (TableName)
);

-- Procedure para carga incremental por versão
CREATE PROCEDURE etl.usp_LoadIncrementalByVersion
    @TableName varchar(100)
AS
BEGIN
    DECLARE @LastVersion bigint;
    DECLARE @CurrentVersion bigint;

    -- Obter versões
    SELECT 
        @LastVersion = LastProcessedVersion,
        @CurrentVersion = CurrentVersion
    FROM meta.VersionControl
    WHERE TableName = @TableName;

    -- Processar registros
    INSERT INTO stg.Customer (
        Customer_BK,
        CustomerName,
        Email,
        Phone,
        Version
    )
    SELECT 
        CustomerID,
        CustomerName,
        Email,
        Phone,
        Version
    FROM source.Customer
    WHERE Version > @LastVersion;

    -- Atualizar controle
    UPDATE meta.VersionControl
    SET 
        LastProcessedVersion = @CurrentVersion,
        LastUpdateDate = GETDATE()
    WHERE TableName = @TableName;
END;
```

## 2. Change Data Capture (CDC)

### 2.1 Configuração do CDC
```sql
-- Habilitar CDC no banco
EXEC sys.sp_cdc_enable_db;

-- Habilitar CDC na tabela
EXEC sys.sp_cdc_enable_table
    @source_schema = N'dbo',
    @source_name = N'Customer',
    @role_name = N'CDC_Reader',
    @supports_net_changes = 1;

-- Verificar configuração
SELECT 
    s.name as SchemaName,
    t.name as TableName,
    is_tracked_by_cdc
FROM sys.tables t
JOIN sys.schemas s ON t.schema_id = s.schema_id
WHERE is_tracked_by_cdc = 1;
```

### 2.2 Processamento de Mudanças
```sql
-- Procedure para processar mudanças CDC
CREATE PROCEDURE etl.usp_ProcessCDCChanges
    @TableName varchar(100)
AS
BEGIN
    DECLARE @from_lsn binary(10);
    DECLARE @to_lsn binary(10);

    -- Obter LSN range
    SELECT @from_lsn = last_processed_lsn
    FROM meta.CDCControl
    WHERE TableName = @TableName;

    SET @to_lsn = sys.fn_cdc_get_max_lsn();

    -- Processar mudanças
    INSERT INTO stg.CustomerChanges (
        Customer_BK,
        CustomerName,
        Email,
        Phone,
        ChangeType,
        ChangeLSN
    )
    SELECT 
        __$source_id as Customer_BK,
        CustomerName,
        Email,
        Phone,
        CASE __$operation
            WHEN 1 THEN 'Delete'
            WHEN 2 THEN 'Insert'
            WHEN 3 THEN 'Update (Before)'
            WHEN 4 THEN 'Update (After)'
        END as ChangeType,
        __$start_lsn as ChangeLSN
    FROM cdc.fn_cdc_get_all_changes_dbo_Customer(
        @from_lsn,
        @to_lsn,
        'all'
    );

    -- Atualizar controle
    UPDATE meta.CDCControl
    SET 
        last_processed_lsn = @to_lsn,
        last_process_date = GETDATE()
    WHERE TableName = @TableName;
END;
```

## 3. Change Tracking

### 3.1 Configuração
```sql
-- Habilitar Change Tracking no banco
ALTER DATABASE YourDatabase
SET CHANGE_TRACKING = ON
(CHANGE_RETENTION = 2 DAYS, AUTO_CLEANUP = ON);

-- Habilitar Change Tracking na tabela
ALTER TABLE dbo.Customer
ENABLE CHANGE_TRACKING
WITH (TRACK_COLUMNS_UPDATED = ON);

-- Verificar configuração
SELECT 
    t.name as TableName,
    t.is_tracked_by_ct
FROM sys.tables t
WHERE t.is_tracked_by_ct = 1;
```

### 3.2 Processamento de Mudanças
```sql
-- Procedure para processar Change Tracking
CREATE PROCEDURE etl.usp_ProcessChangeTracking
    @TableName varchar(100)
AS
BEGIN
    DECLARE @last_sync_version bigint;

    -- Obter última versão processada
    SELECT @last_sync_version = LastSyncVersion
    FROM meta.ChangeTrackingControl
    WHERE TableName = @TableName;

    -- Processar mudanças
    INSERT INTO stg.CustomerChanges (
        Customer_BK,
        ChangeType,
        ChangeVersion
    )
    SELECT 
        t.CustomerID,
        CASE
            WHEN ct.SYS_CHANGE_OPERATION = 'I' THEN 'Insert'
            WHEN ct.SYS_CHANGE_OPERATION = 'U' THEN 'Update'
            WHEN ct.SYS_CHANGE_OPERATION = 'D' THEN 'Delete'
        END,
        ct.SYS_CHANGE_VERSION
    FROM Customer t
    RIGHT OUTER JOIN
        CHANGETABLE(CHANGES Customer, @last_sync_version) AS ct
        ON t.CustomerID = ct.CustomerID;

    -- Atualizar controle
    UPDATE meta.ChangeTrackingControl
    SET 
        LastSyncVersion = CHANGE_TRACKING_CURRENT_VERSION(),
        LastSyncDate = GETDATE()
    WHERE TableName = @TableName;
END;
```

## 4. Merge

### 4.1 Implementação de Merge
```sql
-- Procedure para merge de dados
CREATE PROCEDURE etl.usp_MergeCustomer
AS
BEGIN
    MERGE dim.Customer as target
    USING stg.Customer as source
    ON target.Customer_BK = source.Customer_BK

    WHEN MATCHED AND (
        target.CustomerName <> source.CustomerName OR
        ISNULL(target.Email,'') <> ISNULL(source.Email,'') OR
        ISNULL(target.Phone,'') <> ISNULL(source.Phone,'')
    ) THEN
        UPDATE SET
            CustomerName = source.CustomerName,
            Email = source.Email,
            Phone = source.Phone,
            Updated_DT = GETDATE()

    WHEN NOT MATCHED BY TARGET THEN
        INSERT (
            Customer_BK,
            CustomerName,
            Email,
            Phone,
            Created_DT,
            Updated_DT
        )
        VALUES (
            source.Customer_BK,
            source.CustomerName,
            source.Email,
            source.Phone,
            GETDATE(),
            GETDATE()
        )

    WHEN NOT MATCHED BY SOURCE AND target.Current_FLG = 1 THEN
        UPDATE SET
            Current_FLG = 0,
            Updated_DT = GETDATE();

    -- Logging do resultado
    INSERT INTO meta.MergeLog (
        TableName,
        InsertCount,
        UpdateCount,
        DeleteCount,
        ErrorCount,
        ProcessDate
    )
    SELECT
        'Customer',
        @InsertCount = @@ROWCOUNT,
        @UpdateCount = @@ROWCOUNT,
        @DeleteCount = @@ROWCOUNT,
        0,
        GETDATE();
END;
```

### 4.2 Logging de Mudanças
```sql
-- Tabela de log de mudanças
CREATE TABLE meta.ChangeLog (
    ChangeLog_SK bigint IDENTITY(1,1),
    TableName varchar(100) NOT NULL,
    BusinessKey varchar(100) NOT NULL,
    ColumnName varchar(100) NOT NULL,
    OldValue nvarchar(max) NULL,
    NewValue nvarchar(max) NULL,
    ChangeType varchar(20) NOT NULL,
    ChangeDate datetime2 NOT NULL,
    CONSTRAINT PK_ChangeLog PRIMARY KEY (ChangeLog_SK)
);

-- Trigger para logging de mudanças
CREATE TRIGGER tr_Customer_ChangeLog
ON dim.Customer
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    -- Inserções
    INSERT INTO meta.ChangeLog (
        TableName,
        BusinessKey,
        ColumnName,
        OldValue,
        NewValue,
        ChangeType,
        ChangeDate
    )
    SELECT
        'Customer',
        i.Customer_BK,
        'CustomerName',
        NULL,
        i.CustomerName,
        'Insert',
        GETDATE()
    FROM inserted i
    WHERE NOT EXISTS (SELECT 1 FROM deleted);

    -- Atualizações
    INSERT INTO meta.ChangeLog (
        TableName,
        BusinessKey,
        ColumnName,
        OldValue,
        NewValue,
        ChangeType,
        ChangeDate
    )
    SELECT
        'Customer',
        i.Customer_BK,
        'CustomerName',
        d.CustomerName,
        i.CustomerName,
        'Update',
        GETDATE()
    FROM inserted i
    JOIN deleted d ON i.Customer_SK = d.Customer_SK
    WHERE i.CustomerName <> d.CustomerName;

    -- Deleções
    INSERT INTO meta.ChangeLog (
        TableName,
        BusinessKey,
        ColumnName,
        OldValue,
        NewValue,
        ChangeType,
        ChangeDate
    )
    SELECT
        'Customer',
        d.Customer_BK,
        'CustomerName',
        d.CustomerName,
        NULL,
        'Delete',
        GETDATE()
    FROM deleted d
    WHERE NOT EXISTS (SELECT 1 FROM inserted);
END;
```

## Exercícios Práticos

### Exercício 1: Controle de Carga
1. Implementar controle por data
2. Desenvolver controle por versão
3. Criar logs de processamento

### Exercício 2: CDC e Change Tracking
1. Configurar CDC em tabelas
2. Implementar Change Tracking
3. Processar mudanças

### Exercício 3: Merge e Logging
1. Desenvolver processo de merge
2. Implementar logging detalhado
3. Criar relatórios de mudança

## Material Complementar

### Documentação
- CDC Best Practices
- Change Tracking Guide
- Merge Patterns Guide

### Scripts Úteis
- Monitoramento de mudanças
- Validação de processo
- Limpeza de logs

## Próxima Aula
Na próxima aula, abordaremos Qualidade de Dados, incluindo validação, limpeza e padronização.

## Avaliação
1. Quiz sobre carga incremental
2. Implementação prática
3. Análise de cenários