# Aula 14 - Control Flow e Execução

## Objetivos da Aula
- Compreender os diferentes tipos de containers
- Implementar precedências complexas
- Trabalhar com For Each Loop
- Utilizar Execute SQL Task
- Desenvolver File System Task

## 1. Containers

### 1.1 Sequence Container
```sql
-- Exemplo de variáveis para sequence
DECLARE @SequenceStart datetime2 = GETDATE();
DECLARE @SequenceEnd datetime2;

-- Logging início sequência
INSERT INTO meta.ProcessLog (
    ContainerName,
    StartTime,
    Status
)
VALUES (
    'MainSequence',
    @SequenceStart,
    'Started'
);

-- Atualizar fim sequência
UPDATE meta.ProcessLog
SET 
    EndTime = @SequenceEnd,
    Status = 'Completed'
WHERE 
    ContainerName = 'MainSequence'
    AND StartTime = @SequenceStart;
```

### 1.2 For Loop Container
```sql
-- Configuração de loop
DECLARE @Counter int = 1;
DECLARE @MaxIterations int = 10;

WHILE @Counter <= @MaxIterations
BEGIN
    -- Lógica de processamento
    EXEC etl.usp_ProcessBatch 
        @BatchNumber = @Counter,
        @BatchSize = 1000;
        
    -- Incrementar contador
    SET @Counter += 1;
END;

-- Exemplo de metadata para controle
CREATE TABLE meta.BatchControl (
    BatchID int IDENTITY(1,1),
    BatchNumber int NOT NULL,
    StartTime datetime2 NOT NULL,
    EndTime datetime2 NULL,
    RowsProcessed int NULL,
    Status varchar(20) NOT NULL,
    CONSTRAINT PK_BatchControl PRIMARY KEY (BatchID)
);
```

### 1.3 Foreach Loop Container
```sql
-- Preparação para foreach em arquivos
CREATE TABLE meta.FileControl (
    FileID int IDENTITY(1,1),
    FileName varchar(100) NOT NULL,
    FilePath varchar(500) NOT NULL,
    ProcessDate datetime2 NOT NULL,
    Status varchar(20) NOT NULL,
    CONSTRAINT PK_FileControl PRIMARY KEY (FileID)
);

-- Procedure para processar arquivo
CREATE PROCEDURE etl.usp_ProcessFile
    @FilePath varchar(500),
    @FileName varchar(100)
AS
BEGIN
    -- Registrar início
    INSERT INTO meta.FileControl (
        FileName,
        FilePath,
        ProcessDate,
        Status
    )
    VALUES (
        @FileName,
        @FilePath,
        GETDATE(),
        'Processing'
    );

    -- Processar arquivo
    -- [Lógica de processamento aqui]

    -- Atualizar status
    UPDATE meta.FileControl
    SET Status = 'Completed'
    WHERE FileName = @FileName
    AND ProcessDate = CAST(GETDATE() AS date);
END;
```

## 2. Precedence Constraints

### 2.1 Tipos de Precedência
```sql
-- Configuração de constraints
-- Success Constraint
@[User::TaskStatus] == "Success"

-- Completion Constraint
@[User::TaskStatus] != "Running"

-- Failure Constraint
@[User::TaskStatus] == "Failed"

-- Expression Constraint
@[User::RowCount] > 0 && @[User::ErrorCount] == 0
```

### 2.2 Expressões Complexas
```sql
-- Expressões com múltiplas condições
(@[User::TaskStatus] == "Success" || @[User::TaskStatus] == "Warning") && 
@[User::ProcessDate] == GETDATE() && 
@[User::RowCount] > 0

-- Validações de arquivo
EXISTS(@[User::FilePath]) && 
RIGHT(@[User::FileName], 4) == ".csv" && 
@[User::FileSize] > 0

-- Controle de horário
DATEPART("hh", GETDATE()) >= 20 || 
DATEPART("hh", GETDATE()) <= 4
```

## 3. Execute SQL Task

### 3.1 Tipos de SQLs
```sql
-- DDL Operations
CREATE TABLE #TempProcess (
    ID int IDENTITY(1,1),
    ProcessName varchar(100),
    StartTime datetime2,
    Status varchar(20)
);

-- DML Operations
INSERT INTO #TempProcess (ProcessName, StartTime, Status)
SELECT 
    ?, 
    GETDATE(), 
    'Started';

-- Stored Procedures
EXEC maintenance.usp_PrepareEnvironment
    @ProcessDate = ?,
    @ProcessType = ?;
```

