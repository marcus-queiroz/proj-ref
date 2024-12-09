# Aula 10 - Modelagem para Casos Específicos

## Objetivos da Aula
- Desenvolver modelos para área financeira
- Implementar modelagem para RH
- Criar modelos para vendas e marketing
- Compreender padrões por indústria
- Implementar boas práticas específicas

## 1. Modelagem Financeira

### 1.1 Dimensões Financeiras Principais

```sql
-- Dimensão Plano de Contas
CREATE TABLE dim.ChartOfAccounts (
    Account_SK bigint IDENTITY(1,1),
    Account_BK varchar(20) NOT NULL,
    AccountCode varchar(50) NOT NULL,
    AccountName nvarchar(100) NOT NULL,
    AccountType varchar(20) NOT NULL,
    AccountGroup varchar(50) NOT NULL,
    AccountLevel int NOT NULL,
    ParentAccount_SK bigint NULL,
    IsActive bit NOT NULL,
    
    CONSTRAINT PK_ChartOfAccounts PRIMARY KEY (Account_SK),
    CONSTRAINT FK_ChartOfAccounts_Parent 
        FOREIGN KEY (ParentAccount_SK) 
        REFERENCES dim.ChartOfAccounts(Account_SK)
);

-- Dimensão Centro de Custo
CREATE TABLE dim.CostCenter (
    CostCenter_SK bigint IDENTITY(1,1),
    CostCenter_BK varchar(20) NOT NULL,
    CostCenterCode varchar(20) NOT NULL,
    CostCenterName nvarchar(100) NOT NULL,
    Department nvarchar(50) NOT NULL,
    ResponsiblePerson nvarchar(100) NOT NULL,
    BudgetType varchar(20) NOT NULL,
    
    CONSTRAINT PK_CostCenter PRIMARY KEY (CostCenter_SK)
);
```

### 1.2 Fatos Financeiros

```sql
-- Fato Lançamentos Contábeis
CREATE TABLE fact.GeneralLedger (
    Transaction_SK bigint IDENTITY(1,1),
    Date_SK int NOT NULL,
    Account_SK bigint NOT NULL,
    CostCenter_SK bigint NOT NULL,
    Currency_SK int NOT NULL,
    
    TransactionNumber varchar(50) NOT NULL,
    TransactionType varchar(20) NOT NULL,
    DebitAmount decimal(18,2) NOT NULL,
    CreditAmount decimal(18,2) NOT NULL,
    ExchangeRate decimal(18,6) NOT NULL,
    
    SourceSystem varchar(20) NOT NULL,
    LoadDate datetime2 NOT NULL,
    
    CONSTRAINT PK_GeneralLedger PRIMARY KEY (Transaction_SK)
);

-- Fato Orçamento
CREATE TABLE fact.Budget (
    Budget_SK bigint IDENTITY(1,1),
    Date_SK int NOT NULL,
    Account_SK bigint NOT NULL,
    CostCenter_SK bigint NOT NULL,
    Version_SK int NOT NULL,
    
    PlannedAmount decimal(18,2) NOT NULL,
    ActualAmount decimal(18,2) NULL,
    Variance decimal(18,2) NULL,
    VariancePercentage decimal(5,2) NULL,
    
    CONSTRAINT PK_Budget PRIMARY KEY (Budget_SK)
);
```

### 1.3 Análises Financeiras

```sql
-- View para Demonstrativo de Resultados
CREATE VIEW analysis.IncomeStatement
AS
WITH AccountBalances AS (
    SELECT 
        d.Year,
        d.Month,
        a.AccountType,
        a.AccountGroup,
        SUM(gl.DebitAmount - gl.CreditAmount) as Balance
    FROM fact.GeneralLedger gl
    JOIN dim.Date d ON gl.Date_SK = d.Date_SK
    JOIN dim.ChartOfAccounts a ON gl.Account_SK = a.Account_SK
    GROUP BY
        d.Year,
        d.Month,
        a.AccountType,
        a.AccountGroup
)
SELECT
    Year,
    Month,
    AccountType,
    AccountGroup,
    Balance,
    SUM(Balance) OVER (
        PARTITION BY Year, Month 
        ORDER BY AccountType, AccountGroup
        ROWS UNBOUNDED PRECEDING
    ) as RunningTotal
FROM AccountBalances;
```

## 2. Modelagem para RH

### 2.1 Dimensões de RH

