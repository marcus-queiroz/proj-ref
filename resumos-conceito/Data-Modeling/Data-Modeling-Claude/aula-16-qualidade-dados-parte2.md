## Exercícios Práticos (Continuação)

### Exercício 2: Limpeza e Padronização
1. Implementar funções de limpeza para:
   - Nomes (capitalização, remoção de caracteres especiais)
   - Endereços (formatação padrão)
   - Telefones (formato consistente)
   - Emails (validação e normalização)

2. Desenvolver processo de padronização:
```sql
-- Template para processo de padronização
CREATE PROCEDURE etl.usp_StandardizeData
    @TableName varchar(100)
AS
BEGIN
    -- Implementar limpeza básica
    -- Aplicar regras de padronização
    -- Validar resultados
    -- Gerar log de processamento
END;
```

### Exercício 3: Deduplicação
1. Criar processo de identificação de duplicatas:
   - Match exato
   - Match fuzzy
   - Match por múltiplos critérios

2. Implementar resolução de duplicatas:
```sql
-- Template para resolução
CREATE PROCEDURE etl.usp_ResolveDuplicates
    @GroupID int,
    @MasterRecord_BK varchar(20)
AS
BEGIN
    -- Identificar registros do grupo
    -- Definir registro principal
    -- Atualizar referências
    -- Gerar log de resolução
END;
```

### Exercício 4: Auditoria
1. Desenvolver dashboard de qualidade
2. Criar alertas automáticos
3. Implementar relatórios de tendência

## Material Complementar

### 1. Scripts de Monitoramento

```sql
-- Monitoramento de qualidade
CREATE PROCEDURE monitor.usp_QualityCheck
AS
BEGIN
    -- Verificar regras de validação
    SELECT 
        TableName,
        COUNT(*) as ErrorCount,
        MAX(ErrorDate) as LastError
    FROM meta.ValidationErrors
    GROUP BY TableName;

    -- Verificar duplicatas
    SELECT 
        TableName,
        COUNT(*) as DuplicateCount,
        COUNT(DISTINCT GroupID) as GroupCount
    FROM meta.DuplicateCustomers
    WHERE ResolutionStatus = 'Pending'
    GROUP BY TableName;

    -- Verificar padronização
    SELECT 
        TableName,
        ColumnName,
        COUNT(*) as NonStandardCount
    FROM meta.NonStandardValues
    GROUP BY TableName, ColumnName;
END;
```

### 2. Documentação de Processos

#### 2.1 Fluxo de Validação
1. Configuração de regras
2. Execução de validações
3. Tratamento de erros
4. Reprocessamento

#### 2.2 Processo de Limpeza
1. Identificação de padrões
2. Aplicação de transformações
3. Validação de resultados
4. Documentação de exceções

#### 2.3 Gestão de Duplicatas
1. Critérios de identificação
2. Processo de resolução
3. Atualização de referências
4. Manutenção de histórico

### 3. Boas Práticas

#### 3.1 Validação de Dados
- Definir critérios claros
- Documentar regras
- Manter histórico de validações
- Implementar processo de correção

#### 3.2 Limpeza e Padronização
- Criar biblioteca de funções
- Manter dicionário de termos
- Documentar transformações
- Validar resultados

#### 3.3 Deduplicação
- Definir critérios de match
- Implementar processo de revisão
- Manter histórico de decisões
- Documentar exceções

## Próxima Aula
Na próxima aula, abordaremos Logging e Monitoramento, incluindo estratégias de logging, tabelas de controle e monitoramento de execução.

## Avaliação

### 1. Quiz Teórico
- Conceitos de qualidade de dados
- Estratégias de validação
- Processos de limpeza
- Técnicas de deduplicação

### 2. Projeto Prático
Implementar processo completo de qualidade para um conjunto de dados, incluindo:
1. Validação
2. Limpeza
3. Padronização
4. Deduplicação
5. Relatórios de qualidade

### 3. Critérios de Avaliação
- Completude da solução
- Qualidade do código
- Documentação
- Performance
- Facilidade de manutenção