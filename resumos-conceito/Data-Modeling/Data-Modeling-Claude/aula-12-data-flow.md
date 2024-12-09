# Aula 12 - Data Flow Tasks

## Objetivos da Aula
- Compreender os componentes do Data Flow
- Trabalhar com diferentes fontes de dados
- Implementar transformações básicas
- Configurar destinos de dados
- Desenvolver tratamento de erros

## 1. Fontes de Dados (Sources)

### 1.1 OLE DB Source
```sql
-- Exemplo de query para OLE DB Source
SELECT 
    c.CustomerID,
    c.CustomerName,
    c.CustomerType,
    a.AddressLine1,
    a.AddressLine2,
    a.City,
    a.StateProvince,
    a.Country
FROM 
    Sales.Customer c
    LEFT JOIN Sales.CustomerAddress a ON c.CustomerID = a.CustomerID
WHERE 
    c.ModifiedDate >= ?
ORDER BY 
    c.CustomerID;

-- Parâmetros de configuração
SELECT 
    AccessMode,
    CASE AccessMode
        WHEN 0 THEN 'Table or View'
        WHEN 1 THEN 'Table Name Variable'
        WHEN 2 THEN 'SQL Command'
        WHEN 3 THEN 'SQL Command from Variable'
    END as AccessModeDescription
FROM 
    INFORMATION_SCHEMA.COLUMNS
WHERE 
    1 = 0; -- Apenas para documentação
```

### 1.2 Flat File Source
```xml
<!-- Exemplo de configuração de Flat File -->
<?xml version="1.0"?>
<DTS:ConnectionManager xmlns:DTS="www.microsoft.com/SqlServer/Dts">
  <DTS:Property DTS:Name="ConnectionString">
    <DTS:Property DTS:Name="Format">Delimited</DTS:Property>
    <DTS:Property DTS:Name="ColumnNamesInFirstDataRow">True</DTS:Property>
    <DTS:Property DTS:Name="HeaderRowDelimiter">\n</DTS:Property>
    <DTS:Property DTS:Name="RowDelimiter">\n</DTS:Property>
    <DTS:Property DTS:Name="DataRowsToSkip">0</DTS:Property>
    <DTS:Property DTS:Name="TextQualifier">"</DTS:Property>
    <DTS:Property DTS:Name="CodePage">1252</DTS:Property>
  </DTS:Property>
</DTS:ConnectionManager>
```

### 1.3 Excel Source
```xml
<!-- Exemplo de configuração de Excel -->
<?xml version="1.0"?>
<DTS:ConnectionManager xmlns:DTS="www.microsoft.com/SqlServer/Dts">
  <DTS:Property DTS:Name="ConnectionString">
    <DTS:Property DTS:Name="Provider">Microsoft.ACE.OLEDB.12.0</DTS:Property>
    <DTS:Property DTS:Name="ExtendedProperties">
      Excel 12.0;HDR=YES;IMEX=1;
    </DTS:Property>
  </DTS:Property>
</DTS:ConnectionManager>
```

## 2. Transformações Básicas

### 2.1 Derived Column
```sql
-- Exemplos de expressões para Derived Column
-- Concatenação
[FirstName] + " " + [LastName]

-- Manipulação de data
(DT_DBTIMESTAMP)(SUBSTRING([DateString],1,4) + "-" + 
                 SUBSTRING([DateString],5,2) + "-" + 
                 SUBSTRING([DateString],7,2))

-- Conversão de tipos
(DT_DECIMAL,10,2)[StringValue]

-- Caso condicional
ISNULL([Value]) ? "N/A" : [Value]
```

### 2.2 Conditional Split
```sql
-- Exemplos de condições para Conditional Split
-- Filtro de clientes
[CustomerType] == "Corporate" && [Revenue] > 100000

-- Tratamento de erros
ISNULL([Email]) || [Email] == "" ? "Invalid" : "Valid"

-- Classificação de valores
[Amount] > 10000 ? "High" : 
[Amount] > 1000 ? "Medium" : "Low"
```

### 2.3 Lookup
```sql
-- Query para componente Lookup
SELECT 
    Customer_SK,
    Customer_BK
FROM 
    dim.Customer

-- Configuração de cache
FullCache: Carrega todos os dados na memória
NoCache: Consulta o banco para cada linha
PartialCache: Cache com número máximo de linhas
```

## 3. Transformações Avançadas

