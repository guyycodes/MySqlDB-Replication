# MySQL Master-Slave Replication Setup
## Step 1: Prepare the Master Server
### Configure the MySQL Master Server:

1. Edit the MySQL configuration file on the master server (typically located at `/etc/mysql/mysql.conf.d/mysqld.cnf` on Linux). 
or on mac: with homebrew try : 
```sudo nano /opt/homebrew/etc/my.cnf```
2. Make the following changes:
   - Ensure the master and slave point to the same master_log_file
   ```bash
   // get the master log file which the master points towards

   ```

   ```ini
   [mysqld]
   server-id = 1
   port = 3306
   log_bin = /usr/local/var/log/mysql/mysql-bin
   binlog_do_db = name_of_your_database  # Specify the database you want to replicate
   datadir = /opt/homebrew/var/mysql/  # This is where your MySQL data is stored
   pid-file = /opt/homebrew/var/mysql/Guys-MacBook-Pro.local.pid # This points to the file that stores the server's process ID

   # Only allow connections from localhost
   bind-address = 127.0.0.1
   mysqlx-bind-address = 127.0.0.1
   // for running 2 Database instance on the same machine, you will need to setup differing Unix sockets, pid-file, datadir, log_bin files for each instances and specify different ports.
   ```
   - To locate your files use these commands after youve logged into MySql via the command line: itll show where mysql is currently expecting to find its config files (these commands were tested for mac)
   ```sql
   mysql -u root -p  
   SHOW VARIABLES LIKE '%dir%';
   SHOW VARIABLES LIKE 'pid_file';
   SHOW VARIABLES LIKE 'log_bin%';
   SHOW VARIABLES LIKE 'socket';
   ```
   - if the log_bin doesnt exist create it and give it permissions: ex: for mac
   ```bash
   sudo mkdir -p /usr/local/var/log/mysql
   sudo chown -R _mysql:admin /usr/local/var/log/mysql
   // gives ownership to the MySQL user (_mysql)
   ```
   - `server-id` must be a unique number among all your MySQL servers.
   - `log_bin` enables binary logging, which is necessary for replication.

### Create a Replication User on the Master:

   1. Log into MySQL on the master:

      ```bash
      mysql -u root -p
      ```

   2. Run the following SQL commands to create a replication user, ive user the username 'replicator' as an example, you might want to use something relevant to your use case.

      ```sql
      // do this if you want to use SSL
      CREATE USER 'replicator'@'%' IDENTIFIED WITH caching_sha2_password BY 'password';
      GRANT REPLICATION SLAVE ON 'DB_NAME'.* TO 'replicator'@'%';
      FLUSH PRIVILEGES;

      // do this if your not using SSL and both db are on the same machine
      CREATE USER 'replicator'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';
      GRANT REPLICATION SLAVE ON 'DB_NAME'.* TO 'replicator'@'localhost';
      FLUSH PRIVILEGES;

      // use this to alter a configuration already created
      ALTER USER 'replicator'@'%' IDENTIFIED WITH mysql_native_password BY 'Colorado1!';
      FLUSH PRIVILEGES;
      ```
      Replace `'password'` with a strong password.

   3. Restart MySQL to apply these changes:

      ```java
      sudo systemctl restart mysql
      // or if using mac
      brew services restart mysql
      ```

## Step 2: Prepare the Slave Server
Similar to the master, ensure that the slave has all necessary directories created with appropriate permissions, especially for datadir, log_bin (if logging on the slave db), and the pid-file
### Configure the MySQL Slave Server:

1. Create the my2.cnf config file: 
   ```bash
   cp /opt/homebrew/etc/my.cnf /opt/homebrew/etc/my2.cnf
   ```
   - Remove the config for the master inside my2.cnf and replace with the slave confi as shown below.

2. Youll need to mkdir any new directories required by the salve server
   - ensure your directores have proper ownership (paths may differ for your setup):
   ```bash
   ls -l /opt/homebrew/var/mysql_slave/ # check permissions
   mkdir -p /opt/homebrew/var/mysql_slave/ # make the directory
   ```
3. Make the following changes:
   ``` bash
   // to appropriate correct ownership ex: sudo chown -R _mysql:admin /opt/homebrew/var/mysql_slave/
   sudo chown -R <user>:<group> /opt/homebrew/var/mysql_slave/
   sudo chmod -R 755 /opt/homebrew/var/mysql_slave/
   ```
