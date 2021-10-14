-- physycal level
gcloud compute instances list

gcloud compute ssh postgres14
pg_lsclusters

-- add another cluster
pg_createcluster 14 main2
sudo pg_createcluster 14 main2
sudo nano /etc/postgresql/14/main/postgresql.conf

sudo su postgres
cd /var/lib/postgresql/14/main
ls -la
ls -l base
# - command in psql
# SELECT oid, datname,dattablespace FROM pg_database;
# CREATE DATABASE book;
# SELECT * FROM pg_tablespace;

-- make a catalog for new tablespace
exit
sudo su
sudo mkdir /home/postgres
sudo chown postgres /home/postgres
sudo su postgres
cd /home/postgres
mkdir tmptblspc

psql
# CREATE tablespace ts location '/home/postgres/tmptblspc';
-- list of tablespaces
# \db
# CREATE DATABASE app TABLESPACE ts;
# \c app
-- look for default tablespace
# \l+ 

exit
cd tmptblspc/
ls
cd PG_14_202107181/
ls -la

cd 16385
ls -la

psql
# \c app
# CREATE TABLE test (i int);
# CREATE TABLE test2 (i int) TABLESPACE pg_default;
# SELECT tablename, tablespace FROM pg_tables WHERE schemaname = 'public';
-- change tablespace for TABLE
# ALTER TABLE test SET TABLESPACE pg_default;
# SELECT oid, spcname FROM pg_tablespace; -- oid унимальный номер, по кторому можем найти файлы
# SELECT oid, datname,dattablespace FROM pg_database;

-- where is table
# SELECT pg_relation_filepath('test2');

-- SHOW size of database
# SELECT pg_database_size('app');

-- pretty size
# SELECT pg_size_pretty(pg_database_size('app'));

-- full size of TABLE
# SELECT pg_size_pretty(pg_total_relation_size('test2'));

-- only size of data
# SELECT pg_size_pretty(pg_TABLE_size('test2'));

-- ... only indexes
# SELECT pg_size_pretty(pg_indexes_size('test2'));

-- size of visability map fro TABLE test2
# SELECT pg_size_pretty(pg_relation_size('test2','vm'));

-- size of tablespace
# SELECT pg_size_pretty(pg_tablespace_size('ts'));

-- try broke ts
exit
cd /home/postgres
mkdir tmptblspc2
psql
# CREATE tablespace ts2 location '/home/postgres/tmptblspc2';

# \c app
# ALTER TABLE test SET TABLESPACE ts2;

-- drop directory with new ts
exit
rm -rf tmptblspc2
psql
\c app
\dt
SELECT * FROM test;

exit
rm -rf tmptblspc
psql
\c app

DROP DATABASE app;


-- open access
-- try access
psql -h 34.134.57.202 -U postgres -W

netstat -a | grep postgresql

-- uncomment listen_addresses = '*'
sudo nano /etc/postgresql/14/main/postgresql.conf

-- host    all             all             0.0.0.0/0               md5
sudo nano /etc/postgresql/14/main/pg_hba.conf

-- change password
ALTER USER postgres PASSWORD 'Postgres123#';

-- restart server
sudo pg_ctlcluster 14 main restart

-- try access
psql -h 34.134.57.202 -U postgres -W


-- pg_updatecluster
sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-13

pg_lsclusters

pg_upgradecluster 13 main

help pg_upgradecluster

sudo pg_upgradecluster 13 main upgrade13

sudo pg_renamecluster 13 main main13

pg_lsclusters

sudo -u postgres psql -p 5434

CREATE ROLE testpass PASSWORD 'testpass' LOGIN;

sudo -u postgres psql -p 5434 -U testpass -h localhost -d postgres -W

sudo pg_upgradecluster 13 main13

pg_lsclusters

sudo pg_dropcluster 13 main13

sudo cat /etc/postgresql/14/main13/pg_hba.conf

sudo nano /etc/postgresql/14/main13/pg_hba.conf

sudo pg_ctlcluster 14 main13 restart
-- error
sudo -u postgres psql -p 5434 -U testpass -h localhost -d postgres -W


sudo -u postgres psql -p 5434
ALTER USER testpass PASSWORD 'testpass';
exit

sudo -u postgres psql -p 5434 -U testpass -h localhost -d postgres -W

-- change back to md5

sudo nano /etc/postgresql/14/main13/pg_hba.conf

sudo pg_ctlcluster 14 main13 restart
sudo -u postgres psql -p 5434 -U testpass -h localhost -d postgres -W


sudo pg_ctlcluster 14 main13 stop
sudo pg_dropcluster 14 main13