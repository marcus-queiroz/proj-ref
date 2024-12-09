# Aula 8 - Tipos Especiais de Fatos

## Objetivos da Aula
- Compreender e implementar Bridge Tables
- Desenvolver Factless Fact Tables
- Trabalhar com Fatos Agregados
- Implementar Fatos Consolidados
- Tratar relacionamentos muitos-para-muitos

## 1. Bridge Tables

### 1.1 Conceito e Uso
- Resolução de relacionamentos muitos-para-muitos
- Grupos dinâmicos de dimensões
- Hierarquias alternativas
- Relacionamentos complexos

### 1.2 Implementação Básica
```sql
-- Dimensão de Produto
CREATE TABLE dim.Product (
    Product_SK bigint IDENTITY(1,1),
    Product_BK varchar(20) NOT NULL,
    ProductName nvarchar(100) NOT NULL,
    CONSTRAINT PK_Product PRIMARY KEY (Product_SK)
);

-- Dimensão de Componente
CREATE TABLE dim.Component (
    Component_SK bigint IDENTITY(1,1),
    Component_BK varchar(20) NOT NULL,
    ComponentName nvarchar(100) NOT NULL,
    CONSTRAINT PK_Component PRIMARY KEY (Component_SK)
);

-- Bridge Table para Produtos e Componentes
CREATE TABLE brg.ProductComponent (
    Product_SK bigint NOT NULL,
    Component_SK bigint NOT NULL,
    EffectiveDate date NOT NULL,
    ExpirationDate date NULL,
    Quantity int NOT NULL,
    CONSTRAINT PK_ProductComponent 
        PRIMARY KEY (Product_SK, Component_SK, EffectiveDate),
    CONSTRAINT FK_Bridge_Product 
        FOREIGN KEY (Product_SK) 
        REFERENCES dim.Product (Product_SK),
    CONSTRAINT FK_Bridge_Component 
        FOREIGN KEY (Component_SK) 
        REFERENCES dim.Component (Component_SK)
);
```

### 1.3 Implementação Avançada com Grupos
```sql
-- Dimensão de Grupo
CREATE TABLE dim.ProductGroup (
    ProductGroup_SK bigint IDENTITY(1,1),
    GroupName nvarchar(100) NOT NULL,
    GroupType varchar(50) NOT NULL,
    CONSTRAINT PK_ProductGroup PRIMARY KEY (ProductGroup_SK)
);

-- Bridge Table para Grupos Dinâmicos
CREATE TABLE brg.ProductGroupBridge (
    ProductGroup_SK bigint NOT NULL,
    Product_SK bigint NOT NULL,
    EffectiveDate date NOT NULL,
    ExpirationDate date NULL,
    CONSTRAINT PK_ProductGroupBridge 
        PRIMARY KEY (ProductGroup_SK, Product_SK, EffectiveDate)
);

-- Procedimento para manutenção de grupos
CREATE PROCEDURE dim.usp_MaintainProductGroups
    @GroupName nvarchar(100),
    @Products varchar(max)  -- Lista de Product_SKs em formato CSV
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @Now date = GETDATE();
    DECLARE @GroupSK bigint;

    -- Obter ou criar grupo
    SELECT @GroupSK = ProductGroup_SK
    FROM dim.ProductGroup
    WHERE GroupName = @GroupName;

    IF @GroupSK IS NULL
    BEGIN
        INSERT INTO dim.ProductGroup (GroupName, GroupType)
        VALUES (@GroupName, 'Dynamic');
        SET @GroupSK = SCOPE_IDENTITY();
    END;

    -- Expirar registros antigos
    UPDATE brg.ProductGroupBridge
    SET ExpirationDate = @Now
    WHERE ProductGroup_SK = @GroupSK
    AND ExpirationDate IS NULL;

    -- Inserir novos registros
    INSERT INTO brg.ProductGroupBridge (
        ProductGroup_SK, Product_SK, EffectiveDate
    )
    SELECT 
        @GroupSK,
        Product_SK,
        @Now
    FROM STRING_SPLIT(@Products, ',') s
    INNER JOIN dim.Product p 
        ON p.Product_SK = CAST(s.value AS bigint);
END;
```

## 2. Factless Fact Tables

### 2.1 Conceito
- Registro de eventos sem medidas
- Cobertura de eventos
- Análise de ocorrências
- Oportunidades e relacionamentos

