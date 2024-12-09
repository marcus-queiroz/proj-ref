# Aula 16 - Qualidade de Dados

## Objetivos da Aula
- Implementar validação de dados
- Desenvolver processos de limpeza
- Criar padronização de dados
- Trabalhar com deduplicação
- Implementar auditorias de qualidade

## 1. Validação de Dados

### 1.1 Regras de Validação
```sql
-- Tabela de regras de validação
CREATE TABLE meta.ValidationRules (
    Rule_SK bigint IDENTITY(1,1),
    TableName varchar(100) NOT NULL,
    ColumnName varchar(100) NOT NULL,
    RuleType varchar(50) NOT NULL,
    RuleDefinition nvarchar(max) NOT NULL,
    ErrorMessage nvarchar(500) NOT NULL,
    Severity int NOT NULL,
    IsActive bit NOT NULL,
    CONSTRAINT PK_ValidationRules PRIMARY KEY (Rule_SK)
);

-- Procedure para validação dinâmica
CREATE PROCEDURE etl.usp_ValidateData
    @TableName varchar(100)
AS
BEGIN
    DECLARE @SQL nvarchar(max);
    DECLARE @ColumnName varchar(100);
    DECLARE @RuleDefinition nvarchar(max);
    DECLARE @ErrorMessage nvarchar(500);
    
    -- Cursor para regras de validação
    DECLARE rule_cursor CURSOR FOR
    SELECT 
        ColumnName,
        RuleDefinition,
        ErrorMessage
    FROM meta.ValidationRules
    WHERE TableName = @TableName
    AND IsActive = 1;

    OPEN rule_cursor;
    FETCH NEXT FROM rule_cursor 
    INTO @ColumnName, @RuleDefinition, @ErrorMessage;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Construir SQL dinâmico
        SET @SQL = N'
        INSERT INTO meta.ValidationErrors (
            TableName,
            ColumnName,
            BusinessKey,
            ErrorMessage,
            ErrorDate
        )
        SELECT 
            ''' + @TableName + ''',
            ''' + @ColumnName + ''',
            Business_Key,
            ''' + @ErrorMessage + ''',
            GETDATE()
        FROM ' + @TableName + '
        WHERE ' + @RuleDefinition;

        -- Executar validação
        EXEC sp_executesql @SQL;

        FETCH NEXT FROM rule_cursor 
        INTO @ColumnName, @RuleDefinition, @ErrorMessage;
    END;

    CLOSE rule_cursor;
    DEALLOCATE rule_cursor;
END;
```

### 1.2 Implementação de Regras
```sql
-- Inserir regras de validação
INSERT INTO meta.ValidationRules (
    TableName,
    ColumnName,
    RuleType,
    RuleDefinition,
    ErrorMessage,
    Severity,
    IsActive
)
VALUES
-- Validação de Email
('stg.Customer', 'Email', 'Pattern',
 'Email NOT LIKE ''%_@_%.__%''',
 'Invalid email format', 2, 1),

-- Validação de Telefone
('stg.Customer', 'Phone', 'Pattern',
 'Phone NOT LIKE ''[0-9][0-9][0-9]-[0-9][0-9][0-9]-[0-9][0-9][0-9][0-9]''',
 'Invalid phone format', 2, 1),

-- Validação de Valores
('stg.Sales', 'Amount', 'Range',
 'Amount <= 0 OR Amount > 1000000',
 'Amount out of valid range', 1, 1);
```

## 2. Limpeza de Dados

### 2.1 Funções de Limpeza
```sql
-- Função de limpeza de texto
CREATE FUNCTION dbo.CleanString
(
    @InputString nvarchar(max)
)
RETURNS nvarchar(max)
AS
BEGIN
    DECLARE @OutputString nvarchar(max);

    -- Remover espaços extras
    SET @OutputString = LTRIM(RTRIM(@InputString));
    
    -- Remover caracteres especiais
    SET @OutputString = REPLACE(@OutputString, CHAR(9), ' ');  -- Tab
    SET @OutputString = REPLACE(@OutputString, CHAR(10), ' '); -- LF
    SET @OutputString = REPLACE(@OutputString, CHAR(13), ' '); -- CR
    
    -- Substituir múltiplos espaços por um único
    WHILE CHARINDEX('  ', @OutputString) > 0
        SET @OutputString = REPLACE(@OutputString, '  ', ' ');

    RETURN @OutputString;
END;

-- Função de limpeza de telefone
CREATE FUNCTION dbo.CleanPhone
(
    @Phone varchar(20)
)
RETURNS varchar(20)
AS
BEGIN
    DECLARE @CleanPhone varchar(20);
    
    -- Remover caracteres não numéricos
    SET @CleanPhone = REPLACE(REPLACE(REPLACE(REPLACE(
        @Phone, '(', ''), ')', ''), '-', ''), ' ', '');
    
    -- Formatar número
    IF LEN(@CleanPhone) = 10
        SET @CleanPhone = SUBSTRING(@CleanPhone, 1, 3) + '-' +
                         SUBSTRING(@CleanPhone, 4, 3) + '-' +
                         SUBSTRING(@CleanPhone, 7, 4);
    
    RETURN @CleanPhone;
END;
```

