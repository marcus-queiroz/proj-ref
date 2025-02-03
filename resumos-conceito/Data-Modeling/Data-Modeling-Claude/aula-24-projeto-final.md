# Aula 24 - Projeto Final

## Objetivos da Aula
- Implementação completa de um DW
- Modelagem dimensional
- Desenvolvimento ETL
- Otimização
- Documentação

## 1. Especificação do Projeto

### 1.1 Requisitos do Negócio
```sql
-- Tabela de requisitos
CREATE TABLE project.Requirements (
    Requirement_SK bigint IDENTITY(1,1),
    RequirementType varchar(50) NOT NULL,
    Description nvarchar(500) NOT NULL,
    Priority int NOT NULL,
    Status varchar(20) NOT NULL,
    CONSTRAINT PK_Requirements PRIMARY KEY (Requirement_SK)
);

-- Inserir requisitos principais
INSERT INTO project.Requirements (
    RequirementType,
    Description,
    Priority,
    Status
)
VALUES
('Vendas', 'Análise de vendas por produto, região e tempo', 1, 'Approved'),
('Financeiro', 'Análise de margem e lucratividade', 1, 'Approved'),
('Cliente', 'Segmentação e análise de comportamento', 2, 'Approved'),
('Estoque', 'Controle de estoque e giro', 2, 'Approved'),
('Marketing', 'Análise de campanhas e retorno', 3, 'Approved');
```

### 1.2 Estrutura de Dados
```sql
-- Mapeamento de fontes
CREATE TABLE project.SourceSystems (
    Source_SK bigint IDENTITY(1,1),
    SourceSystem varchar(50) NOT NULL,
    ConnectionString nvarchar(500) NOT NULL,
    UpdateFrequency varchar(20) NOT NULL,
    LastUpdate datetime2 NULL,
    Status varchar(20) NOT NULL,
    CONSTRAINT PK_SourceSystems PRIMARY KEY (Source_SK)
);

-- Mapeamento de tabelas
CREATE TABLE project.SourceTables (
    SourceTable_SK bigint IDENTITY(1,1),
    Source_SK bigint NOT NULL,
    SchemaName varchar(50) NOT NULL,
    TableName varchar(100) NOT NULL,
    LoadType varchar(20) NOT NULL,
    Filter nvarchar(500) NULL,
    CONSTRAINT PK_SourceTables PRIMARY KEY (SourceTable_SK),
    CONSTRAINT FK_SourceTables_System 
        FOREIGN KEY (Source_SK) 
        REFERENCES project.SourceSystems(Source_SK)
);
```

## 2. Modelagem Dimensional

### 2.1 Dimensões
```sql
-- Dimensão Cliente
CREATE TABLE dim.Customer (
    Customer_SK bigint IDENTITY(1,1),
    Customer_BK varchar(20) NOT NULL,
    CustomerName nvarchar(100) NOT NULL,
    CustomerType varchar(20) NOT NULL,
    Segment varchar(50) NOT NULL,
    Email varchar(100) NULL,
    Phone varchar(20) NULL,
    Address nvarchar(200) NULL,
    City nvarchar(100) NULL,
    State nvarchar(50) NULL,
    Country nvarchar(50) NULL,
    ValidFrom datetime2 NOT NULL,
    ValidTo datetime2 NULL,
    Current_FLG bit NOT NULL,
    CONSTRAINT PK_Customer PRIMARY KEY (Customer_SK)
);

-- Dimensão Produto
CREATE TABLE dim.Product (
    Product_SK bigint IDENTITY(1,1),
    Product_BK varchar(20) NOT NULL,
    ProductName nvarchar(100) NOT NULL,
    Category varchar(50) NOT NULL,
    Subcategory varchar(50) NOT NULL,
    Brand varchar(50) NOT NULL,
    UnitPrice decimal(18,2) NOT NULL,
    UnitCost decimal(18,2) NOT NULL,
    ValidFrom datetime2 NOT NULL,
    ValidTo datetime2 NULL,
    Current_FLG bit NOT NULL,
    CONSTRAINT PK_Product PRIMARY KEY (Product_SK)
);

-- Dimensão Tempo
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
    YearMonth int NOT NULL,
    IsWeekend bit NOT NULL,
    IsHoliday bit NOT NULL,
    CONSTRAINT PK_Date PRIMARY KEY (Date_SK)
);
```

