## Exercícios Práticos (Continuação)

### Exercício 1: Financeiro
1. Implemente um modelo para:
   - Controle orçamentário
   - Fluxo de caixa
   - Centro de custos
   
2. Desenvolva as seguintes análises:
```sql
-- Template para análise orçamentária
WITH BudgetAnalysis AS (
    SELECT 
        d.Year,
        d.Month,
        cc.CostCenterName,
        SUM(b.PlannedAmount) as Planned,
        SUM(b.ActualAmount) as Actual,
        SUM(b.Variance) as Variance
    FROM fact.Budget b
    JOIN dim.Date d ON b.Date_SK = d.Date_SK
    JOIN dim.CostCenter cc ON b.CostCenter_SK = cc.CostCenter_SK
    GROUP BY d.Year, d.Month, cc.CostCenterName
)
SELECT * FROM BudgetAnalysis;
```

### Exercício 2: RH
1. Desenvolva um modelo para:
   - Recrutamento e seleção
   - Treinamentos
   - Avaliações de desempenho

2. Crie análises para:
```sql
-- Template para análise de headcount
SELECT 
    d.Year,
    d.Month,
    dep.DepartmentName,
    COUNT(DISTINCT e.Employee_SK) as Headcount,
    SUM(p.BaseSalary) as TotalPayroll,
    AVG(pe.OverallScore) as AvgPerformance
FROM dim.Employee e
JOIN dim.Date d ON e.HireDate <= d.FullDate 
    AND (e.TerminationDate IS NULL OR e.TerminationDate > d.FullDate)
JOIN dim.Department dep ON e.Department_SK = dep.Department_SK
JOIN fact.Payroll p ON e.Employee_SK = p.Employee_SK
JOIN fact.Performance pe ON e.Employee_SK = pe.Employee_SK
GROUP BY d.Year, d.Month, dep.DepartmentName;
```

### Exercício 3: Vendas e Marketing
1. Implemente um modelo para:
   - Análise de vendas multicanal
   - Campanhas de marketing
   - Análise de clientes

2. Desenvolva análises de:
```sql
-- Template para análise de vendas por canal
WITH SalesAnalysis AS (
    SELECT 
        d.Year,
        d.Month,
        ch.ChannelName,
        p.CategoryName,
        COUNT(DISTINCT s.Customer_SK) as UniqueCustomers,
        COUNT(DISTINCT s.OrderNumber) as Orders,
        SUM(s.SalesAmount) as Revenue,
        SUM(s.Margin) as Margin
    FROM fact.Sales s
    JOIN dim.Date d ON s.Date_SK = d.Date_SK
    JOIN dim.Channel ch ON s.Channel_SK = ch.Channel_SK
    JOIN dim.Product p ON s.Product_SK = p.Product_SK
    GROUP BY d.Year, d.Month, ch.ChannelName, p.CategoryName
)
SELECT * FROM SalesAnalysis;
```

## Material Complementar

### 1. Scripts de Validação e Monitoramento

#### 1.1 Validação Financeira
```sql
CREATE PROCEDURE audit.usp_ValidateFinancialData
AS
BEGIN
    -- Verificar balanceamento de débito/crédito
    SELECT 
        Date_SK,
        SUM(DebitAmount) as TotalDebits,
        SUM(CreditAmount) as TotalCredits,
        SUM(DebitAmount) - SUM(CreditAmount) as Difference
    FROM fact.GeneralLedger
    GROUP BY Date_SK
    HAVING ABS(SUM(DebitAmount) - SUM(CreditAmount)) > 0.01;
    
    -- Verificar integridade orçamentária
    SELECT 
        Date_SK,
        Account_SK,
        SUM(PlannedAmount) as TotalPlanned,
        SUM(ActualAmount) as TotalActual
    FROM fact.Budget
    GROUP BY Date_SK, Account_SK
    HAVING ABS(SUM(PlannedAmount)) < ABS(SUM(ActualAmount)) * 2;
END;
```

#### 1.2 Validação de RH
```sql
CREATE PROCEDURE audit.usp_ValidateHRData
AS
BEGIN
    -- Verificar consistência de datas
    SELECT 
        Employee_SK,
        HireDate,
        TerminationDate
    FROM dim.Employee
    WHERE TerminationDate IS NOT NULL 
    AND TerminationDate < HireDate;
    
    -- Verificar consistência salarial
    SELECT 
        e.Employee_SK,
        e.BaseSalary,
        p.MinSalary,
        p.MaxSalary
    FROM dim.Employee e
    JOIN dim.Position p ON e.Position_SK = p.Position_SK
    WHERE e.BaseSalary < p.MinSalary 
    OR e.BaseSalary > p.MaxSalary;
END;
```

#### 1.3 Validação de Vendas
```sql
CREATE PROCEDURE audit.usp_ValidateSalesData
AS
BEGIN
    -- Verificar margens negativas
    SELECT 
        Sale_SK,
        SalesAmount,
        Cost,
        Margin
    FROM fact.Sales
    WHERE Margin < 0;
    
    -- Verificar consistência de preços
    SELECT 
        s.Sale_SK,
        s.UnitPrice,
        p.UnitPrice as ListPrice,
        ABS(s.UnitPrice - p.UnitPrice) / p.UnitPrice * 100 as PriceDiffPercentage
    FROM fact.Sales s
    JOIN dim.Product p ON s.Product_SK = p.Product_SK
    WHERE ABS(s.UnitPrice - p.UnitPrice) / p.UnitPrice > 0.25;
END;
```

### 2. Documentação Específica por Área

#### 2.1 Financeira
- Plano de contas padrão
- Regras de consolidação
- Políticas contábeis
- Requisitos regulatórios

#### 2.2 RH
- Políticas de remuneração
- Estrutura organizacional
- Processos de avaliação
- Requisitos legais

#### 2.3 Vendas
- Política comercial
- Regras de desconto
- Territórios de venda
- Metas e comissionamento

## Próxima Aula
Na próxima aula, começaremos a parte de ETL com SQL Server Integration Services (SSIS), abordando os fundamentos e arquitetura do SSIS.

## Avaliação
1. Quiz sobre modelagem específica por área
2. Projeto prático de implementação
3. Documentação do modelo

### Critérios de Avaliação
- Adequação ao negócio
- Performance das consultas
- Qualidade da documentação
- Uso de boas práticas