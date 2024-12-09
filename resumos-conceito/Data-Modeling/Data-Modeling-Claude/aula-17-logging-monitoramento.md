# Aula 17 - Logging e Monitoramento

## Objetivos da Aula
- Implementar estratégias de logging
- Criar tabelas de controle eficientes
- Desenvolver monitoramento de execução
- Trabalhar com tratamento de erros
- Implementar sistema de notificações

## 1. Estratégias de Logging

### 1.1 Estrutura de Logging
```sql
-- Tabela principal de log
CREATE TABLE meta.ProcessLog (
    Log_SK bigint IDENTITY(1,1),
    PackageName nvarchar(100) NOT NULL,
    ProcessName nvarchar(100) NOT NULL,
    StartTime datetime2 NOT NULL,
    EndTime datetime2 NULL,
    Duration AS DATEDIFF(SECOND, StartTime, EndTime),
    Status varchar(20) NOT NULL,
    RowsProcessed int NULL,
    ErrorCode int NULL,
    ErrorMessage nvarchar(max) NULL,
    CONSTRAINT PK_ProcessLog PRIMARY KEY (Log_SK)
);

-- Log detalhado
CREATE TABLE meta.ProcessLogDetail (
    LogDetail_SK bigint IDENTITY(1,1),
    Log_SK bigint NOT NULL,
    StepName nvarchar(100) NOT NULL,
    StepStartTime datetime2 NOT NULL,
    StepEndTime datetime2 NULL,
    StepDuration AS DATEDIFF(SECOND, StepStartTime, StepEndTime),
    StepStatus varchar(20) NOT NULL,
    StepRowsProcessed int NULL,
    StepMessage nvarchar(max) NULL,
    CONSTRAINT PK_ProcessLogDetail PRIMARY KEY (LogDetail_SK),
    CONSTRAINT FK_ProcessLogDetail_ProcessLog 
        FOREIGN KEY (Log_SK) 
        REFERENCES meta.ProcessLog(Log_SK)
);
```

### 1.2 Procedures de Logging
```sql
-- Iniciar processo
CREATE PROCEDURE meta.usp_StartProcessLog
    @PackageName nvarchar(100),
    @ProcessName nvarchar(100)
AS
BEGIN
    DECLARE @Log_SK bigint;

    INSERT INTO meta.ProcessLog (
        PackageName,
        ProcessName,
        StartTime,
        Status
    )
    VALUES (
        @PackageName,
        @ProcessName,
        GETDATE(),
        'Running'
    );

    SET @Log_SK = SCOPE_IDENTITY();
    RETURN @Log_SK;
END;

-- Registrar passo
CREATE PROCEDURE meta.usp_LogProcessStep
    @Log_SK bigint,
    @StepName nvarchar(100),
    @RowsProcessed int = NULL
AS
BEGIN
    INSERT INTO meta.ProcessLogDetail (
        Log_SK,
        StepName,
        StepStartTime,
        StepStatus,
        StepRowsProcessed
    )
    VALUES (
        @Log_SK,
        @StepName,
        GETDATE(),
        'Running',
        @RowsProcessed
    );

    RETURN SCOPE_IDENTITY();
END;

-- Finalizar processo
CREATE PROCEDURE meta.usp_EndProcessLog
    @Log_SK bigint,
    @Status varchar(20),
    @ErrorCode int = NULL,
    @ErrorMessage nvarchar(max) = NULL
AS
BEGIN
    UPDATE meta.ProcessLog
    SET 
        EndTime = GETDATE(),
        Status = @Status,
        ErrorCode = @ErrorCode,
        ErrorMessage = @ErrorMessage
    WHERE Log_SK = @Log_SK;
END;
```

## 2. Tabelas de Controle

### 2.1 Controle de Execução
```sql
-- Controle de pacotes
CREATE TABLE meta.PackageControl (
    Package_SK bigint IDENTITY(1,1),
    PackageName nvarchar(100) NOT NULL,
    LastExecutionDate datetime2 NULL,
    LastStatus varchar(20) NULL,
    IsEnabled bit NOT NULL,
    ExecutionOrder int NOT NULL,
    MaxRetries int NOT NULL,
    RetryInterval int NOT NULL,
    CONSTRAINT PK_PackageControl PRIMARY KEY (Package_SK)
);

-- Controle de dependências
CREATE TABLE meta.PackageDependency (
    PackageDependency_SK bigint IDENTITY(1,1),
    Package_SK bigint NOT NULL,
    DependentPackage_SK bigint NOT NULL,
    DependencyType varchar(20) NOT NULL,
    IsRequired bit NOT NULL,
    CONSTRAINT PK_PackageDependency PRIMARY KEY (PackageDependency_SK),
    CONSTRAINT FK_PackageDependency_Package 
        FOREIGN KEY (Package_SK) 
        REFERENCES meta.PackageControl(Package_SK),
    CONSTRAINT FK_PackageDependency_Dependent 
        FOREIGN KEY (DependentPackage_SK) 
        REFERENCES meta.PackageControl(Package_SK)
);
```

