# Replication with Docker MySQL Images
Setting up MySQL Replication S ⭢ R1 and S ⭢ R2 with Docker MySQL images

#### Replication enables data from one MySQL database server (the source) to be copied to one or more MySQL database servers (the replicas).

## Quick Start
Basic steps to spin up a minimal MySQL master-slave replication environment for experimentation:

### Prerequisites
- Docker installed and running

For detailed explanations, optional configuration, and troubleshooting, refer to the full article below.

## Full article
### [Setting up MySQL Replication (S ⭢ R1) and (S ⭢ R2) with Docker MySQL images](https://medium.com/@wagnerjfr/setting-up-mysql-replication-s-r1-and-s-r2-with-docker-mysql-images-80fdc06ed07f)

-------------------------

### Steps (Quick Start)
1. **Pull MySQL 5.7 image**
   ```bash
   docker pull mysql/mysql-server:5.7
   ```

2. **Create Docker network**
   ```bash
   docker network create replicanet
   ```

3. **Start containers (1 source + 2 replicas)**
   ```bash
   docker run -d --rm --name=source --net=replicanet --hostname=source \
     -e MYSQL_ROOT_PASSWORD=mypass \
     mysql/mysql-server:5.7 \
     --server-id=1 \
     --log-bin='mysql-bin-1.log'

   docker run -d --rm --name=replica1 --net=replicanet --hostname=replica1 \
     -e MYSQL_ROOT_PASSWORD=mypass \
     mysql/mysql-server:5.7 \
     --server-id=2

   docker run -d --rm --name=replica2 --net=replicanet --hostname=replica2 \
     -e MYSQL_ROOT_PASSWORD=mypass \
     mysql/mysql-server:5.7 \
     --server-id=3
   ```
   Wait for all containers to show `(healthy)` in `docker ps -a`.

4. **Configure source node**
   ```bash
   docker exec -it source mysql -uroot -pmypass \
     -e "CREATE USER 'repl'@'%' IDENTIFIED BY 'replicapass';" \
     -e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';" \
     -e "SHOW MASTER STATUS;"
   ```
   Note the `File` (e.g., `mysql-bin-1.000003`) and `Position` values from the output.

5. **Configure replicas**
   Replace `MASTER_LOG_FILE` with the `File` value from the previous step:
   ```bash
   for N in 1 2; do
     docker exec -it replica$N mysql -uroot -pmypass \
       -e "CHANGE MASTER TO MASTER_HOST='source', MASTER_USER='repl', \
         MASTER_PASSWORD='replicapass', MASTER_LOG_FILE='mysql-bin-1.000003';" \
       -e "START SLAVE;"
   done
   ```

6. **Verify replication**
   Create a test database on the source:
   ```bash
   docker exec -it source mysql -uroot -pmypass -e "CREATE DATABASE TEST;"
   ```
   Check replicas for the new database:
   ```bash
   for N in 1 2; do
     docker exec -it replica$N mysql -uroot -pmypass -e "SHOW DATABASES;"
   done
   ```

7. **Cleanup**
   ```bash
   docker stop source replica1 replica2
   docker network rm replicanet
   docker rmi mysql/mysql-server:5.7
   ```