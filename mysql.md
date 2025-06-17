#!/bin/bash

# Configuración
MYSQL_USER="tu_usuario"
MYSQL_PASS="tu_contraseña"
MYSQL_HOST="localhost"

# Verificar uptime
echo "=== Uptime de MySQL ==="
mysql -u$MYSQL_USER -p$MYSQL_PASS -h$MYSQL_HOST -e "SHOW GLOBAL STATUS LIKE 'Uptime';"

# Verificar procesos y bloqueos
echo "=== Procesos activos ==="
mysql -u$MYSQL_USER -p$MYSQL_PASS -h$MYSQL_HOST -e "SHOW PROCESSLIST;"

echo "=== Bloqueos actuales ==="
mysql -u$MYSQL_USER -p$MYSQL_PASS -h$MYSQL_HOST -e "SHOW ENGINE INNODB STATUS\G" | grep -i "waiting"

# Verificar transacciones distribuidas
echo "=== Transacciones distribuidas ==="
mysql -u$MYSQL_USER -p$MYSQL_PASS -h$MYSQL_HOST -e "SELECT COUNT(*) FROM information_schema.innodb_trx;"

# Verificar espacio en los archivos de datos
echo "=== Espacio utilizado por los datafiles ==="
mysql -u$MYSQL_USER -p$MYSQL_PASS -h$MYSQL_HOST -e "SELECT TABLE_SCHEMA, SUM(data_length+index_length)/1024/1024 AS 'Tamaño MB' FROM information_schema.tables GROUP BY TABLE_SCHEMA;"

echo "Análisis completado."


#!/bin/bash

YEAR=$(date +%Y)
MES=$(date +%m)
DAY=$(date +%d)
FECHA="${DAY}${MES}${YEAR}"

DATABASES=("caja_naranjal" "caja_yeison" "caja_camilo" "caja_quilichao" "caja_huaquillas" 
           "caja_duran" "caja_villamil" "caja_girardota" "caja_bahia" "series" 
           "caja_sangil" "caja_sevilla" "caja_cerrito" "caja_zarzal" "caja_tuquerres" "caja_ipiales")

# Para seguridad del borrado accidental de archivos coloco a los nombres el sufijo _dbbk (database backup)
for db in "${DATABASES[@]}"; do
   echo "Respaldando Base de Datos $db ..."
   mysqldump -R -u respaldos -p"B1ng0JL**2022" "$db" > "${db}${FECHA}_dbbk.sql" && \
   7z a -sdel -mx5 "${db}${FECHA}_dbbk.zip" "${db}${FECHA}_dbbk.sql"
done

echo "Verificando respaldos en el directorio de mas de 2 dias para eliminar ..."
find . -name "*_dbbk.zip" -mtime +2 -exec echo "eliminando respaldo {} de hace 2 dias..." \; -exec rm -f {} \;
