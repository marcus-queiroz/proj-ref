# Aula 6 - Slowly Changing Dimensions (SCD) - Parte 2

## Objetivos da Aula
- Implementar SCD Tipo 3
- Compreender e implementar SCD Tipo 4 (Mini-dimensões)
- Implementar SCD Tipo 6 (Híbrido)
- Desenvolver estratégias de dimensões híbridas
- Criar processos de carga para tipos avançados de SCD

## 1. SCD Tipo 3 (Atributos Anteriores)

### 1.1 Características
- Mantém valor anterior específico
- Número limitado de mudanças
- Uso de colunas específicas para histórico
- Ideal para tracking de mudanças pontuais

### 1.2 Implementação
```sql
-- Estrutura da dimensão Tipo 3
CREATE TABLE dim.Customer_Type3 (
    Customer_SK bigint IDENTITY(1,1),
    Customer_BK varchar(20) NOT NULL,
    CustomerName nvarchar(100) NOT NULL,
    
    -- Atributos atuais
    Current_Address nvarchar(200) NULL,
    Current_City nvarchar(100) NULL,
    Current_Phone varchar(20) NULL,
    
    -- Atributos anteriores
    Previous_Address nvarchar(200) NULL,
    Previous_City nvarchar(100) NULL,
    Previous_Phone varchar(20) NULL,
    
    -- Datas de mudança
    Address_Change_Date datetime2 NULL,
    
    Updated_DT datetime2 NOT NULL,
    CONSTRAINT PK_Customer_Type3 PRIMARY KEY (Customer_SK)
);

-- Processo de atualização
CREATE PROCEDURE dim.usp_UpdateCustomer_Type3
    @Customer_BK varchar(20),
    @CustomerName nvarchar(100),
    @Address nvarchar(200),
    @City nvarchar(100),
    @Phone varchar(20)
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @Now datetime2 = GETDATE();

    IF EXISTS (SELECT 1 FROM dim.Customer_Type3 WHERE Customer_BK = @Customer_BK)
    BEGIN
        UPDATE dim.Customer_Type3
        SET 
            -- Preserva valores anteriores
            Previous_Address = CASE 
                WHEN Current_Address <> @Address 
                THEN Current_Address 
                ELSE Previous_Address END,
            Previous_City = CASE 
                WHEN Current_City <> @City 
                THEN Current_City 
                ELSE Previous_City END,
            Previous_Phone = CASE 
                WHEN Current_Phone <> @Phone 
                THEN Current_Phone 
                ELSE Previous_Phone END,
            
            -- Atualiza valores atuais
            Current_Address = @Address,
            Current_City = @City,
            Current_Phone = @Phone,
            
            -- Atualiza data de mudança se houve alteração
            Address_Change_Date = CASE 
                WHEN Current_Address <> @Address 
                THEN @Now 
                ELSE Address_Change_Date END,
                
            Updated_DT = @Now
        WHERE Customer_BK = @Customer_BK;
    END
    ELSE
    BEGIN
        INSERT INTO dim.Customer_Type3 (
            Customer_BK, CustomerName,
            Current_Address, Current_City, Current_Phone,
            Updated_DT
        )
        VALUES (
            @Customer_BK, @CustomerName,
            @Address, @City, @Phone,
            @Now
        );
    END
END;
```

## 2. SCD Tipo 4 (Mini-dimensões)

### 2.1 Características
- Separação de atributos que mudam em diferentes velocidades
- Criação de dimensões auxiliares
- Otimização de espaço
- Melhor performance em queries

