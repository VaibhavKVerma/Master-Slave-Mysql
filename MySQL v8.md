# Master-Slave MySQL

## Prerequisites

1. **Download MySQL and MySQL Workbench**  
   [MySQL Downloads](https://dev.mysql.com/downloads/)  

2. **Default Local Instance**  
   MySQL automatically provides a local instance running at PORT `3306`, so we will not use this port.

3. **Install Docker**  
   [Docker Downloads](https://www.docker.com/products/docker-desktop)

4. **Download MySQL Docker Image**  
   Get the MySQL image from [Docker Hub](https://hub.docker.com/_/mysql).

---

## Setup Instructions

```
docker network create mysql-replication
```

### 1. Master Database Instance
Run the following command to create a MySQL container for the **master database**:
```bash
docker run --name master \
  --network mysql-replication \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_ROOT_HOST=% \
  -p 3307:3306 \
  -d mysql:8.0
```
Log into master container using:
```bash
docker exec -it master mysql -u root -p
```
Run these commands
```
CREATE USER 'repl'@'%' IDENTIFIED BY 'repl_password';
```
1. Creates a new user named `repl` with the password `repl_password`.
2. `repl` : The username for the replication user.
3. `%` : The host part, which means the user can connect from any host (wildcard).If you'd like to restrict access to a specific IP or hostname (e.g., the slave server's IP), you can replace % with that IP or hostname.
4. `IDENTIFIED BY` : Sets the password for the user to repl_password.
```
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
```
1. `REPLICATION SLAVE` : This privilege allows the user to connect to the master server and read binary log events. The slave uses this data to replicate changes from the master.
2. `*.*`: The privilege applies to all databases and tables on the server. If you'd like to restrict the user to specific databases, you can replace `*.*` with the database name (e.g., db_name.*).
```
FLUSH PRIVILEGES;
```
1. Reloads the privilege tables in MySQL to ensure the changes made with GRANT or CREATE USER take effect immediately.


```
docker ps -a
```

Copy the **my.cnf** to local
```
docker cp master_container_id:/etc/my.cnf ./Downloads
```

Edit the file - The server-id parameter in MySQL configuration is essential for identifying individual servers in a replication setup. In the context of MySQL Master-Slave replication, each server must have a unique server-id to ensure proper communication and data synchronization between them. Master database often has server-id is 1
```
log_bin=mysql-bin
server-id=1
```

Copy the file back to docker
```
docker cp ./mysql/master/my.cnf master_container_id:/etc
```

Also to get master IP Address

```
docker inspect master_container_id
```

Copy **IP Address**

Restart master container
```
docker restart master
```

Log into master container using:
```bash
docker exec -it master mysql -u root -p
```

```
SHOW MASTER STATUS;
```

Copy the **FileName** and **FilePosition**.

### 2. Slave Database Instance
Run the following command to create a MySQL container for the **slave database**:
```bash
docker run --name slave \
  --network mysql-replication \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_ROOT_HOST=% \
  -p 3308:3306 \
  -d mysql:8.0
```

```
docker ps -a
```

Copy the **my.cnf** to local
```
docker cp slave_container_id:/etc/my.cnf ./Downloads
```

Edit the file - The server-id parameter in MySQL configuration is essential for identifying individual servers in a replication setup. In the context of MySQL Master-Slave replication, each server must have a unique server-id to ensure proper communication and data synchronization between them. Master database often has server-id is 1
```
log_bin=mysql-bin
server-id=2
```

Copy the file back to docker
```
docker cp ./mysql/master/my.cnf slave_container_id:/etc
```

```
docker restart slave
docker exec -it slave mysql -u root -p
```

Stop the slave

For version 8.x
```
STOP SLAVE;
```

Execute this:
```
CHANGE MASTER TO MASTER_HOST='172.18.0.2',MASTER_PORT=3306,MASTER_USER='root',MASTER_PASSWORD='password',MASTER_LOG_FILE='FileName',MASTER_LOG_POS=FilePosition,GET_MASTER_PUBLIC_KEY=1;
```

```
START SLAVE;
SHOW SLAVE STATUS\G;
```