4. Determine the current binary log the Master is pointing towards: 
   - See image: ExampleBinLogs.png
   - Notate the log file and the position

   ```sql
   mysql -u root -p
   SHOW BINARY LOGS;
   SHOW MASTER STATUS; 
   ```
   ```ini
   // slave config for my2.conf
   [mysqld]
   server-id = 2
   port = 3307 # must differ from other server-id
   log_bin = /usr/local/var/log/mysql/mysql2-bin # different from master
   relay-log = /usr/local/var/log/mysql/mysql2-relay-bin
   socket = /tmp/mysqld2.sock
   relay-log-index = /usr/local/var/log/mysql/mysql2-relay-bin.index
   read_only = 1
   datadir = /opt/homebrew/var/mysql_slave/
   pid-file = /opt/homebrew/var/mysql/Guys-MacBook-Pro-mysql2.local.pid
   expire_logs_days = 10
   replicate-do-db = STJDA  # Optional, only if replicating specific databases
   bind-address = 127.0.0.1  # Adjust based on security/network requirements
   mysqlx=0 
   ```

   5. Restart MySQL on the slave:

   ```bash
   sudo systemctl restart mysql
      // using mac & homebrew
   brew services restart mysql
   ```

   - `server-id` should be set to a unique number (different from the master).
   - `read_only = 1` makes the slave server read-only for all users except those with the SUPER privilege.

   3. Initialize the data directory on the slave after seeting up the my2.cnf (configuration for the slave db)

      ```bash
      mysqld --initialize --user=<user> --datadir=/opt/homebrew/var/mysql_slave/ 

      Example: mysqld --initialize --user=guymorganb --datadir=/opt/homebrew/var/mysql_slave/ 

      // make sure you notate the temporary password Example: "A temporary password is generated for root@localhost: aRVTgJwSg6>l"

      mysqld_safe --defaults-file=/opt/homebrew/etc/my2.cnf &
      // note: this will hang up the terminal if it succeds and youll need a second termianl to point the slave db to the master db
      ////////////////////////////////////////////////////////////////////
      // if you need to clear the my_slave dir and start over:
      rm -rf /opt/homebrew/var/mysql_slave/*
      ```

   4. Log into the Slave server:
   ```bash
      mysql -u root -p --port=3307 --socket=/tmp/mysqld2.sock
      // itll ask you for that temporary password from the previous step
      // youll need to reset your password to interact with the salve db
      ALTER USER 'root'@'localhost' IDENTIFIED BY 'your_new_password';
   ```

   5. Check your socket connection & process ID to ensure your running a separate instance of MySQL from the master
   ```sql
   SHOW VARIABLES LIKE 'socket'
   SHOW VARIABLES LIKE 'pid_file';
   ```
   - you can also manuall check the process id's and the ports for the master and slave to ensure they are different using this command:
   ```bash
   ps aux | grep mysqld
   ```

### Configure Replication on the Slave: i.e Point the Slave db towards the Master db.

1. Log into MySQL on the slave and lock the tables to prevent any changes (ensure no writes are happening during this period, while you note the log position, if it’s a production environment) : This will output the current binary log file and position. Record these values (File and Position)

   ```bash
   mysql -u root -p

   FLUSH TABLES WITH READ LOCK;
   SHOW MASTER STATUS;
   ```

2. Run the following commands to point the slave to the master:

   ```sql
   CHANGE MASTER TO
   MASTER_HOST='127.0.0.1',
   MASTER_USER='replicator',
   MASTER_PASSWORD='<password>',
   MASTER_LOG_FILE='mysql-bin.000011',
   MASTER_LOG_POS=613;
   START SLAVE;
   ```

   Replace `'master_ip_address'`, `'replicator_password'`, `'recorded_log_file_name'`, and `recorded_log_position` with the actual values. You can get the log file name and position from the master by examining the master's binary log.

### Verify Slave Status:

1. Check the slave's replication status:

After starting the MySQL slave instance, check that the PID file is created and contains the correct process ID:
   
   ```bash
   // start the slave server if you havent already
   mysqld_safe --defaults-file=/path/to/my2.cnf &

   // your path may be different
   cat /opt/homebrew/var/mysql/<pathToYour.pid>.pid
   ```
   ```sql
   SHOW SLAVE STATUS\G
   ```
2. Ensure that `Slave_IO_Running` and `Slave_SQL_Running` both say "Yes".

## Step 3: Creating a snapshot of aan existing database to copy to the slave
If you already had data in your master database before setting up replication, the data won't automatically replicate to your slave database.

1. **On the master, flush all tables and block write statements by running:**

```sql
FLUSH TABLES WITH READ LOCK;
```
2. **While keeping the session running, in another terminal window, dump your database:**
The `--master-data` flag automatically appends the change master to command to the output. 
```bash
mysqldump -u root -p --databases <dbName> --master-data > <dbName>dbdump.db
```

3. **Then unlock tables (this is in the window from step 1):**

```sql
UNLOCK TABLES;
```
You can exit the MySQL shell.

4. **Transfer your `dbdump.db` file to your slave machine.** You can use the `scp` command or any preferred method.

5. **On the slave machine, import the database dump: Ive provided an example for running 2 db's on the same machine**

```bash
// first stop the replica IO_THREAD, log ingto SQL command line of the Slave db and use:
STOP SLAVE IO_THREAD;

// then...exit MySQL command line and go back to your devices command line and use:
mysql -u root -p --port=3307 --socket=/tmp/mysqld2.sock < /path/to/your/<dbName>dbdump.db

// finally start up the thread again: 
START SLAVE IO_THREAD;
```

6. **Then check the slave status again from MYSQL command line:**

```sql
SHOW SLAVE STATUS\G;
SHOW DATABASES;
USE <DBNAME>;
SHOW TABLES;

// Final Check
SHOW SLAVE STATUS\G

// if the process listed below arent running try:
START SLAVE SQL_THREAD;
``` 
- Slave_IO_Running: Should be Yes if the IO thread is running (reading the binary log from the master).
- Slave_SQL_Running: Should be Yes if the SQL thread is running (executing the replicated SQL statements).
- Last_Error: Shows the last error registered by the replication process, if any.
- Seconds_Behind_Master: Indicates how far behind the slave is from the master in terms of replication lag.