### 3.2 Parâmetros e Variáveis
```sql
-- Mapeamento de parâmetros
DECLARE @ProcessName nvarchar(100) = ?;
DECLARE @StartDate datetime2 = ?;
DECLARE @EndDate datetime2 = ?;

-- Retorno de resultados
SELECT @RowCount = COUNT(*)
FROM staging.ProcessControl
WHERE ProcessName = @ProcessName
AND ProcessDate BETWEEN @StartDate AND @EndDate;

-- Múltiplos resultsets
SELECT @TaskCount = COUNT(*)
FROM staging.TaskControl;

SELECT @ErrorCount = COUNT(*)
FROM staging.ErrorLog;
```

## 4. File System Task

### 4.1 Operações de Arquivo
```csharp
// Exemplo de operações em C#
public void Main()
{
    string sourceFile = Dts.Variables["SourceFile"].Value.ToString();
    string destFile = Dts.Variables["DestFile"].Value.ToString();
    
    try
    {
        // Copiar arquivo
        if (File.Exists(sourceFile))
        {
            File.Copy(sourceFile, destFile, true);
            Dts.TaskResult = (int)ScriptResults.Success;
        }
        else
        {
            // Log erro
            string errorMessage = $"File not found: {sourceFile}";
            Dts.Events.FireError(0, "File Operation", errorMessage, "", 0);
            Dts.TaskResult = (int)ScriptResults.Failure;
        }
    }
    catch (Exception ex)
    {
        Dts.Events.FireError(0, "File Operation", ex.Message, "", 0);
        Dts.TaskResult = (int)ScriptResults.Failure;
    }
}
```

### 4.2 Estrutura de Diretórios
```sql
-- Tabela de controle de diretórios
CREATE TABLE meta.DirectoryControl (
    DirectoryID int IDENTITY(1,1),
    DirectoryPath varchar(500) NOT NULL,
    DirectoryType varchar(50) NOT NULL,
    Created datetime2 NOT NULL,
    LastAccessed datetime2 NULL,
    Status varchar(20) NOT NULL,
    CONSTRAINT PK_DirectoryControl PRIMARY KEY (DirectoryID)
);

-- Procedure para manutenção
CREATE PROCEDURE maintenance.usp_ManageDirectories
    @RootPath varchar(500),
    @RetentionDays int
AS
BEGIN
    -- Criar estrutura
    DECLARE @Directories TABLE (
        DirectoryPath varchar(500),
        DirectoryType varchar(50)
    );

    INSERT INTO @Directories VALUES
    (@RootPath + '\Input', 'Source'),
    (@RootPath + '\Process', 'Working'),
    (@RootPath + '\Archive', 'Archive'),
    (@RootPath + '\Error', 'Error');

    -- Criar diretórios
    -- [Lógica de criação aqui]

    -- Limpar arquivos antigos
    -- [Lógica de limpeza aqui]
END;
```

## 5. Implementação de Fluxo Complexo

### 5.1 Orquestração de Processos
```sql
-- Tabela de controle de processos
CREATE TABLE meta.ProcessControl (
    ProcessID int IDENTITY(1,1),
    ProcessName varchar(100) NOT NULL,
    ParentProcessID int NULL,
    StartTime datetime2 NOT NULL,
    EndTime datetime2 NULL,
    Status varchar(20) NOT NULL,
    ErrorMessage nvarchar(max) NULL,
    CONSTRAINT PK_ProcessControl PRIMARY KEY (ProcessID),
    CONSTRAINT FK_ProcessControl_Parent 
        FOREIGN KEY (ParentProcessID) 
        REFERENCES meta.ProcessControl(ProcessID)
);

-- Procedure para gestão de processo
CREATE PROCEDURE meta.usp_ManageProcess
    @ProcessName varchar(100),
    @ParentProcessID int = NULL
AS
BEGIN
    DECLARE @ProcessID int;

    -- Iniciar processo
    INSERT INTO meta.ProcessControl (
        ProcessName,
        ParentProcessID,
        StartTime,
        Status
    )
    VALUES (
        @ProcessName,
        @ParentProcessID,
        GETDATE(),
        'Started'
    );

    SET @ProcessID = SCOPE_IDENTITY();

    -- Retornar ID
    SELECT @ProcessID as ProcessID;
END;
```

## Exercícios Práticos

### Exercício 1: Containers
1. Implementar sequence container
2. Criar loop container
3. Desenvolver foreach container
4. Configurar logging

### Exercício 2: Precedências
1. Implementar diferentes tipos
2. Criar expressões complexas
3. Testar cenários de erro

### Exercício 3: Tasks
1. Desenvolver Execute SQL Task
2. Implementar File System Task
3. Criar processo completo

## Material Complementar

### Documentação
- Control Flow Best Practices
- Task Configuration Guide
- Error Handling Patterns

### Scripts Úteis
- Templates de controle
- Scripts de manutenção
- Procedures de logging

## Próxima Aula
Na próxima aula, abordaremos Carga Incremental, incluindo estratégias de detecção de mudanças e CDC.

## Avaliação
1. Quiz sobre control flow
2. Implementação prática
3. Análise de cenários