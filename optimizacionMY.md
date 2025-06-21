select  concat('ANALYZE table ', table_name, ';') from information_schema.`TABLES` where table_schema = 'caja_portoviejo'

SELECT 
    *
FROM 
    sys.statements_with_full_table_scans
		where db = 'caja_portoviejo'
ORDER BY 
    no_index_used_count DESC
LIMIT 10;


SELECT *
FROM 
    sys.statements_with_full_table_scans
WHERE 
    da_name  IN ('caja_portoviejo')
ORDER BY 
    no_index_used_count DESC
LIMIT 20;

SELECT 
    schema_name AS database_name,
    digest_text AS query_pattern,
    query_sample_text AS example_query,
    count_star AS executions,
    ROUND(sum_timer_wait/1000000000000, 2) AS total_latency_sec,
    ROUND(avg_timer_wait/1000000000000, 2) AS avg_latency_sec,
    sum_no_index_used AS full_scans_count,
    ROUND(sum_rows_examined/count_star) AS avg_rows_examined,
    first_seen,
    last_seen
FROM 
    performance_schema.events_statements_summary_by_digest
WHERE 
    sum_no_index_used > 0
    AND schema_name IN ('caja_portoviejo')
ORDER BY 
    sum_no_index_used DESC
LIMIT 10;


SHOW INDEX FROM cartones;
SHOW CREATE TABLE cartones;

ALTER TABLE cartones REBUILD INDEX UK_Cartones_idProgramacion;

SHOW TABLE STATUS LIKE 'cartones';

OPTIMIZE TABLE cartones;

EXPLAIN SELECT * FROM cartones WHERE idvendedor = 1;

SELECT 
    TABLE_NAME,
    INDEX_NAME,
    SEQ_IN_INDEX,
    COLUMN_NAME,
    CARDINALITY,
    INDEX_TYPE,
    COMMENT
FROM 
    INFORMATION_SCHEMA.STATISTICS
WHERE 
    TABLE_SCHEMA = 'caja_portoviejo'
    AND TABLE_NAME = 'cartones'
ORDER BY 
    INDEX_NAME, SEQ_IN_INDEX;