### 2.2 Controle de Dados
```sql
-- Controle de carga
CREATE TABLE meta.LoadControl (
    LoadControl_SK bigint IDENTITY(1,1),
    TableName varchar(100) NOT NULL,
    LastLoadDate datetime2 NOT NULL,
    LastLoadRows int NOT NULL,
    LastLoadStatus varchar(20) NOT NULL,
    NextLoadDate datetime2 NULL,
    LoadFrequency varchar(20) NOT NULL,
    CONSTRAINT PK_LoadControl PRIMARY KEY (LoadControl_SK)
);

-- Controle de versão
CREATE TABLE meta.VersionControl (
    Version_SK bigint IDENTITY(1,1),
    TableName varchar(100) NOT NULL,
    VersionNumber varchar(20) NOT NULL,
    ChangeDescription nvarchar(500) NOT NULL,
    AppliedDate datetime2 NOT NULL,
    AppliedBy nvarchar(100) NOT NULL,
    ScriptName varchar(100) NOT NULL,
    CONSTRAINT PK_VersionControl PRIMARY KEY (Version_SK)
);
```

## 3. Monitoramento de Execução

### 3.1 Views de Monitoramento
```sql
-- Status atual
CREATE VIEW monitor.CurrentStatus
AS
SELECT 
    p.PackageName,
    p.ProcessName,
    p.StartTime,
    DATEDIFF(MINUTE, p.StartTime, GETDATE()) as RunningMinutes,
    p.Status,
    d.StepName as CurrentStep,
    d.StepStartTime,
    d.StepRowsProcessed
FROM meta.ProcessLog p
INNER JOIN meta.ProcessLogDetail d 
    ON p.Log_SK = d.Log_SK
WHERE p.Status = 'Running'
AND d.StepEndTime IS NULL;

-- Histórico de execução
CREATE VIEW monitor.ExecutionHistory
AS
WITH StepSummary AS (
    SELECT 
        Log_SK,
        COUNT(*) as TotalSteps,
        SUM(StepRowsProcessed) as TotalRowsProcessed,
        MAX(CASE WHEN StepStatus = 'Failed' THEN 1 ELSE 0 END) as HasFailedSteps
    FROM meta.ProcessLogDetail
    GROUP BY Log_SK
)
SELECT 
    p.PackageName,
    p.ProcessName,
    p.StartTime,
    p.EndTime,
    p.Duration,
    p.Status,
    s.TotalSteps,
    s.TotalRowsProcessed,
    s.HasFailedSteps
FROM meta.ProcessLog p
LEFT JOIN StepSummary s ON p.Log_SK = s.Log_SK;
```

### 3.2 Alertas e Notificações
```sql
-- Configuração de alertas
CREATE TABLE meta.AlertConfig (
    Alert_SK bigint IDENTITY(1,1),
    AlertName varchar(100) NOT NULL,
    AlertType varchar(50) NOT NULL,
    AlertCondition nvarchar(max) NOT NULL,
    AlertMessage nvarchar(max) NOT NULL,
    EmailRecipients varchar(max) NULL,
    IsEnabled bit NOT NULL,
    CONSTRAINT PK_AlertConfig PRIMARY KEY (Alert_SK)
);

-- Procedure de verificação
CREATE PROCEDURE monitor.usp_CheckAlerts
AS
BEGIN
    DECLARE @AlertName varchar(100);
    DECLARE @AlertCondition nvarchar(max);
    DECLARE @AlertMessage nvarchar(max);
    DECLARE @EmailRecipients varchar(max);
    DECLARE @SQL nvarchar(max);
    DECLARE @ShouldAlert bit;

    -- Cursor para alertas
    DECLARE alert_cursor CURSOR FOR
    SELECT 
        AlertName,
        AlertCondition,
        AlertMessage,
        EmailRecipients
    FROM meta.AlertConfig
    WHERE IsEnabled = 1;

    OPEN alert_cursor;
    FETCH NEXT FROM alert_cursor 
    INTO @AlertName, @AlertCondition, @AlertMessage, @EmailRecipients;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Verificar condição
        SET @SQL = N'
        IF EXISTS(' + @AlertCondition + ')
            SET @ShouldAlert = 1
        ELSE
            SET @ShouldAlert = 0';

        EXEC sp_executesql @SQL, 
            N'@ShouldAlert bit OUTPUT', 
            @ShouldAlert OUTPUT;

        -- Enviar alerta se necessário
        IF @ShouldAlert = 1
        BEGIN
            EXEC msdb.dbo.sp_send_dbmail
                @profile_name = 'ETL Profile',
                @recipients = @EmailRecipients,
                @subject = @AlertName,
                @body = @AlertMessage;
        END;

        FETCH NEXT FROM alert_cursor 
        INTO @AlertName, @AlertCondition, @AlertMessage, @EmailRecipients;
    END;

    CLOSE alert_cursor;
    DEALLOCATE alert_cursor;
END;
```