### 2.2 Fatos
```sql
-- Fato Vendas
CREATE TABLE fact.Sales (
    Sale_SK bigint IDENTITY(1,1),
    Date_SK int NOT NULL,
    Customer_SK bigint NOT NULL,
    Product_SK bigint NOT NULL,
    Quantity int NOT NULL,
    UnitPrice decimal(18,2) NOT NULL,
    UnitCost decimal(18,2) NOT NULL,
    DiscountAmount decimal(18,2) NOT NULL,
    SalesAmount decimal(18,2) NOT NULL,
    CostAmount decimal(18,2) NOT NULL,
    MarginAmount decimal(18,2) NOT NULL,
    CONSTRAINT PK_Sales PRIMARY KEY (Sale_SK),
    CONSTRAINT FK_Sales_Date 
        FOREIGN KEY (Date_SK) 
        REFERENCES dim.Date(Date_SK),
    CONSTRAINT FK_Sales_Customer 
        FOREIGN KEY (Customer_SK) 
        REFERENCES dim.Customer(Customer_SK),
    CONSTRAINT FK_Sales_Product 
        FOREIGN KEY (Product_SK) 
        REFERENCES dim.Product(Product_SK)
) ON PS_Sales(Date_SK);

-- Fato Estoque
CREATE TABLE fact.Inventory (
    Inventory_SK bigint IDENTITY(1,1),
    Date_SK int NOT NULL,
    Product_SK bigint NOT NULL,
    OpeningQuantity int NOT NULL,
    PurchaseQuantity int NOT NULL,
    SalesQuantity int NOT NULL,
    AdjustmentQuantity int NOT NULL,
    ClosingQuantity int NOT NULL,
    InventoryValue decimal(18,2) NOT NULL,
    CONSTRAINT PK_Inventory PRIMARY KEY (Inventory_SK),
    CONSTRAINT FK_Inventory_Date 
        FOREIGN KEY (Date_SK) 
        REFERENCES dim.Date(Date_SK),
    CONSTRAINT FK_Inventory_Product 
        FOREIGN KEY (Product_SK) 
        REFERENCES dim.Product(Product_SK)
) ON PS_Inventory(Date_SK);
```

## 3. Processos ETL

### 3.1 Stage Layer
```sql
-- Criação de staging
CREATE PROCEDURE etl.usp_CreateStaging
AS
BEGIN
    -- Stage Cliente
    CREATE TABLE stg.Customer (
        Customer_BK varchar(20) NOT NULL,
        CustomerName nvarchar(100) NOT NULL,
        CustomerType varchar(20) NOT NULL,
        Segment varchar(50) NOT NULL,
        Email varchar(100) NULL,
        Phone varchar(20) NULL,
        Address nvarchar(200) NULL,
        City nvarchar(100) NULL,
        State nvarchar(50) NULL,
        Country nvarchar(50) NULL,
        LoadDate datetime2 NOT NULL
    );

    -- Stage Produto
    CREATE TABLE stg.Product (
        Product_BK varchar(20) NOT NULL,
        ProductName nvarchar(100) NOT NULL,
        Category varchar(50) NOT NULL,
        Subcategory varchar(50) NOT NULL,
        Brand varchar(50) NOT NULL,
        UnitPrice decimal(18,2) NOT NULL,
        UnitCost decimal(18,2) NOT NULL,
        LoadDate datetime2 NOT NULL
    );

    -- Stage Vendas
    CREATE TABLE stg.Sales (
        Customer_BK varchar(20) NOT NULL,
        Product_BK varchar(20) NOT NULL,
        SaleDate date NOT NULL,
        Quantity int NOT NULL,
        UnitPrice decimal(18,2) NOT NULL,
        UnitCost decimal(18,2) NOT NULL,
        DiscountAmount decimal(18,2) NOT NULL,
        LoadDate datetime2 NOT NULL
    );
END;
```

### 3.2 Carga Dimensional
```sql
-- Procedure para carga de dimensões
CREATE PROCEDURE etl.usp_LoadDimensions
    @LoadType varchar(10) = 'FULL'
AS
BEGIN
    -- Carregar dimensão cliente (SCD Tipo 2)
    MERGE dim.Customer as target
    USING stg.Customer as source
    ON target.Customer_BK = source.Customer_BK
        AND target.Current_FLG = 1

    WHEN MATCHED AND (
        target.CustomerName <> source.CustomerName OR
        target.CustomerType <> source.CustomerType OR
        target.Segment <> source.Segment OR
        ISNULL(target.Email,'') <> ISNULL(source.Email,'') OR
        ISNULL(target.Phone,'') <> ISNULL(source.Phone,'') OR
        ISNULL(target.Address,'') <> ISNULL(source.Address,'') OR
        ISNULL(target.City,'') <> ISNULL(source.City,'') OR
        ISNULL(target.State,'') <> ISNULL(source.State,'') OR
        ISNULL(target.Country,'') <> ISNULL(source.Country,'')
    ) THEN
        UPDATE SET
            ValidTo = GETDATE(),
            Current_FLG = 0

    WHEN NOT MATCHED BY TARGET THEN
        INSERT (
            Customer_BK,
            CustomerName,
            CustomerType,
            Segment,
            Email,
            Phone,
            Address,
            City,
            State,
            Country,
            ValidFrom,
            ValidTo,
            Current_FLG
        )
        VALUES (
            source.Customer_BK,
            source.CustomerName,
            source.CustomerType,
            source.Segment,
            source.Email,
            source.Phone,
            source.Address,
            source.City,
            source.State,
            source.Country,
            GETDATE(),
            NULL,
            1
        );

    -- Inserir novos registros para mudanças
    INSERT INTO dim.Customer (
        Customer_BK,
        CustomerName,
        CustomerType,
        Segment,
        Email,
        Phone,
        Address,
        City,
        State,
        Country,
        ValidFrom,
        ValidTo,
        Current_FLG
    )
    SELECT 
        source.Customer_BK,
        source.CustomerName,
        source.CustomerType,
        source.Segment,
        source.Email,
        source.Phone,
        source.Address,
        source.City,
        source.State,
        source.Country,
        GETDATE(),
        NULL,
        1
    FROM stg.Customer source
    JOIN dim.Customer target 
        ON source.Customer_BK = target.Customer_BK
    WHERE target.Current_FLG = 0
    AND target.ValidTo = GETDATE();
END;
```