SELECT 
    table_name,
    index_name,
    round(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb,
    stat_description
FROM 
    mysql.innodb_index_stats
WHERE 
    database_name = 'caja_portoviejo'
    AND table_name = 'cartones'
    AND stat_name = 'size';
		
		
EXPLAIN SELECT * FROM cartones WHERE idvendedor = 30;


SELECT * /*
    index_name,
    rows_read,
    rows_changed,
    rows_changed_x_indexes*/
FROM 
    performance_schema.table_io_waits_summary_by_index_usage
WHERE 
    object_schema = 'caja_portoviejo'
    AND object_name = 'cartones';

SELECT 
    object_schema AS database_name,
    object_name AS table_name,
    index_name
FROM 
    performance_schema.table_io_waits_summary_by_index_usage
WHERE 
    index_name IS NOT NULL
    AND count_star = 0
    AND object_schema IN  ('caja_portoviejo')
ORDER BY 
    object_schema, object_name;

SELECT 
    t.TABLE_SCHEMA AS database_name,
    t.TABLE_NAME AS table_name,
    s.INDEX_NAME AS index_name,
    s.CARDINALITY
FROM 
    INFORMATION_SCHEMA.TABLES t
JOIN 
    INFORMATION_SCHEMA.STATISTICS s 
    ON t.TABLE_SCHEMA = s.TABLE_SCHEMA AND t.TABLE_NAME = s.TABLE_NAME
LEFT JOIN 
    performance_schema.table_io_waits_summary_by_index_usage u
    ON t.TABLE_SCHEMA = u.OBJECT_SCHEMA 
    AND t.TABLE_NAME = u.OBJECT_NAME 
    AND s.INDEX_NAME = u.INDEX_NAME
WHERE 
    t.TABLE_SCHEMA  IN ('caja_portoviejo')
    AND u.COUNT_STAR IS NULL OR u.COUNT_STAR = 0
ORDER BY 
    t.TABLE_SCHEMA, t.TABLE_NAME;


SELECT *
FROM 
    sys.schema_unused_indexes
		WHERE OBJECT_SCHEMA = 'caja_portoviejo'
		
		
		
		SELECT 
    object_schema,
    object_name,
    index_name,
    count_star
FROM 
    performance_schema.table_io_waits_summary_by_index_usage
WHERE 
    index_name IS NOT NULL
    AND count_star < 10  -- Ajusta este umbral según tu caso
		AND OBJECT_SCHEMA = 'caja_portoviejo'
ORDER BY 
    count_star ASC;


EXPLAIN select * from cartones where IdProgramacionJuego = 300
SHOW VARIABLES LIKE 'slow_query_log%';

SELECT * FROM mysql.slow_log ORDER BY query_time DESC LIMIT 10;


SELECT 
    digest_text AS query,
    count_star AS executions,
    ROUND(sum_timer_wait/1000000000000, 6) AS total_time_sec,
    ROUND(avg_timer_wait/1000000000000, 6) AS avg_time_sec,
    ROUND(max_timer_wait/1000000000000, 6) AS max_time_sec
FROM 
    performance_schema.events_statements_summary_by_digest
WHERE 
    digest_text NOT LIKE '%performance_schema%'
ORDER BY 
    avg_time_sec DESC
LIMIT 10;


SELECT * FROM sys.statement_analysis
where db = 'caja_portoviejo'
ORDER BY avg_latency DESC
LIMIT 30;

SELECT 
    query,
    db,
    exec_count,
    total_latency,
    avg_latency,
    rows_sent_avg,
    rows_examined_avg,
    last_seen
FROM 
    sys.x$statement_analysis
ORDER BY 
    avg_latency DESC
LIMIT 10;

select * from vendedoreshojas where
ANALYZE TABLE cartones;
OPTIMIZE TABLE nombre_de_la_tabla;

OPTIMIZE TABLE vendedoreshojas;

OPTIMIZE TABLE vendedores;


ALTER TABLE nombre_de_la_tabla REBUILD INDEX nombre_del_indice;

SELECT
    TABLE_NAME,
    INDEX_NAME,
    COLUMN_NAME,
    CARDINALITY
FROM
    information_schema.statistics
WHERE
    TABLE_SCHEMA = 'caja_portoviejo'
ORDER BY
    TABLE_NAME,
    INDEX_NAME,
    SEQ_IN_INDEX;


SELECT database_name, table_name, index_name, 
       ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb,
       stat_description
FROM mysql.innodb_index_stats
WHERE stat_name = 'size' AND database_name = 'caja_portoviejo';

SELECT table_schema, table_name, 
       data_length/1024/1024 AS data_length_mb,
       index_length/1024/1024 AS index_length_mb,
       data_free/1024/1024 AS data_free_mb
FROM information_schema.tables
WHERE table_schema = 'caja_portoviejo' AND data_free > 0;

SHOW VARIABLES LIKE 'innodb%';
SHOW STATUS LIKE 'Innodb%';

Ratio de acierto del buffer pool:

SELECT 
    (1 - (variable_value / (
        SELECT variable_value 
        FROM performance_schema.global_status 
        WHERE variable_name = 'Innodb_buffer_pool_read_requests'
    ))) * 100 AS 'Buffer Pool Hit Ratio (%)'
FROM performance_schema.global_status 
WHERE variable_name = 'Innodb_buffer_pool_reads';

SELECT 
    ROUND((1 - (SELECT variable_value 
               FROM performance_schema.global_status 
               WHERE variable_name = 'Innodb_buffer_pool_reads') / 
           (SELECT variable_value 
            FROM performance_schema.global_status 
            WHERE variable_name = 'Innodb_buffer_pool_read_requests')) * 100, 2) 
    AS 'Buffer Pool Hit Ratio (%)';
		
		
		
		SELECT 
    ROUND((1 - i_reads.i_reads / i_requests.i_requests) * 100, 2) AS 'Hit Ratio (%)',
    i_requests.i_requests AS 'Total Read Requests',
    i_reads.i_reads AS 'Physical Disk Reads',
    FORMAT(i_reads.i_reads / i_requests.i_requests * 100, 2) AS 'Miss Ratio (%)'
FROM 
    (SELECT variable_value AS i_requests 
     FROM performance_schema.global_status 
     WHERE variable_name = 'Innodb_buffer_pool_read_requests') AS i_requests,
    (SELECT variable_value AS i_reads 
     FROM performance_schema.global_status 
     WHERE variable_name = 'Innodb_buffer_pool_reads') AS i_reads;



SELECT * 
FROM mysql.innodb_index_stats
WHERE stat_name = 'size' AND database_name = 'caja_portoviejo';`


SHOW INDEX FROM cartones;
SHOW CREATE TABLE cartones;

ALTER TABLE cartones REBUILD INDEX UK_Cartones_idProgramacion;

UK_Cartones_idProgramacion
idx_cartones_IdVendedor
FK_Cartones_IdProgramacionJuego

SHOW TABLE STATUS LIKE 'cartones';

OPTIMIZE TABLE cartones;


EXPLAIN SELECT * FROM cartones WHERE idvendedor = 1;

SELECT 
    TABLE_NAME,
    INDEX_NAME,
    SEQ_IN_INDEX,
    COLUMN_NAME,
    CARDINALITY,
    INDEX_TYPE,
    COMMENT
FROM 
    INFORMATION_SCHEMA.STATISTICS
WHERE 
    TABLE_SCHEMA = 'caja_portoviejo'
    AND TABLE_NAME = 'cartones'
ORDER BY 
    INDEX_NAME, SEQ_IN_INDEX;

SELECT 
    table_name,
    index_name,
    round(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb,
    stat_description
FROM 
    mysql.innodb_index_stats
WHERE 
    database_name = 'caja_portoviejo'
    AND table_name = 'cartones'
    AND stat_name = 'size';
		
		
EXPLAIN SELECT * FROM cartones WHERE idvendedor = 30;


SELECT * /*
    index_name,
    rows_read,
    rows_changed,
    rows_changed_x_indexes*/
FROM 
    performance_schema.table_io_waits_summary_by_index_usage
WHERE 
    object_schema = 'caja_portoviejo'
    AND object_name = 'cartones';

SELECT 
    object_schema AS database_name,
    object_name AS table_name,
    index_name
FROM 
    performance_schema.table_io_waits_summary_by_index_usage
WHERE 
    index_name IS NOT NULL
    AND count_star = 0
    AND object_schema IN  ('caja_portoviejo')
ORDER BY 
    object_schema, object_name;

SELECT 
    t.TABLE_SCHEMA AS database_name,
    t.TABLE_NAME AS table_name,
    s.INDEX_NAME AS index_name,
    s.CARDINALITY
FROM 
    INFORMATION_SCHEMA.TABLES t
JOIN 
    INFORMATION_SCHEMA.STATISTICS s 
    ON t.TABLE_SCHEMA = s.TABLE_SCHEMA AND t.TABLE_NAME = s.TABLE_NAME
LEFT JOIN 
    performance_schema.table_io_waits_summary_by_index_usage u
    ON t.TABLE_SCHEMA = u.OBJECT_SCHEMA 
    AND t.TABLE_NAME = u.OBJECT_NAME 
    AND s.INDEX_NAME = u.INDEX_NAME
WHERE 
    t.TABLE_SCHEMA  IN ('caja_portoviejo')
    AND u.COUNT_STAR IS NULL OR u.COUNT_STAR = 0
ORDER BY 
    t.TABLE_SCHEMA, t.TABLE_NAME;


SELECT *
FROM 
    sys.schema_unused_indexes
		WHERE OBJECT_SCHEMA = 'caja_portoviejo'
		
		
		
		SELECT 
    object_schema,
    object_name,
    index_name,
    count_star
FROM 
    performance_schema.table_io_waits_summary_by_index_usage
WHERE 
    index_name IS NOT NULL
    AND count_star < 10  -- Ajusta este umbral según tu caso
		AND OBJECT_SCHEMA = 'caja_portoviejo'
ORDER BY 
    count_star ASC;


EXPLAIN select * from cartones where IdProgramacionJuego = 300
SHOW VARIABLES LIKE 'slow_query_log%';

SELECT * FROM mysql.slow_log ORDER BY query_time DESC LIMIT 10;


SELECT 
    digest_text AS query,
    count_star AS executions,
    ROUND(sum_timer_wait/1000000000000, 6) AS total_time_sec,
    ROUND(avg_timer_wait/1000000000000, 6) AS avg_time_sec,
    ROUND(max_timer_wait/1000000000000, 6) AS max_time_sec
FROM 
    performance_schema.events_statements_summary_by_digest
WHERE 
    digest_text NOT LIKE '%performance_schema%'
ORDER BY 
    avg_time_sec DESC
LIMIT 10;


SELECT `p` . `IdProgramacionJuego` , `p` . `CartonFinal` , `p` . `CartonInicial` , `p` . `CartonesAleatorios` , `p` . `CartonesEnJuego` , `p` . `CartonesPorHoja` , `p` . `CartonesPorLinea` , `p` . `CartonesPorPagina` , `p` . `CartonesPromocion` , `p` . `Estado` , `p` . `FechaCierre` , `p` . `FechaProgramada` , `p` . `HojaFinal` , `p` . `HojaInicial` , `p` . `IdJuego` , `p` . `IdSerie` , `p` . `IdVendedorCierre` , `p` . `PrevioCierre` , `p` . `ResultadoFinal` , `p` . `TipoJuego` , `p` . `TotalCartones` , `p` . `TotalModulos` , `p` . `TotalPremios` , `p` . `ValorCarton` , `p` . `ValorModulo` , `l` . `IdLisTablas` , `l` . `Defecto` , `l` . `Fabrica` , `l` . `Simbolo` , `l` . `SqlConsulta` , `l` . `Tabla` , `l` . `TipoJuego` , `l` . `Visible` , `c` . `IdCartonImpresion` , `c` . `IdJuego` , `c` . `IdProgramacionJuego` , `c` . `IdVendedor` , `c` . `Numero` , `c` . `NumeroHoja` , `c` . `NumeroLinea` , `c` . `NumeroPagina` , `c` . `P1` ,


SELECT `p` . `IdPerfil` FROM ` ... p` WHERE `p` . `IdUsuario` = ?

SELECT * FROM sys.statement_analysis
where db = 'caja_portoviejo'
ORDER BY avg_latency DESC
LIMIT 30;

SELECT 
    query,
    db,
    exec_count,
    total_latency,
    avg_latency,
    rows_sent_avg,
    rows_examined_avg,
    last_seen
FROM 
    sys.x$statement_analysis
ORDER BY 
    avg_latency DESC
LIMIT 10;

select * from vendedoreshojas where
ANALYZE TABLE cartones;
OPTIMIZE TABLE nombre_de_la_tabla;

OPTIMIZE TABLE vendedoreshojas;

OPTIMIZE TABLE vendedores;


ALTER TABLE nombre_de_la_tabla REBUILD INDEX nombre_del_indice;

SELECT
    TABLE_NAME,
    INDEX_NAME,
    COLUMN_NAME,
    CARDINALITY
FROM
    information_schema.statistics
WHERE
    TABLE_SCHEMA = 'caja_portoviejo'
ORDER BY
    TABLE_NAME,
    INDEX_NAME,
    SEQ_IN_INDEX;


SELECT database_name, table_name, index_name, 
       ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb,
       stat_description
FROM mysql.innodb_index_stats
WHERE stat_name = 'size' AND database_name = 'caja_portoviejo';

SELECT table_schema, table_name, 
       data_length/1024/1024 AS data_length_mb,
       index_length/1024/1024 AS index_length_mb,
       data_free/1024/1024 AS data_free_mb
FROM information_schema.tables
WHERE table_schema = 'caja_portoviejo' AND data_free > 0;

SHOW VARIABLES LIKE 'innodb%';
SHOW STATUS LIKE 'Innodb%';

Ratio de acierto del buffer pool:

SELECT 
    (1 - (variable_value / (
        SELECT variable_value 
        FROM performance_schema.global_status 
        WHERE variable_name = 'Innodb_buffer_pool_read_requests'
    ))) * 100 AS 'Buffer Pool Hit Ratio (%)'
FROM performance_schema.global_status 
WHERE variable_name = 'Innodb_buffer_pool_reads';

SELECT 
    ROUND((1 - (SELECT variable_value 
               FROM performance_schema.global_status 
               WHERE variable_name = 'Innodb_buffer_pool_reads') / 
           (SELECT variable_value 
            FROM performance_schema.global_status 
            WHERE variable_name = 'Innodb_buffer_pool_read_requests')) * 100, 2) 
    AS 'Buffer Pool Hit Ratio (%)';
		
		
		
		SELECT 
    ROUND((1 - i_reads.i_reads / i_requests.i_requests) * 100, 2) AS 'Hit Ratio (%)',
    i_requests.i_requests AS 'Total Read Requests',
    i_reads.i_reads AS 'Physical Disk Reads',
    FORMAT(i_reads.i_reads / i_requests.i_requests * 100, 2) AS 'Miss Ratio (%)'
FROM 
    (SELECT variable_value AS i_requests 
     FROM performance_schema.global_status 
     WHERE variable_name = 'Innodb_buffer_pool_read_requests') AS i_requests,
    (SELECT variable_value AS i_reads 
     FROM performance_schema.global_status 
     WHERE variable_name = 'Innodb_buffer_pool_reads') AS i_reads;



SELECT * 
FROM mysql.innodb_index_stats
WHERE stat_name = 'size' AND database_name = 'caja_portoviejo';
