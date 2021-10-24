-- Write Ahead Log
SHOW fsync;
SHOW wal_sync_method;
SET fsync = on;

SELECT pg_current_wal_insert_lsn();

-- buffer cache size
SELECT setting, unit FROM pg_settings WHERE name = 'shared_buffers'; 

-- decrease size of buffer pool
ALTER SYSTEM SET shared_buffers = 200;

-- restart after changes
sudo pg_ctlcluster 14 main restart


--
CREATE DATABASE buffer_temp;
\c buffer_temp
CREATE TABLE test(i int);

-- generate values
INSERT INTO test SELECT s.id FROM generate_series(1,100) AS s(id); 
SELECT * FROM test limit 10;

-- extension for view buffer pool
CREATE EXTENSION pg_buffercache; 

CREATE VIEW pg_buffercache_v AS
SELECT bufferid,
       (SELECT c.relname FROM pg_class c WHERE  pg_relation_filenode(c.oid) = b.relfilenode ) relname,
       CASE relforknumber
         WHEN 0 THEN 'main'
         WHEN 1 THEN 'fsm'
         WHEN 2 THEN 'vm'
       END relfork,
       relblocknumber,
       isdirty,
       usagecount
FROM   pg_buffercache b
WHERE  b.reldatabase IN (    0, (SELECT oid FROM pg_database WHERE datname = current_database()) )
AND    b.usagecount IS NOT NULL;

SELECT * FROM test limit 10;
SELECT * FROM pg_buffercache_v WHERE relname='test';
UPDATE test set i = 2 WHERE i = 1;

-- see dirty line
SELECT * FROM pg_buffercache_v WHERE relname='test';


-- warm pages usagecount > 3
SELECT c.relname,
  count(*) blocks,
  round( 100.0 * 8192 * count(*) / pg_table_size(c.oid) ) "% of rel",
  round( 100.0 * 8192 * count(*) FILTER (WHERE b.usagecount > 3) / pg_table_size(c.oid) ) "% hot"
FROM pg_buffercache b
  JOIN pg_class c ON pg_relation_filenode(c.oid) = b.relfilenode
WHERE  b.reldatabase IN (
         0, (SELECT oid FROM pg_database WHERE datname = current_database())
       )
AND    b.usagecount is not null
GROUP BY c.relname, c.oid
ORDER BY 2 DESC
LIMIT 10;

-- generate values with text for bigger size of pages
CREATE TABLE test_text(t text);
INSERT INTO test_text SELECT 'строка '||s.id FROM generate_series(1,500) AS s(id); 
SELECT * FROM test_text limit 10;
SELECT * FROM test_text;
SELECT * FROM pg_buffercache_v WHERE relname='test_text';


-- prewarm pages
-- restart cluster for clear buffer pool
sudo pg_ctlcluster 14 main restart

sudo -u postgres psql
\c buffer_temp
SELECT * FROM pg_buffercache_v WHERE relname='test_text';
CREATE EXTENSION pg_prewarm;
SELECT pg_prewarm('test_text');
SELECT * FROM pg_buffercache_v WHERE relname='test_text';



-------------WAL-----------------
\c buffer_temp
SELECT * FROM pg_ls_waldir() LIMIT 10;
CREATE EXTENSION pageinspect;

BEGIN;
-- current lsn position
SELECT pg_current_wal_insert_lsn();
-- what wal file we have
SELECT pg_walfile_name('0/182BCA8');

-- after UPDATE lsn changes
SELECT lsn FROM page_header(get_raw_page('test_text',0));
commit;
UPDATE test_text set t = '1' WHERE t = 'строка 1';
SELECT pg_current_wal_insert_lsn();
SELECT lsn FROM page_header(get_raw_page('test_text',0));
SELECT '0/182E0D8'::pg_lsn - '0/17E0E80'::pg_lsn;

-- look on wal file
/usr/lib/postgresql/14/bin/pg_waldump -p /var/lib/postgresql/14/main/pg_wal -s 0/17E0E80 -e 0/182E0D8 000000010000000000000001



---Checkpoint----
-- look cluster info
/usr/lib/postgresql/14/bin/pg_controldata /var/lib/postgresql/14/main/
SELECT pg_current_wal_insert_lsn();
CHECKPOINT;
SELECT pg_current_wal_insert_lsn();
/usr/lib/postgresql/14/bin/pg_waldump -p /var/lib/postgresql/14/main/pg_wal -s 0/179E6F8 -e 0/179E7E0 000000010000000000000001

-- let`s stop cluster emergency
\c buffer_temp
INSERT INTO test_text values('error');

sudo pg_ctlcluster 14 main stop -m immediate

sudo pg_ctlcluster 14 main start

sudo -u postgres psql

\c buffer_temp
SELECT * FROM test_text WHERE t = 'error';


--  bgwriter statistics
SELECT * FROM pg_stat_bgwriter \gx



-- let`s try tests in syncronous & asynchronous modes
pgbench -i buffer_temp
pgbench -P 1 -T 10 buffer_temp

ALTER SYSTEM SET synchronous_commit = off;

pgbench -P 1 -T 10 buffer_temp

SELECT pg_reload_conf(); 
sudo pg_ctlcluster 14 main reload

