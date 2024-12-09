### **Parte 1: Explicação Conceitual e Funcionamento**

O objetivo desta solução é criar um sistema eficiente e escalável para processar arquivos de diferentes clientes, garantindo controle centralizado e evitando conflitos entre os processos. A solução combina o SQL Server e o SSIS para gerenciar o fluxo de trabalho e dividir a carga entre diferentes pacotes de importação (workers).

---

### **1. Objetivo do Sistema**

1. **Centralizar o Controle**:
    - Gerenciar quais arquivos devem ser processados, qual cliente está relacionado a esses arquivos e qual worker está responsável por processá-los.
2. **Evitar Conflitos**:
    - Garantir que um grupo de arquivos de um cliente (`ClienteA`) seja processado por um único worker, evitando execução simultânea para o mesmo cliente.
3. **Dividir a Carga**:
    - Distribuir os processos entre múltiplos pacotes (workers), otimizando o tempo de processamento.

---

### **2. Funcionamento Geral**

A solução é composta por três elementos principais:

1. **Tabela de Controle (`FileControl`)**:
    
    - Gerencia quais grupos de tabelas (ex.: `ClienteA`, `ClienteB`) precisam ser processados e qual worker foi designado.
2. **Tabela de Parâmetros dos Arquivos (`FileParameters`)**:
    
    - Armazena os detalhes de cada arquivo dentro de um processo (como nome do arquivo, parâmetros adicionais, status, caminho do diretório).
    - Relaciona os arquivos com o processo correspondente.
3. **Pacotes SSIS e Lógica SQL**:
    
    - **Pacote Master**:
		- Chama o worker designado (definido na procedure).
		- O worker processa os arquivos associados ao cliente e atualiza a tabela `FileControl` ao final.
    - **Pacotes Workers**:
        - Cada worker é um pacote genérico que processa os arquivos de um único cliente ou grupo.

---

### **3. Fluxo de Trabalho**

#### **Passo 1: Registro de Processos e Arquivos**

- Quando a aplicação identifica arquivos para processamento:
    - Cria um registro na tabela `FileControl` para o grupo de arquivos (cliente/processo).
    - Insere os detalhes dos arquivos na tabela `FileParameters`, vinculados ao processo registrado.

#### **Passo 2: Seleção de Processos**

- Um **job no SQL Server Agent** executa uma procedure periodicamente.
- A procedure verifica na tabela `FileControl` se há processos pendentes.
    - Se houver, ela:
        - Seleciona o próximo processo pendente.
        - Atribui um worker disponível a esse processo.
        - Atualiza o status do processo para `InProgress`.

#### **Passo 3: Execução do Worker**

- O pacote SSIS Master:
    - Chama a procedure para obter os detalhes do processo (qual worker e cliente).
    - Executa o pacote worker correspondente, passando o `ProcessID` como parâmetro.

#### **Passo 4: Processamento dos Arquivos**

- O worker SSIS processa os arquivos relacionados ao `ProcessID`:
    - Atualiza o status dos arquivos conforme são processados.
    - Ao finalizar, atualiza o status do processo na tabela `FileControl` para `Completed` ou `Failed`.

---

### **4. Controle de Concorrência**

- **Atribuição Exclusiva**:
    
    - Cada processo (cliente) é atribuído a apenas um worker.
    - O SQL Server garante que apenas um processo será marcado como `InProgress` por vez, usando bloqueios e transações.
- **Evitar Processos Travados**:
    
    - Um script periódicos redefine processos travados:
        - Processos com status `InProgress` por muito tempo são reatribuídos como `Pending`.

---

### **5. Vantagens da Solução**

1. **Escalabilidade**:
    - É possível adicionar mais pacotes workers para processar arquivos simultaneamente.
2. **Resiliência**:
    - Processos travados ou com falha podem ser reprocessados sem afetar outros processos.
3. **Organização Centralizada**:
    - Todo o controle está centralizado nas tabelas `FileControl` e `FileParameters`, permitindo visibilidade do estado de cada arquivo e processo.
4. **Flexibilidade**:
    - A lógica de atribuição de workers pode ser adaptada conforme as necessidades (ex.: prioridade por cliente, tipo de dado).

---

Na próxima parte, vou detalhar a **implementação técnica**: estrutura de tabelas, procedimentos armazenados, configuração dos pacotes SSIS, e exemplos de código SQL e C#. 😊




