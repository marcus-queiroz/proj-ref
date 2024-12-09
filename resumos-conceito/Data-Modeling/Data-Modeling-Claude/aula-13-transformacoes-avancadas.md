# Aula 13 - Transformações Avançadas

## Objetivos da Aula
- Dominar transformações complexas no SSIS
- Implementar Lookup avançado
- Trabalhar com Merge e Merge Join
- Utilizar Script Component
- Desenvolver transformações customizadas

## 1. Lookup Avançado

### 1.1 Cache Modes e Otimização
```sql
-- Configuração de cache para Lookup
-- Full Cache (Todos os dados em memória)
CREATE TABLE #LookupCache (
    Key_SK bigint PRIMARY KEY,
    Key_BK varchar(20),
    Description nvarchar(100),
    INDEX IX_Key_BK (Key_BK)
);

INSERT INTO #LookupCache
SELECT Key_SK, Key_BK, Description
FROM dim.LookupTable;

-- Partial Cache (Cache com limite de memória)
DECLARE @CacheSize int = 1000000; -- 1 milhão de registros
SELECT TOP (@CacheSize)
    Key_SK, Key_BK, Description
FROM dim.LookupTable
ORDER BY LastAccessDate DESC;

-- No Cache (Consulta direta)
SELECT Key_SK, Key_BK, Description
FROM dim.LookupTable
WHERE Key_BK = ?;
```

### 1.2 Multiple Lookup Implementations
```sql
-- Lookup em cascata
-- Primeira lookup
SELECT Customer_SK, Customer_BK
FROM dim.Customer;

-- Segunda lookup (baseada no resultado da primeira)
SELECT Address_SK, Customer_SK
FROM dim.CustomerAddress
WHERE Customer_SK = ?;

-- Terceira lookup
SELECT Contact_SK, Customer_SK
FROM dim.CustomerContact
WHERE Customer_SK = ?;
```

## 2. Merge e Merge Join

### 2.1 Merge Transformations
```sql
-- Preparação dos dados para Merge
-- Stream 1 (deve estar ordenado)
SELECT 
    CustomerID,
    TransactionDate,
    Amount
FROM source.Transactions
ORDER BY CustomerID, TransactionDate;

-- Stream 2 (deve estar ordenado)
SELECT 
    CustomerID,
    LastUpdateDate,
    Balance
FROM source.Balances
ORDER BY CustomerID, LastUpdateDate;

-- Configuração do Merge
IsSorted: True
JoinType: Full/Left/Inner
MaxBuffersPerInput: 10
```

### 2.2 Merge Join Avançado
```sql
-- Exemplo de Merge Join complexo
-- Query principal
SELECT 
    o.OrderID,
    o.OrderDate,
    o.CustomerID,
    o.TotalAmount
FROM Sales.Orders o
ORDER BY o.CustomerID;

-- Query de junção 1
SELECT 
    c.CustomerID,
    c.CustomerName,
    c.CustomerType
FROM Sales.Customers c
ORDER BY c.CustomerID;

-- Query de junção 2
SELECT 
    p.OrderID,
    p.ProductID,
    p.Quantity,
    p.UnitPrice
FROM Sales.OrderDetails p
ORDER BY p.OrderID;

-- Tratamento de não correspondências
ISNULL(JoinColumn, DefaultValue) as ResultColumn
```

## 3. Script Component

### 3.1 Script Transformation
```csharp
// Exemplo de Script Component como transformação
public class ScriptMain : UserComponent
{
    public override void Input0_ProcessInputRow(Input0Buffer Row)
    {
        // Processamento de data
        if (!Row.DateColumn_IsNull)
        {
            try
            {
                DateTime dt = DateTime.ParseExact(
                    Row.DateColumn, 
                    "yyyyMMdd",
                    CultureInfo.InvariantCulture
                );
                Row.ProcessedDate = dt;
            }
            catch
            {
                Row.ErrorFlag = true;
            }
        }

        // Processamento de valores numéricos
        if (!Row.AmountColumn_IsNull)
        {
            decimal amount = Row.AmountColumn;
            if (amount < 0)
            {
                Row.NegativeAmount = Math.Abs(amount);
                Row.TransactionType = "Debit";
            }
            else
            {
                Row.NegativeAmount = 0;
                Row.TransactionType = "Credit";
            }
        }
    }
}
```

