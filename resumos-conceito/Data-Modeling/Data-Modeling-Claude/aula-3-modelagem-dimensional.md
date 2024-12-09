# Aula 3 - Modelagem Dimensional Básica

## Objetivos da Aula
- Compreender os conceitos fundamentais de modelagem dimensional
- Entender diferentes tipos de fatos e suas aplicações
- Aprender sobre dimensões e seus componentes
- Dominar o conceito de granularidade
- Implementar modelos básicos no SQL Server

## 1. Fundamentos da Modelagem Dimensional

### 1.1 Conceitos Básicos
- Modelagem dimensional vs Modelagem normalizada
- Elementos principais:
  - Fatos (medidas)
  - Dimensões (contexto)
  - Atributos
  - Hierarquias

### 1.2 Vantagens da Modelagem Dimensional
- Facilidade de entendimento
- Performance em consultas
- Flexibilidade para mudanças
- Previsibilidade do modelo

### 1.3 Star Schema vs Snowflake
- Características do Star Schema
  - Dimensões desnormalizadas
  - Relacionamentos diretos
  - Menor complexidade de joins

- Características do Snowflake
  - Dimensões normalizadas
  - Relacionamentos em cascata
  - Economia de espaço

## 2. Tabelas Fato

### 2.1 Tipos de Tabelas Fato

#### Fatos Transacionais
- Registro de eventos
- Granularidade por transação
- Exemplo: vendas, pedidos, pagamentos
```sql
CREATE TABLE FACT_Sales (
    Sale_SK bigint IDENTITY(1,1),
    Date_SK int NOT NULL,
    Product_SK bigint NOT NULL,
    Customer_SK bigint NOT NULL,
    Store_SK bigint NOT NULL,
    SaleAmount decimal(18,2) NOT NULL,
    Quantity int NOT NULL,
    UnitPrice decimal(18,2) NOT NULL,
    UnitCost decimal(18,2) NOT NULL,
    TransactionTime datetime2 NOT NULL
);
```

#### Fatos Periódicos (Snapshots)
- Capturas regulares de estado
- Granularidade temporal definida
- Exemplo: saldos mensais, estoque diário
```sql
CREATE TABLE FACT_InventoryDaily (
    Date_SK int NOT NULL,
    Product_SK bigint NOT NULL,
    Store_SK bigint NOT NULL,
    OpeningQuantity int NOT NULL,
    ClosingQuantity int NOT NULL,
    MinQuantity int NOT NULL,
    MaxQuantity int NOT NULL,
    AvgQuantity decimal(18,2) NOT NULL
);
```

#### Fatos Acumulativos (Accumulating Snapshots)
- Múltiplos eventos no tempo
- Acompanhamento de processo
- Exemplo: ciclo de pedido, processo de manufatura
```sql
CREATE TABLE FACT_OrderFulfillment (
    Order_SK bigint IDENTITY(1,1),
    Customer_SK bigint NOT NULL,
    Product_SK bigint NOT NULL,
    OrderDate_SK int NOT NULL,
    ShipDate_SK int NULL,
    DeliveryDate_SK int NULL,
    OrderAmount decimal(18,2) NOT NULL,
    ShippingCost decimal(18,2) NULL,
    Status varchar(20) NOT NULL
);
```

### 2.2 Medidas

#### Tipos de Medidas
- Aditivas (podem ser somadas em todas as dimensões)
- Semi-aditivas (podem ser somadas em algumas dimensões)
- Não-aditivas (não podem ser somadas)

#### Exemplos de Implementação
```sql
-- Medidas aditivas
SaleAmount decimal(18,2),
Quantity int,

-- Medidas semi-aditivas
Balance decimal(18,2),
InventoryLevel int,

-- Medidas não-aditivas
UnitPrice decimal(18,2),
PercentageMargin decimal(5,2)
```

## 3. Dimensões

### 3.1 Anatomia de uma Dimensão
```sql
CREATE TABLE DIM_Customer (
    Customer_SK bigint IDENTITY(1,1),  -- Surrogate Key
    Customer_BK varchar(20) NOT NULL,   -- Business Key
    
    -- Atributos descritivos
    CustomerName nvarchar(100) NOT NULL,
    CustomerType varchar(20) NOT NULL,
    
    -- Atributos de endereço
    Address nvarchar(200),
    City nvarchar(100),
    State nvarchar(50),
    Country nvarchar(50),
    PostalCode varchar(20),
    
    -- Atributos de contato
    Email varchar(100),
    Phone varchar(20),
    
    -- Atributos de controle
    Created_DT datetime2 NOT NULL,
    Updated_DT datetime2 NOT NULL,
    Current_FLG bit NOT NULL
);
```

### 3.2 Tipos de Atributos
- Descritivos
- Hierárquicos
- Categóricos
- Métricos
- Controle

### 3.3 Hierarquias
```sql
-- Hierarquia geográfica
CREATE TABLE DIM_Geography (
    Geography_SK bigint IDENTITY(1,1),
    
    -- Níveis hierárquicos
    Country nvarchar(50),
    Region nvarchar(50),
    State nvarchar(50),
    City nvarchar(100),
    District nvarchar(100),
    
    -- Códigos de cada nível
    CountryCode char(2),
    StateCode char(2),
    CityCode varchar(10),
    
    -- Atributos de controle
    Valid_From datetime2 NOT NULL,
    Valid_To datetime2 NOT NULL,
    Current_FLG bit NOT NULL
);
```

## 4. Granularidade

### 4.1 Definição de Granularidade
- Nível mais baixo de detalhe
- Impacto nas análises
- Compromisso entre detalhe e volume

### 4.2 Exemplos de Granularidade
- Por transação
- Por dia
- Por produto
- Por cliente
- Combinações múltiplas

### 4.3 Impacto nas Medidas
- Agregações possíveis
- Cálculos derivados
- Limitações analíticas

## Exercícios Práticos

### Exercício 1: Modelagem Básica
1. Crie um modelo dimensional para um cenário de vendas:
   - Fato de vendas
   - Dimensões: produto, cliente, tempo, loja
   - Defina granularidade
   - Identifique medidas

### Exercício 2: Implementação
1. Implemente o modelo no SQL Server:
   - Scripts de criação
   - Constraints
   - Relacionamentos
   - Dados de exemplo

### Exercício 3: Consultas
1. Desenvolva consultas analíticas:
   - Vendas por período
   - Análise de clientes
   - Rankings de produtos
   - Tendências temporais

## Material Complementar

### Leitura Recomendada
- The Data Warehouse Toolkit (Kimball) - Capítulos 1-3
- Building a Dimensional Model (Microsoft)
- Star Schema The Complete Reference

### Scripts Úteis
- Templates para criação de dimensões
- Templates para criação de fatos
- Verificação de integridade dimensional

## Próxima Aula
Na próxima aula, aprenderemos sobre o ambiente SQL Server na prática, incluindo configuração, backup e restore.

## Avaliação
1. Quiz sobre conceitos dimensionais
2. Projeto prático de modelagem
3. Apresentação de modelo