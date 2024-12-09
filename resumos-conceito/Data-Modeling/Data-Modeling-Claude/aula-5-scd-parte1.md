# Aula 5 - Slowly Changing Dimensions (SCD) - Parte 1

## Objetivos da Aula
- Compreender o conceito de Slowly Changing Dimensions
- Implementar SCD Tipo 1
- Implementar SCD Tipo 2
- Desenvolver estratégias de chaves
- Criar processos de carga para SCDs

## 1. Fundamentos de Slowly Changing Dimensions

### 1.1 O que são SCDs?
- Dimensões que mudam ao longo do tempo
- Necessidade de rastreamento histórico
- Impacto nas análises
- Tipos principais de mudança

### 1.2 Quando Usar Cada Tipo
- Tipo 1: Sobrescrita
  - Dados de referência
  - Correções de erros
  - Quando histórico não é necessário

- Tipo 2: Histórico completo
  - Mudanças importantes para análise
  - Necessidade de auditoria
  - Análises temporais

## 2. SCD Tipo 1 (Sobrescrita)

### 2.1 Características
- Substitui valores antigos
- Mantém apenas versão atual
- Sem histórico de mudanças
- Implementação simples

### 2.2 Implementação Básica
```sql
-- Estrutura da dimensão Tipo 1
CREATE TABLE dim.Customer_Type1 (
    Customer_SK bigint IDENTITY(1,1),
    Customer_BK varchar(20) NOT NULL,
    CustomerName nvarchar(100) NOT NULL,
    Email varchar(100) NULL,
    Phone varchar(20) NULL,
    Address nvarchar(200) NULL,
    City nvarchar(100) NULL,
    Updated_DT datetime2 NOT NULL,
    CONSTRAINT PK_Customer_Type1 PRIMARY KEY (Customer_SK)
);

-- Processo de atualização Tipo 1
CREATE PROCEDURE dim.usp_UpdateCustomer_Type1
    @Customer_BK varchar(20),
    @CustomerName nvarchar(100),
    @Email varchar(100),
    @Phone varchar(20),
    @Address nvarchar(200),
    @City nvarchar(100)
AS
BEGIN
    SET NOCOUNT ON;

    IF EXISTS (SELECT 1 FROM dim.Customer_Type1 WHERE Customer_BK = @Customer_BK)
    BEGIN
        -- Atualiza registro existente
        UPDATE dim.Customer_Type1
        SET CustomerName = @CustomerName,
            Email = @Email,
            Phone = @Phone,
            Address = @Address,
            City = @City,
            Updated_DT = GETDATE()
        WHERE Customer_BK = @Customer_BK;
    END
    ELSE
    BEGIN
        -- Insere novo registro
        INSERT INTO dim.Customer_Type1 
        (Customer_BK, CustomerName, Email, Phone, Address, City, Updated_DT)
        VALUES 
        (@Customer_BK, @CustomerName, @Email, @Phone, @Address, @City, GETDATE());
    END
END;
```

### 2.3 Processo de Carga em Lote
```sql
-- Merge para carga em lote
MERGE dim.Customer_Type1 AS target
USING stg.Customer AS source
ON target.Customer_BK = source.Customer_BK

-- Atualiza registros existentes
WHEN MATCHED AND (
    target.CustomerName <> source.CustomerName OR
    ISNULL(target.Email,'') <> ISNULL(source.Email,'') OR
    ISNULL(target.Phone,'') <> ISNULL(source.Phone,'') OR
    ISNULL(target.Address,'') <> ISNULL(source.Address,'') OR
    ISNULL(target.City,'') <> ISNULL(source.City,'')
) THEN
    UPDATE SET
        CustomerName = source.CustomerName,
        Email = source.Email,
        Phone = source.Phone,
        Address = source.Address,
        City = source.City,
        Updated_DT = GETDATE()

-- Insere novos registros
WHEN NOT MATCHED THEN
    INSERT (Customer_BK, CustomerName, Email, Phone, Address, City, Updated_DT)
    VALUES (source.Customer_BK, source.CustomerName, source.Email, 
            source.Phone, source.Address, source.City, GETDATE());
```

## 3. SCD Tipo 2 (Histórico)

### 3.1 Características
- Mantém histórico completo
- Múltiplas versões do mesmo registro
- Controle de vigência
- Flags de registro atual

