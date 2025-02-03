```sql
-- Continuação do procedimento de restore
    INSERT INTO @LogBackups
    SELECT 
        BackupFile,
        BackupDate
    FROM maintenance.BackupHistory
    WHERE DatabaseName = @DatabaseName
    AND BackupType = 'LOG'
    AND BackupDate > COALESCE(
        (SELECT BackupDate
         FROM maintenance.BackupHistory
         WHERE BackupFile = @LastDiffBackup),
        (SELECT BackupDate
         FROM maintenance.BackupHistory
         WHERE BackupFile = @LastFullBackup)
    )
    AND BackupDate <= @TargetDate
    ORDER BY BackupDate;

    -- Restaurar full backup
    RESTORE DATABASE @DatabaseName
    FROM DISK = @LastFullBackup
    WITH NORECOVERY;

    -- Restaurar differential backup se existir
    IF @LastDiffBackup IS NOT NULL
    BEGIN
        RESTORE DATABASE @DatabaseName
        FROM DISK = @LastDiffBackup
        WITH NORECOVERY;
    END;

    -- Restaurar log backups
    DECLARE @LogBackup varchar(500);
    DECLARE log_cursor CURSOR FOR
    SELECT BackupFile
    FROM @LogBackups
    ORDER BY BackupDate;

    OPEN log_cursor;
    FETCH NEXT FROM log_cursor INTO @LogBackup;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        RESTORE LOG @DatabaseName
        FROM DISK = @LogBackup
        WITH NORECOVERY;

        FETCH NEXT FROM log_cursor INTO @LogBackup;
    END;

    CLOSE log_cursor;
    DEALLOCATE log_cursor;

    -- Recuperar database
    RESTORE DATABASE @DatabaseName WITH RECOVERY;
END;
```

## Exercícios Práticos

### Exercício 1: Configuração de Hardware
1. Análise de requisitos:
   ```sql
   -- Template para análise de requisitos
   SELECT 
       (SELECT COUNT(*) FROM fact.Sales) as TotalRows,
       CAST(SUM(used_page_count * 8.0 / 1024) as decimal(10,2)) as UsedSpaceMB,
       MAX(row_count) as MaxTableSize
   FROM sys.dm_db_partition_stats;
   ```

2. Implementar configurações:
   ```sql
   -- Template para configurações
   EXEC config.usp_ConfigureMemory 
       @MaxMemoryMB = 16384,  -- 16GB
       @MinMemoryMB = 4096;   -- 4GB
   ```

3. Monitorar performance:
   ```sql
   -- Template para monitoramento
   EXEC monitor.usp_CPUUsage;
   EXEC monitor.usp_MemoryUsage;
   EXEC monitor.usp_IOUsage;
   ```

### Exercício 2: Alta Disponibilidade
1. Configurar Always On:
   ```sql
   -- Template de configuração
   EXEC config.usp_ConfigureAvailabilityGroup
       @AGName = 'DW_AG',
       @PrimaryServer = 'SQLPRIMARY',
       @SecondaryServer = 'SQLSECONDARY';
   ```

2. Implementar monitoramento:
   ```sql
   -- Template de monitoramento
   SELECT * FROM monitor.AGStatus;
   ```

3. Testar failover:
   ```sql
   -- Template de teste
   ALTER AVAILABILITY GROUP [DW_AG] FAILOVER;
   ```

### Exercício 3: Disaster Recovery
1. Implementar estratégia de backup:
   ```sql
   -- Template de backup
   EXEC maintenance.usp_ConfigureBackup
       @DatabaseName = 'DWDatabase',
       @BackupPath = 'D:\Backup';
   ```

2. Testar recuperação:
   ```sql
   -- Template de restore
   EXEC maintenance.usp_RestoreDatabase
       @DatabaseName = 'DWDatabase',
       @BackupPath = 'D:\Backup',
       @TargetDate = '2024-03-15 14:30:00';
   ```

3. Documentar processo:
   ```sql
   -- Template de documentação
   INSERT INTO meta.RecoveryProcedures (
       ProcedureName,
       Description,
       Steps,
       LastTested,
       TestedBy
   )
   VALUES (
       'Full Recovery',
       'Process for full database recovery',
       'Step 1: ...',
       GETDATE(),
       SYSTEM_USER
   );
   ```

## Material Complementar

### Documentação
1. Hardware Configuration Guide
   - CPU recommendations
   - Memory settings
   - Storage configuration

2. High Availability Best Practices
   - Always On setup
   - Failover configuration
   - Monitoring guidelines

3. Disaster Recovery Procedures
   - Backup strategy
   - Recovery processes
   - Testing procedures

### Scripts Úteis
1. Monitoramento:
```sql
-- CPU Usage
SELECT * FROM sys.dm_os_schedulers
WHERE scheduler_id < 255;

-- Memory Usage
SELECT * FROM sys.dm_os_process_memory;

-- IO Statistics
SELECT * FROM sys.dm_io_virtual_file_stats(NULL, NULL);
```

2. Manutenção:
```sql
-- Index Maintenance
ALTER INDEX ALL ON TableName REBUILD;

-- Statistics Update
UPDATE STATISTICS TableName WITH FULLSCAN;

-- DBCC Checks
DBCC CHECKDB('DatabaseName');
```

3. Backup Validation:
```sql
-- Verify Backup
RESTORE VERIFYONLY FROM DISK = 'BackupFile';

-- Backup History
SELECT * FROM msdb.dbo.backupset
ORDER BY backup_start_date DESC;
```

## Próxima Aula
Na próxima aula, abordaremos o Projeto Final, onde aplicaremos todos os conceitos aprendidos em um projeto completo de Data Warehouse.

## Avaliação
1. Quiz sobre arquitetura e escalabilidade
2. Projeto prático de implementação
3. Documentação de configurações