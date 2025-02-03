# Aula 23 - Arquitetura e Escalabilidade

## Objetivos da Aula
- Compreender arquitetura física
- Configurar hardware adequadamente
- Implementar compressão de dados
- Desenvolver alta disponibilidade
- Trabalhar com disaster recovery

## 1. Arquitetura Física

### 1.1 Configuração de Hardware
```sql
-- View para análise de recursos
CREATE VIEW monitor.HardwareUsage
AS
SELECT 
    cpu_count as CPUCount,
    scheduler_count as SchedulerCount,
    hyperthread_ratio as HyperthreadRatio,
    physical_memory_kb / 1024 as PhysicalMemoryMB,
    virtual_memory_kb / 1024 as VirtualMemoryMB,
    committed_kb / 1024 as CommittedMemoryMB,
    committed_target_kb / 1024 as CommittedTargetMB
FROM sys.dm_os_sys_info;

-- Procedure para configuração de memória
CREATE PROCEDURE config.usp_ConfigureMemory
    @MaxMemoryMB int,
    @MinMemoryMB int
AS
BEGIN
    -- Configurar memória máxima
    EXEC sp_configure 'show advanced options', 1;
    RECONFIGURE;
    
    EXEC sp_configure 'max server memory (MB)', @MaxMemoryMB;
    EXEC sp_configure 'min server memory (MB)', @MinMemoryMB;
    RECONFIGURE;
    
    -- Verificar configuração
    SELECT 
        name,
        value,
        value_in_use
    FROM sys.configurations
    WHERE name IN (
        'max server memory (MB)',
        'min server memory (MB)'
    );
END;
```

### 1.2 Filegroups e Files
```sql
-- Procedure para configurar filegroups
CREATE PROCEDURE config.usp_ConfigureFileGroups
    @DatabaseName sysname,
    @DataPath varchar(500),
    @LogPath varchar(500)
AS
BEGIN
    DECLARE @SQL nvarchar(max);
    
    -- Adicionar filegroups
    SET @SQL = '
    ALTER DATABASE ' + QUOTENAME(@DatabaseName) + '
    ADD FILEGROUP [DATA_FG];
    
    ALTER DATABASE ' + QUOTENAME(@DatabaseName) + '
    ADD FILEGROUP [INDEX_FG];
    
    ALTER DATABASE ' + QUOTENAME(@DatabaseName) + '
    ADD FILEGROUP [ARCHIVE_FG];';
    
    EXEC sp_executesql @SQL;
    
    -- Adicionar arquivos
    SET @SQL = '
    ALTER DATABASE ' + QUOTENAME(@DatabaseName) + '
    ADD FILE (
        NAME = ''' + @DatabaseName + '_Data1'',
        FILENAME = ''' + @DataPath + '\' + @DatabaseName + '_Data1.ndf'',
        SIZE = 10GB,
        MAXSIZE = UNLIMITED,
        FILEGROWTH = 1GB
    ) TO FILEGROUP [DATA_FG];';
    
    EXEC sp_executesql @SQL;
    
    -- Configurar log
    SET @SQL = '
    ALTER DATABASE ' + QUOTENAME(@DatabaseName) + '
    MODIFY FILE (
        NAME = ''' + @DatabaseName + '_Log'',
        SIZE = 5GB,
        MAXSIZE = UNLIMITED,
        FILEGROWTH = 1GB
    );';
    
    EXEC sp_executesql @SQL;
END;
```

## 2. Configuração de Hardware

