# Aula 4 - SQL Server na Prática

## Objetivos da Aula
- Configurar adequadamente o ambiente SQL Server
- Compreender a estrutura de databases e schemas
- Aprender scripts básicos de DDL e DML
- Implementar práticas de versionamento
- Dominar rotinas de backup e restore

## 1. Configuração do Ambiente

### 1.1 Instalação do SQL Server
- Requisitos de sistema
- Opções de instalação
- Componentes principais:
  - Database Engine
  - Integration Services
  - Analysis Services
  - Reporting Services

### 1.2 Configuração de Instância
```sql
-- Verificar instâncias instaladas
SELECT @@SERVERNAME AS ServerName,
       @@VERSION AS SQLServerVersion;

-- Configurações básicas
sp_configure 'show advanced options', 1;
RECONFIGURE;

-- Configurar memória
sp_configure 'max server memory (MB)', 8192;
sp_configure 'min server memory (MB)', 2048;
RECONFIGURE;
```

### 1.3 Configuração de Segurança
```sql
-- Criar login
CREATE LOGIN [DWUser] WITH PASSWORD = 'StrongPassword123';

-- Criar usuário no database
USE [DWDatabase]
CREATE USER [DWUser] FOR LOGIN [DWUser];

-- Atribuir permissões
GRANT SELECT, INSERT, UPDATE, DELETE 
ON SCHEMA::dbo 
TO [DWUser];
```

## 2. Estrutura de Databases e Schemas

### 2.1 Criação de Database
```sql
-- Criar database para Data Warehouse
CREATE DATABASE DWDatabase
ON PRIMARY
( 
    NAME = 'DWDatabase',
    FILENAME = 'C:\SQLData\DWDatabase.mdf',
    SIZE = 10GB,
    MAXSIZE = UNLIMITED,
    FILEGROWTH = 1GB
)
LOG ON
(
    NAME = 'DWDatabase_log',
    FILENAME = 'C:\SQLData\DWDatabase_log.ldf',
    SIZE = 5GB,
    MAXSIZE = UNLIMITED,
    FILEGROWTH = 1GB
);
```

### 2.2 Estrutura de Schemas
```sql
-- Criar schemas para organização
CREATE SCHEMA stg;  -- staging
CREATE SCHEMA dim;  -- dimensões
CREATE SCHEMA fact; -- fatos
CREATE SCHEMA meta; -- metadados
GO

-- Exemplo de uso
CREATE TABLE stg.Customer (
    CustomerID int,
    CustomerName nvarchar(100)
);

CREATE TABLE dim.Customer (
    Customer_SK bigint IDENTITY(1,1),
    CustomerID int,
    CustomerName nvarchar(100)
);
```

## 3. Scripts Básicos (DDL e DML)

### 3.1 Data Definition Language (DDL)
```sql
-- Criar tabela
CREATE TABLE dim.Date (
    Date_SK int NOT NULL,
    FullDate date NOT NULL,
    DayOfWeek tinyint NOT NULL,
    DayName varchar(10) NOT NULL,
    Month tinyint NOT NULL,
    MonthName varchar(10) NOT NULL,
    Quarter tinyint NOT NULL,
    Year smallint NOT NULL,
    CONSTRAINT PK_Date PRIMARY KEY (Date_SK)
);

-- Alterar tabela
ALTER TABLE dim.Date
ADD IsWeekend bit NOT NULL DEFAULT(0);

-- Criar índice
CREATE NONCLUSTERED INDEX IX_Date_FullDate
ON dim.Date(FullDate);
```

### 3.2 Data Manipulation Language (DML)
```sql
-- Inserir dados
INSERT INTO dim.Date (Date_SK, FullDate, DayOfWeek, DayName, Month, MonthName, Quarter, Year)
VALUES (20240101, '2024-01-01', 1, 'Monday', 1, 'January', 1, 2024);

-- Atualizar dados
UPDATE dim.Date
SET IsWeekend = 1
WHERE DayOfWeek IN (6,7);

-- Excluir dados
DELETE FROM dim.Date
WHERE Year < 2020;
```

