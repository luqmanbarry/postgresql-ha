# Setting up HA PostgreSql

## Prepare master db server

1.  Setup directory structure for master db server

2. Standup master db server

`
    docker network create pgdb-ha-network --subnet "10.0.0.1/24"
    docker-compose -f docker-compose.master.yaml up --force-recreate
    docker-compose -f docker-compose.master.yaml down
    cp db/config/pg_hba.conf db/config/postgresql.conf db/master-data
    docker-compose -f docker-compose.master.yaml up --force-recreate
`
#### Note: 3,4,5 are execute if pb_hba.conf and postgresql.conf files do not exist

3. Prepare posgres.conf file 

`
    docker run -i --rm postgres cat /usr/share/postgresql/postgresql.conf.sample > ./postgres.conf
`

    Note: Use https://pgtune.leopard.in.ua for VM/DB tune up values

4. Add Values below to postres.conf file to enable replication

`
    # Replication
    wal_level = replica
    hot_standby = on
    max_wal_senders = 10
    max_replication_slots = 10
    hot_standby_feedback = on
`

5. Create pg_hba.conf file, use docs for more fine tuning

`
    Follow link: https://www.postgresql.org/docs/10/auth-pg-hba-conf.html
`



## Prepare the worker servers

1.  Create the replicator user on primary db -- postgresdb01

`
    docker exec -it postgresdb01 psql -U pgdbuser
    CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'pgdbpassword';
`

2. Create the physical replication slot on master for each container

`
    SELECT * FROM pg_create_physical_replication_slot('replication_slot_slave1');
`

3. Run below query to confirm physical replication has been created

`
    SELECT * FROM pg_replication_slots;
    \q; -- Quite Postgres shell
`

4. Restart master server for new configs to be picked up

`
    docker-compose -f docker-compose.master.yaml down
    docker-compose -f docker-compose.master.yaml up --force-recreate

`

4. Create master backup for slave restore

`
    docker exec -it postgresdb01 /bin/bash
    pg_basebackup -D /tmp/postgresslave1 -S replication_slot_slave1 -X stream -P -U replicator -Fp -R
    ls /tmp/ -- confirm folder has been created
    exit -- Exit container
`

5. Copy pg_basebackup output directories to host machine for data mounting into worker containers

`
    docker cp postgresdb01:/tmp/postgresslave1/ ./db/worker-data
    Repeat above command to the number of worker containers
`

## Stand up the worker container while mounting the ./db/worker-data directory
Note: This backup directory can be anywhere

1. Edit recovery.conf file from pg_basebackup dir(worker-data) to set correct master server host IP/DNS, set failover file as well

`
    In config key: primary_conninfo, edit value of host to point to master host(ie: host=postgresdb01)
    Example below:

    standby_mode = 'on'
    primary_conninfo = 'host=postgresdb01 user=replicator passfile=''/root/.pgpass'' port=5432 sslmode=prefer sslcompression=1 krbsrvname=postgres target_session_attrs=any'
    primary_slot_name = 'replication_slot_slave2'
    trigger_file = '/var/lib/postgresql/data/failover'
`

2. In db/worker-data directory, edit postgresql.conf file to enable hot_standby

`
    wal_level = hot_standby # allow queries while in read only recovery mode
	max_wal_senders = 5 # this sends the wal files, how many slaves can connect and pull docker
	hot_standby = on # allow queries while in read only recovery mode
`


3. Launch the workers docker-compose file, validate replications is activated

`
    docker-compose -f docker-compose.slaves.yaml up --force-recreate
    docker exec -it postgresdb02 /bin/bash
    psql -h postgresdb02 -p 5432 -U pgdbuser
    select * from pg_stat_replication;

`

4. Validate data replication on worker containers

    a. Create few tables in master server

`
    docker run --rm --name postgres-test --network=pgdb-ha-network -it postgres:10 /bin/bash
    psql -h postgresdb01 -p 5432 -U pgdbuser
    Enter password when prompted
    \l;    -- list databases
    CREATE DATABASE pghatest;
    \c pghatest; -- Switch to database
    CREATE TABLE todo (id SERIAL, name varchar(100) NOT NULL, description varchar(200),PRIMARY KEY(id));
    \dt; -- Show tables
    insert into todo(name, description) values('TODO #1', 'Describing TODO #1');
    select * from todo; -- Display content of todo table
    \q   -- Quit PostgreSql shell
`

    b. Connect to worker(R\O) containers to confirm replication is in full swing

`
    docker exec -it postgresdb02 /bin/bash
    psql -h postgresdb02 -p 5432 -U pgdbuser
    Enter password when prompted
    \l    -- list databases
    \c pghatest;   -- Switch to database
    \dt; -- Show tables
    select * from todo; -- Display content of todo table
    insert into todo(name, description) values('NO TODO #1', 'NO TODO #1 coz R/O Only');
    Since postgresdb02 is in R/O mode, insert statement will fail with: ERROR:  cannot execute INSERT in a read-only transaction
    \q   -- Quit PostgreSql shell
`

## Failover Steps

1. Validate recovery.conf file in data direcotry of vm/container taking over, if file absent create one with below content

`
    standby_mode = 'on'
    primary_conninfo = 'host=<FAILOVER_HOST> user=replicator passfile=''/root/.pgpass'' port=5432 sslmode=prefer sslcompression=1 krbsrvname=postgres target_session_attrs=any'
    primary_slot_name = 'replication_slot_slave1'
    trigger_file = '/var/lib/postgresql/data/failover'
`

2. Edit recovery.conf file for each worker vm to point to correct master host

3. Restart the VMs for which recovery.conf file was edited

4. Create failover file is it does not exist
Example: For setting up postgresdb02 as master, execute following;

`
    docker exec -it postgresdb02 /bin/bash
    touch '/var/lib/postgresql/data/failover'
`

4. Restart the VMs for which recovery.conf file was edited