### 2.2 Processo de Limpeza
```sql
-- Procedure de limpeza de dados
CREATE PROCEDURE etl.usp_CleanCustomerData
AS
BEGIN
    -- Atualizar dados limpos
    UPDATE stg.Customer
    SET
        CustomerName = dbo.CleanString(CustomerName),
        Email = LOWER(dbo.CleanString(Email)),
        Phone = dbo.CleanPhone(Phone),
        Address = dbo.CleanString(Address);

    -- Logging de limpeza
    INSERT INTO meta.CleaningLog (
        TableName,
        RowsCleaned,
        CleaningDate
    )
    VALUES (
        'Customer',
        @@ROWCOUNT,
        GETDATE()
    );
END;
```

## 3. Padronização de Dados

### 3.1 Tabelas de Referência
```sql
-- Tabela de padrões
CREATE TABLE meta.StandardValues (
    Standard_SK bigint IDENTITY(1,1),
    CategoryName varchar(50) NOT NULL,
    SourceValue varchar(100) NOT NULL,
    StandardValue varchar(100) NOT NULL,
    IsActive bit NOT NULL,
    CONSTRAINT PK_StandardValues PRIMARY KEY (Standard_SK)
);

-- Inserir valores padrão
INSERT INTO meta.StandardValues 
(CategoryName, SourceValue, StandardValue, IsActive)
VALUES
-- Estados
('State', 'SP', 'São Paulo', 1),
('State', 'S.Paulo', 'São Paulo', 1),
('State', 'Sao Paulo', 'São Paulo', 1),

-- Tipos de Cliente
('CustomerType', 'PF', 'Person', 1),
('CustomerType', 'Person', 'Person', 1),
('CustomerType', 'Individual', 'Person', 1),
('CustomerType', 'PJ', 'Company', 1),
('CustomerType', 'Company', 'Company', 1),
('CustomerType', 'Corporate', 'Company', 1);
```

### 3.2 Processo de Padronização
```sql
-- Procedure de padronização
CREATE PROCEDURE etl.usp_StandardizeCustomerData
AS
BEGIN
    -- Padronizar estados
    UPDATE c
    SET c.State = s.StandardValue
    FROM stg.Customer c
    JOIN meta.StandardValues s 
        ON c.State = s.SourceValue
        AND s.CategoryName = 'State';

    -- Padronizar tipos de cliente
    UPDATE c
    SET c.CustomerType = s.StandardValue
    FROM stg.Customer c
    JOIN meta.StandardValues s 
        ON c.CustomerType = s.SourceValue
        AND s.CategoryName = 'CustomerType';

    -- Logging
    INSERT INTO meta.StandardizationLog (
        TableName,
        RowsStandardized,
        StandardizationDate
    )
    VALUES (
        'Customer',
        @@ROWCOUNT,
        GETDATE()
    );
END;
```

## 4. Deduplicação

### 4.1 Identificação de Duplicatas
```sql
-- Procedure para identificar duplicatas
CREATE PROCEDURE etl.usp_IdentifyDuplicates
AS
BEGIN
    -- Criar tabela temporária para duplicatas
    CREATE TABLE #Duplicates (
        GroupID int,
        Customer_BK varchar(20),
        CustomerName varchar(100),
        Email varchar(100),
        Phone varchar(20),
        MatchType varchar(50)
    );

    -- Encontrar duplicatas por email
    INSERT INTO #Duplicates
    SELECT 
        ROW_NUMBER() OVER(ORDER BY Email) as GroupID,
        Customer_BK,
        CustomerName,
        Email,
        Phone,
        'Email Match' as MatchType
    FROM stg.Customer c
    WHERE Email IN (
        SELECT Email
        FROM stg.Customer
        WHERE Email IS NOT NULL
        GROUP BY Email
        HAVING COUNT(*) > 1
    );

    -- Encontrar duplicatas por nome e telefone
    INSERT INTO #Duplicates
    SELECT 
        ROW_NUMBER() OVER(ORDER BY CustomerName, Phone) + 1000 as GroupID,
        Customer_BK,
        CustomerName,
        Email,
        Phone,
        'Name and Phone Match' as MatchType
    FROM stg.Customer c
    WHERE NOT EXISTS (
        SELECT 1 FROM #Duplicates d
        WHERE c.Customer_BK = d.Customer_BK
    )
    AND EXISTS (
        SELECT 1
        FROM stg.Customer c2
        WHERE c.CustomerName = c2.CustomerName
        AND c.Phone = c2.Phone
        AND c.Customer_BK <> c2.Customer_BK
    );

    -- Inserir resultados na tabela de duplicatas
    INSERT INTO meta.DuplicateCustomers (
        GroupID,
        Customer_BK,
        CustomerName,
        Email,
        Phone,
        MatchType,
        IdentificationDate
    )
    SELECT 
        GroupID,
        Customer_BK,
        CustomerName,
        Email,
        Phone,
        MatchType,
        GETDATE()
    FROM #Duplicates;

    -- Limpeza
    DROP TABLE #Duplicates;
END;
```