### 2.1 CPU e Memória
```sql
-- Procedure para monitoramento de CPU
CREATE PROCEDURE monitor.usp_CPUUsage
AS
BEGIN
    -- Uso de CPU por banco
    SELECT 
        DB_NAME(database_id) as DatabaseName,
        SUM(total_worker_time) / 1000.0 as CPU_ms,
        SUM(execution_count) as ExecutionCount,
        SUM(total_worker_time) * 1.0 / 
            SUM(execution_count) / 1000.0 as AvgCPU_ms
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY (
        SELECT CAST(SUBSTRING(sql_handle, 1, 2) as int) as database_id
    ) AS db
    GROUP BY database_id
    ORDER BY CPU_ms DESC;

    -- Uso de CPU por query
    SELECT TOP 10
        SUBSTRING(
            qt.text,
            (qs.statement_start_offset/2) + 1,
            ((CASE qs.statement_end_offset
                WHEN -1 THEN DATALENGTH(qt.text)
                ELSE qs.statement_end_offset
            END - qs.statement_start_offset)/2) + 1
        ) as QueryText,
        qs.execution_count,
        qs.total_worker_time / 1000.0 as CPU_ms,
        qs.total_worker_time * 1.0 / 
            qs.execution_count / 1000.0 as AvgCPU_ms
    FROM sys.dm_exec_query_stats qs
    CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
    ORDER BY qs.total_worker_time DESC;
END;

-- Procedure para monitoramento de memória
CREATE PROCEDURE monitor.usp_MemoryUsage
AS
BEGIN
    -- Uso de memória por buffer pool
    SELECT 
        DB_NAME(database_id) as DatabaseName,
        COUNT(*) * 8 / 1024.0 as BufferMB,
        COUNT(*) * 8.0 / 
            SUM(COUNT(*)) OVER() * 100 as BufferPercent
    FROM sys.dm_os_buffer_descriptors
    GROUP BY database_id
    ORDER BY BufferMB DESC;

    -- Uso de memória por objeto
    SELECT TOP 10
        OBJECT_NAME(p.object_id) as ObjectName,
        p.index_id,
        COUNT(*) * 8 / 1024.0 as BufferMB,
        COUNT(DISTINCT p.partition_number) as Partitions
    FROM sys.allocation_units a
    JOIN sys.dm_os_buffer_descriptors b 
        ON a.allocation_unit_id = b.allocation_unit_id
    JOIN sys.partitions p 
        ON a.container_id = p.hobt_id
    WHERE b.database_id = DB_ID()
    GROUP BY p.object_id, p.index_id
    ORDER BY BufferMB DESC;
END;
```

### 2.2 I/O e Storage
```sql
-- Procedure para monitoramento de I/O
CREATE PROCEDURE monitor.usp_IOUsage
AS
BEGIN
    -- I/O por arquivo
    SELECT 
        DB_NAME(database_id) as DatabaseName,
        file_id,
        num_of_reads,
        num_of_writes,
        io_stall_read_ms,
        io_stall_write_ms,
        size_on_disk_bytes / 1024.0 / 1024.0 as SizeMB,
        CAST(100.0 * io_stall_read_ms /
            (io_stall_read_ms + io_stall_write_ms) as decimal(5,2))
            as ReadLatencyPercent
    FROM sys.dm_io_virtual_file_stats(NULL, NULL)
    ORDER BY (io_stall_read_ms + io_stall_write_ms) DESC;

    -- I/O por tabela
    SELECT TOP 10
        OBJECT_NAME(object_id) as TableName,
        range_scan_count,
        page_io_latch_wait_count,
        page_io_latch_wait_in_ms,
        CAST(100.0 * range_scan_count /
            SUM(range_scan_count) OVER() as decimal(5,2))
            as ScanPercent
    FROM sys.dm_db_index_operational_stats(
        DB_ID(), NULL, NULL, NULL)
    WHERE object_id > 100
    ORDER BY page_io_latch_wait_in_ms DESC;
END;
```

## 3. Compressão de Dados

### 3.1 Estratégias de Compressão
```sql
-- Procedure para análise de compressão
CREATE PROCEDURE maintenance.usp_AnalyzeCompression
    @TableName nvarchar(128)
AS
BEGIN
    -- Estimar compressão
    EXEC sp_estimate_data_compression_savings
        @schema_name = 'dbo',
        @object_name = @TableName,
        @index_id = NULL,
        @partition_number = NULL,
        @data_compression = 'PAGE';

    -- Analisar espaço atual
    SELECT 
        OBJECT_NAME(object_id) as TableName,
        data_compression_desc,
        partition_number,
        rows,
        used_page_count * 8 / 1024.0 as UsedSpaceMB
    FROM sys.dm_db_partition_stats p
    JOIN sys.partitions pt 
        ON p.partition_id = pt.partition_id
    WHERE object_id = OBJECT_ID(@TableName);
END;

-- Procedure para implementar compressão
CREATE PROCEDURE maintenance.usp_ImplementCompression
    @TableName nvarchar(128),
    @CompressionType varchar(20)
AS
BEGIN
    DECLARE @SQL nvarchar(max);
    
    -- Comprimir tabela
    SET @SQL = '
    ALTER TABLE ' + @TableName + '
    REBUILD PARTITION = ALL
    WITH (DATA_COMPRESSION = ' + @CompressionType + ');';
    
    EXEC sp_executesql @SQL;
    
    -- Comprimir índices
    SET @SQL = '
    ALTER INDEX ALL ON ' + @TableName + '
    REBUILD
    WITH (DATA_COMPRESSION = ' + @CompressionType + ');';
    
    EXEC sp_executesql @SQL;
END;
```

### 3.2 Monitoramento de Compressão
```sql
-- View para análise de compressão
CREATE VIEW monitor.CompressionStatus
AS
SELECT 
    OBJECT_SCHEMA_NAME(object_id) as SchemaName,
    OBJECT_NAME(object_id) as TableName,
    index_id,
    partition_number,
    data_compression_desc,
    rows,
    used_page_count * 8 / 1024.0 as UsedSpaceMB
FROM sys.dm_db_partition_stats p
JOIN sys.partitions pt 
    ON p.partition_id = pt.partition_id
WHERE data_compression > 0;
```