### 2.2 Implementação
```sql
-- Dimensão principal
CREATE TABLE dim.Customer_Type4 (
    Customer_SK bigint IDENTITY(1,1),
    Customer_BK varchar(20) NOT NULL,
    CustomerName nvarchar(100) NOT NULL,
    CustomerProfile_SK bigint NOT NULL,  -- FK para mini-dimensão
    Created_DT datetime2 NOT NULL,
    Updated_DT datetime2 NOT NULL,
    CONSTRAINT PK_Customer_Type4 PRIMARY KEY (Customer_SK)
);

-- Mini-dimensão (perfil do cliente)
CREATE TABLE dim.CustomerProfile (
    CustomerProfile_SK bigint IDENTITY(1,1),
    IncomeRange varchar(20) NOT NULL,
    AgeGroup varchar(20) NOT NULL,
    LifestyleSegment varchar(30) NOT NULL,
    CreditRating char(1) NOT NULL,
    Valid_From datetime2 NOT NULL,
    Valid_To datetime2 NULL,
    Current_FLG bit NOT NULL,
    CONSTRAINT PK_CustomerProfile PRIMARY KEY (CustomerProfile_SK)
);

-- Processo de atualização da mini-dimensão
CREATE PROCEDURE dim.usp_UpdateCustomerProfile
    @Customer_BK varchar(20),
    @IncomeRange varchar(20),
    @AgeGroup varchar(20),
    @LifestyleSegment varchar(30),
    @CreditRating char(1)
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @Now datetime2 = GETDATE();
    DECLARE @ProfileSK bigint;
    DECLARE @CurrentProfileSK bigint;

    -- Verifica se já existe perfil com estas características
    SELECT @ProfileSK = CustomerProfile_SK
    FROM dim.CustomerProfile
    WHERE IncomeRange = @IncomeRange
    AND AgeGroup = @AgeGroup
    AND LifestyleSegment = @LifestyleSegment
    AND CreditRating = @CreditRating
    AND Current_FLG = 1;

    -- Se não existe, cria novo perfil
    IF @ProfileSK IS NULL
    BEGIN
        INSERT INTO dim.CustomerProfile (
            IncomeRange, AgeGroup, LifestyleSegment, CreditRating,
            Valid_From, Current_FLG
        )
        VALUES (
            @IncomeRange, @AgeGroup, @LifestyleSegment, @CreditRating,
            @Now, 1
        );
        
        SET @ProfileSK = SCOPE_IDENTITY();
    END

    -- Atualiza referência na dimensão principal
    UPDATE dim.Customer_Type4
    SET CustomerProfile_SK = @ProfileSK,
        Updated_DT = @Now
    WHERE Customer_BK = @Customer_BK;
END;
```

## 3. SCD Tipo 6 (Híbrido)

### 3.1 Características
- Combinação de Tipos 1, 2 e 3
- Máxima flexibilidade
- Rastreamento completo de mudanças
- Maior complexidade de implementação

