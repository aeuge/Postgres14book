sudo su postgres
psql
CREATE DATABASE dvdrental;
cd $HOME
wget --quiet https://www.postgresqltutorial.com/wp-content/uploads/2019/05/dvdrental.zip
unzip dvdrental.zip
pg_restore -U postgres -d dvdrental dvdrental.tar
psql
\c dvdrental

SELECT count(*) FROM rental;
SELECT * FROM rental LIMIT 3;    

\timing
SELECT * FROM rental WHERE customer_id = 333 LIMIT 3;    

CREATE INDEX idx_rental_customer_id ON rental(customer_id);
SELECT * FROM rental WHERE customer_id = 333 LIMIT 3;    

EXPLAIN SELECT * FROM rental WHERE customer_id = 333 LIMIT 3;    
DROP INDEX idx_rental_customer_id;
EXPLAIN SELECT * FROM rental WHERE customer_id = 333 LIMIT 3;    

SELECT customer_id, count(customer_id) FROM rental GROUP BY customer_id ORDER BY 2 LIMIT 3;

SELECT * FROM rental WHERE customer_id = 318 LIMIT 3;    

CREATE TABLE test AS 
SELECT generate_series AS id
	, generate_series::text || (random() * 10)::text AS col2 
    , (array['Yes', 'No', 'Maybe'])[floor(random() * 3 + 1)] AS is_okay
FROM generate_series(1, 1000000);

SELECT * FROM test WHERE id = 1;

CREATE INDEX idx_test_id ON test(id);
SELECT * FROM test WHERE id = 1;

DROP INDEX idx_test_id;
CREATE INDEX idx_test_id_is_okay ON test(id, is_okay);

EXPLAIN SELECT * FROM test WHERE id = 1 AND is_okay = 'True';
EXPLAIN SELECT * FROM test WHERE is_okay = 'True' AND id = 1;
EXPLAIN SELECT * FROM test WHERE id = 1;
EXPLAIN SELECT * FROM test WHERE is_okay = 'True';


EXPLAIN SELECT * FROM test ORDER BY id, is_okay;
EXPLAIN SELECT * FROM test ORDER BY id DESC, is_okay DESC;
EXPLAIN SELECT * FROM test ORDER BY id, is_okay DESC;
EXPLAIN SELECT * FROM test ORDER BY is_okay;

-- functional
DROP INDEX idx_test_id_is_okay;
CREATE INDEX idx_test_id_is_okay ON test(lower(is_okay));

EXPLAIN SELECT * FROM test WHERE lower(is_okay) = 'true';
EXPLAIN SELECT * FROM test WHERE is_okay = 'True';

-- part
DROP INDEX idx_test_id_is_okay;
CREATE INDEX idx_test_id_is_okay ON test(id) WHERE id < 50;

EXPLAIN SELECT * FROM test WHERE id < 5;
EXPLAIN SELECT * FROM test WHERE id = 100;

-- include
DROP INDEX idx_test_id_is_okay;
CREATE INDEX idx_test_id_is_okay ON test(id) INCLUDE (is_okay);

EXPLAIN SELECT id, is_okay FROM test WHERE id = 5;
EXPLAIN SELECT id, col2 FROM test WHERE id = 100;

-- full text
SELECT to_tsvector('fat cats ate fat rats') @@ to_tsquery('fat & rat');

SELECT * FROM address LIMIT 10;

SELECT count(*)
FROM address
WHERE to_tsvector(address) @@ to_tsquery('Street');

SELECT count(*)
FROM address
WHERE address LIKE '%Street%';

CREATE INDEX idx_address_btree ON address(address);
CREATE INDEX idx_address_gin ON address USING GIN (to_tsvector('english', address));

SELECT count(*)
FROM address
WHERE to_tsvector('english', address) @@ to_tsquery('english','Street')


-- reindex
psql -c "create database index_test"
pgbench -i -s 10 index_test

psql 
\с index_test
\dt
CREATE EXTENSION pgstattuple;
\d+ pgbench_accounts
SELECT * FROM pgstatindex('pgbench_accounts_pkey') \gx
update pgbench_accounts set bid = bid + 1000000;
SELECT * FROM pgstatindex('pgbench_accounts_pkey') \gx
update pgbench_accounts set aid = aid + 1000000;
SELECT * FROM pgstatindex('pgbench_accounts_pkey') \gx
VACUUM pgbench_accounts;
SELECT * FROM pgstatindex('pgbench_accounts_pkey') \gx
REINDEX INDEX CONCURRENTLY pgbench_accounts_pkey;
SELECT * FROM pgstatindex('pgbench_accounts_pkey') \gx