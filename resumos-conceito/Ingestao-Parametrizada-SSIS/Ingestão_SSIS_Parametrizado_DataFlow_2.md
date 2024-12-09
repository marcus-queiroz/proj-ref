# Solução Final para Gerenciamento de Processos com Workers e SSIS

## **1. Fluxo Geral da Solução**

### **1.1. Registro de Processos**

1. A aplicação web insere novos processos no banco utilizando a procedure `[stage].[RegisterProcess]`.
    
    - Essa procedure verifica se já existe um processo com o mesmo nome em status `'Pending'` ou `'InProgress'`. Caso exista, impede a duplicidade.
        
    - Se não houver duplicidade, o processo é registrado com:
        
        - `Status = 'Pending'`.
            
        - Data de criação (`CreatedAt`).
            

**Exemplo de Registro:**

```sql
EXEC [stage].[RegisterProcess] @ProcessName = 'ProcessoA';
```

**Resultado no Banco:**

|ProcessID|ProcessName|Status|CreatedAt|WorkerAssigned|StartTime|EndTime|
|---|---|---|---|---|---|---|
|1|ProcessoA|Pending|2024-12-06 10:00:00.000|NULL|NULL|NULL|

---

### **1.2. Atribuição de Processos a Workers**

1. A procedure `[stage].[AssignProcessToWorker]` é executada periodicamente por um **job no SQL Server Agent** (ex.: a cada 1 minuto).
    
2. Ela verifica os processos pendentes e distribui para Workers disponíveis:
    
    - Um Worker é identificado pelo nome dinâmico, como `Worker_1`, `Worker_2`, etc.
        
    - A procedure assegura que apenas Workers que não estão ocupados (`Status = 'InProgress'`) sejam considerados.
        
3. Quando um processo é atribuído, a procedure:
    
    - Atualiza o status do processo para `'InProgress'`.
        
    - Define o `WorkerAssigned` como o Worker correspondente.
        
    - Registra o `StartTime`.
        

**Exemplo de Execução:**

```
EXEC [stage].[AssignProcessToWorker] @NumberOfWorkers = 3;
```

**Resultado no Banco:**

|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|ProcessID|ProcessName|Status|CreatedAt|WorkerAssigned|StartTime|EndTime|
|1|ProcessoA|InProgress|2024-12-06 10:00:00.000|Worker_1|2024-12-06 10:01:00.000|NULL|
|2|ProcessoB|InProgress|2024-12-06 10:00:30.000|Worker_2|2024-12-06 10:01:00.000|NULL|

---

### **1.3. Execução do Worker (Pacote SSIS)**

1. Cada Worker é representado por um **job no SQL Server Agent** que executa o pacote SSIS correspondente (ex.: `Worker_1` executa o pacote `Worker1Package.dtsx`).
    
2. O pacote SSIS realiza as seguintes etapas:
    
    - **Descobrir o processo atribuído:**
        
        - A tarefa **Executar SQL** chama a procedure `[stage].[ProcessWorkerTask]` para obter o `ProcessID` e `ProcessName` do processo atribuído ao Worker.
            
        - Caso não haja processo atribuído, a procedure retorna valores padrão (ex.: `NULL` ou uma mensagem indicativa).
            
    - **Processar o trabalho:**
        
        - Com base no `ProcessID`, o pacote executa a lógica de negócio (ETL, integração, etc.).
            
    - **Atualizar o status:**
        
        - Após concluir o trabalho, o pacote atualiza o status do processo para `'Completed'`.
            

**Exemplo de Chamada no SSIS:**

```sql
EXEC [stage].[ProcessWorkerTask] @WorkerName = 'Worker_1';
```

**Resultado da Procedure:**

|   |   |
|---|---|
|ProcessID|ProcessName|
|1|ProcessoA|

---

### **1.4. Finalização**

1. O pacote SSIS, após processar o trabalho, atualiza o status do processo no banco de dados para `'Completed'`:
    
    ```sql
    UPDATE [stage].[FileControl]
    SET Status = 'Completed', EndTime = GETDATE()
    WHERE ProcessID = ?;
    ```
    
2. Isso garante que o processo seja marcado como concluído e o Worker fique disponível para novos trabalhos.
    

**Resultado Final no Banco:**

|   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|
|ProcessID|ProcessName|Status|CreatedAt|WorkerAssigned|StartTime|EndTime|
|1|ProcessoA|Completed|2024-12-06 10:00:00.000|Worker_1|2024-12-06 10:01:00.000|2024-12-06 10:05:00.000|