## 4. Alta Disponibilidade

### 4.1 Always On
```sql
-- Procedure para configuração de AG
CREATE PROCEDURE config.usp_ConfigureAvailabilityGroup
    @AGName sysname,
    @PrimaryServer varchar(128),
    @SecondaryServer varchar(128)
AS
BEGIN
    -- Criar endpoint
    CREATE ENDPOINT [Hadr_endpoint]
    STATE = STARTED
    AS TCP (
        LISTENER_PORT = 5022
    )
    FOR DATABASE_MIRRORING (
        ROLE = ALL,
        AUTHENTICATION = WINDOWS NEGOTIATE
    );
    
    -- Criar AG
    CREATE AVAILABILITY GROUP [@AGName]
    FOR DATABASE [YourDatabase]
    REPLICA ON
        @PrimaryServer WITH (
            ENDPOINT_URL = 'TCP://primary:5022',
            FAILOVER_MODE = AUTOMATIC,
            AVAILABILITY_MODE = SYNCHRONOUS_COMMIT
        ),
        @SecondaryServer WITH (
            ENDPOINT_URL = 'TCP://secondary:5022',
            FAILOVER_MODE = AUTOMATIC,
            AVAILABILITY_MODE = SYNCHRONOUS_COMMIT
        );
END;
```

### 4.2 Monitoramento de HA
```sql
-- View para monitoramento de AG
CREATE VIEW monitor.AGStatus
AS
SELECT 
    ag.name as AGName,
    ar.replica_server_name,
    drs.database_name,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc,
    CAST(drs.log_send_queue_size / 1024.0 as decimal(10,2)) 
        as LogSendQueueMB,
    CAST(drs.redo_queue_size / 1024.0 as decimal(10,2)) 
        as RedoQueueMB,
    drs.last_commit_time
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_replicas ar 
    ON drs.replica_id = ar.replica_id
JOIN sys.availability_groups ag 
    ON ar.group_id = ag.group_id;
```

## 5. Disaster Recovery

### 5.1 Backup Strategy
```sql
-- Procedure para configuração de backup
CREATE PROCEDURE maintenance.usp_ConfigureBackup
    @DatabaseName sysname,
    @BackupPath varchar(500)
AS
BEGIN
    -- Backup full
    BACKUP DATABASE @DatabaseName
    TO DISK = @BackupPath + '\' + @DatabaseName + 
        '_FULL_' + FORMAT(GETDATE(), 'yyyyMMdd') + '.bak'
    WITH COMPRESSION, CHECKSUM, FORMAT;
    
    -- Backup diferencial
    BACKUP DATABASE @DatabaseName
    TO DISK = @BackupPath + '\' + @DatabaseName + 
        '_DIFF_' + FORMAT(GETDATE(), 'yyyyMMdd') + '.bak'
    WITH DIFFERENTIAL, COMPRESSION, CHECKSUM;
    
    -- Backup de log
    BACKUP LOG @DatabaseName
    TO DISK = @BackupPath + '\' + @DatabaseName + 
        '_LOG_' + FORMAT(GETDATE(), 'yyyyMMddHHmm') + '.trn'
    WITH COMPRESSION, CHECKSUM;
END;
```

### 5.2 Recovery
```sql
-- Procedure para restore
CREATE PROCEDURE maintenance.usp_RestoreDatabase
    @DatabaseName sysname,
    @BackupPath varchar(500),
    @TargetDate datetime2
AS
BEGIN
    DECLARE @LastFullBackup varchar(500);
    DECLARE @LastDiffBackup varchar(500);
    DECLARE @LogBackups TABLE (
        BackupFile varchar(500),
        BackupDate datetime2
    );
    
    -- Encontrar backups necessários
    SELECT TOP 1 @LastFullBackup = BackupFile
    FROM maintenance.BackupHistory
    WHERE DatabaseName = @DatabaseName
    AND BackupType = 'FULL'
    AND BackupDate <= @TargetDate
    ORDER BY BackupDate DESC;
    
    SELECT TOP 1 @LastDiffBackup = BackupFile
    FROM maintenance.BackupHistory
    WHERE DatabaseName = @DatabaseName
    AND BackupType = 'DIFF'
    AND BackupDate <= @TargetDate
    AND BackupDate > (
        SELECT BackupDate
        FROM maintenance.BackupHistory
        WHERE BackupFile = @LastFullBackup
    )
    ORDER BY BackupDate DESC;
    
    INSERT INTO @LogBackups
    