### 3.1 Aggregate
```sql
-- Exemplos de agregações
GROUP BY [Category], [Region]
SUM([Amount]) as TotalAmount
COUNT(*) as RowCount
MAX([Date]) as LastDate
MIN([Date]) as FirstDate

-- Configurações de memória
AutoAdjustBufferSize: True/False
Keys: Número de chaves únicas estimadas
```

### 3.2 Merge Join
```sql
-- Preparação para Merge Join
-- Query 1 (ordenada por chave)
SELECT 
    OrderID,
    CustomerID,
    OrderDate
FROM 
    Sales.Orders
ORDER BY 
    CustomerID

-- Query 2 (ordenada por chave)
SELECT 
    CustomerID,
    CustomerName
FROM 
    Sales.Customers
ORDER BY 
    CustomerID
```

### 3.3 Union All
```sql
-- Exemplo de mapeamento de colunas
Source1: CustomerID, Name, Email, 'System1' as Source
Source2: ClientID, ClientName, EmailAddress, 'System2' as Source

-- Configurações
AutoPassThrough: True/False
ValidateExternalMetadata: True/False
```

## 4. Destinos de Dados (Destinations)

### 4.1 OLE DB Destination
```sql
-- Configurações de tabela destino
CREATE TABLE stg.Customer (
    CustomerID int,
    CustomerName nvarchar(100),
    CustomerType varchar(20),
    Email varchar(100),
    LoadDate datetime2 DEFAULT(GETDATE())
);

-- Opções de carregamento
Table lock
Check constraints
Keep nulls
Enable identity insert
```

### 4.2 Flat File Destination
```xml
<!-- Configuração de arquivo de saída -->
<DTS:PropertyExpression>
  @[User::OutputPath] + "\\" + 
  "Customer_" + 
  RIGHT("0" + (DT_STR,2,1252)MONTH(GETDATE()),2) + 
  RIGHT("0" + (DT_STR,2,1252)DAY(GETDATE()),2) + 
  ".txt"
</DTS:PropertyExpression>
```

## 5. Tratamento de Erros

### 5.1 Error Output
```sql
-- Tabela para erros
CREATE TABLE meta.DataFlowErrors (
    ErrorID bigint IDENTITY(1,1),
    PackageName nvarchar(100),
    ComponentName nvarchar(100),
    ErrorCode int,
    ErrorColumn nvarchar(100),
    ErrorDescription nvarchar(max),
    ErrorData nvarchar(max),
    LoadDate datetime2 DEFAULT(GETDATE())
);

-- Redirecionamento de erros
ErrorCode: Código do erro
ErrorColumn: Coluna que causou o erro
```

### 5.2 Logging
```sql
-- Procedure para log de erros
CREATE PROCEDURE meta.usp_LogDataFlowError
    @PackageName nvarchar(100),
    @ComponentName nvarchar(100),
    @ErrorCode int,
    @ErrorColumn nvarchar(100),
    @ErrorDescription nvarchar(max),
    @ErrorData nvarchar(max)
AS
BEGIN
    INSERT INTO meta.DataFlowErrors (
        PackageName,
        ComponentName,
        ErrorCode,
        ErrorColumn,
        ErrorDescription,
        ErrorData
    )
    VALUES (
        @PackageName,
        @ComponentName,
        @ErrorCode,
        @ErrorColumn,
        @ErrorDescription,
        @ErrorData
    );
END;
```

## Exercícios Práticos

### Exercício 1: Fontes de Dados
1. Configurar diferentes fontes:
   - OLE DB Source
   - Flat File Source
   - Excel Source
2. Implementar validações
3. Tratar erros de fonte

### Exercício 2: Transformações
1. Implementar:
   - Derived Column
   - Conditional Split
   - Lookup
2. Criar transformações compostas
3. Otimizar performance

### Exercício 3: Destinos
1. Configurar diferentes destinos
2. Implementar logging
3. Tratar erros de carga

## Material Complementar

### Documentação
- Data Flow Task Best Practices
- Performance Tuning Guide
- Error Handling Guide

### Scripts Úteis
- Templates de transformação
- Scripts de validação
- Procedures de logging

## Próxima Aula
Na próxima aula, abordaremos Transformações Avançadas no SSIS, incluindo agregações complexas e manipulação de dados.

## Avaliação
1. Quiz sobre Data Flow Tasks
2. Implementação prática
3. Análise de performance