### 4.2 Resolução de Duplicatas
```sql
-- Procedure para resolver duplicatas
CREATE PROCEDURE etl.usp_ResolveDuplicates
AS
BEGIN
    -- Criar tabela de mapeamento
    CREATE TABLE #DuplicateMapping (
        SourceCustomer_BK varchar(20),
        TargetCustomer_BK varchar(20)
    );

    -- Identificar registros principais por grupo
    INSERT INTO #DuplicateMapping
    SELECT 
        d.Customer_BK as SourceCustomer_BK,
        FIRST_VALUE(d.Customer_BK) OVER (
            PARTITION BY d.GroupID 
            ORDER BY c.Created_DT
        ) as TargetCustomer_BK
    FROM meta.DuplicateCustomers d
    JOIN stg.Customer c ON d.Customer_BK = c.Customer_BK
    WHERE d.ResolutionStatus = 'Pending';

    -- Atualizar referências
    UPDATE f
    SET Customer_BK = m.TargetCustomer_BK
    FROM fact.Sales f
    JOIN #DuplicateMapping m 
        ON f.Customer_BK = m.SourceCustomer_BK;

    -- Marcar duplicatas como resolvidas
    UPDATE d
    SET 
        ResolutionStatus = 'Resolved',
        ResolutionDate = GETDATE()
    FROM meta.DuplicateCustomers d
    JOIN #DuplicateMapping m 
        ON d.Customer_BK = m.SourceCustomer_BK;

    -- Limpeza
    DROP TABLE #DuplicateMapping;
END;
```

## 5. Auditorias de Qualidade

### 5.1 Métricas de Qualidade
```sql
-- Procedure para gerar métricas
CREATE PROCEDURE meta.usp_GenerateQualityMetrics
AS
BEGIN
    INSERT INTO meta.QualityMetrics (
        TableName,
        MetricDate,
        TotalRows,
        InvalidRows,
        DuplicateRows,
        NullPercentage,
        StandardizedPercentage
    )
    SELECT
        'Customer' as TableName,
        GETDATE() as MetricDate,
        COUNT(*) as TotalRows,
        (SELECT COUNT(*) 
         FROM meta.ValidationErrors v 
         WHERE v.TableName = 'Customer') as InvalidRows,
        (SELECT COUNT(*) 
         FROM meta.DuplicateCustomers 
         WHERE ResolutionStatus = 'Pending') as DuplicateRows,
        (SELECT AVG(CASE 
            WHEN Email IS NULL OR Phone IS NULL THEN 100.0 
            ELSE 0.0 END)
         FROM stg.Customer) as NullPercentage,
        (SELECT COUNT(*) * 100.0 / COUNT(*)
         FROM stg.Customer c
         JOIN meta.StandardValues s 
             ON c.State = s.StandardValue) as StandardizedPercentage
    FROM stg.Customer;
END;
```

### 5.2 Relatórios de Qualidade
```sql
-- View para relatório de qualidade
CREATE VIEW report.DataQualityDashboard
AS
WITH MetricsTrend AS (
    SELECT
        TableName,
        MetricDate,
        TotalRows,
        InvalidRows * 100.0 / NULLIF(TotalRows, 0) as InvalidPercentage,
        DuplicateRows * 100.0 / NULLIF(TotalRows, 0) as DuplicatePercentage,
        NullPercentage,
        StandardizedPercentage
    FROM meta.QualityMetrics
),
ValidationSummary AS (
    SELECT
        TableName,
        ColumnName,
        COUNT(*) as ErrorCount,
        MAX(ErrorDate) as LastErrorDate
    FROM meta.ValidationErrors
    GROUP BY TableName, ColumnName
)
SELECT
    m.TableName,
    m.MetricDate,
    m.TotalRows,
    m.InvalidPercentage,
    m.DuplicatePercentage,
    m.NullPercentage,
    m.StandardizedPercentage,
    v.ErrorCount,
    v.LastErrorDate
FROM MetricsTrend m
LEFT JOIN ValidationSummary v 
    ON m.TableName = v.TableName;
```

## Exercícios Práticos

### Exercício 1: Validação
1. Criar regras de validação
2. Implementar processo de validação
3. Gerar relatório de erros

### Exercício 2: Limpeza e