### 2.2 Implementação
```sql
-- Factless fact table para cobertura de eventos
CREATE TABLE fact.EventCoverage (
    Date_SK int NOT NULL,
    Store_SK bigint NOT NULL,
    Promotion_SK bigint NOT NULL,
    EventType_SK int NOT NULL,
    CONSTRAINT PK_EventCoverage 
        PRIMARY KEY (Date_SK, Store_SK, Promotion_SK, EventType_SK)
);

-- Factless fact table para oportunidades de venda
CREATE TABLE fact.SalesOpportunity (
    Date_SK int NOT NULL,
    Product_SK bigint NOT NULL,
    Customer_SK bigint NOT NULL,
    Campaign_SK bigint NOT NULL,
    ContactType_SK int NOT NULL,
    ContactDate datetime2 NOT NULL,
    CONSTRAINT PK_SalesOpportunity 
        PRIMARY KEY (Date_SK, Product_SK, Customer_SK, Campaign_SK)
);

-- Exemplo de análise de cobertura
CREATE VIEW analysis.EventCoverageAnalysis AS
WITH DateRange AS (
    SELECT Date_SK
    FROM dim.Date
    WHERE FullDate BETWEEN '2024-01-01' AND '2024-12-31'
),
StoreDays AS (
    SELECT d.Date_SK, s.Store_SK
    FROM DateRange d
    CROSS JOIN dim.Store s
    WHERE s.Status = 'Active'
)
SELECT 
    d.Month,
    d.MonthName,
    s.StoreRegion,
    COUNT(sd.Date_SK) as PossibleEvents,
    COUNT(ec.Date_SK) as ActualEvents,
    CAST(COUNT(ec.Date_SK) AS float) / COUNT(sd.Date_SK) as CoverageRatio
FROM StoreDays sd
INNER JOIN dim.Date d ON sd.Date_SK = d.Date_SK
INNER JOIN dim.Store s ON sd.Store_SK = s.Store_SK
LEFT JOIN fact.EventCoverage ec 
    ON sd.Date_SK = ec.Date_SK 
    AND sd.Store_SK = ec.Store_SK
GROUP BY 
    d.Month,
    d.MonthName,
    s.StoreRegion;
```

## 3. Fatos Agregados

### 3.1 Conceito
- Pré-agregação de medidas
- Otimização de performance
- Diferentes níveis de granularidade
- Estratégias de atualização

### 3.2 Implementação
```sql
-- Fato base no nível transacional
CREATE TABLE fact.SalesTransaction (
    Sale_SK bigint IDENTITY(1,1),
    Date_SK int NOT NULL,
    Product_SK bigint NOT NULL,
    Store_SK bigint NOT NULL,
    Customer_SK bigint NOT NULL,
    Quantity int NOT NULL,
    Amount decimal(18,2) NOT NULL,
    Cost decimal(18,2) NOT NULL,
    CONSTRAINT PK_SalesTransaction PRIMARY KEY (Sale_SK)
);

-- Fato agregado diário por produto e loja
CREATE TABLE fact.SalesDailyProduct (
    Date_SK int NOT NULL,
    Product_SK bigint NOT NULL,
    Store_SK bigint NOT NULL,
    TotalQuantity int NOT NULL,
    TotalAmount decimal(18,2) NOT NULL,
    TotalCost decimal(18,2) NOT NULL,
    NumberOfTransactions int NOT NULL,
    CONSTRAINT PK_SalesDailyProduct 
        PRIMARY KEY (Date_SK, Product_SK, Store_SK)
);

-- Procedure para manter agregações
CREATE PROCEDURE fact.usp_MaintainSalesAggregations
    @StartDate date,
    @EndDate date
AS
BEGIN
    SET NOCOUNT ON;

    -- Atualizar agregação diária
    WITH DailySales AS (
        SELECT 
            Date_SK,
            Product_SK,
            Store_SK,
            SUM(Quantity) as TotalQuantity,
            SUM(Amount) as TotalAmount,
            SUM(Cost) as TotalCost,
            COUNT(*) as NumberOfTransactions
        FROM fact.SalesTransaction
        WHERE Date_SK BETWEEN 
            CAST(REPLACE(@StartDate, '-', '') AS int) AND
            CAST(REPLACE(@EndDate, '-', '') AS int)
        GROUP BY
            Date_SK,
            Product_SK,
            Store_SK
    )
    MERGE fact.SalesDailyProduct AS target
    USING DailySales AS source
    ON  target.Date_SK = source.Date_SK
        AND target.Product_SK = source.Product_SK
        AND target.Store_SK = source.Store_SK
    
    WHEN MATCHED THEN
        UPDATE SET
            TotalQuantity = source.TotalQuantity,
            TotalAmount = source.TotalAmount,
            TotalCost = source.TotalCost,
            NumberOfTransactions = source.NumberOfTransactions
    
    WHEN NOT MATCHED THEN
        INSERT (
            Date_SK, Product_SK, Store_SK,
            TotalQuantity, TotalAmount, TotalCost, NumberOfTransactions
        )
        VALUES (
            source.Date_SK, source.Product_SK, source.Store_SK,
            source.TotalQuantity, source.TotalAmount, 
            source.TotalCost, source.NumberOfTransactions
        );
END;
```

## 4. Fatos Consolidados

### 4.1 Conceito
- Combinação de diferentes fontes
- Harmonização de granularidade
- Regras de negócio complexas
- Visão unificada

