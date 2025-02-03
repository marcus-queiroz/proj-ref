
# Documentação do Monitoramento de ETL no SSIS

## Introdução

Este sistema de monitoramento foi projetado para rastrear e auditar todas as execuções de processos ETL no SQL Server Integration Services (SSIS). Ele fornece uma estrutura robusta para registrar cada estágio do processo, permitindo um rastreamento preciso, propagação automática de falhas e coleta de métricas operacionais.

### O que este monitoramento resolve?

- **Garantia de rastreamento completo**: Cada execução do ETL é registrada, desde o início até a conclusão.
    
- **Gerenciamento de dependências**: Valida se subprocessos necessários foram concluídos antes de continuar.
    
- **Detecção e propagação de falhas**: Se uma etapa falhar, essa informação é refletida no nível do subprocesso e processo principal.
    
- **Coleta de métricas operacionais**: Registra tempo de execução, volumetria de registros processados e arquivos utilizados.
    
- **Centralização do log**: Todas as execuções ficam armazenadas em tabelas, permitindo auditoria e análise histórica.

---

## Estrutura do Monitoramento

O sistema é baseado em três níveis de granularidade para permitir uma visão detalhada da execução do ETL:

- **Nível de Processo**: Representa a execução completa do ETL, desde o início até o fim.
    
- **Nível de Subprocesso**: Divide o ETL em partes menores e interdependentes. Cada subprocesso pode ter múltiplos eventos.
    
- **Nível de Evento**: Armazena registros detalhados de cada atividade realizada dentro de um subprocesso, incluindo execuções de queries, cargas de dados e chamadas de procedures.
    

Cada nível é armazenado em uma tabela de log, garantindo visibilidade total e possibilitando auditorias futuras.

---

## Tabelas de Log

O sistema utiliza **três tabelas principais** para armazenar logs de execução, garantindo que cada etapa do processo ETL seja devidamente monitorada.

### 1. [ctrl].[ETLEvents_Process] – Processos ETL

**Objetivo:**

- Mantém um histórico de todas as execuções do ETL, permitindo rastreamento de falhas e análise de performance.
    
- Permite associar múltiplos workers à mesma execução, facilitando paralelismo.

|**Coluna**|**Descrição**|
|---|---|
|`EventID`|Identificador único do evento|
|`ProcessInstanceId`|GUID único gerado para cada execução do ETL|
|`Worker`|Identificador do worker responsável pela execução|
|`ProcessName`|Nome do processo ETL sendo executado|
|`EventDateTime`|Data e hora de início do processo|
|`DurationInSeconds`|Tempo total de execução do processo|
|`RecordCount`|Quantidade de registros processados|
|`NumberFiles`|Quantidade de arquivos manipulados|
|`EventStatus`|Status final (`In Progress`, `Success`, `Failure`)|

---

### 2. [ctrl].[ETLEvents_SubProcess] – Subprocessos

**Objetivo:**

- Permite quebrar grandes execuções ETL em partes menores, melhorando a rastreabilidade e o controle de dependências.
    
- Propaga falhas automaticamente, garantindo que subprocessos impactem o processo corretamente.

|**Coluna**|**Descrição**|
|---|---|
|`EventID`|Identificador único do evento|
|`ProcessInstanceId`|GUID do processo ao qual o subprocesso pertence|
|`Worker`|Identificador do worker|
|`ProcessName`|Nome do processo ETL principal|
|`SubProcessId`|Identificador único do subprocesso|
|`SubProcessName`|Nome do subprocesso (exemplo: 'Carga Tabela A', 'Transformação X')|
|`PrecedenceSubProcess`|Nome do subprocesso anterior, se houver dependências|
|`EventDateTime`|Data e hora de início do subprocesso|
|`DurationInSeconds`|Tempo total de execução do subprocesso|
|`EventStatus`|Status do subprocesso (`In Progress`, `Success`, `Failure`)|

---

### 3. [ctrl].[ETLEvents] – Eventos

**Objetivo:**

- Registra cada ação individual dentro de um subprocesso, permitindo rastreabilidade detalhada.
    
- Permite associar operações a tabelas e procedures, garantindo monitoramento granular.

|**Coluna**|**Descrição**|
|---|---|
|`EventID`|Identificador único do evento|
|`ProcessInstanceId`|GUID do processo|
|`Worker`|Identificador do worker|
|`ProcessName`|Nome do processo principal|
|`SubProcessId`|Identificador do subprocesso ao qual o evento pertence|
|`TableId`|Identificador da tabela processada (se aplicável)|
|`PrecedenceSubProcess`|Nome do subprocesso anterior|
|`PrecedenceTableId`|Nome da tabela anterior (se aplicável)|
|`OperationType`|Tipo da operação (`Full Load`, `Procedure`, etc.)|
|`TableName`|Nome da tabela processada|
|`ProcedureName`|Nome da procedure executada (se aplicável)|
|`EventStatus`|Status final do evento (`Success`, `Failure`)|
|`ErrorMessage`|Detalhes do erro, se houver|

---

## Como os logs funcionam na prática?

O SSIS insere registros nessas tabelas automaticamente à medida que os processos e subprocessos são executados. Isso permite:

- Consultar logs para diagnosticar falhas rapidamente.
    
- Rastrear dependências entre subprocessos e evitar que falhas passem despercebidas.
    
- Obter métricas detalhadas sobre tempo de execução e volumetria.
    
- Propagar erros automaticamente do nível de evento até o nível de processo.



