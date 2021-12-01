-- wal-g & Postgres14
gcloud beta compute --project=celtic-house-266612 instances create walg --zone=us-central1-a --machine-type=e2-medium --subnet=default --network-tier=PREMIUM --maintenance-policy=MIGRATE --service-account=933982307116-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --image=ubuntu-2104-hirsute-v20211119 --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-ssd --boot-disk-device-name=postgres --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
gcloud compute ssh walg
sudo apt update && sudo apt-mark hold linux-image-5.11.0-1023-gcp && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-14
sudo pg_lsclusters

-- wal-g v1.1.1-rc
-- https://github.com/wal-g/wal-g
-- install

wget https://github.com/wal-g/wal-g/releases/download/v1.1.1-rc/wal-g-pg-ubuntu-20.04-amd64.tar.gz && tar -zxvf wal-g-pg-ubuntu-20.04-amd64.tar.gz && sudo mv wal-g-pg-ubuntu-20.04-amd64 /usr/local/bin/wal-g

-- sudo ls -l /usr/local/bin/wal-g

sudo mkdir /home/backups && sudo chown postgres:postgres /home/backups

-- create configuration file for wal-g
sudo su postgres
nano ~/.walg.json

-- https://github.com/wal-g/wal-g/blob/master/docs/PostgreSQL.md
-- https://github.com/wal-g/wal-g/blob/master/docs/STORAGES.md

{
    "WALG_FILE_PREFIX": "/home/backups",
    "WALG_COMPRESSION_METHOD": "brotli",
    "WALG_DELTA_MAX_STEPS": "5",
    "PGDATA": "/var/lib/postgresql/14/main",
    "PGHOST": "/var/run/postgresql/.s.PGSQL.5432"
}

mkdir /var/lib/postgresql/14/main/log

-- postgresql.conf
echo "wal_level=replica" >> /var/lib/postgresql/14/main/postgresql.auto.conf
echo "archive_mode=on" >> /var/lib/postgresql/14/main/postgresql.auto.conf
echo "archive_command='wal-g wal-push \"%p\" >> /var/lib/postgresql/14/main/log/archive_command.log 2>&1' " >> /var/lib/postgresql/14/main/postgresql.auto.conf 
echo "archive_timeout=60" >> /var/lib/postgresql/14/main/postgresql.auto.conf 
echo "restore_command='wal-g wal-fetch \"%f\" \"%p\" >> /var/lib/postgresql/14/main/log/restore_command.log 2>&1' " >> /var/lib/postgresql/14/main/postgresql.auto.conf

cat ~/14/main/postgresql.auto.conf

-- restart PostgreSQL
pg_ctlcluster 14 main stop
pg_ctlcluster 14 main start

-- create test db
psql -c "CREATE DATABASE testw;"

-- create test table
psql testw -c "create table test(i int);"
psql testw -c "insert into test values (10), (20), (30);"
psql testw -c "select * from test;"


-- make backup
wal-g backup-push /var/lib/postgresql/14/main

-- cat /var/log/postgresql/postgresql-14-main.log
-- cat /var/lib/postgresql/14/main/log/archive_command.log

psql testw -c "UPDATE test SET i = 3 WHERE i = 30"

-- make delta
wal-g backup-push /var/lib/postgresql/14/main

wal-g backup-list

-- restore 
pg_createcluster 14 main2
rm -rf /var/lib/postgresql/14/main2

wal-g backup-fetch /var/lib/postgresql/14/main2 LATEST


-- create signal file for recovery wal
touch "/var/lib/postgresql/14/main2/recovery.signal"
pg_ctlcluster 14 main2 start

psql -p 5433 testw -c "select * from test;"


-- GStore
rm $HOME/.walg.json
nano ~/.walg.json
{
    "WALG_GS_PREFIX": "gs://walgg",
    "GOOGLE_APPLICATION_CREDENTIALS" : "/var/lib/postgresql/celtic-house-266612-65d95e64c26a.json",
    "WALG_COMPRESSION_METHOD": "brotli",
    "WALG_DELTA_MAX_STEPS": "5",
    "PGDATA": "/var/lib/postgresql/14/main",
    "PGHOST": "/var/run/postgresql/.s.PGSQL.5432"
}
exit

gcloud compute instances list

scp /mnt/c/download/celtic-house-266612-65d95e64c26a.json aeugene@35.232.80.250:/home/aeugene/

sudo cp /home/aeugene/celtic-house-266612-65d95e64c26a.json /var/lib/postgresql/celtic-house-266612-65d95e64c26a.json
sudo chown postgres:postgres /var/lib/postgresql/celtic-house-266612-65d95e64c26a.json

sudo su postgres

wal-g backup-push /var/lib/postgresql/14/main

wal-g backup-list

gcloud compute instances delete walg