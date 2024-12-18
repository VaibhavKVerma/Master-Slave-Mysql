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
  -d mysql:latest
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
   SHOW BINARY LOG STATUS;
```
Copy the **FileName** and **FilePosition**.

### 2. Slave Database Instance
Run the following command to create a MySQL container for the **slave database**:
```bash
docker run --name slave \
  --network mysql-replication
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_ROOT_HOST=% \
  -p 3308:3306 \
  -d mysql:latest
```

```
   docker exec -it slave mysql -u root -p
```

Stop the slave
```
   STOP REPLICA;
```

Execute this:
```
   CHANGE REPLICATION SOURCE TO SOURCE_HOST='master-host',SOURCE_PORT=3307,SOURCE_USER='repl',SOURCE_PASSWORD='repl_password',SOURCE_LOG_FILE='FileName',SOURCE_LOG_POS=FilePosition;
```

```
   START REPLICA;
   SHOW REPLICA STATUS\G;
```


