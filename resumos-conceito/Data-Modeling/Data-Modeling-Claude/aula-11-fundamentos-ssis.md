# Aula 11 - Fundamentos de SSIS

## Objetivos da Aula
- Compreender a arquitetura do SSIS
- Conhecer os componentes principais
- Criar pacotes básicos
- Entender Control Flow vs Data Flow
- Trabalhar com variáveis e parâmetros

## 1. Arquitetura SSIS

### 1.1 Componentes Principais
- Integration Services Service
- SSIS Catalog
- Package Execution Engine
- Runtime Engine
- Data Flow Engine

### 1.2 Ambiente de Desenvolvimento
- Visual Studio com SSIS
- SQL Server Data Tools (SSDT)
- Project Deployment Model
- Package Deployment Model

### 1.3 Estrutura de Projetos
```plaintext
Solution
├── Project
│   ├── Connection Managers
│   ├── Packages
│   │   ├── Package1.dtsx
│   │   └── Package2.dtsx
│   ├── Parameters
│   └── Project.params
└── Project.ispac
```

## 2. Componentes do SSIS

### 2.1 Connection Managers
- Conexões de Banco de Dados
  - OLE DB
  - ADO.NET
  - ODBC
- Conexões de Arquivo
  - Flat File
  - Excel
  - XML
- Conexões de Rede
  - FTP
  - HTTP
  - SMB

### 2.2 Variables
```sql
-- Exemplo de uso de variáveis em script task
Public Sub Main()
    Dim strFileName As String
    strFileName = Dts.Variables("UserFolderPath").Value.ToString() & _
                 Dts.Variables("FileName").Value.ToString()
    
    ' Usar a variável
    If System.IO.File.Exists(strFileName) Then
        Dts.Variables("FileExists").Value = True
    Else
        Dts.Variables("FileExists").Value = False
    End If
    
    Dts.TaskResult = ScriptResults.Success
End Sub
```

### 2.3 Parameters
```sql
-- Exemplo de configuração de parâmetros
CREATE PROCEDURE config.usp_SetSSISParameters
    @ProjectName nvarchar(100),
    @EnvironmentName nvarchar(100)
AS
BEGIN
    DECLARE @reference_id bigint

    -- Obter reference_id do ambiente
    SELECT @reference_id = reference_id
    FROM catalog.environment_references
    WHERE project_name = @ProjectName
    AND environment_name = @EnvironmentName

    -- Configurar parâmetros
    EXEC catalog.set_environment_variable_value
        @variable_name = N'DatabaseName',
        @environment_name = @EnvironmentName,
        @value = N'DWDatabase'

    EXEC catalog.set_environment_variable_value
        @variable_name = N'FileLocation',
        @environment_name = @EnvironmentName,
        @value = N'C:\ETL\Files'
END
```

## 3. Control Flow

### 3.1 Containers
- Sequence Container
- For Loop Container
- Foreach Loop Container
- Task Host Container

### 3.2 Tasks Principais
```sql
-- Execute SQL Task
DECLARE @SQL nvarchar(max)
SET @SQL = N'
    TRUNCATE TABLE stg.Customer;
    INSERT INTO stg.Customer (
        CustomerID,
        CustomerName,
        CustomerType
    )
    SELECT 
        CustomerID,
        CustomerName,
        CustomerType
    FROM source.Customer
    WHERE ModifiedDate >= @LastLoadDate;
'
EXEC sp_executesql @SQL, 
    N'@LastLoadDate datetime',
    @LastLoadDate = @[User::LastLoadDate]

-- File System Task
-- Exemplo de script para mover arquivo
File.Move(
    Dts.Variables["SourceFile"].Value.ToString(),
    Dts.Variables["DestinationFile"].Value.ToString()
)

-- Expression Task
-- Exemplo de expressão
@[User::FullPath] = @[User::FolderPath] + "\\" + @[User::FileName]
```

### 3.3 Precedence Constraints
- Tipos de Precedência
  - Success
  - Failure
  - Completion
- Expressões de Precedência
- Múltiplas Precedências

## 4. Data Flow

### 4.1 Source Components
- OLE DB Source
- Excel Source
- Flat File Source
- XML Source
- Custom Source

### 4.2 Transformation Components
- Derived Column
- Conditional Split
- Lookup
- Aggregate
- Sort
- Union All
- Merge
- Merge Join

### 4.3 Destination Components
- OLE DB Destination
- Flat File Destination
- Excel Destination
- SQL Server Destination

