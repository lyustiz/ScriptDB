# ScriptDB

## Sqlserver

## Administracion
- SQL Server 2022 Configuration Manager -  Srvicios
- SQL Server Management Studio  - Adminstracion
- Data Migration Asssistant  - Migracion
- Copy Databese Wizard - MIgracion

## DBCC
1. Verificación de Integridad
- Verifica la integridad de toda la base de datos.
~~~
DBCC CHECKDB('NombreBD') WITH NO_INFOMSGS;
~~~
 - Opciones de reparación (REPAIR)
--- REPAIR_ALLOW_DATA_LOSS**, REPAIR_FAST, REPAIR_REBUILD*
 - Opciones de verificación
--- NOINDEX, EXTENDED_LOGICAL_CHECKS, TABLOCK,PHYSICAL_ONLY*
 - Opciones de reporte
--- WITH NO_INFOMSGS*, ALL_ERRORMSGS, ESTIMATEONLY*
 ~~~
 SELECT * FROM sys.objets WHERE object_id = ID (identificar objeto corrupto por id)
 ALTER DATABASE dbname SET SINGLE USER WITH ROOLBACK IMMEDIATE;
 DBCC CHACHDB('dbname', REAPAIR_REBUILD)
 ALTER DATABASE dbname SET MULTI_USER WITH ROOLBACK IMMEDIATE;
 ~~~
- Revisa una tabla específica.
~~~
DBCC CHECKTABLE('Tabla') WITH NO_INFOMSGS;
~~~
- Examina la asignación de páginas en la BD.
~~~
DBCC CHECKALLOC('NombreBD');
~~~
2. Mantenimiento de Archivos
- Reduce el tamaño de la base de datos.
~~~
DBCC SHRINKDATABASE('NombreBD', 10); --
~~~
- Reduce un archivo específico (ej. .mdf o .ldf).
~~~
DBCC SHRINKFILE('ArchivoLog', 100);
~~~
- SHRINK puede causar fragmentación → Rebuild índices después.
4. Limpieza de Caché
- Borra el buffer cache (útil para pruebas).
~~~
DBCC DROPCLEANBUFFERS;
~~~
- Limpia el caché de ejecución.
~~~
DBCC FREEPROCCACHE;
~~~
5. Estadísticas y Diagnóstico
- Muestra estadísticas del log de transacciones.
~~~
DBCC SQLPERF(LOGSPACE);
~~~
- Muestra estadísticas de índices.
~~~
DBCC SHOW_STATISTICS('Tabla', 'Índice');
~~~
- Muestra configuraciones de la sesión actual.
~~~
DBCC USEROPTIONS;
~~~
6. Depuración (Trace Flags)
- Activa/desactiva flags de seguimiento.
~~~
DBCC TRACEON(1222, -1);  -- Registra deadlocks en el error log
DBCC TRACEOFF(1222, -1);
~~~
- Algunos comandos requieren permisos de sysadmin.
  
### Versiones
~~~
SELECT @@version;
SELECT SERVERPROPERTY('productversion'), SERVERPROPERTY('productlevel'), SERVERPROPERTY('edition')
~~~
 SQL Server 2016 (versión 13.0), SQL Server 2017 (versión 14.0), SQL Server 2019 (versión 15.0), SQL Server 2022 (versión 16.0). 