### **Parte 2: Implementação Técnica**

Aqui está o detalhamento técnico para implementar a solução discutida anteriormente. Essa parte abrange a estrutura das tabelas, procedures no SQL Server e configuração dos pacotes SSIS (Master e Workers).

---

### **1. Estrutura das Tabelas**

#### **1.1. Tabela `FileControl`**

Gerencia os processos de importação (um para cada cliente ou grupo).
```sql
CREATE TABLE FileControl (
    ProcessID INT IDENTITY PRIMARY KEY,       -- Identificador único do processo
    ProcessName NVARCHAR(255),                -- Nome do cliente ou grupo (ex.: ClienteA)
    WorkerAssigned NVARCHAR(50),              -- Nome do worker designado
    Status NVARCHAR(50),                      -- Status: Pending, InProgress, Completed, Failed
    CreatedAt DATETIME DEFAULT GETDATE(),     -- Data/hora de criação do processo
    StartTime DATETIME NULL,                  -- Data/hora em que o processamento começou
    EndTime DATETIME NULL                     -- Data/hora em que o processamento terminou
);
```

#### **1.2. Tabela `FileParameters`**

Armazena os detalhes dos arquivos relacionados a cada processo.
```sql
CREATE TABLE FileParameters (
    FileID INT IDENTITY PRIMARY KEY,          -- Identificador único do arquivo
    ProcessID INT,                            -- Relacionado ao ProcessID da tabela FileControl
    FileName NVARCHAR(255),                   -- Nome do arquivo
    Parameter1 NVARCHAR(255),                 -- Parâmetro adicional (se necessário)
    Parameter2 NVARCHAR(255),                 -- Outro parâmetro adicional
    Status NVARCHAR(50),                      -- Status: Pending, InProgress, Completed, Failed
    CreatedAt DATETIME DEFAULT GETDATE(),     -- Data/hora em que o arquivo foi registrado
    StartTime DATETIME NULL,                  -- Data/hora de início do processamento
    EndTime DATETIME NULL                     -- Data/hora de conclusão do processamento
);
```


---

### **2. Procedures no SQL Server**

#### **2.1. Procedure: Registrar Processos e Arquivos**

Esta procedure insere os processos e os arquivos relacionados.
```sql
CREATE PROCEDURE RegisterProcessAndFiles
    @ProcessName NVARCHAR(255),
    @FileList NVARCHAR(MAX) -- Lista de arquivos separada por vírgula
AS
BEGIN
    BEGIN TRAN;

    -- Inserir o processo na tabela FileControl
    INSERT INTO FileControl (ProcessName, Status)
    VALUES (@ProcessName, 'Pending');

    DECLARE @ProcessID INT = SCOPE_IDENTITY();

    -- Dividir a lista de arquivos e inseri-los na tabela FileParameters
    DECLARE @File NVARCHAR(255);
    DECLARE @Pos INT;

    WHILE CHARINDEX(',', @FileList) > 0
    BEGIN
        SET @Pos = CHARINDEX(',', @FileList);
        SET @File = LTRIM(RTRIM(LEFT(@FileList, @Pos - 1)));
        SET @FileList = SUBSTRING(@FileList, @Pos + 1, LEN(@FileList));

        INSERT INTO FileParameters (ProcessID, FileName, Status)
        VALUES (@ProcessID, @File, 'Pending');
    END;

    -- Inserir o último arquivo
    INSERT INTO FileParameters (ProcessID, FileName, Status)
    VALUES (@ProcessID, LTRIM(RTRIM(@FileList)), 'Pending');

    COMMIT TRAN;
END;
```


---

#### **2.2. Procedure: Atribuir Worker a um Processo**

Esta procedure bloqueia um processo e atribui um worker para execução.
```sql
CREATE PROCEDURE AssignWorkerToProcess
AS
BEGIN
    BEGIN TRAN;

    DECLARE @ProcessID INT, @ProcessName NVARCHAR(255), @WorkerAssigned NVARCHAR(50);

    -- Selecionar o próximo processo pendente
    SELECT TOP 1 @ProcessID = ProcessID, @ProcessName = ProcessName
    FROM FileControl WITH (ROWLOCK, UPDLOCK, READPAST)
    WHERE Status = 'Pending'
    ORDER BY CreatedAt;

    -- Se não houver processos pendentes, encerrar
    IF @ProcessID IS NULL
    BEGIN
        COMMIT TRAN;
        RETURN;
    END;

    -- Atribuir o worker
    SET @WorkerAssigned = 'Worker_' + @ProcessName;

    -- Atualizar o processo para InProgress e associar o worker
    UPDATE FileControl
    SET Status = 'InProgress', WorkerAssigned = @WorkerAssigned, StartTime = GETDATE()
    WHERE ProcessID = @ProcessID;

    -- Retornar os detalhes do processo para o SSIS
    SELECT @ProcessID AS ProcessID, @ProcessName AS ProcessName, @WorkerAssigned AS WorkerAssigned;

    COMMIT TRAN;
END;
```


