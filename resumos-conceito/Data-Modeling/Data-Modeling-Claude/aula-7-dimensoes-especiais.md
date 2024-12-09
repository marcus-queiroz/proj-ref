# Aula 7 - Tipos Especiais de Dimensões

## Objetivos da Aula
- Compreender e implementar Role-Playing Dimensions
- Desenvolver Junk Dimensions
- Trabalhar com Dimensões Degeneradas
- Implementar Dimensões Conformadas
- Entender Dimensões Ragged e Snowflaked

## 1. Role-Playing Dimensions

### 1.1 Conceito
- Mesma dimensão usada em diferentes contextos
- Reutilização de estrutura
- Diferentes perspectivas analíticas
- Economia de espaço e manutenção

### 1.2 Implementação com Views
```sql
-- Dimensão base de data
CREATE TABLE dim.Date (
    Date_SK int NOT NULL,
    FullDate date NOT NULL,
    DayOfWeek tinyint NOT NULL,
    DayName varchar(10) NOT NULL,
    DayOfMonth tinyint NOT NULL,
    Month tinyint NOT NULL,
    MonthName varchar(10) NOT NULL,
    Quarter tinyint NOT NULL,
    Year smallint NOT NULL,
    CONSTRAINT PK_Date PRIMARY KEY (Date_SK)
);

-- Views para diferentes papéis
CREATE VIEW dim.OrderDate AS
SELECT 
    Date_SK as OrderDate_SK,
    FullDate as OrderFullDate,
    DayOfWeek as OrderDayOfWeek,
    DayName as OrderDayName,
    Month as OrderMonth,
    MonthName as OrderMonthName,
    Quarter as OrderQuarter,
    Year as OrderYear
FROM dim.Date;

CREATE VIEW dim.ShipDate AS
SELECT 
    Date_SK as ShipDate_SK,
    FullDate as ShipFullDate,
    DayOfWeek as ShipDayOfWeek,
    DayName as ShipDayName,
    Month as ShipMonth,
    MonthName as ShipMonthName,
    Quarter as ShipQuarter,
    Year as ShipYear
FROM dim.Date;

-- Exemplo de uso em fato
CREATE TABLE fact.Sales (
    Sale_SK bigint IDENTITY(1,1),
    OrderDate_SK int NOT NULL,
    ShipDate_SK int NULL,
    -- outras colunas
    CONSTRAINT FK_Sales_OrderDate FOREIGN KEY (OrderDate_SK)
        REFERENCES dim.Date (Date_SK),
    CONSTRAINT FK_Sales_ShipDate FOREIGN KEY (ShipDate_SK)
        REFERENCES dim.Date (Date_SK)
);
```

## 2. Junk Dimensions

### 2.1 Conceito
- Combinação de atributos de baixa cardinalidade
- Redução de dimensões
- Otimização de espaço
- Simplificação do modelo

### 2.2 Implementação
```sql
-- Criação da Junk Dimension
CREATE TABLE dim.OrderAttributes (
    OrderAttribute_SK int IDENTITY(1,1),
    OrderPriority varchar(10) NOT NULL,
    ShipMode varchar(20) NOT NULL,
    OrderStatus varchar(15) NOT NULL,
    PaymentMethod varchar(20) NOT NULL,
    CONSTRAINT PK_OrderAttributes PRIMARY KEY (OrderAttribute_SK)
);

-- Procedimento para popular/manter Junk Dimension
CREATE PROCEDURE dim.usp_MaintainOrderAttributes
AS
BEGIN
    SET NOCOUNT ON;

    -- Inserir novas combinações da staging
    INSERT INTO dim.OrderAttributes (
        OrderPriority, ShipMode, OrderStatus, PaymentMethod
    )
    SELECT DISTINCT
        s.OrderPriority,
        s.ShipMode,
        s.OrderStatus,
        s.PaymentMethod
    FROM stg.Orders s
    LEFT JOIN dim.OrderAttributes d
        ON s.OrderPriority = d.OrderPriority
        AND s.ShipMode = d.ShipMode
        AND s.OrderStatus = d.OrderStatus
        AND s.PaymentMethod = d.PaymentMethod
    WHERE d.OrderAttribute_SK IS NULL;
END;

-- Uso na tabela fato
CREATE TABLE fact.Sales (
    Sale_SK bigint IDENTITY(1,1),
    OrderAttribute_SK int NOT NULL,
    -- outras colunas
    CONSTRAINT FK_Sales_OrderAttributes FOREIGN KEY (OrderAttribute_SK)
        REFERENCES dim.OrderAttributes (OrderAttribute_SK)
);
```

