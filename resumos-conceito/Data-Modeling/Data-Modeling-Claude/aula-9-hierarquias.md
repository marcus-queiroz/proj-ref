# Aula 9 - Hierarquias e Relacionamentos

## Objetivos da Aula
- Compreender diferentes tipos de hierarquias
- Implementar hierarquias naturais
- Desenvolver hierarquias paralelas
- Trabalhar com hierarquias recursivas
- Implementar navegação em hierarquias

## 1. Hierarquias Naturais

### 1.1 Conceito
- Estruturas hierárquicas fixas
- Níveis predeterminados
- Relacionamentos um-para-muitos
- Granularidade definida

### 1.2 Implementação Básica
```sql
-- Hierarquia Geográfica
CREATE TABLE dim.Geography (
    Geography_SK bigint IDENTITY(1,1),
    
    -- Chaves naturais por nível
    Country_BK varchar(2) NOT NULL,
    Region_BK varchar(5) NOT NULL,
    State_BK varchar(2) NOT NULL,
    City_BK varchar(10) NOT NULL,
    
    -- Nomes e atributos por nível
    CountryName nvarchar(100) NOT NULL,
    RegionName nvarchar(100) NOT NULL,
    StateName nvarchar(100) NOT NULL,
    CityName nvarchar(100) NOT NULL,
    
    -- Atributos adicionais
    Population int NULL,
    TimeZone varchar(50) NULL,
    
    -- Controle
    Created_DT datetime2 NOT NULL,
    Updated_DT datetime2 NOT NULL,
    
    CONSTRAINT PK_Geography PRIMARY KEY (Geography_SK)
);

-- Índices para suporte à navegação
CREATE NONCLUSTERED INDEX IX_Geography_Natural
ON dim.Geography(Country_BK, Region_BK, State_BK, City_BK);

-- View para análise em diferentes níveis
CREATE VIEW analysis.GeographyHierarchy
AS
SELECT DISTINCT
    Country_BK,
    CountryName,
    NULL as Region_BK,
    NULL as RegionName,
    NULL as State_BK,
    NULL as StateName,
    NULL as City_BK,
    NULL as CityName,
    1 as Level
FROM dim.Geography

UNION ALL

SELECT DISTINCT
    Country_BK,
    CountryName,
    Region_BK,
    RegionName,
    NULL,
    NULL,
    NULL,
    NULL,
    2
FROM dim.Geography

UNION ALL

SELECT DISTINCT
    Country_BK,
    CountryName,
    Region_BK,
    RegionName,
    State_BK,
    StateName,
    NULL,
    NULL,
    3
FROM dim.Geography

UNION ALL

SELECT DISTINCT
    Country_BK,
    CountryName,
    Region_BK,
    RegionName,
    State_BK,
    StateName,
    City_BK,
    CityName,
    4
FROM dim.Geography;
```

## 2. Hierarquias Paralelas

### 2.1 Conceito
- Múltiplas hierarquias na mesma dimensão
- Diferentes perspectivas de análise
- Relacionamentos independentes
- Navegação flexível

### 2.2 Implementação
```sql
-- Dimensão de Produto com Hierarquias Paralelas
CREATE TABLE dim.Product (
    Product_SK bigint IDENTITY(1,1),
    Product_BK varchar(20) NOT NULL,
    ProductName nvarchar(100) NOT NULL,
    
    -- Hierarquia de Categorização
    Category_BK varchar(10) NOT NULL,
    CategoryName nvarchar(100) NOT NULL,
    Subcategory_BK varchar(10) NOT NULL,
    SubcategoryName nvarchar(100) NOT NULL,
    
    -- Hierarquia de Manufatura
    Manufacturer_BK varchar(10) NOT NULL,
    ManufacturerName nvarchar(100) NOT NULL,
    ProductLine_BK varchar(10) NOT NULL,
    ProductLineName nvarchar(100) NOT NULL,
    
    -- Hierarquia de Marketing
    Brand_BK varchar(10) NOT NULL,
    BrandName nvarchar(100) NOT NULL,
    MarketSegment varchar(50) NOT NULL,
    
    CONSTRAINT PK_Product PRIMARY KEY (Product_SK)
);

-- Views para diferentes hierarquias
CREATE VIEW analysis.ProductByCategory
AS
SELECT DISTINCT
    CategoryName,
    SubcategoryName,
    ProductName
FROM dim.Product;

CREATE VIEW analysis.ProductByManufacturer
AS
SELECT DISTINCT
    ManufacturerName,
    ProductLineName,
    ProductName
FROM dim.Product;

CREATE VIEW analysis.ProductByMarketing
AS
SELECT DISTINCT
    MarketSegment,
    BrandName,
    ProductName
FROM dim.Product;
```