---

### **3. Configuração dos Pacotes SSIS**

#### **3.1. Pacote Master**

O Master gerencia a distribuição de processos e invoca o worker designado.

##### **Componentes Principais**:

1. **Execute SQL Task**:
    
    - Chama a procedure `AssignWorkerToProcess`.
    - Armazena os resultados (ProcessID, ProcessName, WorkerAssigned) em variáveis SSIS.
2. **Expression Task ou Condicional**:
    
    - Determina qual worker será executado com base na variável `WorkerAssigned`.
3. **Execute Package Task**:
    
    - Invoca o worker designado.
    - Passa o `ProcessID` como parâmetro para o worker.

---

#### **3.2. Pacotes Workers**

Os workers processam os arquivos de um único cliente.

##### **Componentes Principais**:

1. **Execute SQL Task**:
    
    - Seleciona os arquivos relacionados ao `ProcessID`:
```sql
SELECT FileName, Parameter1, Parameter2
FROM FileParameters
WHERE ProcessID = ? AND Status = 'Pending';
```
        
    - Atualiza o status dos arquivos para `InProgress`:
```sql
UPDATE FileParameters
SET Status = 'InProgress', StartTime = GETDATE()
WHERE ProcessID = ? AND Status = 'Pending';
```
        
2. **Data Flow Task**:
    
    - Processa os arquivos usando os parâmetros carregados.
3. **Execute SQL Task**:
    
    - Após o processamento, atualiza o status dos arquivos para `Completed` ou `Failed`.
4. **Finalizar o Processo**:
    
    - Após processar todos os arquivos, atualiza o status do processo na tabela `FileControl`:
```sql
UPDATE FileControl
SET Status = 'Completed', EndTime = GETDATE()
WHERE ProcessID = ?;
```
        

---

### **4. Job no SQL Server Agent**

O SQL Server Agent roda periodicamente para iniciar o pacote Master.

##### **Configuração**:

1. **Job Step 1**: Execute o pacote Master.
2. **Frequência**: Agende para rodar a cada 1 minuto.

---

### **5. Controle de Falhas**

#### **5.1. Timeout para Processos Travados**

Scripts periódicos redefinem processos ou arquivos travados:
```sql
UPDATE FileControl
SET Status = 'Pending', WorkerAssigned = NULL, StartTime = NULL
WHERE Status = 'InProgress' AND DATEDIFF(MINUTE, StartTime, GETDATE()) > 60;
```

```sql
UPDATE FileParameters
SET Status = 'Pending', StartTime = NULL
WHERE Status = 'InProgress' AND DATEDIFF(MINUTE, StartTime, GETDATE()) > 60;
```


---

### **Resumo**

- **SQL Server** gerencia a fila de processos e bloqueia execuções simultâneas.
- **SSIS Master** distribui os processos para os workers.
- **SSIS Workers** processam os arquivos de um cliente.
- **Resiliência** é garantida com scripts de timeout e mecanismos de retry.








### **Parte 3: Monitoramento e Logging**

O monitoramento e o logging são cruciais para garantir a visibilidade do funcionamento da solução, identificar problemas rapidamente e rastrear o desempenho. Essa parte detalha como implementar uma estratégia de monitoramento eficiente.

---

### **1. Objetivos do Monitoramento e Logging**

- **Visibilidade**: Saber quais processos estão sendo executados e seu status.
- **Rastreamento de Erros**: Capturar falhas e detalhar o contexto para facilitar a depuração.
- **Métricas de Desempenho**: Identificar gargalos ou processos que levam mais tempo do que o esperado.
- **Histórico de Execuções**: Ter um registro confiável de todas as execuções para auditorias e análises.

---

### **2. Estrutura de Logging**

#### **2.1. Logging no Banco de Dados**

Crie tabelas para registrar logs detalhados dos processos e arquivos.

