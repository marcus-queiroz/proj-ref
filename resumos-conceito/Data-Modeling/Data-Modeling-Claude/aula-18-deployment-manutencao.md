# Aula 18 - Deployment e Manutenção

## Objetivos da Aula
- Compreender estratégias de deployment
- Implementar versionamento de pacotes
- Gerenciar ambientes (DEV, QA, PROD)
- Desenvolver manutenção de jobs
- Implementar backup e recuperação

## 1. Estratégias de Deployment

### 1.1 Gestão de Ambientes
```sql
-- Configuração de ambientes
CREATE TABLE meta.Environment (
    Environment_SK bigint IDENTITY(1,1),
    EnvironmentName varchar(20) NOT NULL,
    Description nvarchar(200) NOT NULL,
    IsActive bit NOT NULL,
    CreatedDate datetime2 NOT NULL,
    CONSTRAINT PK_Environment PRIMARY KEY (Environment_SK)
);

-- Configurações por ambiente
CREATE TABLE meta.EnvironmentConfig (
    Config_SK bigint IDENTITY(1,1),
    Environment_SK bigint NOT NULL,
    ConfigKey varchar(100) NOT NULL,
    ConfigValue nvarchar(500) NOT NULL,
    IsEncrypted bit NOT NULL,
    LastModified datetime2 NOT NULL,
    CONSTRAINT PK_EnvironmentConfig PRIMARY KEY (Config_SK),
    CONSTRAINT FK_EnvironmentConfig_Environment 
        FOREIGN KEY (Environment_SK) 
        REFERENCES meta.Environment(Environment_SK)
);

-- Procedure para obter configuração
CREATE PROCEDURE meta.usp_GetEnvironmentConfig
    @EnvironmentName varchar(20),
    @ConfigKey varchar(100)
AS
BEGIN
    SELECT 
        CASE 
            WHEN IsEncrypted = 1 
            THEN DECRYPTBYKEY(CONVERT(varbinary(max), ConfigValue))
            ELSE ConfigValue 
        END as ConfigValue
    FROM meta.EnvironmentConfig ec
    JOIN meta.Environment e ON ec.Environment_SK = e.Environment_SK
    WHERE e.EnvironmentName = @EnvironmentName
    AND ec.ConfigKey = @ConfigKey;
END;
```

### 1.2 Script de Deployment
```sql
-- Controle de deployment
CREATE TABLE meta.DeploymentControl (
    Deployment_SK bigint IDENTITY(1,1),
    Environment_SK bigint NOT NULL,
    DeploymentVersion varchar(20) NOT NULL,
    DeploymentDate datetime2 NOT NULL,
    DeployedBy nvarchar(100) NOT NULL,
    Status varchar(20) NOT NULL,
    Comments nvarchar(max) NULL,
    CONSTRAINT PK_DeploymentControl PRIMARY KEY (Deployment_SK)
);

-- Procedure de deployment
CREATE PROCEDURE deploy.usp_DeployPackages
    @EnvironmentName varchar(20),
    @Version varchar(20),
    @DeployedBy nvarchar(100)
AS
BEGIN
    DECLARE @Environment_SK bigint;
    DECLARE @Deployment_SK bigint;

    -- Obter Environment_SK
    SELECT @Environment_SK = Environment_SK
    FROM meta.Environment
    WHERE EnvironmentName = @EnvironmentName;

    -- Registrar início do deployment
    INSERT INTO meta.DeploymentControl (
        Environment_SK,
        DeploymentVersion,
        DeploymentDate,
        DeployedBy,
        Status
    )
    VALUES (
        @Environment_SK,
        @Version,
        GETDATE(),
        @DeployedBy,
        'In Progress'
    );

    SET @Deployment_SK = SCOPE_IDENTITY();

    BEGIN TRY
        -- Executar scripts de deployment
        -- [Scripts específicos do ambiente aqui]

        -- Atualizar status
        UPDATE meta.DeploymentControl
        SET Status = 'Success'
        WHERE Deployment_SK = @Deployment_SK;
    END TRY
    BEGIN CATCH
        -- Registrar erro e reverter
        UPDATE meta.DeploymentControl
        SET 
            Status = 'Failed',
            Comments = ERROR_MESSAGE()
        WHERE Deployment_SK = @Deployment_SK;

        THROW;
    END CATCH;
END;
```

## 2. Versionamento de Pacotes

