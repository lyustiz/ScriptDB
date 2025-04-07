# ScriptDB

##Sqlserver
BACKUP DATABASE [NombreBaseDatos] TO DISK = 'C:\Ruta\Backup.bak'
BACKUP DATABASE [NombreBaseDatos] TO DISK = 'C:\Ruta\BackupDiff.bak' WITH DIFFERENTIAL
BACKUP LOG [NombreBaseDatos] TO DISK = 'C:\Ruta\BackupLog.trn'
BACKUP DATABASE [NombreBaseDatos] FILE = 'NombreArchivo' TO DISK = 'C:\Ruta\BackupFile.bak'
BACKUP DATABASE [NombreBaseDatos] TO DISK = 'C:\Ruta\BackupCopy.bak' WITH COPY_ONLY
SELECT name, recovery_model_desc FROM sys.databases WHERE name = 'NombreBaseDatos';
ALTER DATABASE [NombreBaseDatos] SET RECOVERY FULL;
restaurables con RESTORE VERIFYONLY.
RESTORE DATABASE [NombreBaseDatos] FROM DISK = 'C:\Ruta\Backup.bak' WITH RECOVERY
ALTER INDEX ALL ON [NombreTabla] REBUILD;
UPDATE STATISTICS [NombreTabla];
DBCC CHECKDB ('NombreBaseDatos')
;DBCC SHRINKFILE ('NombreArchivoLog', 100)
SELECT * FROM sys.dm_exec_requests WHERE status = 'running';
EXEC sp_readerrorlog;
SELECT * FROM sys.dm_tran_locks WHERE resource_type = 'OBJECT'; 