## Procedures de Monitoramento

O controle do processo ETL é realizado por três procedures principais, que gerenciam a inicialização, execução e finalização dos subprocessos e eventos:

1. **ctrl.SensorStart**: Marca o início de um evento ou subprocesso e gerencia dependências.
    
2. **ctrl.SensorEnd**: Finaliza eventos e subprocessos, registrando tempo de execução e status.
    
3. **ctrl.EndPipeline**: Garante que o processo completo seja encerrado corretamente, propagando erros.
    

Além disso, existe uma procedure auxiliar:

- **ctrl.pInsETLEvents**: Recebe registros de execução das procedures do ETL e insere no log.

---

## 1. SensorStart – Início do Evento

### O que essa procedure resolve?

- Registra o início de um subprocesso no log.
- Garante que um subprocesso anterior tenha sido finalizado antes de iniciar o próximo.
- Propaga erros ocorridos em subprocessos anteriores, evitando execuções indevidas.

### Fluxo de Execução

1. **Verifica se há um subprocesso ainda em andamento**
    
    - Se existir, finaliza o anterior antes de iniciar um novo.
    - Isso garante que subprocessos não fiquem "abertos" indefinidamente.
2. **Registra a inicialização do subprocesso atual**
    
    - Insere um novo registro na tabela `[ctrl].[ETLEvents_SubProcess]`.
    - Marca o evento com `EventStatus = 'In Progress'`.
3. **Propaga falhas caso existam eventos com `Failure` no subprocesso anterior**
    
    - Se um subprocesso anterior falhou, o status do atual já começa como `Failure`.
4. **Registra a execução do evento individual (nível de tabela)**
    
    - Insere um registro na `[ctrl].[ETLEvents]`, permitindo rastreamento detalhado.

**Exemplo prático:**

- O ETL está processando a **Tabela A**.
- O subprocesso **"Carga Inicial"** falha.
- O próximo subprocesso recebe o status `Failure` e não executa sem intervenção manual.

## 2. SensorEnd – Finalização do Evento

### O que essa procedure resolve?

- Finaliza subprocessos e eventos individuais no log.
- Calcula tempo de execução (`DurationInSeconds`).
- Propaga falhas se houver erros em eventos individuais.

### Fluxo de Execução

1. **Verifica se o evento realmente foi iniciado**
    
    - Se não encontrar um registro correspondente em `[ctrl].[ETLEvents]`, retorna erro.
2. **Calcula o tempo de execução (`DurationInSeconds`)**
    
    - Obtém a diferença entre o tempo inicial (`EventDateTime`) e o horário atual.
3. **Determina o status final (`Success` ou `Failure`)**
    
    - Se o evento foi marcado como `Failure`, mantém esse status.
    - Caso contrário, registra `Success`.
4. **Propaga falhas para o subprocesso**
    
    - Se um evento individual falhar, o subprocesso também é marcado como `Failure`.

**Exemplo prático:**

- A **Tabela B** está sendo carregada.
- O evento individual (`INSERT INTO ...`) falha.
- O status do evento é atualizado para `Failure`.
- O subprocesso inteiro recebe o status `Failure`.

## 3. EndPipeline – Finalização do Processo Completo

### O que essa procedure resolve?

- Garante que todos os subprocessos sejam encerrados corretamente.
- Propaga falhas de subprocessos para o nível do processo.
- Registra a duração total da execução.

### Fluxo de Execução

1. **Finaliza subprocessos ainda abertos**
    
    - Se algum subprocesso ainda estiver `In Progress`, ele é encerrado com `Success` ou `Failure`.
2. **Propaga erros de subprocessos para o processo principal**
    
    - Se qualquer subprocesso falhou, o status do processo principal é atualizado para `Failure`.
3. **Calcula tempo total do processo**
    
    - Obtém a diferença entre `EventDateTime` inicial e horário atual.
4. **Finaliza o registro na `[ctrl].[ETLEvents_Process]`**
    
    - Define `EventStatus = 'Success'` ou `Failure`, dependendo do que ocorreu nos subprocessos.

**Exemplo prático:**

- O ETL tem **5 subprocessos**.
- O **Subprocesso 3 falha**.
- O processo inteiro é marcado como `Failure`, evitando resultados inconsistentes.

## 4. pInsETLEvents – Registro de Logs de Procedures do ETL

### O que essa procedure resolve?

- Permite que procedures customizadas do ETL registrem logs diretamente na tabela `[ctrl].[ETLEvents]`.
- Calcula automaticamente tempo de execução (`DurationInSeconds`).
- Propaga falhas se um erro for detectado.

Essa procedure é chamada por outras procedures do ETL para registrar eventos no log, sem depender do SSIS.

**Exemplo prático:**

- A procedure `stage.pETLProcedureTemplate` executa um `MERGE` e registra a quantidade de registros afetados.
- Ela chama `pInsETLEvents` passando `RecordCount`, `EventStatus` e `ErrorMessage`.
- O log é atualizado automaticamente.

## Conclusão

O sistema de monitoramento implementado garante rastreabilidade total da execução do ETL no SSIS.

- Registra cada etapa do processo.
- Calcula tempo de execução e volumetria.
- Propaga falhas automaticamente, evitando execuções indevidas.
- Permite auditoria completa, centralizando logs em tabelas SQL.

Com essa estrutura, qualquer falha pode ser detectada rapidamente, e o histórico de execuções pode ser analisado para melhorar a eficiência do ETL.













