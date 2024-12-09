# Aula 2 - Tipos de Dados e Estruturas

## Objetivos da Aula
- Compreender os tipos de dados disponíveis no SQL Server
- Aprender boas práticas de nomenclatura
- Entender estruturas de tabelas e relacionamentos
- Conhecer os principais tipos de constraints
- Introduzir conceitos básicos de indexação

## 1. Tipos de Dados no SQL Server

### 1.1 Tipos Numéricos
- **Inteiros**
  - tinyint (0 a 255) - 1 byte
  - smallint (-32,768 a 32,767) - 2 bytes
  - int (-2^31 a 2^31 - 1) - 4 bytes
  - bigint (-2^63 a 2^63 - 1) - 8 bytes

- **Decimais/Monetários**
  - decimal/numeric(p,s) - precisão e escala definidas
  - money - 8 bytes
  - smallmoney - 4 bytes
  - float - 4 ou 8 bytes
  - real - 4 bytes

### 1.2 Tipos de Caractere
- **Comprimento Fixo**
  - char(n) - até 8000 caracteres
  - nchar(n) - até 4000 caracteres Unicode

- **Comprimento Variável**
  - varchar(n) - até 8000 caracteres
  - nvarchar(n) - até 4000 caracteres Unicode
  - varchar(max) - até 2GB
  - nvarchar(max) - até 2GB

### 1.3 Tipos de Data e Hora
- date - apenas data
- time - apenas hora
- datetime - data e hora (precisão 3.33ms)
- datetime2 - data e hora (precisão 100ns)
- datetimeoffset - datetime2 com fuso horário
- smalldatetime - data e hora (precisão 1min)

### 1.4 Outros Tipos
- bit - valores booleanos
- binary/varbinary - dados binários
- uniqueidentifier - GUID
- xml - dados XML
- geography/geometry - dados espaciais

## 2. Boas Práticas de Nomenclatura

### 2.1 Convenções Gerais
- Use nomes significativos e descritivos
- Evite abreviações ambíguas
- Mantenha consistência no padrão
- Use singular para tabelas
- Prefixos e sufixos quando necessário

### 2.2 Padrões para Data Warehouse
- Prefixos para tipo de tabela
  - DIM_ para dimensões
  - FACT_ para fatos
  - STG_ para staging
  - BRG_ para bridge tables

- Sufixos comuns
  - _ID para chaves
  - _SK para surrogate keys
  - _BK para business keys
  - _DT para datas
  - _TS para timestamps

### 2.3 Exemplos Práticos
```sql
-- Dimensão Cliente
CREATE TABLE DIM_Customer (
    Customer_SK bigint IDENTITY(1,1),
    Customer_BK varchar(20),
    CustomerName nvarchar(100),
    Created_DT datetime2,
    Updated_TS datetimeoffset
);

-- Fato Vendas
CREATE TABLE FACT_Sales (
    Sale_SK bigint IDENTITY(1,1),
    Customer_SK bigint,
    Product_SK bigint,
    Date_SK int,
    SaleAmount decimal(18,2),
    Quantity int
);
```

## 3. Estruturas de Tabelas e Relacionamentos

### 3.1 Componentes de Tabela
- Colunas e tipos de dados
- Chaves primárias
- Chaves estrangeiras
- Índices
- Constraints
- Partições (quando aplicável)

### 3.2 Relacionamentos
- Um para Um (1:1)
- Um para Muitos (1:N)
- Muitos para Muitos (N:N)
- Auto-relacionamentos

### 3.3 Implementação de Relacionamentos
```sql
-- Criando relacionamento
ALTER TABLE FACT_Sales
ADD CONSTRAINT FK_Sales_Customer
FOREIGN KEY (Customer_SK)
REFERENCES DIM_Customer(Customer_SK);
```

## 4. Constraints

### 4.1 Tipos de Constraints
- PRIMARY KEY
- FOREIGN KEY
- UNIQUE
- CHECK
- DEFAULT
- NOT NULL

### 4.2 Exemplos de Implementação
```sql
CREATE TABLE DIM_Product (
    Product_SK bigint IDENTITY(1,1),
    Product_BK varchar(20),
    ProductName nvarchar(100),
    UnitPrice decimal(18,2),
    CONSTRAINT PK_Product PRIMARY KEY (Product_SK),
    CONSTRAINT UQ_Product_BK UNIQUE (Product_BK),
    CONSTRAINT CHK_UnitPrice CHECK (UnitPrice >= 0),
    CONSTRAINT DF_CreateDate DEFAULT GETDATE() FOR Created_DT
);
```

## 5. Introdução a Índices

### 5.1 Tipos Básicos de Índices
- Clustered
- Non-clustered
- Unique
- Columnstore
- Filtered

### 5.2 Considerações sobre Índices
- Um índice clustered por tabela
- Impacto em INSERT/UPDATE/DELETE
- Manutenção e fragmentação
- Uso de memória
- Custo/benefício

### 5.3 Exemplo Básico
```sql
-- Índice clustered (geralmente criado com PK)
CREATE CLUSTERED INDEX IX_Customer_SK
ON DIM_Customer(Customer_SK);

-- Índice non-clustered
CREATE NONCLUSTERED INDEX IX_Customer_BK
ON DIM_Customer(Customer_BK);
```

## Exercícios Práticos

### Exercício 1: Criação de Estruturas
1. Crie uma dimensão de Produto com:
   - Surrogate key
   - Business key
   - Atributos descritivos
   - Controle temporal
   - Constraints apropriadas

2. Crie uma tabela fato de Vendas com:
   - Chaves estrangeiras
   - Métricas
   - Índices apropriados

### Exercício 2: Análise de Tipos de Dados
1. Analise um modelo existente
2. Identifique possíveis melhorias
3. Justifique suas escolhas

### Exercício 3: Implementação de Constraints
1. Adicione constraints a tabelas existentes
2. Teste violações de constraint
3. Corrija dados problemáticos

## Material Complementar

### Documentação
- SQL Server Data Types (Microsoft Docs)
- Best Practices for Data Warehouse Design
- SQL Server Index Design Guide

### Scripts Úteis
- Análise de uso de tipos de dados
- Verificação de integridade de constraints
- Análise de fragmentação de índices

## Próxima Aula
Na próxima aula, abordaremos os conceitos básicos de modelagem dimensional, incluindo fatos, dimensões e granularidade.

## Avaliação
1. Quiz sobre tipos de dados
2. Exercício prático de modelagem
3. Análise de caso de uso real