1. **Tabela de Logs de Processos (`ProcessLog`)**
    
    - Registra eventos relacionados aos processos de importação (nível macro).
```sql
CREATE TABLE ProcessLog (
    LogID INT IDENTITY PRIMARY KEY,       -- Identificador único do log
    ProcessID INT,                        -- Relacionado ao FileControl
    Message NVARCHAR(500),                -- Mensagem de log
    LogLevel NVARCHAR(50),                -- Nível: Info, Warning, Error
    LogTime DATETIME DEFAULT GETDATE()    -- Data/hora do log
);
```
    
    **Exemplo de Registros**:
    
    |LogID|ProcessID|Message|LogLevel|LogTime|
    |---|---|---|---|---|
    |1|101|Worker_ClienteA iniciado|Info|2024-11-25 08:00:00|
    |2|101|Arquivo Clientes.xlsx falhou|Error|2024-11-25 08:01:00|
    
2. **Tabela de Logs de Arquivos (`FileLog`)**
    
    - Registra eventos específicos para cada arquivo processado.
```sql
CREATE TABLE FileLog (
    LogID INT IDENTITY PRIMARY KEY,       -- Identificador único do log
    FileID INT,                           -- Relacionado ao FileParameters
    Message NVARCHAR(500),                -- Mensagem de log
    LogLevel NVARCHAR(50),                -- Nível: Info, Warning, Error
    LogTime DATETIME DEFAULT GETDATE()    -- Data/hora do log
);
```
    
    **Exemplo de Registros**:
    
    |LogID|FileID|Message|LogLevel|LogTime|
    |---|---|---|---|---|
    |1|1001|Arquivo validado com sucesso|Info|2024-11-25 08:00:30|
    |2|1002|Erro ao conectar ao banco|Error|2024-11-25 08:02:00|
    

---

### **3. Implementação de Logging no SSIS**

1. **Configuração do Provedor de Logs no SSIS**:
    
    - Configure o provedor de logs para gravar eventos importantes em uma tabela SQL.
    - No **SSIS**:
        - Vá até a aba **SSIS > Logging**.
        - Selecione **SSIS log provider for SQL Server**.
        - Escolha os eventos a serem registrados (ex.: `OnError`, `OnInformation`).
2. **Logs Personalizados no SSIS**:
    
    - Use scripts personalizados para gravar logs diretamente nas tabelas:
```csharp
Dts.Events.FireInformation(0, "WorkerPackage", "Processando o arquivo Clientes.xlsx", "", 0, ref fireAgain);

string query = "INSERT INTO FileLog (FileID, Message, LogLevel) VALUES (?, ?, ?)";
using (SqlConnection conn = new SqlConnection(Dts.Connections["DBConnection"].ConnectionString))
{
    SqlCommand cmd = new SqlCommand(query, conn);
    cmd.Parameters.AddWithValue("@FileID", fileID);
    cmd.Parameters.AddWithValue("@Message", "Arquivo processado com sucesso.");
    cmd.Parameters.AddWithValue("@LogLevel", "Info");
    conn.Open();
    cmd.ExecuteNonQuery();
}
```
        

---

### **4. Dashboards para Monitoramento**

#### **4.1. Relatórios SQL Server Reporting Services (SSRS)**

Crie relatórios dinâmicos com base nas tabelas de log:

1. **Visão Geral dos Processos**:
    
    - Total de processos pendentes, em execução, completados e falhos.
    - Gráficos de desempenho (tempo médio por processo).
2. **Visão Detalhada por Cliente**:
    
    - Relatório por cliente, mostrando os arquivos processados, tempo médio e falhas.

#### **4.2. Power BI para Monitoramento Visual**

1. Conecte-se ao banco SQL Server.
2. Crie gráficos dinâmicos:
    - **Status dos Processos**: Pizza mostrando `Pending`, `InProgress`, `Completed`.
    - **Falhas por Cliente**: Barra mostrando falhas em processos por cliente.
    - **Tempo Médio por Arquivo**: Linha temporal comparando clientes.

---

### **5. Alertas Automatizados**

#### **5.1. Alertas por Email**

Configure alertas para erros críticos:

- Use o SQL Server Database Mail:
```sql
EXEC msdb.dbo.sp_send_dbmail
    @profile_name = 'DefaultProfile',
    @recipients = 'admin@company.com',
    @subject = 'Erro no WorkerPackage',
    @body = 'O arquivo Clientes.xlsx falhou no processamento.';
```  
    