### Backup
~~~
BACKUP DATABASE [NombreBaseDatos] TO DISK = 'C:\Ruta\Backup.bak';
~~~
~~~
BACKUP DATABASE [NombreBaseDatos] TO DISK = 'C:\Ruta\BackupDiff.bak' WITH DIFFERENTIAL;
~~~
~~~
BACKUP LOG [NombreBaseDatos] TO DISK = 'C:\Ruta\BackupLog.trn'.
~~~
~~~
BACKUP DATABASE [NombreBaseDatos] FILE = 'NombreArchivo' TO DISK = 'C:\Ruta\BackupFile.bak'.
~~~
~~~
BACKUP DATABASE [NombreBaseDatos] TO DISK = 'C:\Ruta\BackupCopy.bak' WITH COPY_ONLY.
~~~
~~~
SELECT name, recovery_model_desc FROM sys.databases WHERE name = 'NombreBaseDatos';
~~~
~~~
ALTER DATABASE [NombreBaseDatos] SET RECOVERY FULL;
~~~
- restaurables con RESTORE VERIFYONLY.
~~~
- RESTORE DATABASE [NombreBaseDatos] FROM DISK = 'C:\Ruta\Backup.bak' WITH RECOVERY
~~~
~~~
ALTER INDEX ALL ON [NombreTabla] REBUILD;
~~~
~~~
UPDATE STATISTICS [NombreTabla];
~~~
~~~
DBCC CHECKDB ('NombreBaseDatos')
~~~
~~~
DBCC SHRINKFILE ('NombreArchivoLog', 100)
~~~
~~~
SELECT * FROM sys.dm_exec_requests WHERE status = 'running';
~~~
~~~
EXEC sp_readerrorlog;
~~~
~~~
SELECT * FROM sys.dm_tran_locks WHERE resource_type = 'OBJECT';
~~~

### Indices
- Muestra estadísticas detalladas de un índice.
~~~
DBCC SHOW_STATISTICS('NombreTabla', 'NombreIndice');
~~~
~~~
-- Verificación rápida (recomendada semanalmente)
DBCC CHECKDB('NombreBD') WITH PHYSICAL_ONLY, NO_INFOMSGS;

-- Verificación completa (recomendada mensualmente)
DBCC CHECKDB('NombreBD') WITH NO_INFOMSGS;

-- muestra información sobre fragmentación
SELECT * FROM sys.dm_db_index_physical_stats(
    DB_ID('NombreBD'), 
    OBJECT_ID('NombreTabla'), 
    NULL, NULL, 'DETAILED');

-- Reorganización (operación en línea, menos intrusiva)
ALTER INDEX [NombreIndice] ON [NombreTabla] REORGANIZE;

-- Reconstrucción (más completa pero puede bloquear la tabla)
ALTER INDEX [NombreIndice] ON [NombreTabla] REBUILD;

-- Actualización de estadísticas:
UPDATE STATISTICS [NombreTabla] [NombreIndice] WITH FULLSCAN;

-- DMv
sys.dm_db_index_physical_stats
sys.dm_db_index_usage_stats
sys.dm_db_index_operational_stats
sys.dm_db_missing_index_stats
~~~
- Tipos  CLUSTER - NONCLUSTER


# DMVs y DMFs en SQL Server: Dynamic Management Views and Functions

Los **Objetos de Administración Dinámica (DMVs/DMFs)** son vistas y funciones que proporcionan información crítica sobre el estado y rendimiento de SQL Server. Son herramientas esenciales para el monitoreo y solución de problemas.

## Principales categorías de DMVs

### 1. DMVs de rendimiento
```sql
-- Consultas más costosas
SELECT TOP 10 
    qs.total_elapsed_time/qs.execution_count AS avg_elapsed_time,
    qs.total_logical_reads/qs.execution_count AS avg_logical_reads,
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
          WHEN -1 THEN DATALENGTH(qt.text)
          ELSE qs.statement_end_offset
          END - qs.statement_start_offset)/2)+1) AS query_text
FROM sys.dm_exec_query_stats AS qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) AS qt
ORDER BY avg_elapsed_time DESC;
```

### 2. DMVs de índices
```sql
-- Fragmentación de índices
SELECT 
    OBJECT_NAME(ind.object_id) AS TableName,
    ind.name AS IndexName,
    ips.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, NULL) ips
INNER JOIN sys.indexes ind ON ips.object_id = ind.object_id AND ips.index_id = ind.index_id
WHERE ips.avg_fragmentation_in_percent > 30
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

### 3. DMVs de conexiones
```sql
-- Conexiones activas
SELECT 
    c.session_id,
    s.login_name,
    s.host_name,
    s.program_name,
    c.connect_time,
    s.status