## 3. Dimensões Degeneradas

### 3.1 Conceito
- Atributos dimensionais sem dimensão física
- Normalmente identificadores de transação
- Economia de joins
- Simplificação do modelo

### 3.2 Implementação
```sql
CREATE TABLE fact.Sales (
    Sale_SK bigint IDENTITY(1,1),
    OrderNumber varchar(20) NOT NULL,  -- Dimensão degenerada
    InvoiceNumber varchar(20) NOT NULL,  -- Dimensão degenerada
    -- outras colunas
    CONSTRAINT PK_Sales PRIMARY KEY (Sale_SK)
);

-- Índices para suporte a análises
CREATE NONCLUSTERED INDEX IX_Sales_OrderNumber
ON fact.Sales(OrderNumber)
INCLUDE (Sale_SK);

CREATE NONCLUSTERED INDEX IX_Sales_InvoiceNumber
ON fact.Sales(InvoiceNumber)
INCLUDE (Sale_SK);
```

## 4. Dimensões Conformadas

### 4.1 Conceito
- Dimensões compartilhadas entre fatos
- Padronização de análises
- Consistência de informações
- Facilidade de manutenção

### 4.2 Implementação
```sql
-- Dimensão cliente conformada
CREATE TABLE dim.Customer (
    Customer_SK bigint IDENTITY(1,1),
    Customer_BK varchar(20) NOT NULL,
    CustomerName nvarchar(100) NOT NULL,
    CustomerType varchar(20) NOT NULL,
    CustomerSegment varchar(30) NOT NULL,
    -- outros atributos
    CONSTRAINT PK_Customer PRIMARY KEY (Customer_SK)
);

-- Uso em diferentes fatos
CREATE TABLE fact.Sales (
    Sale_SK bigint IDENTITY(1,1),
    Customer_SK bigint NOT NULL,
    -- outras colunas
    CONSTRAINT FK_Sales_Customer FOREIGN KEY (Customer_SK)
        REFERENCES dim.Customer (Customer_SK)
);

CREATE TABLE fact.ServiceCalls (
    ServiceCall_SK bigint IDENTITY(1,1),
    Customer_SK bigint NOT NULL,
    -- outras colunas
    CONSTRAINT FK_ServiceCalls_Customer FOREIGN KEY (Customer_SK)
        REFERENCES dim.Customer (Customer_SK)
);
```

## 5. Dimensões Ragged e Snowflaked

### 5.1 Dimensões Ragged (Hierarquias Irregulares)
```sql
CREATE TABLE dim.Geography (
    Geography_SK bigint IDENTITY(1,1),
    Geography_BK varchar(20) NOT NULL,
    GeographyName nvarchar(100) NOT NULL,
    GeographyLevel varchar(20) NOT NULL,
    ParentGeography_SK bigint NULL,
    Level1_Name nvarchar(100) NULL,
    Level2_Name nvarchar(100) NULL,
    Level3_Name nvarchar(100) NULL,
    Level4_Name nvarchar(100) NULL,
    CONSTRAINT PK_Geography PRIMARY KEY (Geography_SK),
    CONSTRAINT FK_Geography_Parent FOREIGN KEY (ParentGeography_SK)
        REFERENCES dim.Geography (Geography_SK)
);

-- Procedure para construir hierarquia
CREATE PROCEDURE dim.usp_BuildGeographyHierarchy
AS
BEGIN
    SET NOCOUNT ON;
    
    WITH GeographyCTE AS (
        -- Nível base
        SELECT 
            Geography_SK,
            Geography_BK,
            GeographyName,
            GeographyLevel,
            ParentGeography_SK,
            CAST(GeographyName AS nvarchar(500)) as HierarchyPath,
            1 as Level
        FROM dim.Geography
        WHERE ParentGeography_SK IS NULL

        UNION ALL

        -- Níveis recursivos
        SELECT 
            g.Geography_SK,
            g.Geography_BK,
            g.GeographyName,
            g.GeographyLevel,
            g.ParentGeography_SK,
            CAST(cte.HierarchyPath + ' > ' + g.GeographyName AS nvarchar(500)),
            cte.Level + 1
        FROM dim.Geography g
        INNER JOIN GeographyCTE cte 
            ON g.ParentGeography_SK = cte.Geography_SK
    )
    UPDATE g
    SET 
        Level1_Name = CASE WHEN r.Level = 1 THEN r.GeographyName ELSE g.Level1_Name END,
        Level2_Name = CASE WHEN r.Level = 2 THEN r.GeographyName ELSE g.Level2_Name END,
        Level3_Name = CASE WHEN r.Level = 3 THEN r.GeographyName ELSE g.Level3_Name END,
        Level4_Name = CASE WHEN r.Level = 4 THEN r.GeographyName ELSE g.Level4_Name END
    FROM dim.Geography g
    INNER JOIN GeographyCTE r 
        ON g.Geography_SK = r.Geography_SK;
END;
```