- Configure os alertas em triggers na tabela de log para enviar notificações em erros:
```sql
CREATE TRIGGER trg_FileLog_Error
ON FileLog
AFTER INSERT
AS
BEGIN
    IF EXISTS (SELECT 1 FROM inserted WHERE LogLevel = 'Error')
    BEGIN
        EXEC msdb.dbo.sp_send_dbmail
            @profile_name = 'DefaultProfile',
            @recipients = 'admin@company.com',
            @subject = 'Erro Crítico no Processo',
            @body = 'Erro identificado. Verifique os logs detalhados.';
    END
END;
```
    

#### **5.2. Monitoramento de Performance**

- Configure alertas para longas execuções:
```sql
SELECT ProcessID, DATEDIFF(MINUTE, StartTime, GETDATE()) AS ExecutionTime
FROM FileControl
WHERE Status = 'InProgress' AND DATEDIFF(MINUTE, StartTime, GETDATE()) > 60;
```
    
- Dispare uma ação para reavaliar ou notificar longas execuções.
    

---

### **6. Estratégia de Retenção de Logs**

1. **Rotação de Logs**:
    
    - Mova logs antigos para uma tabela de histórico:
```sql
INSERT INTO ProcessLog_History SELECT * FROM ProcessLog WHERE LogTime < DATEADD(MONTH, -6, GETDATE());
DELETE FROM ProcessLog WHERE LogTime < DATEADD(MONTH, -6, GETDATE());
```
        
2. **Armazenamento em Arquivos**:
    
    - Exporte logs críticos para um arquivo CSV ou JSON, se necessário.

---

### **Resumo**

- **Tabelas de Log** registram eventos detalhados para processos e arquivos.
- **Dashboards** em SSRS ou Power BI oferecem visibilidade clara.
- **Alertas** automáticos notificam falhas e problemas de desempenho.
- **Rotação de Logs** mantém a solução eficiente a longo prazo








### **Parte 4: Planejamento para Manutenções e Dinâmica de Trabalho**

Para garantir que a solução seja sustentável a longo prazo e minimamente impactada por mudanças no ambiente ou nos requisitos, é essencial adotar práticas de **manutenção planejada** e implementar uma **dinâmica de trabalho bem definida**, incluindo estratégias de **CI/CD (Integração Contínua e Entrega Contínua)** para o ciclo de produção.

---

### **1. Planejamento para Manutenções**

#### **1.1. Manutenções Preventivas**

- **Verificação de Logs e Status**:
    
    - Periodicamente, revise as tabelas de logs e controle (`FileControl`, `FileParameters`) para identificar processos travados ou falhas recorrentes.
    - Execute scripts automatizados para redefinir processos pendentes:
```sql
UPDATE FileControl
SET Status = 'Pending', WorkerAssigned = NULL, StartTime = NULL
WHERE Status = 'InProgress' AND DATEDIFF(MINUTE, StartTime, GETDATE()) > 60;
```
        
- **Rotação de Dados**:
    
    - Limpe registros antigos ou mova-os para tabelas de histórico para evitar crescimento excessivo:
```sql
INSERT INTO FileControl_History SELECT * FROM FileControl WHERE EndTime < DATEADD(MONTH, -6, GETDATE());
DELETE FROM FileControl WHERE EndTime < DATEADD(MONTH, -6, GETDATE());
```
        
- **Validação de Pacotes SSIS**:
    
    - Verifique regularmente se os pacotes SSIS estão funcionais e conectados às fontes corretas (ex.: bases de dados ou pastas de arquivos).

#### **1.2. Manutenções Corretivas**

- **Erro em Pacotes ou Scripts**:
    
    - Registre exceções críticas em logs (`ProcessLog` ou `FileLog`).
    - Crie uma política de reprocessamento automático para arquivos falhos:
```sql
UPDATE FileParameters
SET Status = 'Pending'
WHERE Status = 'Failed';

```
        
- **Atualizações de Requisitos**:
    
    - Alterações no formato dos arquivos ou requisitos de transformação podem exigir ajustes nos workers.
    - Implemente **testes unitários** e de integração para validar novas transformações antes da implantação.

#### **1.3. Planejamento de Capacity**

- Estime o crescimento dos dados e número de clientes:
    - Adicione mais workers conforme necessário para equilibrar a carga.
    - Atualize o ambiente de banco de dados ou SSIS para suportar maior paralelismo.