FROM sys.dm_exec_connections c
JOIN sys.dm_exec_sessions s ON c.session_id = s.session_id;
```


## DMVs útiles para administración diaria

### Monitorización de bloqueos
```sql
SELECT
    tl.request_session_id,
    wt.blocking_session_id,
    OBJECT_NAME(p.object_id) AS BlockedObjectName,
    tl.resource_type,
    tl.request_mode,
    tl.request_status
FROM sys.dm_tran_locks tl
JOIN sys.partitions p ON tl.resource_associated_entity_id = p.hobt_id
JOIN sys.dm_os_waiting_tasks wt ON tl.lock_owner_address = wt.resource_address;
```

### Uso de memoria
```sql
SELECT
    type,
    pages_kb/1024 AS pages_mb,
    virtual_memory_committed_kb/1024 AS virtual_memory_mb
FROM sys.dm_os_memory_clerks
ORDER BY pages_kb DESC;
```

### Planes de ejecución en caché
```sql
SELECT TOP 20
    qs.execution_count,
    qs.total_logical_reads/qs.execution_count AS avg_logical_reads,
    qs.total_elapsed_time/qs.execution_count AS avg_elapsed_time,
    SUBSTRING(qt.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
          WHEN -1 THEN DATALENGTH(qt.text)
          ELSE qs.statement_end_offset
          END - qs.statement_start_offset)/2)+1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
ORDER BY qs.total_logical_reads DESC;
```
### Identificar cuellos de botella en E/S
```sql
SELECT 
    DB_NAME(vfs.database_id) AS BaseDeDatos,
    mf.physical_name AS ArchivoFisico,
    vfs.num_of_reads AS Lecturas,
    vfs.num_of_bytes_read AS BytesLeidos,
    vfs.io_stall_read_ms AS EsperaLectura_ms,
    vfs.num_of_writes AS Escrituras,
    vfs.num_of_bytes_written AS BytesEscritos,
    vfs.io_stall_write_ms AS EsperaEscritura_ms,
    vfs.io_stall AS EsperaTotal,
    vfs.size_on_disk_bytes / 1024 / 1024 AS TamañoEnDisco_MB
FROM 
    sys.dm_io_virtual_file_stats(NULL, NULL) AS vfs
JOIN 
    sys.master_files AS mf ON vfs.database_id = mf.database_id 
    AND vfs.file_id = mf.file_id
ORDER BY 
    vfs.io_stall DESC;
```

### Calcular latencia promedio de E/S
```sql
SELECT 
    DB_NAME(vfs.database_id) AS BaseDeDatos,
    mf.physical_name AS ArchivoFisico,
    CASE 
        WHEN vfs.num_of_reads = 0 THEN 0 
        ELSE vfs.io_stall_read_ms / vfs.num_of_reads 
    END AS LatenciaLecturaPromedio,
    CASE 
        WHEN vfs.num_of_writes = 0 THEN 0 
        ELSE vfs.io_stall_write_ms / vfs.num_of_writes 
    END AS LatenciaEscrituraPromedio,
    CASE 
        WHEN (vfs.num_of_reads = 0 AND vfs.num_of_writes = 0) THEN 0 
        ELSE vfs.io_stall / (vfs.num_of_reads + vfs.num_of_writes) 
    END AS LatenciaTotalPromedio
FROM 
    sys.dm_io_virtual_file_stats(NULL, NULL) AS vfs
JOIN 
    sys.master_files AS mf ON vfs.database_id = mf.database_id 
    AND vfs.file_id = mf.file_id
ORDER BY 
    LatenciaTotalPromedio DESC;