### 5.2 Dimensões Snowflaked
```sql
-- Exemplo de dimensão snowflaked para produto
CREATE TABLE dim.ProductCategory (
    ProductCategory_SK bigint IDENTITY(1,1),
    CategoryName nvarchar(50) NOT NULL,
    CategoryDescription nvarchar(200) NULL,
    CONSTRAINT PK_ProductCategory PRIMARY KEY (ProductCategory_SK)
);

CREATE TABLE dim.ProductSubcategory (
    ProductSubcategory_SK bigint IDENTITY(1,1),
    ProductCategory_SK bigint NOT NULL,
    SubcategoryName nvarchar(50) NOT NULL,
    SubcategoryDescription nvarchar(200) NULL,
    CONSTRAINT PK_ProductSubcategory PRIMARY KEY (ProductSubcategory_SK),
    CONSTRAINT FK_ProductSubcategory_Category 
        FOREIGN KEY (ProductCategory_SK)
        REFERENCES dim.ProductCategory (ProductCategory_SK)
);

CREATE TABLE dim.Product (
    Product_SK bigint IDENTITY(1,1),
    ProductSubcategory_SK bigint NOT NULL,
    ProductName nvarchar(100) NOT NULL,
    -- outros atributos
    CONSTRAINT PK_Product PRIMARY KEY (Product_SK),
    CONSTRAINT FK_Product_Subcategory 
        FOREIGN KEY (ProductSubcategory_SK)
        REFERENCES dim.ProductSubcategory (ProductSubcategory_SK)
);

-- View desnormalizada
CREATE VIEW dim.ProductComplete AS
SELECT 
    p.Product_SK,
    p.ProductName,
    ps.SubcategoryName,
    ps.SubcategoryDescription,
    pc.CategoryName,
    pc.CategoryDescription
FROM dim.Product p
INNER JOIN dim.ProductSubcategory ps 
    ON p.ProductSubcategory_SK = ps.ProductSubcategory_SK
INNER JOIN dim.ProductCategory pc 
    ON ps.ProductCategory_SK = pc.ProductCategory_SK;
```

## Exercícios Práticos

### Exercício 1: Role-Playing Dimensions
1. Implemente dimensão de data
2. Crie diferentes views para papéis
3. Utilize em tabela fato
4. Desenvolva consultas analíticas

### Exercício 2: Junk Dimensions
1. Identifique atributos candidatos
2. Implemente junk dimension
3. Desenvolva processo de manutenção
4. Analise benefícios

### Exercício 3: Dimensões Especiais
1. Implemente hierarquia geográfica
2. Crie dimensão snowflaked
3. Desenvolva consultas complexas
4. Compare performances

## Material Complementar

### Documentação
- Patterns de modelagem dimensional
- Best practices para hierarquias
- Otimização de consultas dimensionais

### Scripts Úteis
- Análise de cardinalidade
- Validação de hierarquias
- Manutenção de dimensões

## Próxima Aula
Na próxima aula, abordaremos Tipos Especiais de Fatos, incluindo bridge tables, factless fact tables e fatos agregados.

## Avaliação
1. Quiz sobre tipos especiais de dimensões
2. Implementação prática
3. Análise de casos de uso