### 3.2 Implementação Básica
```sql
-- Estrutura da dimensão Tipo 2
CREATE TABLE dim.Customer_Type2 (
    Customer_SK bigint IDENTITY(1,1),
    Customer_BK varchar(20) NOT NULL,
    CustomerName nvarchar(100) NOT NULL,
    Email varchar(100) NULL,
    Phone varchar(20) NULL,
    Address nvarchar(200) NULL,
    City nvarchar(100) NULL,
    Valid_From datetime2 NOT NULL,
    Valid_To datetime2 NULL,
    Current_FLG bit NOT NULL,
    Version_NBR int NOT NULL,
    CONSTRAINT PK_Customer_Type2 PRIMARY KEY (Customer_SK)
);

-- Índices recomendados
CREATE NONCLUSTERED INDEX IX_Customer_Type2_BK 
ON dim.Customer_Type2(Customer_BK)
INCLUDE (Current_FLG);

CREATE NONCLUSTERED INDEX IX_Customer_Type2_Dates 
ON dim.Customer_Type2(Valid_From, Valid_To)
INCLUDE (Customer_BK, Current_FLG);
```

### 3.3 Processo de Atualização
```sql
CREATE PROCEDURE dim.usp_UpdateCustomer_Type2
    @Customer_BK varchar(20),
    @CustomerName nvarchar(100),
    @Email varchar(100),
    @Phone varchar(20),
    @Address nvarchar(200),
    @City nvarchar(100)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @Now datetime2 = GETDATE();
    DECLARE @Version int;

    -- Verifica se existe versão atual
    SELECT @Version = ISNULL(MAX(Version_NBR), 0)
    FROM dim.Customer_Type2
    WHERE Customer_BK = @Customer_BK;

    -- Verifica se houve mudança
    IF EXISTS (
        SELECT 1 
        FROM dim.Customer_Type2
        WHERE Customer_BK = @Customer_BK
        AND Current_FLG = 1
        AND (
            CustomerName <> @CustomerName OR
            ISNULL(Email,'') <> ISNULL(@Email,'') OR
            ISNULL(Phone,'') <> ISNULL(@Phone,'') OR
            ISNULL(Address,'') <> ISNULL(@Address,'') OR
            ISNULL(City,'') <> ISNULL(@City,'')
        )
    )
    BEGIN
        -- Expira registro atual
        UPDATE dim.Customer_Type2
        SET Valid_To = @Now,
            Current_FLG = 0
        WHERE Customer_BK = @Customer_BK
        AND Current_FLG = 1;

        -- Insere nova versão
        INSERT INTO dim.Customer_Type2 (
            Customer_BK, CustomerName, Email, Phone, 
            Address, City, Valid_From, Valid_To, 
            Current_FLG, Version_NBR
        )
        VALUES (
            @Customer_BK, @CustomerName, @Email, @Phone,
            @Address, @City, @Now, NULL,
            1, @Version + 1
        );
    END
    ELSE IF @Version = 0
    BEGIN
        -- Insere primeiro registro
        INSERT INTO dim.Customer_Type2 (
            Customer_BK, CustomerName, Email, Phone,
            Address, City, Valid_From, Valid_To,
            Current_FLG, Version_NBR
        )
        VALUES (
            @Customer_BK, @CustomerName, @Email, @Phone,
            @Address, @City, @Now, NULL,
            1, 1
        );
    END
END;
```

## 4. Estratégias de Chaves

### 4.1 Surrogate Keys
- Geração de chaves
- Manutenção de integridade
- Relacionamento com fatos

### 4.2 Business Keys
- Identificação natural
- Unicidade
- Rastreabilidade

### 4.3 Versioning
- Controle de versões
- Numeração sequencial
- Flags de controle

## Exercícios Práticos

### Exercício 1: Implementação Tipo 1
1. Crie uma dimensão de produto Tipo 1
2. Implemente processo de atualização
3. Teste diferentes cenários de mudança

### Exercício 2: Implementação Tipo 2
1. Crie uma dimensão de cliente Tipo 2
2. Implemente controle de versões
3. Teste cenários de mudança histórica

### Exercício 3: Análise de Dados
1. Consulte histórico de mudanças
2. Analise impacto nas fatos
3. Implemente consultas temporais

## Material Complementar

### Documentação
- SQL Server SCD Patterns
- Performance Tuning for SCDs
- Best Practices for Historical Data

### Scripts Úteis
- Verificação de integridade histórica
- Análise de crescimento dimensional
- Limpeza de dados históricos

## Próxima Aula
Na próxima aula, continuaremos com Slowly Changing Dimensions (SCD) Parte 2, abordando os tipos 3, 4 e 6.

## Avaliação
1. Quiz sobre conceitos de SCD
2. Implementação prática de SCDs
3. Análise de cenários de uso