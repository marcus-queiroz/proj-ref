## Exercícios Práticos

### Exercício 1: Bridge Tables
1. Desenvolva uma bridge table para:
   - Produtos e suas categorias (relacionamento M:M)
   - Inclua atributos de vigência
   - Implemente controles de versão
   - Crie procedimento de manutenção

2. Implemente consultas utilizando a bridge table:
```sql
-- Exemplo de consulta a ser implementada
WITH ProductCategories AS (
    SELECT 
        p.Product_SK,
        p.ProductName,
        STRING_AGG(c.CategoryName, ', ') as Categories
    FROM dim.Product p
    JOIN brg.ProductCategory b ON p.Product_SK = b.Product_SK
    JOIN dim.Category c ON b.Category_SK = c.Category_SK
    WHERE GETDATE() BETWEEN b.EffectiveDate AND ISNULL(b.ExpirationDate, '9999-12-31')
    GROUP BY p.Product_SK, p.ProductName
)
SELECT * FROM ProductCategories;
```

### Exercício 2: Factless Fact Tables
1. Crie uma factless fact table para:
   - Registro de contatos com cliente
   - Cobertura de promoções
   - Oportunidades de venda

2. Desenvolva análises usando a factless fact table:
```sql
-- Template de análise a ser implementada
SELECT 
    d.MonthName,
    c.CustomerSegment,
    COUNT(*) as NumberOfContacts,
    COUNT(DISTINCT c.Customer_SK) as CustomersContacted
FROM fact.CustomerContacts fc
JOIN dim.Date d ON fc.Date_SK = d.Date_SK
JOIN dim.Customer c ON fc.Customer_SK = c.Customer_SK
GROUP BY 
    d.MonthName,
    c.CustomerSegment;
```

### Exercício 3: Fatos Agregados
1. Implemente um conjunto de fatos agregados:
   - Vendas diárias por produto
   - Vendas mensais por categoria
   - Vendas anuais por região

2. Crie processo de manutenção das agregações:
```sql
-- Template de processo a ser implementado
CREATE PROCEDURE fact.usp_UpdateAggregations
    @StartDate date,
    @EndDate date,
    @AggregationType varchar(20)
AS
BEGIN
    -- Implementar lógica de agregação baseada no tipo
    -- Incluir tratamento de erro
    -- Implementar log de execução
    -- Validar resultados
END;
```

### Exercício 4: Fatos Consolidados
1. Desenvolva um fato consolidado que integre:
   - Vendas de diferentes canais
   - Diferentes granularidades
   - Métricas calculadas

2. Implemente processo de consolidação:
   - Tratamento de diferentes fontes
   - Harmonização de dados
   - Validação de resultados

## Material Complementar

### Documentação
1. Design Patterns para Fatos Especiais
   - Bridge tables patterns
   - Factless fact patterns
   - Agregation patterns
   - Consolidation patterns

2. Performance Optimization
   - Indexação para fatos especiais
   - Estratégias de particionamento
   - Compressão de dados
   - Manutenção de índices

3. Best Practices
   - Quando usar cada tipo de fato
   - Estratégias de implementação
   - Considerações de maintenance
   - Padrões de validação

### Scripts Úteis

#### 1. Análise de Bridge Tables
```sql
-- Verificar integridade de bridge table
CREATE PROCEDURE audit.usp_CheckBridgeIntegrity
AS
BEGIN
    -- Verificar registros órfãos
    -- Verificar sobreposição de datas
    -- Verificar consistência de grupos
    -- Gerar relatório de auditoria
END;
```

#### 2. Monitoramento de Agregações
```sql
-- Monitorar consistência de agregações
CREATE PROCEDURE audit.usp_ValidateAggregations
AS
BEGIN
    -- Comparar totais agregados vs detalhados
    -- Verificar missing data
    -- Validar cálculos
    -- Gerar relatório de discrepâncias
END;
```

#### 3. Manutenção de Fatos
```sql
-- Template para manutenção de fatos
CREATE PROCEDURE maintenance.usp_FactMaintenance
    @FactTable varchar(100),
    @RetentionPeriod int
AS
BEGIN
    -- Implementar archiving
    -- Rebuild/reorganize índices
    -- Update estatísticas
    -- Log de manutenção
END;
```

## Próxima Aula
Na próxima aula, abordaremos Hierarquias e Relacionamentos, incluindo hierarquias naturais, paralelas e recursivas.

## Avaliação
1. Quiz sobre tipos especiais de fatos
2. Implementação prática de cada tipo
3. Projeto de modelagem completo incluindo diferentes tipos de fatos

### Critérios de Avaliação
- Correto uso dos tipos de fatos
- Implementação de processos de manutenção
- Performance das soluções
- Documentação e boas práticas