---

## **2. Componentes da Solução**

### **2.1. Procedures no Banco de Dados**

#### **[stage].[RegisterProcess]**

Registra novos processos no banco de dados:

```sql
CREATE PROCEDURE [stage].[RegisterProcess]
    @ProcessName NVARCHAR(255)
AS
BEGIN
    IF EXISTS (
        SELECT 1 FROM [stage].[FileControl]
        WHERE ProcessName = @ProcessName AND Status IN ('Pending', 'InProgress')
    )
    BEGIN
        PRINT 'Processo já registrado e pendente ou em execução.';
        RETURN;
    END;

    INSERT INTO [stage].[FileControl] (ProcessName, Status, CreatedAt)
    VALUES (@ProcessName, 'Pending', GETDATE());

    PRINT 'Processo registrado com sucesso.';
END;
```

#### **[stage].[AssignProcessToWorker]**

Atribui processos pendentes a Workers disponíveis:

```sql
CREATE PROCEDURE [stage].[AssignProcessToWorker]
    @NumberOfWorkers INT
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @WorkerIndex INT = 1;
    DECLARE @WorkerName NVARCHAR(50);
    DECLARE @ProcessID INT;

    WHILE @WorkerIndex <= @NumberOfWorkers
    BEGIN
        SET @WorkerName = CONCAT('Worker_', @WorkerIndex);

        IF EXISTS (
            SELECT 1 FROM [stage].[FileControl]
            WHERE WorkerAssigned = @WorkerName AND Status = 'InProgress'
        )
        BEGIN
            SET @WorkerIndex = @WorkerIndex + 1;
            CONTINUE;
        END;

        SELECT TOP 1 @ProcessID = ProcessID
        FROM [stage].[FileControl]
        WHERE Status = 'Pending'
        ORDER BY CreatedAt;

        IF @ProcessID IS NULL BREAK;

        UPDATE [stage].[FileControl]
        SET WorkerAssigned = @WorkerName, Status = 'InProgress', StartTime = GETDATE()
        WHERE ProcessID = @ProcessID;

        SET @WorkerIndex = @WorkerIndex + 1;
    END;
END;
```

#### **[stage].[ProcessWorkerTask]**

Obtém o processo atribuído a um Worker:

```sql
CREATE PROCEDURE [stage].[ProcessWorkerTask]
    @WorkerName NVARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT TOP 1 ProcessID, ProcessName
    FROM [stage].[FileControl]
    WHERE WorkerAssigned = @WorkerName AND Status = 'InProgress';
END;
```

---


## **3. Fluxo no SSIS**

### **3.2. Configuração da Tarefa Executar SQL no SSIS**

1. **Definir o ResultSet**:
    
    - Configure o **ResultSet** como **Linha Única** na tarefa Executar SQL.
        
    - Isso garante que o SSIS possa capturar os resultados da procedure `[stage].[ProcessWorkerTask]`.
        
2. **Mapeamento de Variáveis**:
    
    - Mapeie as colunas retornadas pela procedure (`ProcessID` e `ProcessName`) para variáveis SSIS.
        

**Exemplo:**

|Nome da Variável SSIS|Direção|Nome da Coluna SQL|
|---|---|---|
|User::ProcessID|Output|ProcessID|
|User::ProcessName|Output|ProcessName|

3. **Configuração da Procedure**:
    
    - Certifique-se de que o nome do Worker (ex.: `Worker_1`) é passado como parâmetro para a procedure.
        
    - O valor do parâmetro deve ser associado a uma variável no SSIS (ex.: `User::WorkerName`).
        

**SQL de Exemplo:**

```sql
EXEC [stage].[ProcessWorkerTask] @WorkerName = ?;
```

---

### **3.3. Lógica de Processamento no SSIS**

1. **Fluxo Baseado em Condições:**
    
    - Após capturar as variáveis `ProcessID` e `ProcessName`, insira um **Container de Fluxo Condicional** que verifica se `ProcessID` é nulo.
        
    - Se `ProcessID` for nulo, o Worker não possui tarefas atribuídas e o fluxo deve ser finalizado.
        
2. **Tarefa Data Flow (ETL):**
    
    - Se `ProcessID` não for nulo, utilize um Data Flow Task para processar o trabalho associado ao processo.
        
    - Carregue as informações necessárias com base no `ProcessID`.
        