```

### Verificar tamaño de archivo y uso
```sql
SELECT 
    DB_NAME(vfs.database_id) AS BaseDeDatos,
    mf.name AS NombreLogico,
    mf.physical_name AS ArchivoFisico,
    vfs.size_on_disk_bytes / 1024 / 1024 AS TamañoEnDisco_MB,
    mf.size * 8 / 1024 AS TamañoAsignado_MB,
    vfs.num_of_reads AS Lecturas,
    vfs.num_of_writes AS Escrituras
FROM 
    sys.dm_io_virtual_file_stats(NULL, NULL) AS vfs
JOIN 
    sys.master_files AS mf ON vfs.database_id = mf.database_id 
    AND vfs.file_id = mf.file_id
ORDER BY 
    vfs.size_on_disk_bytes DESC;
```
- 20.000 ms max
### Scripts
~~~
	WITH CTE AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY columna_duplicada1, columna_duplicada2 ORDER BY (SELECT NULL)) AS RN
    FROM tu_tabla
	)
	SELECT *
	FROM CTE
	WHERE RN > 1
~~~	
~~~	
	WITH CTE AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY columna_duplicada1, columna_duplicada2 ORDER BY columna_identificadora ASC) AS RN
    FROM tu_tabla
	)
	DELETE FROM CTE
	WHERE RN > 1;
~~~	
-- Eliminar duplicados conservando la fila con el ID más bajo
~~~
	DELETE FROM tu_tabla
WHERE id NOT IN (
    SELECT MIN(id)
    FROM tu_tabla
    GROUP BY columna_duplicada1, columna_duplicada2
);
~~~
-- Eliminar duplicados conservando la fila con el ID más alto
~~~
DELETE FROM tu_tabla
WHERE id NOT IN (
    SELECT MAX(id)
    FROM tu_tabla
    GROUP BY columna_duplicada1, columna_duplicada2
);
~~~
- Duplicados
~~~
SELECT columna1, columna2, ..., COUNT(*)
FROM tu_tabla
GROUP BY columna1, columna2, ...
HAVING COUNT(*) > 1;
~~~
~~~
SELECT a.*
FROM tu_tabla a
WHERE a.rowid > ANY (
    SELECT b.rowid
    FROM tu_tabla b
    WHERE
        -- Aquí debes especificar las condiciones para considerar dos filas como duplicadas
        a.columna1 = b.columna1 AND
        a.columna2 = b.columna2 AND
        ...
);
~~~
~~~
DELETE FROM tu_tabla
WHERE ROWID NOT IN (
    SELECT MIN(ROWID)
    FROM tu_tabla
    GROUP BY columna1, columna2, ... -- Reemplaza con las columnas que definen un duplicado
);
~~~


# ORACLE
- BLOQUEOS
~~~
SELECT
    lo.session_id,
    s.serial#,
    s.username,
    o.owner,
    o.object_name,
    DECODE(lo.locked_mode,
           1, 'Null',
           2, 'Row-S (SS)',
           3, 'Row-X (SX)',
           4, 'Share',
           5, 'S/Row-X (SSX)',
           6, 'Exclusive',
           'Unknown') AS locked_mode
FROM
    v$locked_object lo,
    dba_objects o,
    v$session s
WHERE
    lo.object_id = o.object_id
    AND lo.session_id = s.sid
ORDER BY
    lo.session_id;
~~~
	
- Sesiones están esperando por un evento relacionado con locks.
~~~
	SELECT
    sid,
    serial#,
    username,
    event,
    seconds_in_wait
FROM
    v$session
WHERE
    event LIKE '%lock%'
ORDER BY
    seconds_in_wait DESC;
~~~	
- Analiza el código SQL de las sesiones bloqueantes: Utiliza la V$SQL o V$SQLAREA vistas
~~~	
	ALTER SYSTEM KILL SESSION 'sid,serial#' IMMEDIATE;
~~~

- SELECT_CATALOG_ROLE o privilegios de sistema específicos como SELECT ANY DICTIONARY