## 5. Implementação Prática

### 5.1 Pacote ETL Básico
```sql
-- 1. Limpar staging
TRUNCATE TABLE stg.Customer;

-- 2. Carregar dados novos
INSERT INTO stg.Customer (
    Customer_BK,
    CustomerName,
    CustomerType,
    Email,
    Phone,
    Address,
    City,
    State,
    Country,
    LoadDate
)
SELECT 
    CustomerID,
    CustomerName,
    CustomerType,
    Email,
    Phone,
    Address,
    City,
    State,
    Country,
    GETDATE()
FROM source.Customer;

-- 3. Atualizar dimensão
MERGE dim.Customer as target
USING stg.Customer as source
ON target.Customer_BK = source.Customer_BK

WHEN MATCHED AND (
    target.CustomerName <> source.CustomerName OR
    target.CustomerType <> source.CustomerType OR
    ISNULL(target.Email,'') <> ISNULL(source.Email,'') OR
    ISNULL(target.Phone,'') <> ISNULL(source.Phone,'') OR
    ISNULL(target.Address,'') <> ISNULL(source.Address,'') OR
    ISNULL(target.City,'') <> ISNULL(source.City,'') OR
    ISNULL(target.State,'') <> ISNULL(source.State,'') OR
    ISNULL(target.Country,'') <> ISNULL(source.Country,'')
) THEN
    UPDATE SET
        CustomerName = source.CustomerName,
        CustomerType = source.CustomerType,
        Email = source.Email,
        Phone = source.Phone,
        Address = source.Address,
        City = source.City,
        State = source.State,
        Country = source.Country,
        Updated_DT = GETDATE()

WHEN NOT MATCHED THEN
    INSERT (
        Customer_BK,
        CustomerName,
        CustomerType,
        Email,
        Phone,
        Address,
        City,
        State,
        Country,
        Created_DT,
        Updated_DT
    )
    VALUES (
        source.Customer_BK,
        source.CustomerName,
        source.CustomerType,
        source.Email,
        source.Phone,
        source.Address,
        source.City,
        source.State,
        source.Country,
        GETDATE(),
        GETDATE()
    );
```

### 5.2 Logging e Auditoria
```sql
-- Tabela de log
CREATE TABLE meta.ETLLog (
    Log_SK bigint IDENTITY(1,1),
    PackageName nvarchar(100) NOT NULL,
    TaskName nvarchar(100) NOT NULL,
    StartTime datetime2 NOT NULL,
    EndTime datetime2 NULL,
    RowsProcessed int NULL,
    Status varchar(20) NOT NULL,
    ErrorMessage nvarchar(max) NULL,
    CONSTRAINT PK_ETLLog PRIMARY KEY (Log_SK)
);

-- Procedure de log
CREATE PROCEDURE meta.usp_LogETLEvent
    @PackageName nvarchar(100),
    @TaskName nvarchar(100),
    @StartTime datetime2,
    @EndTime datetime2 = NULL,
    @RowsProcessed int = NULL,
    @Status varchar(20),
    @ErrorMessage nvarchar(max) = NULL
AS
BEGIN
    INSERT INTO meta.ETLLog (
        PackageName,
        TaskName,
        StartTime,
        EndTime,
        RowsProcessed,
        Status,
        ErrorMessage
    )
    VALUES (
        @PackageName,
        @TaskName,
        @StartTime,
        @EndTime,
        @RowsProcessed,
        @Status,
        @ErrorMessage
    );
END;
```

## Exercícios Práticos

### Exercício 1: Configuração
1. Instalar e configurar SSIS
2. Criar projeto SSIS
3. Configurar conexões
4. Testar ambiente

### Exercício 2: Pacote Básico
1. Criar pacote ETL simples
2. Implementar Control Flow
3. Implementar Data Flow
4. Adicionar logging

### Exercício 3: Variáveis e Parâmetros
1. Criar variáveis
2. Configurar parâmetros
3. Usar expressões
4. Implementar lógica dinâmica

## Material Complementar

### Documentação
- SSIS Developer Guide
- Best Practices for SSIS
- Performance Tuning Guide

### Scripts Úteis
- Templates de packages
- Scripts de manutenção
- Scripts de monitoramento

## Próxima Aula
Na próxima aula, abordaremos Data Flow Tasks em detalhes, incluindo transformações e otimizações.

## Avaliação
1. Quiz sobre fundamentos SSIS
2. Implementação de pacote básico
3. Documentação de package