3. **Atualização do Status:**
    
    - Ao final do processamento, utilize uma tarefa **Executar SQL** para atualizar o status do processo para `'Completed'` e registrar o `EndTime`.
        

**SQL de Atualização:**

```sql
UPDATE [stage].[FileControl]
SET Status = 'Completed', EndTime = GETDATE()
WHERE ProcessID = ?;
```

- A variável `ProcessID` do SSIS deve ser mapeada para o parâmetro da consulta.
    

---

## **4. Monitoramento e Logging**

### **4.1. Estrutura de Logging**

1. **Tabela** `**ProcessLog**`:
    
    - Registra eventos relacionados aos processos.
        

```sql
CREATE TABLE [stage].[ProcessLog] (
    LogID INT IDENTITY PRIMARY KEY,       -- Identificador do log
    ProcessID INT,                        -- ID do processo relacionado
    Message NVARCHAR(500),                -- Descrição do evento
    LogLevel NVARCHAR(50),                -- Nível do log: Info, Warning, Error
    LogTime DATETIME DEFAULT GETDATE()    -- Data e hora do log
);
```

2. **Inserir Logs Personalizados no SSIS**:
    
    - Use scripts no SSIS para registrar informações durante a execução.
        

**Exemplo de Código em Script Task:**

```csharp
Dts.Events.FireInformation(0, "WorkerPackage", "Processamento iniciado para o ProcessID: " + ProcessID, "", 0, ref fireAgain);

string query = "INSERT INTO [stage].[ProcessLog] (ProcessID, Message, LogLevel) VALUES (@ProcessID, @Message, @LogLevel)";
using (SqlConnection conn = new SqlConnection(Dts.Connections["DBConnection"].ConnectionString))
{
    SqlCommand cmd = new SqlCommand(query, conn);
    cmd.Parameters.AddWithValue("@ProcessID", ProcessID);
    cmd.Parameters.AddWithValue("@Message", "Processamento finalizado com sucesso.");
    cmd.Parameters.AddWithValue("@LogLevel", "Info");
    conn.Open();
    cmd.ExecuteNonQuery();
}
```

---

### **4.2. Painéis de Monitoramento**

1. **Crie Painéis no Power BI ou SSRS:**
    
    - Mostre:
        
        - Número de processos pendentes, em execução e concluídos.
            
        - Tempo médio de execução por Worker.
            
        - Processos que excederam o tempo limite.
            
2. **Relatórios no Power BI:**
    
    - Use conexão direta com o SQL Server para atualizar os relatórios em tempo real.
        

---

## **5. Controle de Falhas**

### **5.1. Processos Travados**

- Configure scripts para redefinir processos que permanecem em `'InProgress'` por muito tempo:
    

```sql
UPDATE [stage].[FileControl]
SET Status = 'Pending', WorkerAssigned = NULL, StartTime = NULL
WHERE Status = 'InProgress' AND DATEDIFF(MINUTE, StartTime, GETDATE()) > 60;
```

- Insira logs para rastrear processos redefinidos:
    

```sql
INSERT INTO [stage].[ProcessLog] (ProcessID, Message, LogLevel)
SELECT ProcessID, 'Processo redefinido por timeout', 'Warning'
FROM [stage].[FileControl]
WHERE Status = 'InProgress' AND DATEDIFF(MINUTE, StartTime, GETDATE()) > 60;
```

---

## **6. Testes Automatizados**

### **6.1. Testes de Unidade no Banco**

1. **Teste de Registro de Processos:**
    
    - Execute a procedure `[stage].[RegisterProcess]` com nomes duplicados e diferentes para validar as regras de registro.
        
2. **Teste de Atribuição de Workers:**
    
    - Simule cenários com diferentes números de Workers e processos pendentes.
        
3. **Teste de Timeout:**
    
    - Configure processos manuais como `'InProgress'` e execute o script de timeout para garantir a redefinição.
        

### **6.2. Testes de Integração no SSIS**

1. **Simulação de Fluxo Completo:**
    
    - Registre processos, execute a atribuição de Workers e processe os trabalhos no SSIS.
        
2. **Validação de Logs:**
    
    - Certifique-se de que os logs estejam sendo gerados corretamente em cada etapa.
        

---

## **Resumo**

- **Procedures Bem-Definidas**: Gerenciam registro, atribuição e monitoramento de processos.
    