### 3.3 Carga de Fatos
```sql
-- Procedure para carga de fatos
CREATE PROCEDURE etl.usp_LoadFacts
    @StartDate date,
    @EndDate date
AS
BEGIN
    -- Carregar fato vendas
    INSERT INTO fact.Sales (
        Date_SK,
        Customer_SK,
        Product_SK,
        Quantity,
        UnitPrice,
        UnitCost,
        DiscountAmount,
        SalesAmount,
        CostAmount,
        MarginAmount
    )
    SELECT 
        d.Date_SK,
        c.Customer_SK,
        p.Product_SK,
        s.Quantity,
        s.UnitPrice,
        s.UnitCost,
        s.DiscountAmount,
        (s.Quantity * s.UnitPrice) - s.DiscountAmount,
        (s.Quantity * s.UnitCost),
        (s.Quantity * s.UnitPrice) - s.DiscountAmount - 
            (s.Quantity * s.UnitCost)
    FROM stg.Sales s
    JOIN dim.Date d ON s.SaleDate = d.FullDate
    JOIN dim.Customer c 
        ON s.Customer_BK = c.Customer_BK
        AND s.SaleDate BETWEEN c.ValidFrom AND ISNULL(c.ValidTo, '9999-12-31')
    JOIN dim.Product p 
        ON s.Product_BK = p.Product_BK
        AND s.SaleDate BETWEEN p.ValidFrom AND ISNULL(p.ValidTo, '9999-12-31')
    WHERE s.SaleDate BETWEEN @StartDate AND @EndDate;
END;
```

## 4. Otimização

### 4.1 Índices e Particionamento
```sql
-- Configurar particionamento
CREATE PARTITION FUNCTION PF_Date (int)
AS RANGE RIGHT FOR VALUES 
(20240101, 20240201, 20240301, 20240401);

CREATE PARTITION SCHEME PS_Sales
AS PARTITION PF_Date
ALL TO ([PRIMARY]);

-- Criar índices
CREATE CLUSTERED COLUMNSTORE INDEX CCI_Sales
ON fact.Sales;

CREATE NONCLUSTERED INDEX IX_Sales_Date
ON fact.Sales(Date_SK)
INCLUDE (SalesAmount, MarginAmount);

CREATE NONCLUSTERED INDEX IX_Sales_Customer
ON fact.Sales(Customer_SK)
INCLUDE (SalesAmount, MarginAmount);
```

### 4.2 Agregações
```sql
-- Criar views agregadas
CREATE VIEW mart.SalesByMonth
WITH SCHEMABINDING
AS
SELECT 
    d.Year,
    d.Month,
    COUNT_BIG(*) as TransactionCount,
    SUM(s.SalesAmount) as TotalSales,
    SUM(s.MarginAmount) as TotalMargin
FROM fact.Sales s
JOIN dim.Date d ON s.Date_SK = d.Date_SK
GROUP BY
    d.Year,
    d.Month;

CREATE UNIQUE CLUSTERED INDEX IX_SalesByMonth
ON mart.SalesByMonth(Year, Month);
```

## 5. Documentação

### 5.1 Dicionário de Dados
```sql
-- Criar dicionário de dados
CREATE TABLE meta.DataDictionary (
    Table_SK bigint IDENTITY(1,1),
    SchemaName varchar(50) NOT NULL,
    TableName varchar(100) NOT NULL,
    ColumnName varchar(100) NOT NULL,
    DataType varchar(50) NOT NULL,
    Description nvarchar(500) NOT NULL,
    BusinessRule nvarchar(500) NULL,
    CONSTRAINT PK_DataDictionary PRIMARY KEY (Table_SK)
);

-- Procedure para documentação
CREATE PROCEDURE meta.usp_DocumentTable
    @SchemaName varchar(50),
    @TableName varchar(100)
AS
BEGIN
    INSERT INTO meta.DataDictionary (
        SchemaName,
        TableName,
        ColumnName,
        DataType,
        Description
    )
    SELECT 
        @SchemaName,
        @TableName,
        c.name,
        t.name + 
            CASE 
                WHEN t.name IN ('varchar','n