```sql
-- Dimensão Funcionário
CREATE TABLE dim.Employee (
    Employee_SK bigint IDENTITY(1,1),
    Employee_BK varchar(20) NOT NULL,
    FirstName nvarchar(50) NOT NULL,
    LastName nvarchar(50) NOT NULL,
    BirthDate date NOT NULL,
    Gender char(1) NOT NULL,
    MaritalStatus char(1) NOT NULL,
    
    -- Atributos profissionais
    Position_SK bigint NOT NULL,
    Department_SK bigint NOT NULL,
    HireDate date NOT NULL,
    TerminationDate date NULL,
    EmploymentStatus varchar(20) NOT NULL,
    EmploymentType varchar(20) NOT NULL,
    
    -- Atributos de remuneração
    PayGrade varchar(10) NOT NULL,
    BaseSalary decimal(18,2) NOT NULL,
    
    -- Controle temporal
    Valid_From datetime2 NOT NULL,
    Valid_To datetime2 NULL,
    Current_FLG bit NOT NULL,
    
    CONSTRAINT PK_Employee PRIMARY KEY (Employee_SK)
);

-- Dimensão Cargo
CREATE TABLE dim.Position (
    Position_SK bigint IDENTITY(1,1),
    Position_BK varchar(20) NOT NULL,
    PositionTitle nvarchar(100) NOT NULL,
    JobFamily varchar(50) NOT NULL,
    JobLevel int NOT NULL,
    RequiredEducation varchar(50) NOT NULL,
    MinSalary decimal(18,2) NOT NULL,
    MaxSalary decimal(18,2) NOT NULL,
    
    CONSTRAINT PK_Position PRIMARY KEY (Position_SK)
);
```

### 2.2 Fatos de RH

```sql
-- Fato Folha de Pagamento
CREATE TABLE fact.Payroll (
    Payroll_SK bigint IDENTITY(1,1),
    Date_SK int NOT NULL,
    Employee_SK bigint NOT NULL,
    Position_SK bigint NOT NULL,
    
    BaseSalary decimal(18,2) NOT NULL,
    Overtime decimal(18,2) NOT NULL,
    Bonuses decimal(18,2) NOT NULL,
    Commissions decimal(18,2) NOT NULL,
    Benefits decimal(18,2) NOT NULL,
    Deductions decimal(18,2) NOT NULL,
    TaxesWithheld decimal(18,2) NOT NULL,
    NetPay decimal(18,2) NOT NULL,
    
    CONSTRAINT PK_Payroll PRIMARY KEY (Payroll_SK)
);

-- Fato Avaliações de Desempenho
CREATE TABLE fact.Performance (
    Performance_SK bigint IDENTITY(1,1),
    Date_SK int NOT NULL,
    Employee_SK bigint NOT NULL,
    Evaluator_SK bigint NOT NULL,
    
    OverallScore decimal(3,2) NOT NULL,
    GoalsScore decimal(3,2) NOT NULL,
    SkillsScore decimal(3,2) NOT NULL,
    CompetenciesScore decimal(3,2) NOT NULL,
    Comments nvarchar(max) NULL,
    
    CONSTRAINT PK_Performance PRIMARY KEY (Performance_SK)
);
```

### 2.3 Análises de RH

```sql
-- View para Análise de Turnover
CREATE VIEW analysis.EmployeeTurnover
AS
WITH EmployeeStatus AS (
    SELECT 
        d.Year,
        d.Month,
        dep.DepartmentName,
        COUNT(DISTINCT CASE 
            WHEN e.HireDate <= EOMONTH(DATEFROMPARTS(d.Year, d.Month, 1))
            AND (e.TerminationDate IS NULL 
                OR e.TerminationDate > EOMONTH(DATEFROMPARTS(d.Year, d.Month, 1)))
            THEN e.Employee_SK 
        END) as ActiveEmployees,
        COUNT(DISTINCT CASE 
            WHEN e.HireDate BETWEEN DATEFROMPARTS(d.Year, d.Month, 1)
            AND EOMONTH(DATEFROMPARTS(d.Year, d.Month, 1))
            THEN e.Employee_SK 
        END) as NewHires,
        COUNT(DISTINCT CASE 
            WHEN e.TerminationDate BETWEEN DATEFROMPARTS(d.Year, d.Month, 1)
            AND EOMONTH(DATEFROMPARTS(d.Year, d.Month, 1))
            THEN e.Employee_SK 
        END) as Terminations
    FROM dim.Date d
    CROSS JOIN dim.Department dep
    LEFT JOIN dim.Employee e ON e.Department_SK = dep.Department_SK
    GROUP BY
        d.Year,
        d.Month,
        dep.DepartmentName
)
SELECT
    Year,
    Month,
    DepartmentName,
    ActiveEmployees,
    NewHires,
    Terminations,
    CAST(CAST(Terminations as float) / NULLIF(ActiveEmployees, 0) * 100 as decimal(5,2)) as TurnoverRate
FROM EmployeeStatus;
```

## 3. Modelagem para Vendas e Marketing

### 3.1 Dimensões de Vendas

