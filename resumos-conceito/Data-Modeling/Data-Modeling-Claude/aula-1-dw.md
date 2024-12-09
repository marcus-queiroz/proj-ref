# Aula 1 - Introdução ao Data Warehousing

## Objetivos da Aula
- Compreender os conceitos fundamentais de Data Warehouse
- Conhecer as principais arquiteturas de DW
- Entender a estrutura básica do SQL Server
- Familiarizar-se com as ferramentas do ambiente SQL Server

## 1. Conceitos Fundamentais de Data Warehouse

### 1.1 O que é um Data Warehouse?
Um Data Warehouse (DW) é um repositório central de dados integrados provenientes de múltiplas fontes, projetado especificamente para análise e suporte à decisão. Diferente dos sistemas transacionais (OLTP), um DW é otimizado para consultas analíticas (OLAP).

Características principais:
- Orientado por assunto
- Integrado
- Não volátil
- Variante no tempo

### 1.2 Por que precisamos de um Data Warehouse?
- Consolidação de dados de múltiplas fontes
- Histórico consistente de informações
- Separação entre ambiente analítico e transacional
- Padronização de informações
- Única versão da verdade
- Performance otimizada para análises

### 1.3 Componentes de um Data Warehouse
1. Fonte de dados
   - Sistemas transacionais
   - Arquivos externos
   - APIs
   - Dados não estruturados

2. Área de Stage
   - Zona de pouso dos dados
   - Dados em formato bruto
   - Sem transformações significativas

3. Área de Integração
   - Processamento e transformação
   - Aplicação de regras de negócio
   - Padronização e limpeza

4. Área de Apresentação
   - Data Marts
   - Cubos OLAP
   - Visões consolidadas

## 2. Arquiteturas de Data Warehouse

### 2.1 Arquitetura Kimball
- Abordagem bottom-up
- Data Marts dimensionais interconectados
- Dimensões conformadas
- Bus Matrix Architecture

Vantagens:
- Implementação incremental
- Menor custo inicial
- Resultados mais rápidos
- Mais flexível

Desvantagens:
- Pode gerar redundância
- Maior complexidade na gestão
- Necessidade de forte governança

### 2.2 Arquitetura Inmon
- Abordagem top-down
- Modelo corporativo normalizado
- Data Marts derivados
- Forte integração

Vantagens:
- Maior consistência
- Menor redundância
- Modelo corporativo único

Desvantagens:
- Maior custo inicial
- Implementação mais longa
- Menor flexibilidade

### 2.3 Arquiteturas Híbridas e Modernas
- Data Vault
- Lake House
- Modern Data Stack

## 3. Estrutura do SQL Server

### 3.1 Componentes Principais
- Instância
- Databases
- Schemas
- Objetos de banco
  - Tabelas
  - Views
  - Stored Procedures
  - Functions
  - Indexes

### 3.2 Databases do Sistema
- master
- model
- msdb
- tempdb

### 3.3 Objetos de Usuário
- Databases de usuário
- Schemas personalizados
- Roles e permissões

## 4. Ferramentas SQL Server

### 4.1 SQL Server Management Studio (SSMS)
- Interface principal de administração
- Object Explorer
- Query Editor
- Execution Plans
- Templates

### 4.2 SQL Server Integration Services (SSIS)
- Ferramenta de ETL
- Visual Studio Integration
- Packages
- Projects
- Deployments

### 4.3 SQL Server Analysis Services (SSAS)
- Modelagem multidimensional
- Tabular models
- Cubos OLAP
- DAX e MDX

## Exercícios Práticos

### Exercício 1: Conceitos Básicos
1. Identifique em sua organização:
   - Fontes de dados potenciais para DW
   - Necessidades analíticas
   - Desafios atuais com dados

2. Compare sistemas OLTP vs OLAP:
   - Volume de dados
   - Tipos de consultas
   - Performance
   - Usuários

### Exercício 2: Instalação e Configuração
1. Instale o SQL Server Developer Edition
2. Instale o SQL Server Management Studio
3. Configure uma instância local
4. Crie seu primeiro database

### Exercício 3: Exploração do Ambiente
1. Explore o Object Explorer no SSMS
2. Crie queries simples
3. Examine os databases do sistema
4. Configure backup básico

## Material Complementar

### Leitura Recomendada
- The Data Warehouse Toolkit (Ralph Kimball)
- Building the Data Warehouse (W.H. Inmon)
- Microsoft SQL Server documentation

### Links Úteis
- Microsoft Learn Platform
- SQL Server samples on GitHub
- MSDN Documentation
- Community forums

## Próxima Aula
Na próxima aula, abordaremos em detalhes os tipos de dados e estruturas no SQL Server, incluindo boas práticas de nomenclatura e design de tabelas.

## Avaliação
1. Quiz sobre conceitos básicos de DW
2. Exercício prático de instalação
3. Discussão em grupo sobre casos de uso