## 4. Versionamento

### 4.1 Controle de Versão de Esquema
```sql
-- Tabela de controle de versão
CREATE TABLE meta.SchemaVersion (
    Version_SK int IDENTITY(1,1),
    VersionNumber varchar(20) NOT NULL,
    Description nvarchar(500) NOT NULL,
    ScriptName varchar(100) NOT NULL,
    AppliedDate datetime2 NOT NULL DEFAULT(GETDATE()),
    AppliedBy nvarchar(100) NOT NULL DEFAULT(SYSTEM_USER)
);

-- Registrar nova versão
INSERT INTO meta.SchemaVersion (VersionNumber, Description, ScriptName)
VALUES ('1.0.0', 'Initial schema creation', 'V1.0.0_Initial.sql');
```

### 4.2 Scripts de Migração
```sql
-- Exemplo de script de migração
BEGIN TRANSACTION;
    
    -- Verificar versão atual
    DECLARE @CurrentVersion varchar(20);
    SELECT @CurrentVersion = MAX(VersionNumber)
    FROM meta.SchemaVersion;
    
    -- Aplicar alterações
    IF @CurrentVersion = '1.0.0'
    BEGIN
        -- Alterações do schema
        ALTER TABLE dim.Customer
        ADD Email varchar(100);
        
        -- Registrar nova versão
        INSERT INTO meta.SchemaVersion (VersionNumber, Description, ScriptName)
        VALUES ('1.0.1', 'Added Email to Customer dimension', 'V1.0.1_AddEmail.sql');
    END

COMMIT TRANSACTION;
```

## 5. Backup e Restore

### 5.1 Estratégias de Backup
```sql
-- Backup completo
BACKUP DATABASE DWDatabase
TO DISK = 'C:\Backup\DWDatabase_Full.bak'
WITH INIT, NAME = 'DWDatabase Full Backup';

-- Backup diferencial
BACKUP DATABASE DWDatabase
TO DISK = 'C:\Backup\DWDatabase_Diff.bak'
WITH DIFFERENTIAL, NAME = 'DWDatabase Differential Backup';

-- Backup de log
BACKUP LOG DWDatabase
TO DISK = 'C:\Backup\DWDatabase_Log.trn'
WITH NAME = 'DWDatabase Log Backup';
```

### 5.2 Restore
```sql
-- Restore completo
RESTORE DATABASE DWDatabase
FROM DISK = 'C:\Backup\DWDatabase_Full.bak'
WITH REPLACE, RECOVERY;

-- Restore com vários arquivos
RESTORE DATABASE DWDatabase
FROM DISK = 'C:\Backup\DWDatabase_Full.bak'
WITH NORECOVERY;

RESTORE DATABASE DWDatabase
FROM DISK = 'C:\Backup\DWDatabase_Diff.bak'
WITH NORECOVERY;

RESTORE LOG DWDatabase
FROM DISK = 'C:\Backup\DWDatabase_Log.trn'
WITH RECOVERY;
```

## Exercícios Práticos

### Exercício 1: Configuração
1. Instale o SQL Server Developer Edition
2. Configure uma instância com:
   - Memória apropriada
   - Autenticação mista
   - Diretórios personalizados

### Exercício 2: Criação de Ambiente
1. Crie um database para DW
2. Configure schemas apropriados
3. Implemente controle de versão
4. Crie usuários e permissões

### Exercício 3: Backup e Restore
1. Desenvolva estratégia de backup
2. Implemente rotina de backup
3. Pratique cenários de restore
4. Documente procedimentos

## Material Complementar

### Documentação
- SQL Server Configuration Guide
- Backup and Restore Best Practices
- Security Guidelines

### Scripts Úteis
- Verificação de configuração
- Monitoramento de backup
- Análise de crescimento de database

## Próxima Aula
Na próxima aula, começaremos a trabalhar com Slowly Changing Dimensions (SCD) Parte 1.

## Avaliação
1. Quiz sobre configuração
2. Projeto prático de implementação
3. Documentação de ambiente