- **Pacotes SSIS Otimizados**: Integram-se com o banco para processar e atualizar o status dos trabalhos.
    
- **Controle de Falhas**: Scripts de timeout garantem que processos travados sejam redefinidos automaticamente.
    
- **Monitoramento Avançado**: Logs detalhados e dashboards oferecem visibilidade completa da solução.





---


## **1. Fatores que Impactam o Armazenamento**

### **1.1. Volume de Processos**

- O número de **processos registrados** por minuto é crítico. Se o sistema registra e finaliza muitos processos por minuto (por exemplo, centenas ou milhares), o banco pode crescer rapidamente.

### **1.2. Tamanho das Tabelas**

- A tabela `[stage].[FileControl]` acumulará entradas para cada processo gerenciado. Dependendo do tempo de retenção dos dados, essa tabela pode crescer significativamente.
- A tabela `[stage].[ProcessLog]` (caso o logging seja detalhado) pode consumir muito espaço, especialmente com logs frequentes ou mensagens longas.

### **1.3. Retenção de Dados**

- Se você não implementar um mecanismo de **limpeza ou arquivamento** periódico, os dados históricos continuarão acumulando indefinidamente.

---

## **2. Estratégias para Prevenir Problemas de Armazenamento**

### **2.1. Rotação de Dados**

- **Limpeza Periódica de Dados Antigos:**
    
    - Mova processos antigos (ex.: concluídos há mais de 6 meses) para uma tabela de histórico.
    - Use um job agendado para realizar essa limpeza, por exemplo:
```sql 
INSERT INTO [stage].[FileControl_History]
SELECT * FROM [stage].[FileControl]
WHERE Status = 'Completed' AND EndTime < DATEADD(MONTH, -6, GETDATE());

DELETE FROM [stage].[FileControl]
WHERE Status = 'Completed' AND EndTime < DATEADD(MONTH, -6, GETDATE());
```

        
- **Rotação de Logs:**
    
    - Mova logs antigos para tabelas de histórico ou exporte-os para arquivos externos.
```sql 
INSERT INTO [stage].[ProcessLog_History]
SELECT * FROM [stage].[ProcessLog]
WHERE LogTime < DATEADD(MONTH, -6, GETDATE());

DELETE FROM [stage].[ProcessLog]
WHERE LogTime < DATEADD(MONTH, -6, GETDATE());
```   
        

---

### **2.2. Monitoramento de Crescimento**

Implemente um sistema para monitorar o tamanho das tabelas:

- Use a query abaixo para verificar o tamanho das tabelas frequentemente:
```sql 
EXEC sp_spaceused '[stage].[FileControl]';
EXEC sp_spaceused '[stage].[ProcessLog]';
```    
    
- Configure alertas para quando o tamanho ultrapassar um limite seguro.

---

### **2.3. Compressão de Dados**

Se você estiver usando o **SQL Server Enterprise Edition**, habilite a **compressão de dados** nas tabelas grandes:
```sql 
ALTER TABLE [stage].[FileControl]
REBUILD WITH (DATA_COMPRESSION = PAGE);
```

---

### **2.4. Limitar Logging**

Se o volume de logs for muito alto:

- Restrinja o logging apenas aos eventos mais importantes (ex.: erros e informações críticas).
- Use níveis de log (`Info`, `Warning`, `Error`) para filtrar informações irrelevantes.

---

## **3. Estimativa de Crescimento**

### **Exemplo de Cálculo**

Se você processa 100 processos por minuto e cada registro na tabela `[stage].[FileControl]` ocupa 1 KB:

- **Por dia:** `100 processos/min × 60 min × 24 horas × 1 KB = 144 MB/dia`
- **Por mês:** `144 MB × 30 dias = ~4,32 GB/mês`

Se você adicionar logs detalhados, esse valor pode dobrar ou triplicar.

---

## **4. Recomendações**

1. **Implemente Rotação de Dados e Logs:** Mantenha apenas os dados relevantes no banco e mova históricos para tabelas secundárias.
2. **Configure Alertas:** Monitore o tamanho das tabelas e configure notificações para evitar gargalos de armazenamento.
3. **Faça Auditorias Regulares:** Revise o crescimento das tabelas mensalmente para ajustar estratégias de limpeza.
4. **Use um Arquivo de Log Externo (se necessário):** Para logs detalhados, exporte-os para arquivos JSON/CSV em vez de armazená-los diretamente no banco.