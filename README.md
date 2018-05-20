# Replication with Docker MySQL Images
Setting up MySQL Replication M->S1 and M->S2 with Docker MySQL images

#### Replication enables data from one MySQL database server (the master) to be copied to one or more MySQL database servers (the slaves).

## References
https://dev.mysql.com/doc/refman/8.0/en/replication.html

## Prerequisites

1. Docker installed
2. Do not run while connected to VPN

## Overview

We start by creating a Docker network named **replicanet**, then we are going to pull **mysql 8** from Docker Hub (https://hub.docker.com/r/mysql/mysql-server/) and create a replication topology with 3 nodes (1 master and 2 slaves) in different hosts.

## Pull MySQL Sever Image

To download the MySQL Community Edition image, the command is:
```console
$ docker pull mysql/mysql-server:tag
```
If :tag is omitted, the latest tag is used, and the image for the latest GA version of MySQL Server is downloaded.

Examples:
```
docker pull mysql/mysql-server
docker pull mysql/mysql-server:5.7
docker pull mysql/mysql-server:8.0
```
In this example, we are going to use ***mysql/mysql-server:8.0***

## Creating a Docker network
```console
$ docker network create replicanet
```
Just need to create it once, unless you remove it from docker network.

To see all Docker Network:
```console
$ docker network ls
```
## Creating 3 MySQL 8 containers

Run the commands below in a terminal.
```console
$ docker run -d --rm --name=master --net=replicanet --hostname=master \
  -v $PWD/d0:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mypass \
  mysql/mysql-server:5.7 \
  --server-id=1 \
  --log-bin='mysql-bin-1.log'

$ docker run -d --rm --name=slave1 --net=replicanet --hostname=slave1 \
  -v $PWD/d1:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mypass \
  mysql/mysql-server:5.7 \
  --server-id=2

$ docker run -d --rm --name=slave2 --net=replicanet --hostname=slave2 \
  -v $PWD/d2:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mypass \
  mysql/mysql-server:5.7 \
  --server-id=3
```
It's possible to see whether the containers are started by running:
```console
$ docker ps -a
```
```console
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS                            PORTS                 NAMES
b2b855652b3b        mysql/mysql-server:5.7   "/entrypoint.sh --se…"   3 seconds ago       Up 3 seconds (health: starting)   3306/tcp, 33060/tcp   slave2
8a10c0c92350        mysql/mysql-server:5.7   "/entrypoint.sh --se…"   7 seconds ago       Up 5 seconds (health: starting)   3306/tcp, 33060/tcp   slave1
8f8ceffd4580        mysql/mysql-server:5.7   "/entrypoint.sh --se…"   7 seconds ago       Up 7 seconds (health: starting)   3306/tcp, 33060/tcp   master
```
Servers are still with status **(health: starting)**, wait till they are with state **(healthy)** before running the following commands.
```console
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS                    PORTS                 NAMES
b2b855652b3b        mysql/mysql-server:5.7   "/entrypoint.sh --se…"   30 seconds ago      Up 30 seconds (healthy)   3306/tcp, 33060/tcp   slave2
8a10c0c92350        mysql/mysql-server:5.7   "/entrypoint.sh --se…"   34 seconds ago      Up 32 seconds (healthy)   3306/tcp, 33060/tcp   slave1
8f8ceffd4580        mysql/mysql-server:5.7   "/entrypoint.sh --se…"   34 seconds ago      Up 34 seconds (healthy)   3306/tcp, 33060/tcp   master
```

Now we’re ready start our instances and configure replication.

Let's configure in **master node** the replication user and get the initial replication co-ordinates

```console
$ docker exec -it master mysql -uroot -pmypass \
  -e "CREATE USER 'repl'@'%' IDENTIFIED BY 'slavepass';" \
  -e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';" \
  -e "SHOW MASTER STATUS;"
```
Output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+----------+--------------+------------------+-------------------+
| File               | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+--------------------+----------+--------------+------------------+-------------------+
| mysql-bin-1.000003 |      595 |              |                  |                   |
+--------------------+----------+--------------+------------------+-------------------+
```

Let’s continue with the slave instance. Change (if different) the replication co-ordinates captured in the previous step:
**MASTER_LOG_FILE='mysql-bin-1.000003'**, **MASTER_LOG_POS=595**, before running the below command.

```
for N in 1 2
  do docker exec -it slave$N mysql -uroot -pmypass \
    -e "CHANGE MASTER TO MASTER_HOST='master', MASTER_USER='repl', \
      MASTER_PASSWORD='slavepass', MASTER_LOG_FILE='mysql-bin-1.000003', MASTER_LOG_POS=595;"
  docker exec -it slave$N mysql -uroot -pmypass -e"START SLAVE;" -e"SHOW SLAVE STATUS\G"
done
```
Slave1 output:
```console
*************************** 1. row ***************************
               Slave_IO_State: Checking master version
                  Master_Host: master
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin-1.000003
          Read_Master_Log_Pos: 595
               Relay_Log_File: slave1-relay-bin.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: mysql-bin-1.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
                             ...
```
Slave2 output:
```console
*************************** 1. row ***************************
               Slave_IO_State: Checking master version
                  Master_Host: master
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin-1.000003
          Read_Master_Log_Pos: 595
               Relay_Log_File: slave2-relay-bin.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: mysql-bin-1.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
                             ...
```

Now it's time to test whether data is replicated to slaves.
We are going to create a new database named "TEST" in master.
```console
$ docker exec -it master mysql -uroot -pmypass -e"CREATE DATABASE TEST; USE TEST; SHOW DATABASES;"
```
Output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| TEST               |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

Run the code below to check whether the database was replicated.
```
for N in 1 2
  do docker exec -it slave$N mysql -uroot -pmypass \
  -e "SHOW VARIABLES WHERE Variable_name = 'hostname';" \
  -e "SHOW DATABASES;"
done
```
Output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| hostname      | slave1 |
+---------------+--------+
+--------------------+
| Database           |
+--------------------+
| information_schema |
| TEST               |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| hostname      | slave2 |
+---------------+--------+
+--------------------+
| Database           |
+--------------------+
| information_schema |
| TEST               |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

## Stopping containers, removing created network and image

#### Stopping running container(s):
```console
$ docker stop master slave1 slave2
```
#### Removing the data directories created (they are located in the folder were the containers were run):
```console
$ sudo rm -rf d0 d1 d2
```
#### Removing the created network:
```console
$ docker network rm group1
```
#### Remove the MySQL 8 image:
```console
$ docker rmi mysql/mysql-server:8.0
```