### 2.1 Controle de Versão
```sql
-- Versão de pacotes
CREATE TABLE meta.PackageVersion (
    PackageVersion_SK bigint IDENTITY(1,1),
    PackageName nvarchar(100) NOT NULL,
    VersionNumber varchar(20) NOT NULL,
    CreationDate datetime2 NOT NULL,
    CreatedBy nvarchar(100) NOT NULL,
    Description nvarchar(max) NULL,
    SourceCode nvarchar(max) NOT NULL,
    CONSTRAINT PK_PackageVersion PRIMARY KEY (PackageVersion_SK)
);

-- Histórico de alterações
CREATE TABLE meta.ChangeHistory (
    Change_SK bigint IDENTITY(1,1),
    PackageVersion_SK bigint NOT NULL,
    ChangeType varchar(20) NOT NULL,
    ChangeDescription nvarchar(max) NOT NULL,
    ChangedBy nvarchar(100) NOT NULL,
    ChangeDate datetime2 NOT NULL,
    CONSTRAINT PK_ChangeHistory PRIMARY KEY (Change_SK),
    CONSTRAINT FK_ChangeHistory_PackageVersion 
        FOREIGN KEY (PackageVersion_SK) 
        REFERENCES meta.PackageVersion(PackageVersion_SK)
);
```

### 2.2 Deployment por Versão
```sql
-- Procedure para deployment de versão específica
CREATE PROCEDURE deploy.usp_DeployVersion
    @PackageName nvarchar(100),
    @VersionNumber varchar(20),
    @EnvironmentName varchar(20)
AS
BEGIN
    DECLARE @SourceCode nvarchar(max);
    
    -- Obter código fonte da versão
    SELECT @SourceCode = SourceCode
    FROM meta.PackageVersion
    WHERE PackageName = @PackageName
    AND VersionNumber = @VersionNumber;

    -- Executar deployment
    BEGIN TRY
        -- Preparar ambiente
        EXEC deploy.usp_PrepareEnvironment @EnvironmentName;

        -- Executar código
        EXEC sp_executesql @SourceCode;

        -- Registrar sucesso
        INSERT INTO meta.DeploymentControl (
            Environment_SK,
            DeploymentVersion,
            DeploymentDate,
            DeployedBy,
            Status
        )
        VALUES (
            (SELECT Environment_SK 
             FROM meta.Environment 
             WHERE EnvironmentName = @EnvironmentName),
            @VersionNumber,
            GETDATE(),
            SYSTEM_USER,
            'Success'
        );
    END TRY
    BEGIN CATCH
        -- Registrar falha
        INSERT INTO meta.DeploymentControl (
            Environment_SK,
            DeploymentVersion,
            DeploymentDate,
            DeployedBy,
            Status,
            Comments
        )
        VALUES (
            (SELECT Environment_SK 
             FROM meta.Environment 
             WHERE EnvironmentName = @EnvironmentName),
            @VersionNumber,
            GETDATE(),
            SYSTEM_USER,
            'Failed',
            ERROR_MESSAGE()
        );

        THROW;
    END CATCH;
END;
```

## 3. Manutenção de Jobs

### 3.1 Gestão de Jobs
```sql
-- Configuração de jobs
CREATE TABLE meta.JobConfig (
    Job_SK bigint IDENTITY(1,1),
    JobName nvarchar(100) NOT NULL,
    JobDescription nvarchar(max) NULL,
    ScheduleType varchar(20) NOT NULL,
    ScheduleDefinition varchar(100) NOT NULL,
    IsEnabled bit NOT NULL,
    LastRunDate datetime2 NULL,
    NextRunDate datetime2 NULL,
    CONSTRAINT PK_JobConfig PRIMARY KEY (Job_SK)
);

-- Dependências de jobs
CREATE TABLE meta.JobDependency (
    JobDependency_SK bigint IDENTITY(1,1),
    Job_SK bigint NOT NULL,
    DependentJob_SK bigint NOT NULL,
    IsRequired bit NOT NULL,
    CONSTRAINT PK_JobDependency PRIMARY KEY (JobDependency_SK),
    CONSTRAINT FK_JobDependency_Job 
        FOREIGN KEY (Job_SK) 
        REFERENCES meta.JobConfig(Job_SK),
    CONSTRAINT FK_JobDependency_Dependent 
        FOREIGN KEY (DependentJob_SK) 
        REFERENCES meta.JobConfig(Job_SK)
);
```

### 3.2 Manutenção Automática
```sql
-- Procedure de manutenção
CREATE PROCEDURE maintenance.usp_PerformMaintenance
    @JobName nvarchar(100)
AS
BEGIN
    -- Limpar histórico antigo
    DELETE FROM meta.ProcessLog
    WHERE EndTime < DATEADD(MONTH, -3, GETDATE());

    -- Atualizar estatísticas
    UPDATE STATISTICS dbo.FactTable;

    -- Rebuild/Reorganize índices
    DECLARE @SQL nvarchar(max);
    SELECT @SQL = 
        CASE 
            WHEN avg_fragmentation_in_percent > 30 
            THEN 'ALTER INDEX ' + name + ' ON ' + object_name(object_id) + ' REBUILD'
            WHEN avg_fragmentation_in_percent > 10 
            THEN 'ALTER INDEX ' + name + ' ON ' + object_name(object_id) + ' REORGANIZE'
            ELSE NULL
        END
    FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, NULL) s
    JOIN sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
    WHERE avg_fragmentation_in_percent > 10;

    IF @SQL IS NOT NULL
        EXEC sp_executesql @SQL;

    -- Atualizar job status
    UPDATE meta.JobConfig
    SET 
        LastRunDate = GETDATE(),
        NextRunDate = DATEADD(DAY, 1, GETDATE())
    WHERE JobName = @JobName;
END;
```

