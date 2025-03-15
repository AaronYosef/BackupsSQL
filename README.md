# Procedimiento Almacenado para Backups en SQL Server  

Este repositorio contiene un **Stored Procedure** en **SQL Server** para la ejecución de **backups completos, diferenciales y de logs de transacciones**, almacenándolos en una estructura de carpetas organizada.  

## Características  
- Realiza **backups completos, diferenciales y de logs** según el tipo solicitado.  
- Organiza los archivos de respaldo en carpetas por **base de datos y tipo de backup**.  
- Soporta **infraestructura Windows Server** y **entornos Docker**.  
- Genera nombres de archivos con **timestamps** para evitar sobrescribir respaldos previos.  
- Puede ser **automatizado con SQL Server Agent**.  

## Estructura de Carpetas  
Los backups se almacenan en la siguiente estructura:  
```
Backups/
│── <DatabaseName>/
│   ├── Full/
│   │   ├── Database_FULL_YYYYMMDD_HHMMSS.bak
│   ├── Differential/
│   │   ├── Database_DIFF_YYYYMMDD_HHMMSS.dif
│   ├── Log/
│   │   ├── Database_LOG_YYYYMMDD_HHMMSS.trn
```

## Instalación y Uso  

1. **Crear el Stored Procedure en SQL Server**  
   Copia y ejecuta el siguiente código en **SQL Server Management Studio (SSMS)**:  

   ```sql
   CREATE PROCEDURE sp_BackupDatabase  
       @DatabaseName NVARCHAR(100),  
       @BackupType NVARCHAR(20)  -- Puede ser 'FULL', 'DIFFERENTIAL' o 'LOG'  
   AS  
   BEGIN  
       SET NOCOUNT ON;  

       -- Validar si la base de datos existe  
       IF NOT EXISTS (SELECT 1 FROM sys.databases WHERE name = @DatabaseName)  
       BEGIN  
           PRINT 'Error: La base de datos especificada no existe.';  
           RETURN;  
       END  

       DECLARE @BackupPath NVARCHAR(500);  
       DECLARE @BackupFileName NVARCHAR(500);  
       DECLARE @FileExtension NVARCHAR(10);  
       DECLARE @BackupCommand NVARCHAR(MAX);  
       DECLARE @CurrentDateTime NVARCHAR(20);  

       -- Obtener timestamp para el nombre del archivo  
       SET @CurrentDateTime = FORMAT(GETDATE(), 'yyyyMMdd_HHmmss');  

       -- Definir extensión del archivo según tipo de backup  
       SET @FileExtension = CASE  
           WHEN @BackupType = 'FULL' THEN '.bak'  
           WHEN @BackupType = 'DIFFERENTIAL' THEN '.dif'  
           WHEN @BackupType = 'LOG' THEN '.trn'  
           ELSE NULL  
       END  

       IF @FileExtension IS NULL  
       BEGIN  
           PRINT 'Error: Tipo de backup no válido. Use FULL, DIFFERENTIAL o LOG.';  
           RETURN;  
       END  

       -- Definir ruta de almacenamiento  
       SET @BackupPath = 'C:\Backups\' + @DatabaseName + '\' + @BackupType + '\';  

       -- Crear nombre del archivo  
       SET @BackupFileName = @BackupPath + @DatabaseName + '_' + @BackupType + '_' + @CurrentDateTime + @FileExtension;  

       -- Construcción del comando de respaldo  
       SET @BackupCommand = 'BACKUP DATABASE ' + QUOTENAME(@DatabaseName) +  
                            ' TO DISK = ''' + @BackupFileName + '''' +  
                            CASE  
                                WHEN @BackupType = 'FULL' THEN ' WITH FORMAT, INIT, NAME = ''Full Backup'''  
                                WHEN @BackupType = 'DIFFERENTIAL' THEN ' WITH DIFFERENTIAL, NAME = ''Differential Backup'''  
                                WHEN @BackupType = 'LOG' THEN ' WITH LOG, NAME = ''Transaction Log Backup'''  
                                ELSE ''  
                            END;  

       -- Ejecutar el backup  
       EXEC sp_executesql @BackupCommand;  

       PRINT 'Backup realizado correctamente: ' + @BackupFileName;  
   END  
   ```

2. **Ejecutar el Stored Procedure**  
   Para generar un backup hay que ejecutar:  

   ```sql
   EXEC sp_BackupDatabase 'MiBaseDeDatos', 'FULL';  
   ```  

   ```sql
   EXEC sp_BackupDatabase 'MiBaseDeDatos', 'DIFFERENTIAL';  
   ```  

   ```sql
   EXEC sp_BackupDatabase 'MiBaseDeDatos', 'LOG';  
   ```

## Automatización con SQL Server Agent  
Para ejecutar los backups automáticamente, crea una tarea en **SQL Server Agent** con la siguiente frecuencia:  
- **Backup completo:** Cada 24 horas.  
- **Backup diferencial:** Cada 8 horas.  
- **Backup de logs:** Cada 15 minutos.  

## Implementación en Docker  
Volumen para almacenar los backups:  

```bash
docker run -d --name sqlserver_backup -v /ruta/local/backups:/var/opt/mssql/backups mcr.microsoft.com/mssql/server
```

Ajustastamos la ruta de `@BackupPath` en el procedimiento almacenado:  

```sql
SET @BackupPath = '/var/opt/mssql/backups/' + @DatabaseName + '/' + @BackupType + '/';
```

## Restauración de Backups  
Para restaurar un backup, use el siguiente comando en **SQL Server Management Studio (SSMS)**:  

```sql
RESTORE DATABASE MiBaseDeDatos  
FROM DISK = 'C:\Backups\MiBaseDeDatos\Full\MiBaseDeDatos_FULL_YYYYMMDD_HHMMSS.bak'  
WITH REPLACE, RECOVERY;
```