---



### **2. Estratégias para Mitigação de Erros**

#### **2.1. Testes Automatizados**

1. **Testes de Unidade**:
    - Testar individualmente transformações no banco de dados.
2. **Testes de Integração**:
    - Simular a execução completa de pacotes SSIS usando arquivos de teste.
3. **Testes de Performance**:
    - Avaliar o impacto de múltiplos processos executando simultaneamente.

#### **2.2. Estratégia de Rollback**

- **Banco de Dados**:
    
    - Use transações para evitar alterações parciais:
```sql
BEGIN TRAN;
-- Scripts de alteração
ROLLBACK TRAN; -- Ou COMMIT TRAN, se for bem-sucedido
```
        
- **Pacotes SSIS**:
    
    - Mantenha versões anteriores dos pacotes para reverter rapidamente.

#### **2.3. Acompanhamento Pós-Deploy**

- **Revisão**:
    - Após cada deploy, revise logs e métricas de execução.
- **Checklist de Deploy**:
    - Inclua validações manuais ou automatizadas para garantir que o sistema está funcional.

---

### **Resumo**

A manutenção da solução requer:

1. **Monitoramento contínuo** para identificar problemas rapidamente.
2. **Dinâmica de trabalho organizada**, com controle de versão e automação de deploys.
3. **Planejamento de rollback** para mitigar riscos no ciclo de produção





--- 





### **Resumo mais direto**

1. **Tabela `FileControl`**:
    
    - Gerencia quais grupos de tabelas (ex.: `ClienteA`, `ClienteB`) precisam ser processados e qual worker foi designado.
2. **Tabela `FileParameters`**:
    
    - Armazena os detalhes de cada arquivo (ex.: parâmetros de importação).
3. **Procedure no SQL Server**:
    
    - Bloqueia um processo (`ClienteA`) para um worker.
    - Define qual pacote worker será chamado pelo SSIS.
    - Atualiza o status da tabela `FileControl`.
4. **Pacote SSIS Master**:
    
    - Chama o worker designado (definido na procedure).
    - O worker processa os arquivos associados ao cliente e atualiza a tabela `FileControl` ao final.
5. **Workers SSIS**:
    
    - Cada worker é um pacote genérico que processa os arquivos de um único cliente ou grupo.

---

### **Passo 1: Estrutura das Tabelas**

#### **Tabela `FileControl`**

Gerencia quais processos (clientes) serão executados e os workers associados.
```sql
CREATE TABLE FileControl (
    ProcessID INT IDENTITY PRIMARY KEY,        -- Identificador único do processo
    ProcessName NVARCHAR(255),                -- Nome do cliente ou processo (ex.: ClienteA)
    WorkerAssigned NVARCHAR(50),              -- Nome do worker reservado
    Status NVARCHAR(50),                      -- Status: Pending, InProgress, Completed, Failed
    CreatedAt DATETIME DEFAULT GETDATE(),     -- Data/hora de criação
    StartTime DATETIME NULL,                  -- Data/hora de início
    EndTime DATETIME NULL                     -- Data/hora de conclusão
);
```

#### **Tabela `FileParameters`**

Armazena os detalhes dos arquivos que serão processados pelo worker.
```sql
CREATE TABLE FileParameters (
    FileID INT IDENTITY PRIMARY KEY,          -- Identificador único do arquivo
    ProcessID INT,                            -- Relacionado ao ProcessID da FileControl
    FileName NVARCHAR(255),                   -- Nome do arquivo
    Parameter1 NVARCHAR(255),                 -- Parâmetro de importação 1
    Parameter2 NVARCHAR(255),                 -- Parâmetro de importação 2
    Status NVARCHAR(50),                      -- Status: Pending, InProgress, Completed, Failed
    CreatedAt DATETIME DEFAULT GETDATE(),     -- Data/hora de criação
    StartTime DATETIME NULL,                  -- Data/hora de início
    EndTime DATETIME NULL                     -- Data/hora de conclusão
);
```


---

### **Passo 2: Procedure no SQL Server**