## 4. Backup e Recuperação

### 4.1 Estratégia de Backup
```sql
-- Configuração de backup
CREATE TABLE meta.BackupConfig (
    Backup_SK bigint IDENTITY(1,1),
    BackupType varchar(20) NOT NULL,
    BackupPath varchar(500) NOT NULL,
    RetentionDays int NOT NULL,
    IsEnabled bit NOT NULL,
    LastBackupDate datetime2 NULL,
    CONSTRAINT PK_BackupConfig PRIMARY KEY (Backup_SK)
);

-- Procedure de backup
CREATE PROCEDURE maintenance.usp_PerformBackup
    @BackupType varchar(20)
AS
BEGIN
    DECLARE 
        @BackupPath varchar(500),
        @BackupFile varchar(500),
        @SQL nvarchar(max);

    -- Obter configuração
    SELECT @BackupPath = BackupPath
    FROM meta.BackupConfig
    WHERE BackupType = @BackupType
    AND IsEnabled = 1;

    -- Gerar nome do arquivo
    SET @BackupFile = @BackupPath + '\Backup_' + 
        CONVERT(varchar(8), GETDATE(), 112) + '_' +
        REPLACE(CONVERT(varchar(8), GETDATE(), 108), ':', '') + 
        '.bak';

    -- Executar backup
    SET @SQL = 'BACKUP DATABASE [YourDatabase] TO DISK = ''' + 
        @BackupFile + ''' WITH COMPRESSION, CHECKSUM';

    EXEC sp_executesql @SQL;

    -- Atualizar registro
    UPDATE meta.BackupConfig
    SET LastBackupDate = GETDATE()
    WHERE BackupType = @BackupType;
END;
```

### 4.2 Processo de Recovery
```sql
-- Log de recovery
CREATE TABLE meta.RecoveryLog (
    Recovery_SK bigint IDENTITY(1,1),
    RecoveryDate datetime2 NOT NULL,
    BackupFile varchar(500) NOT NULL,
    RecoveryType varchar(20) NOT NULL,
    Status varchar(20) NOT NULL,
    ErrorMessage nvarchar(max) NULL,
    CONSTRAINT PK_RecoveryLog PRIMARY KEY (Recovery_SK)
);

-- Procedure de recovery
CREATE PROCEDURE maintenance.usp_PerformRecovery
    @BackupFile varchar(500)
AS
BEGIN
    DECLARE @SQL nvarchar(max);

    BEGIN TRY
        -- Verificar backup
        RESTORE VERIFYONLY FROM DISK = @BackupFile;

        -- Executar recovery
        SET @SQL = '
        ALTER DATABASE [YourDatabase] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
        RESTORE DATABASE [YourDatabase] 
        FROM DISK = ''' + @BackupFile + '''
        WITH REPLACE;
        ALTER DATABASE [YourDatabase] SET MULTI_USER;';

        EXEC sp_executesql @SQL;

        -- Registrar sucesso
        INSERT INTO meta.RecoveryLog (
            RecoveryDate,
            BackupFile,
            RecoveryType,
            Status
        )
        VALUES (
            GETDATE(),
            @BackupFile,
            'Full',
            'Success'
        );
    END TRY
    BEGIN CATCH
        -- Registrar erro
        INSERT INTO meta.RecoveryLog (
            RecoveryDate,
            BackupFile,
            RecoveryType,
            Status,
            ErrorMessage
        )
        VALUES (
            GETDATE(),
            @BackupFile,
            'Full',
            'Failed',
            ERROR_MESSAGE()
        );

        THROW;
    END CATCH;
END;
```

## Exercícios Práticos

### Exercício 1: Deployment
1. Configurar ambientes
2. Implementar processo de deployment
3. Testar rollback

### Exercício 2: Versionamento
1. Implementar controle de versão
2. Criar processo de atualização
3. Gerenciar dependências

### Exercício 3: Manutenção
1. Configurar jobs de manutenção
2. Implementar backup
3. Testar recovery

## Material Complementar

### Documentação
- Deployment Best Practices
- Version Control Guidelines
- Maintenance Procedures

### Scripts Úteis
- Scripts de deployment
- Jobs de manutenção
- Procedures de backup

## Próxima Aula
Na próxima aula, abordaremos Indexação para DW, incluindo tipos de índices e estratégias de otimização.

## Avaliação
1. Quiz sobre deployment e manutenção
2. Projeto prático de implementação
3. Plano de manutenção documentado