### 4.2 Implementação
```sql
-- Fato consolidado de vendas
CREATE TABLE fact.SalesConsolidated (
    Date_SK int NOT NULL,
    Product_SK bigint NOT NULL,
    Store_SK bigint NOT NULL,
    Channel_SK bigint NOT NULL,
    
    -- Medidas de diferentes fontes
    RetailQuantity int NOT NULL,
    RetailAmount decimal(18,2) NOT NULL,
    OnlineQuantity int NOT NULL,
    OnlineAmount decimal(18,2) NOT NULL,
    WholesaleQuantity int NOT NULL,
    WholesaleAmount decimal(18,2) NOT NULL,
    
    -- Medidas calculadas
    TotalQuantity int NOT NULL,
    TotalAmount decimal(18,2) NOT NULL,
    
    -- Controle
    LastUpdated datetime2 NOT NULL,
    SourceSystem varchar(20) NOT NULL,
    
    CONSTRAINT PK_SalesConsolidated 
        PRIMARY KEY (Date_SK, Product_SK, Store_SK, Channel_SK)
);

-- Procedure para consolidação
CREATE PROCEDURE fact.usp_ConsolidateSales
    @StartDate date,
    @EndDate date
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @Now datetime2 = GETDATE();

    -- Consolidar dados de diferentes fontes
    WITH SalesUnified AS (
        -- Vendas de varejo
        SELECT
            Date_SK,
            Product_SK,
            Store_SK,
            Channel_SK,
            Quantity as RetailQuantity,
            Amount as RetailAmount,
            0 as OnlineQuantity,
            0 as OnlineAmount,
            0 as WholesaleQuantity,
            0 as WholesaleAmount
        FROM fact.RetailSales
        WHERE Date_SK BETWEEN 
            CAST(REPLACE(@StartDate, '-', '') AS int) AND
            CAST(REPLACE(@EndDate, '-', '') AS int)

        UNION ALL

        -- Vendas online
        SELECT
            Date_SK,
            Product_SK,
            Store_SK,
            Channel_SK,
            0,
            0,
            Quantity as OnlineQuantity,
            Amount as OnlineAmount,
            0,
            0
        FROM fact.OnlineSales
        WHERE Date_SK BETWEEN 
            CAST(REPLACE(@StartDate, '-', '') AS int) AND
            CAST(REPLACE(@EndDate, '-', '') AS int)

        UNION ALL

        -- Vendas atacado
        SELECT
            Date_SK,
            Product_SK,
            Store_SK,
            Channel_SK,
            0,
            0,
            0,
            0,
            Quantity as WholesaleQuantity,
            Amount as WholesaleAmount
        FROM fact.WholesaleSales
        WHERE Date_SK BETWEEN 
            CAST(REPLACE(@StartDate, '-', '') AS int) AND
            CAST(REPLACE(@EndDate, '-', '') AS int)
    )
    -- Agregar e consolidar
    MERGE fact.SalesConsolidated AS target
    USING (
        SELECT
            Date_SK,
            Product_SK,
            Store_SK,
            Channel_SK,
            SUM(RetailQuantity) as RetailQuantity,
            SUM(RetailAmount) as RetailAmount,
            SUM(OnlineQuantity) as OnlineQuantity,
            SUM(OnlineAmount) as OnlineAmount,
            SUM(WholesaleQuantity) as WholesaleQuantity,
            SUM(WholesaleAmount) as WholesaleAmount,
            SUM(RetailQuantity + OnlineQuantity + WholesaleQuantity) as TotalQuantity,
            SUM(RetailAmount + OnlineAmount + WholesaleAmount) as TotalAmount
        FROM SalesUnified
        GROUP BY
            Date_SK,
            Product_SK,
            Store_SK,
            Channel_SK
    ) AS source
    ON  target.Date_SK = source.Date_SK
        AND target.Product_SK = source.Product_SK
        AND target.Store_SK = source.Store_SK
        AND target.Channel_SK = source.Channel_SK
    
    WHEN MATCHED THEN
        UPDATE SET
            RetailQuantity = source.RetailQuantity,
            RetailAmount = source.RetailAmount,
            OnlineQuantity = source.OnlineQuantity,
            OnlineAmount = source.OnlineAmount,
            WholesaleQuantity = source.WholesaleQuantity,
            WholesaleAmount = source.WholesaleAmount,
            TotalQuantity = source.TotalQuantity,
            TotalAmount = source.TotalAmount,
            LastUpdated = @Now
    
    WHEN NOT MATCHED THEN
        INSERT (
            Date_SK, Product_SK, Store_SK, Channel_SK,
            RetailQuantity, RetailAmount,
            OnlineQuantity, OnlineAmount,
            WholesaleQuantity, WholesaleAmount,
            TotalQuantity, TotalAmount,
            LastUpdated, SourceSystem
        )
        VALUES (
            source.Date_SK, source.Product_SK, 
            source.Store_SK, source.Channel_SK,
            source.RetailQuantity, source.RetailAmount,
            source.OnlineQuantity, source.OnlineAmount,
            source.WholesaleQuantity, source.WholesaleAmount,
            source.TotalQuantity, source.TotalAmount,
            @Now, 'Consolidation'
        );
END;
```

## Exerc