## 3. Hierarquias Recursivas

### 3.1 Conceito
- Estruturas auto-referenciadas
- Níveis dinâmicos
- Profundidade variável
- Flexibilidade organizacional

### 3.2 Implementação
```sql
-- Estrutura organizacional recursiva
CREATE TABLE dim.Organization (
    Organization_SK bigint IDENTITY(1,1),
    Organization_BK varchar(20) NOT NULL,
    OrganizationName nvarchar(100) NOT NULL,
    Parent_SK bigint NULL,
    OrganizationType varchar(50) NOT NULL,
    Level int NOT NULL,
    Path varchar(500) NOT NULL,
    
    Created_DT datetime2 NOT NULL,
    Updated_DT datetime2 NOT NULL,
    
    CONSTRAINT PK_Organization PRIMARY KEY (Organization_SK),
    CONSTRAINT FK_Organization_Parent 
        FOREIGN KEY (Parent_SK) 
        REFERENCES dim.Organization(Organization_SK)
);

-- Função para obter hierarquia ascendente
CREATE FUNCTION dim.fn_GetOrganizationAncestors
(
    @OrganizationSK bigint
)
RETURNS TABLE
AS
RETURN
WITH OrganizationCTE AS (
    -- Base: organização inicial
    SELECT 
        Organization_SK,
        Parent_SK,
        OrganizationName,
        0 as Level
    FROM dim.Organization
    WHERE Organization_SK = @OrganizationSK

    UNION ALL

    -- Recursão: adiciona os pais
    SELECT 
        p.Organization_SK,
        p.Parent_SK,
        p.OrganizationName,
        c.Level + 1
    FROM dim.Organization p
    INNER JOIN OrganizationCTE c 
        ON p.Organization_SK = c.Parent_SK
)
SELECT 
    Organization_SK,
    OrganizationName,
    Level
FROM OrganizationCTE;

-- Função para obter hierarquia descendente
CREATE FUNCTION dim.fn_GetOrganizationDescendants
(
    @OrganizationSK bigint
)
RETURNS TABLE
AS
RETURN
WITH OrganizationCTE AS (
    -- Base: organização inicial
    SELECT 
        Organization_SK,
        Parent_SK,
        OrganizationName,
        0 as Level
    FROM dim.Organization
    WHERE Organization_SK = @OrganizationSK

    UNION ALL

    -- Recursão: adiciona os filhos
    SELECT 
        c.Organization_SK,
        c.Parent_SK,
        c.OrganizationName,
        p.Level + 1
    FROM dim.Organization c
    INNER JOIN OrganizationCTE p 
        ON c.Parent_SK = p.Organization_SK
)
SELECT 
    Organization_SK,
    OrganizationName,
    Level
FROM OrganizationCTE;

-- Procedimento para manutenção do path
CREATE PROCEDURE dim.usp_UpdateOrganizationPath
AS
BEGIN
    SET NOCOUNT ON;

    WITH OrganizationPath AS (
        -- Nível raiz
        SELECT
            Organization_SK,
            CAST(Organization_BK AS varchar(500)) as Path,
            1 as Level
        FROM dim.Organization
        WHERE Parent_SK IS NULL

        UNION ALL

        -- Níveis recursivos
        SELECT
            o.Organization_SK,
            CAST(p.Path + '/' + o.Organization_BK AS varchar(500)),
            p.Level + 1
        FROM dim.Organization o
        INNER JOIN OrganizationPath p 
            ON o.Parent_SK = p.Organization_SK
    )
    UPDATE o
    SET 
        Path = p.Path,
        Level = p.Level,
        Updated_DT = GETDATE()
    FROM dim.Organization o
    INNER JOIN OrganizationPath p 
        ON o.Organization_SK = p.Organization_SK;
END;
```

