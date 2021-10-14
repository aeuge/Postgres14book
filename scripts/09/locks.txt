-- locks

SELECT * FROM pg_locks \gx
SELECT pg_backend_pid();
SHOW log_lock_waits;

CREATE DATABASE locks;
\c locks
-- увидим, что pid изменился
SELECT pg_backend_pid();
CREATE TABLE t (i int);

-- VIEW is also RELATION
BEGIN;
SELECT * FROM pg_locks;
CREATE VIEW v AS SELECT * FROM t;
SELECT * FROM pg_locks;
SELECT relname FROM pg_class WHERE oid = 16385;
ROLLBACK;

-- advisory_lock
-- 1 terminal
BEGIN; 
SELECT hashtext('block me');
SELECT pg_advisory_lock(hashtext('block me'));
SELECT pg_backend_pid();

SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks WHERE pid = 6413;

-- 2 terminal
SELECT pg_advisory_lock(hashtext('block me'));

-- 1 terminal
COMMIT; 

-- commit does not release the lock
SELECT pg_advisory_unlock(hashtext('block me'));
-- has it`s own PID



-- 1 terminal
CREATE TABLE accounts(id integer, amount numeric);
INSERT INTO accounts VALUES (1,2000.00), (2,2000.00), (3,2000.00);

-- 2 terminal
\c locks
begin;
SELECT pg_backend_pid();
SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks WHERE pid = 2165;
UPDATE accounts SET amount = amount + 1 WHERE id = 1;

-- look at the locks

-- 1 terminal
CREATE INDEX ON accounts(id);

-- What happens?

-- 2 terminal
SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks WHERE pid = 2165;
SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks;

-- who is blocking us
SELECT pg_blocking_pids(1883); 

-- PG_STAT_ACTIVITY
SELECT * FROM pg_stat_activity WHERE pid = ANY(pg_blocking_pids(1883)) \gx
commit;



-- going to the deep
CREATE EXTENSION pageinspect;
CREATE VIEW accounts_v AS
SELECT '(0,'||lp||')' AS ctid,
       t_xmax as xmax,
       CASE WHEN (t_infomask & 1024) > 0  THEN 't' END AS commited,
       CASE WHEN (t_infomask & 2048) > 0  THEN 't' END AS aborted,
       CASE WHEN (t_infomask & 128) > 0   THEN 't' END AS lock_only,
       CASE WHEN (t_infomask & 4096) > 0  THEN 't' END AS is_multi,
       CASE WHEN (t_infomask2 & 8192) > 0 THEN 't' END AS keys_upd
FROM heap_page_items(get_raw_page('accounts',0))
WHERE lp <= 10
ORDER BY lp;

SELECT * FROM accounts;
SELECT * FROM accounts_v ;
truncate accounts;
INSERT INTO accounts VALUES (1,2000.00), (2,2000.00), (3,2000.00);

BEGIN;
UPDATE accounts set id = 4 WHERE id = 1;
UPDATE accounts set amount = 4000 WHERE id = 3;
SELECT * FROM accounts_v ;


-- 2 terminal
BEGIN;
SELECT * FROM accounts WHERE id = 2 FOR KEY SHARE;
SELECT * FROM accounts WHERE id = 3 FOR KEY SHARE;
SELECT * FROM accounts_v;
CREATE EXTENSION pgrowlocks;
SELECT * FROM pgrowlocks('accounts') \gx

-- Waiting queue
CREATE VIEW locks_v AS
SELECT pid,
       locktype,
       CASE locktype
         WHEN 'relation' THEN relation::REGCLASS::text
         WHEN 'virtualxid' THEN virtualxid::text
         WHEN 'transactionid' THEN transactionid::text
         WHEN 'tuple' THEN relation::REGCLASS::text||':'||tuple::text
       END AS lockid,
       mode,
       granted
FROM pg_locks;


SELECT * FROM locks_v;


-- let's see the locks
SELECT pid, backend_type, wait_event_type, wait_event FROM pg_stat_activity;

-- we usually get a lock, but in this mode we get an error
SELECT * FROM accounts FOR UPDATE NOWAIT; 

-- for multi-threaded data processing
DECLARE c CURSOR FOR SELECT * FROM accounts ORDER BY id FOR UPDATE SKIP LOCKED;
FETCH C; 