```sql
-- Dimensão Cliente
CREATE TABLE dim.Customer (
    Customer_SK bigint IDENTITY(1,1),
    Customer_BK varchar(20) NOT NULL,
    CustomerName nvarchar(100) NOT NULL,
    CustomerType varchar(20) NOT NULL,
    Segment varchar(50) NOT NULL,
    
    -- Informações de contato
    Email varchar(100) NULL,
    Phone varchar(20) NULL,
    
    -- Endereço
    Address nvarchar(200) NULL,
    City nvarchar(100) NULL,
    State nvarchar(50) NULL,
    Country nvarchar(50) NULL,
    PostalCode varchar(20) NULL,
    
    -- Classificação
    CustomerRating int NULL,
    CreditLimit decimal(18,2) NULL,
    AccountManager nvarchar(100) NULL,
    
    -- Controle temporal
    Valid_From datetime2 NOT NULL,
    Valid_To datetime2 NULL,
    Current_FLG bit NOT NULL,
    
    CONSTRAINT PK_Customer PRIMARY KEY (Customer_SK)
);

-- Dimensão Produto
CREATE TABLE dim.Product (
    Product_SK bigint IDENTITY(1,1),
    Product_BK varchar(20) NOT NULL,
    ProductName nvarchar(100) NOT NULL,
    
    -- Categorização
    Category_SK bigint NOT NULL,
    Brand varchar(50) NOT NULL,
    
    -- Atributos
    UnitPrice decimal(18,2) NOT NULL,
    UnitCost decimal(18,2) NOT NULL,
    Weight decimal(10,2) NULL,
    Size varchar(20) NULL,
    Color varchar(20) NULL,
    
    -- Status
    Status varchar(20) NOT NULL,
    LaunchDate date NULL,
    DiscontinuedDate date NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (Product_SK)
);
```

### 3.2 Fatos de Vendas

```sql
-- Fato Vendas
CREATE TABLE fact.Sales (
    Sale_SK bigint IDENTITY(1,1),
    Date_SK int NOT NULL,
    Customer_SK bigint NOT NULL,
    Product_SK bigint NOT NULL,
    SalesRep_SK bigint NOT NULL,
    Territory_SK bigint NOT NULL,
    
    -- Medidas
    Quantity int NOT NULL,
    UnitPrice decimal(18,2) NOT NULL,
    Discount decimal(18,2) NOT NULL,
    SalesAmount decimal(18,2) NOT NULL,
    Cost decimal(18,2) NOT NULL,
    Margin decimal(18,2) NOT NULL,
    
    -- Atributos
    OrderNumber varchar(20) NOT NULL,
    InvoiceNumber varchar(20) NULL,
    
    CONSTRAINT PK_Sales PRIMARY KEY (Sale_SK)
);

-- Fato Campanhas de Marketing
CREATE TABLE fact.MarketingCampaign (
    Campaign_SK bigint IDENTITY(1,1),
    Date_SK int NOT NULL,
    Product_SK bigint NOT NULL,
    Channel_SK bigint NOT NULL,
    Territory_SK bigint NOT NULL,
    
    -- Medidas
    CampaignCost decimal(18,2) NOT NULL,
    Impressions int NOT NULL,
    Clicks int NOT NULL,
    Conversions int NOT NULL,
    Revenue decimal(18,2) NOT NULL,
    ROI decimal(18,2) NOT NULL,
    
    CONSTRAINT PK_MarketingCampaign PRIMARY KEY (Campaign_SK)
);
```

### 3.3 Análises de Vendas

```sql
-- View para Análise de Vendas
CREATE VIEW analysis.SalesPerformance
AS
WITH SalesMetrics AS (
    SELECT 
        d.Year,
        d.Month,
        p.Category_SK,
        c.Segment,
        SUM(s.Quantity) as TotalQuantity,
        SUM(s.SalesAmount) as TotalSales,
        SUM(s.Margin) as TotalMargin,
        COUNT(DISTINCT c.Customer_SK) as UniqueCustomers,
        COUNT(DISTINCT s.OrderNumber) as NumberOfOrders
    FROM fact.Sales s
    JOIN dim.Date d ON s.Date_SK = d.Date_SK
    JOIN dim.Product p ON s.Product_SK = p.Product_SK
    JOIN dim.Customer c ON s.Customer_SK = c.Customer_SK
    GROUP BY
        d.Year,
        d.Month,
        p.Category_SK,
        c.Segment
)
SELECT
    Year,
    Month,
    Category_SK,
    Segment,
    TotalQuantity,
    TotalSales,
    TotalMargin,
    UniqueCustomers,
    NumberOfOrders,
    TotalSales / NULLIF(UniqueCustomers, 0) as AverageCustomerValue,
    TotalSales / NULLIF(NumberOfOrders, 0) as AverageOrderValue,
    TotalMargin / NULLIF(TotalSales, 0) * 100 as MarginPercentage
FROM SalesMetrics;
```

## 4. Padrões por Indústria

### 4.1 Varejo
- Gestão de produtos e categorias
- Análise de vendas por canal
- Gestão de estoque
- Comportamento do consumidor

### 4.2 Manufatura
- Controle de produção
- Gestão de qualidade
- Planejamento de materiais
- Análise de eficiência

### 4.3 Serviços
- Gestão de contratos
- Análise de satisfação
- Indicadores de serviço
- Gestão de recursos

## Exercícios Práticos

### Exercício 1: Financeiro
1. Implemente um modelo para:
   - Controle or