A procedure gerencia a lógica de atribuição de workers e bloqueio de processos.
```sql
CREATE PROCEDURE AssignWorkerToProcess
AS
BEGIN
    BEGIN TRAN;

    DECLARE @ProcessID INT, @ProcessName NVARCHAR(255), @WorkerAssigned NVARCHAR(50);

    -- Seleciona o próximo processo pendente
    SELECT TOP 1 @ProcessID = ProcessID, @ProcessName = ProcessName
    FROM FileControl WITH (ROWLOCK, UPDLOCK, READPAST)
    WHERE Status = 'Pending'
    ORDER BY CreatedAt;

    -- Se nenhum processo estiver pendente, encerra
    IF @ProcessID IS NULL
    BEGIN
        COMMIT TRAN;
        RETURN;
    END;

    -- Define qual worker será atribuído (você pode implementar uma lógica mais complexa aqui)
    SET @WorkerAssigned = 'Worker_' + @ProcessName;

    -- Atualiza o status do processo para InProgress e associa o worker
    UPDATE FileControl
    SET Status = 'InProgress', WorkerAssigned = @WorkerAssigned, StartTime = GETDATE()
    WHERE ProcessID = @ProcessID;

    -- Retorna os detalhes para o SSIS (se necessário)
    SELECT @ProcessID AS ProcessID, @ProcessName AS ProcessName, @WorkerAssigned AS WorkerAssigned;

    COMMIT TRAN;
END;
```


---

### **Passo 3: Pacote SSIS Master**

O pacote Master é responsável por:

1. Chamar a procedure `AssignWorkerToProcess`.
2. Capturar os detalhes do processo (cliente e worker designado).
3. Invocar o worker correspondente.

#### **Configuração no SSIS Master**:

1. **Execute SQL Task**:
    
    - Chama a procedure `AssignWorkerToProcess` e captura os resultados (ProcessID, ProcessName, WorkerAssigned).
    - Armazene os valores em variáveis SSIS:
        - `ProcessID`
        - `ProcessName`
        - `WorkerAssigned`
2. **Expression Task ou Condicional**:
    
    - Use a variável `WorkerAssigned` para determinar qual worker será chamado.
3. **Execute Package Task**:
    
    - Chama o worker designado (`Worker_ClienteA`, `Worker_ClienteB`, etc.).
    - Passe o `ProcessID` e/ou outros parâmetros necessários como variáveis.

---

### **Passo 4: Workers SSIS**

Cada worker é genérico e processa os arquivos associados ao `ProcessID`.

#### **Configuração no Worker**:

1. **Execute SQL Task**:
    
    - Seleciona os arquivos associados ao `ProcessID` e atualiza o status para `InProgress`:
```sql
UPDATE FileParameters
SET Status = 'InProgress', StartTime = GETDATE()
WHERE ProcessID = ? AND Status = 'Pending';
```
        
2. **Data Flow Task**:
    
    - Processa os arquivos, usando os parâmetros armazenados na tabela `FileParameters`.
3. **Finalizar o Processo**:
    
    - Após concluir todos os arquivos, atualiza o status do processo na tabela `FileControl` para `Completed`:
```sql
UPDATE FileControl
SET Status = 'Completed', EndTime = GETDATE()
WHERE ProcessID = ?;
```
        

---

### **Exemplo de Execução**

1. **Job no SQL Server Agent**:
    
    - Executa o pacote Master a cada minuto.
2. **Fluxo do Pacote Master**:
    
    - A procedure atribui um worker a um processo pendente.
    - O pacote Master chama o worker correspondente.
3. **Fluxo do Worker**:
    
    - Processa os arquivos do cliente.
    - Atualiza os status nas tabelas `FileControl` e `FileParameters`.

---

### **Mitigação de Riscos**

1. **Arquivos Presos em `InProgress`**:
    
    - Configure scripts para redefinir processos travados:
```sql
UPDATE FileControl
SET Status = 'Pending', WorkerAssigned = NULL, StartTime = NULL
WHERE Status = 'InProgress' AND DATEDIFF(MINUTE, StartTime, GETDATE()) > 60;
```
        
2. **Registro de Erros**:
    
    - Adicione logs detalhados em caso de falhas no worker.
3. **Controle de Exclusividade**:
    
    - A lógica de bloqueio com `ROWLOCK`, `UPDLOCK`, e `READPAST` garante que apenas um processo será atribuído por vez.

---

### **Resumo**

- **FileControl** controla os processos e workers atribuídos.
- **FileParameters** gerencia os arquivos associados a cada processo.
- A procedure garante que os processos sejam atribuídos sem conflitos.
- O Master SSIS distribui a carga para os workers, garantindo escalabilidade.