### 3.2 Script Source
```csharp
// Exemplo de Script Component como fonte
public override void CreateNewOutputRows()
{
    string[] files = Directory.GetFiles(
        Variables.FolderPath, 
        "*.csv"
    );

    foreach (string file in files)
    {
        string[] lines = File.ReadAllLines(file);
        foreach (string line in lines.Skip(1)) // Skip header
        {
            string[] fields = line.Split(',');
            Output0Buffer.AddRow();
            Output0Buffer.FileName = Path.GetFileName(file);
            Output0Buffer.Field1 = fields[0];
            Output0Buffer.Field2 = fields[1];
            // ... processar outros campos
        }
    }
}
```

### 3.3 Script Destination
```csharp
// Exemplo de Script Component como destino
private StreamWriter _writer;

public override void PreExecute()
{
    _writer = new StreamWriter(
        Variables.OutputFilePath,
        false,
        Encoding.UTF8
    );
    // Escrever cabeçalho
    _writer.WriteLine("Field1,Field2,ProcessedDate,Amount");
}

public override void Input0_ProcessInputRow(Input0Buffer Row)
{
    StringBuilder sb = new StringBuilder();
    sb.Append(Row.Field1);
    sb.Append(",");
    sb.Append(Row.Field2);
    sb.Append(",");
    sb.Append(Row.ProcessedDate.ToString("yyyy-MM-dd"));
    sb.Append(",");
    sb.Append(Row.Amount.ToString("F2"));
    
    _writer.WriteLine(sb.ToString());
}

public override void PostExecute()
{
    if (_writer != null)
    {
        _writer.Close();
        _writer = null;
    }
}
```

## 4. Transformações Customizadas

### 4.1 Aggregate com Expressões Complexas
```sql
-- Configuração de agregações avançadas
-- Exemplo 1: Média móvel
AVG([Amount]) OVER(
    PARTITION BY [CustomerID]
    ORDER BY [TransactionDate]
    ROWS BETWEEN 3 PRECEDING AND CURRENT ROW
) as MovingAverage

-- Exemplo 2: Ranking por grupo
RANK() OVER(
    PARTITION BY [Category]
    ORDER BY [Amount] DESC
) as RankInCategory

-- Exemplo 3: Agregações condicionais
SUM(CASE 
    WHEN [TransactionType] = 'Sale' THEN [Amount]
    WHEN [TransactionType] = 'Return' THEN -[Amount]
    ELSE 0 
END) as NetAmount
```

### 4.2 Conditional Split Complexo
```sql
-- Condições complexas para Split
-- Regra 1: Clientes de alto valor
[YearlyPurchases] > 100000 && 
[CustomerType] == "Corporate" && 
[CreditScore] > 700

-- Regra 2: Clientes em risco
[DaysSinceLastPurchase] > 180 || 
([CreditScore] < 500 && [PastDueAmount] > 1000)

-- Regra 3: Clientes para promoção
[PurchaseFrequency] > 2 && 
[AverageOrderValue] > 500 && 
[ReturnRate] < 0.05
```

## 5. Otimização de Performance

### 5.1 Buffer Configuration
```sql
-- Configurações de buffer
DefaultBufferMaxRows = 10000
DefaultBufferSize = 10485760 -- 10MB
EngineThreads = 5
```

### 5.2 Memory Optimization
```sql
-- Gestão de memória
CREATE TABLE #TempLookup (
    Key_SK bigint,
    Key_BK varchar(20),
    INDEX IX_Temp (Key_SK, Key_BK)
)

-- Particionar dados grandes
DECLARE @BatchSize int = 100000;
DECLARE @CurrentBatch int = 1;

WHILE EXISTS (
    SELECT 1 
    FROM source.LargeTable 
    WHERE BatchID = @CurrentBatch
)
BEGIN
    -- Processar lote
    -- Incrementar batch
    SET @CurrentBatch += 1;
END
```

## Exercícios Práticos

### Exercício 1: Lookup Avançado
1. Implementar diferentes modos de cache
2. Criar lookups em cascata
3. Otimizar performance de lookup

### Exercício 2: Script Component
1. Desenvolver transformação customizada
2. Implementar lógica de negócio complexa
3. Tratar erros e exceções

### Exercício 3: Otimização
1. Configurar buffers
2. Implementar particionamento
3. Monitorar performance

## Material Complementar

### Documentação
- Advanced Transformation Patterns
- Performance Optimization Guide
- Memory Management Best Practices

### Scripts Úteis
- Monitoramento de performance
- Debug de transformações
- Validação de dados

## Próxima Aula
Na próxima aula, abordaremos Control Flow e Execução, incluindo containers, precedências e execução condicional.

## Avaliação
1. Quiz sobre transformações avançadas
2. Projeto prático de implementação
3. Análise de performance