# ScriptDB

## Administracion
- SQL Server 2022 Configuration Manager -  Srvicios
- SQL Server Management Studio  - Adminstracion
## Sqlserver

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