## 4. Navegação em Hierarquias

### 4.1 Técnicas de Navegação
```sql
-- Drill-down dinâmico
CREATE PROCEDURE analysis.usp_DrillDownGeography
    @Level int,
    @Country_BK varchar(2) = NULL,
    @Region_BK varchar(5) = NULL,
    @State_BK varchar(2) = NULL
AS
BEGIN
    SET NOCOUNT ON;

    IF @Level = 1
        SELECT DISTINCT
            Country_BK,
            CountryName,
            SUM(Population) as TotalPopulation
        FROM dim.Geography
        GROUP BY
            Country_BK,
            CountryName;
    
    ELSE IF @Level = 2 AND @Country_BK IS NOT NULL
        SELECT DISTINCT
            Country_BK,
            CountryName,
            Region_BK,
            RegionName,
            SUM(Population) as TotalPopulation
        FROM dim.Geography
        WHERE Country_BK = @Country_BK
        GROUP BY
            Country_BK,
            CountryName,
            Region_BK,
            RegionName;
    
    -- Continuar para outros níveis...
END;

-- Implementação de ancestors e descendants
CREATE FUNCTION analysis.fn_GetAncestorDescendantMetrics
(
    @OrganizationSK bigint,
    @Direction varchar(10)  -- 'UP' or 'DOWN'
)
RETURNS TABLE
AS
RETURN
SELECT 
    o.OrganizationName,
    o.Level,
    m.TotalEmployees,
    m.TotalSales,
    m.AverageSalary
FROM (
    SELECT *
    FROM dim.fn_GetOrganizationAncestors(@OrganizationSK)
    WHERE @Direction = 'UP'
    UNION ALL
    SELECT *
    FROM dim.fn_GetOrganizationDescendants(@OrganizationSK)
    WHERE @Direction = 'DOWN'
) o
CROSS APPLY (
    SELECT
        COUNT(DISTINCT e.Employee_SK) as TotalEmployees,
        SUM(s.SalesAmount) as TotalSales,
        AVG(e.Salary) as AverageSalary
    FROM dim.Employee e
    LEFT JOIN fact.Sales s 
        ON e.Employee_SK = s.Employee_SK
    WHERE e.Organization_SK = o.Organization_SK
) m;
```

## Exercícios Práticos

### Exercício 1: Hierarquias Naturais
1. Implemente uma hierarquia de produtos:
   - Categoria > Subcategoria > Produto
   - Inclua atributos em cada nível
   - Crie views para análise
   - Desenvolva procedures de navegação

### Exercício 2: Hierarquias Paralelas
1. Crie uma dimensão de cliente com múltiplas hierarquias:
   - Geográfica (País > Estado > Cidade)
   - Comercial (Segmento > Grupo > Cliente)
   - Desenvolva análises combinando hierarquias

### Exercício 3: Hierarquias Recursivas
1. Implemente uma estrutura organizacional:
   - Relações pai-filho dinâmicas
   - Cálculo de métricas agregadas
   - Navegação multi-nível
   - Manutenção de paths

## Material Complementar

### Documentação
- Padrões de hierarquia
- Técnicas de navegação
- Otimização de consultas hierárquicas

### Scripts Úteis
- Validação de integridade hierárquica
- Manutenção de paths
- Cálculos agregados em hierarquias

## Próxima Aula
Na próxima aula, abordaremos Modelagem para Casos Específicos, incluindo modelagem financeira, RH e vendas.

## Avaliação
1. Quiz sobre tipos de hierarquias
2. Implementação prática
3. Projeto de modelagem hierárquica