### 3.2 Implementação
```sql
CREATE TABLE dim.Customer_Type6 (
    Customer_SK bigint IDENTITY(1,1),
    Customer_BK varchar(20) NOT NULL,
    
    -- Atributos Tipo 1 (sempre atualizados)
    CustomerName nvarchar(100) NOT NULL,
    Email varchar(100) NULL,
    
    -- Atributos Tipo 2 (histórico completo)
    Address nvarchar(200) NULL,
    City nvarchar(100) NULL,
    
    -- Atributos Tipo 3 (valor anterior)
    Current_PhoneNumber varchar(20) NULL,
    Previous_PhoneNumber varchar(20) NULL,
    PhoneNumber_Change_Date datetime2 NULL,
    
    -- Controle de versão
    Valid_From datetime2 NOT NULL,
    Valid_To datetime2 NULL,
    Current_FLG bit NOT NULL,
    Version_NBR int NOT NULL,
    
    Updated_DT datetime2 NOT NULL,
    CONSTRAINT PK_Customer_Type6 PRIMARY KEY (Customer_SK)
);

-- Processo de atualização híbrido
CREATE PROCEDURE dim.usp_UpdateCustomer_Type6
    @Customer_BK varchar(20),
    @CustomerName nvarchar(100),
    @Email varchar(100),
    @Address nvarchar(200),
    @City nvarchar(100),
    @PhoneNumber varchar(20)
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @Now datetime2 = GETDATE();
    DECLARE @Version int;

    -- Obtém versão atual
    SELECT @Version = ISNULL(MAX(Version_NBR), 0)
    FROM dim.Customer_Type6
    WHERE Customer_BK = @Customer_BK;

    -- Verifica mudanças Tipo 2
    IF EXISTS (
        SELECT 1 
        FROM dim.Customer_Type6
        WHERE Customer_BK = @Customer_BK
        AND Current_FLG = 1
        AND (
            ISNULL(Address,'') <> ISNULL(@Address,'') OR
            ISNULL(City,'') <> ISNULL(@City,'')
        )
    )
    BEGIN
        -- Expira registro atual
        UPDATE dim.Customer_Type6
        SET Valid_To = @Now,
            Current_FLG = 0
        WHERE Customer_BK = @Customer_BK
        AND Current_FLG = 1;

        -- Insere nova versão
        INSERT INTO dim.Customer_Type6 (
            Customer_BK, CustomerName, Email,
            Address, City,
            Current_PhoneNumber, Previous_PhoneNumber,
            PhoneNumber_Change_Date,
            Valid_From, Valid_To, Current_FLG, Version_NBR,
            Updated_DT
        )
        SELECT 
            @Customer_BK, @CustomerName, @Email,
            @Address, @City,
            -- Lógica Tipo 3 para telefone
            CASE 
                WHEN Current_PhoneNumber <> @PhoneNumber
                THEN @PhoneNumber
                ELSE Current_PhoneNumber
            END,
            CASE 
                WHEN Current_PhoneNumber <> @PhoneNumber
                THEN Current_PhoneNumber
                ELSE Previous_PhoneNumber
            END,
            CASE 
                WHEN Current_PhoneNumber <> @PhoneNumber
                THEN @Now
                ELSE PhoneNumber_Change_Date
            END,
            @Now, NULL, 1, @Version + 1,
            @Now
        FROM dim.Customer_Type6
        WHERE Customer_BK = @Customer_BK
        AND Current_FLG = 0
        AND Valid_To = @Now;
    END
    ELSE
    BEGIN
        -- Atualiza apenas atributos Tipo 1 e 3
        UPDATE dim.Customer_Type6
        SET 
            -- Tipo 1
            CustomerName = @CustomerName,
            Email = @Email,
            
            -- Tipo 3
            Previous_PhoneNumber = CASE 
                WHEN Current_PhoneNumber <> @PhoneNumber
                THEN Current_PhoneNumber
                ELSE Previous_PhoneNumber
            END,
            Current_PhoneNumber = @PhoneNumber,
            PhoneNumber_Change_Date = CASE 
                WHEN Current_PhoneNumber <> @PhoneNumber
                THEN @Now
                ELSE PhoneNumber_Change_Date
            END,
            
            Updated_DT = @Now
        WHERE Customer_BK = @Customer_BK
        AND Current_FLG = 1;
    END
END;
```

## Exercícios Práticos

### Exercício 1: Mini-dimensões
1. Identifique candidatos a mini-dimensões
2. Implemente mini-dimensão para perfil de cliente
3. Desenvolva processo de atualização
4. Analise impacto na performance

### Exercício 2: Dimensão Híbrida
1. Crie dimensão com múltiplos tipos de tracking
2. Implemente processo de carga
3. Teste diferentes cenários de mudança
4. Desenvolva consultas analíticas

### Exercício 3: Análise Comparativa
1. Compare performance entre abordagens
2. Analise consumo de espaço
3. Avalie complexidade de manutenção
4. Documente prós e contras

## Material Complementar

### Documentação
- Padrões avançados de SCD
- Otimização de SCDs
- Melhores práticas por tipo

### Scripts Úteis
- Monitoramento de crescimento
- Análise de performance
- Manutenção de histórico

## Próxima Aula
Na próxima aula, abordaremos Tipos Especiais de Dimensões, incluindo role-playing dimensions, junk dimensions e dimensões degeneradas.

## Avaliação
1. Quiz sobre tipos avançados de SCD
2. Implementação de mini-dimensões
3. Projeto de dimensão híbrida