## 4. Tratamento de Erros

### 4.1 Registro de Erros
```sql
-- Log de erros detalhado
CREATE TABLE meta.ErrorLog (
    Error_SK bigint IDENTITY(1,1),
    ErrorDate datetime2 NOT NULL,
    PackageName nvarchar(100) NOT NULL,
    ProcessName nvarchar(100) NOT NULL,
    StepName nvarchar(100) NOT NULL,
    ErrorNumber int NULL,
    ErrorSeverity int NULL,
    ErrorState int NULL,
    ErrorProcedure nvarchar(128) NULL,
    ErrorLine int NULL,
    ErrorMessage nvarchar(max) NOT NULL,
    StackTrace nvarchar(max) NULL,
    CONSTRAINT PK_ErrorLog PRIMARY KEY (Error_SK)
);

-- Procedure para logging de erros
CREATE PROCEDURE meta.usp_LogError
    @PackageName nvarchar(100),
    @ProcessName nvarchar(100),
    @StepName nvarchar(100)
AS
BEGIN
    DECLARE 
        @ErrorNumber int,
        @ErrorSeverity int,
        @ErrorState int,
        @ErrorProcedure nvarchar(128),
        @ErrorLine int,
        @ErrorMessage nvarchar(max);

    -- Capturar detalhes do erro
    SELECT 
        @ErrorNumber = ERROR_NUMBER(),
        @ErrorSeverity = ERROR_SEVERITY(),
        @ErrorState = ERROR_STATE(),
        @ErrorProcedure = ERROR_PROCEDURE(),
        @ErrorLine = ERROR_LINE(),
        @ErrorMessage = ERROR_MESSAGE();

    -- Registrar erro
    INSERT INTO meta.ErrorLog (
        ErrorDate,
        PackageName,
        ProcessName,
        StepName,
        ErrorNumber,
        ErrorSeverity,
        ErrorState,
        ErrorProcedure,
        ErrorLine,
        ErrorMessage,
        StackTrace
    )
    VALUES (
        GETDATE(),
        @PackageName,
        @ProcessName,
        @StepName,
        @ErrorNumber,
        @ErrorSeverity,
        @ErrorState,
        @ErrorProcedure,
        @ErrorLine,
        @ErrorMessage,
        NULL  -- Stack trace pode ser adicionado se disponível
    );

    -- Retornar ID do erro
    RETURN SCOPE_IDENTITY();
END;
```

### 4.2 Retry Logic
```sql
-- Configuração de retry
CREATE PROCEDURE etl.usp_ExecuteWithRetry
    @PackageName nvarchar(100),
    @MaxRetries int = 3,
    @RetryDelay int = 60  -- segundos
AS
BEGIN
    DECLARE @RetryCount int = 0;
    DECLARE @Success bit = 0;

    WHILE @RetryCount < @MaxRetries AND @Success = 0
    BEGIN
        BEGIN TRY
            -- Executar package
            -- [Lógica de execução aqui]
            SET @Success = 1;
        END TRY
        BEGIN CATCH
            SET @RetryCount = @RetryCount + 1;
            
            -- Registrar erro
            EXEC meta.usp_LogError 
                @PackageName,
                'Package Execution',
                'Retry Attempt ' + CAST(@RetryCount as varchar(10));

            -- Aguardar antes de retry
            IF @RetryCount < @MaxRetries
                WAITFOR DELAY @RetryDelay;
        END CATCH
    END;

    -- Se todas as tentativas falharam
    IF @Success = 0
        THROW 50000, 'Maximum retry attempts reached', 1;
END;
```

## Exercícios Práticos

### Exercício 1: Implementação de Logging
1. Criar estrutura completa de logging
2. Implementar procedures de controle
3. Desenvolver views de monitoramento

### Exercício 2: Sistema de Alertas
1. Configurar alertas
2. Implementar verificações
3. Criar notificações

### Exercício 3: Tratamento de Erros
1. Implementar logging de erros
2. Desenvolver retry logic
3. Criar recuperação automática

## Material Complementar

### Documentação
- Best Practices for Logging
- Monitoring Strategies
- Error Handling Patterns

### Scripts Úteis
- Queries de monitoramento
- Scripts de manutenção
- Procedures de diagnóstico

## Próxima Aula
Na próxima aula, abordaremos Deployment e Manutenção, incluindo estratégias de deployment, versionamento e manutenção.

## Avaliação
1. Quiz sobre logging e monitoramento
2. Implementação prática
3. Análise de cenários de erro