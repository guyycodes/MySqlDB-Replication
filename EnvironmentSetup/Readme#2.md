# MySQL Master-Slave Replication Setup

## Step 1: Prepare the Master Server

### Configure the MySQL Master Server:

1. Edit the MySQL configuration file on the master server (typically located at `/etc/mysql/mysql.conf.d/mysqld.cnf` on Linux).
2. Make the following changes:

   ```ini
   [mysqld]
   server-id = 1
   log_bin = /var/log/mysql/mysql-bin.log
   binlog_do_db = name_of_your_database  # Specify the database you want to replicate
   ```

   - `server-id` must be a unique number among all your MySQL servers.
   - `log_bin` enables binary logging, which is necessary for replication.

3. Restart MySQL to apply these changes:

   ```bash
   sudo systemctl restart mysql
   ```

### Create a Replication User on the Master:

1. Log into MySQL on the master server:

   ```bash
   mysql -u root -p
   ```

2. Run the following SQL commands to create a replication user:

   ```sql
   CREATE USER 'replicator'@'%' IDENTIFIED BY 'replicator_password';
   GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
   FLUSH PRIVILEGES;
   ```

   Replace `'replicator_password'` with a strong password.

## Step 2: Prepare the Slave Server

### Configure the MySQL Slave Server:

1. Edit the MySQL configuration file on the slave server.
2. Make the following changes:

   ```ini
   [mysqld]
   server-id = 2
   relay_log = /var/log/mysql/mysql-relay-bin.log
   read_only = 1
   ```

   - `server-id` should be set to a unique number (different from the master).
   - `read_only = 1` makes the slave server read-only for all users except those with the SUPER privilege.

3. Restart MySQL on the slave:

   ```bash
   sudo systemctl restart mysql
   ```

### Configure Replication on the Slave:

1. Log into MySQL on the slave:

   ```bash
   mysql -u root -p
   ```

2. Run the following commands to point the slave to the master:

   ```sql
   CHANGE MASTER TO
   MASTER_HOST='master_ip_address',
   MASTER_USER='replicator',
   MASTER_PASSWORD='replicator_password',
   MASTER_LOG_FILE='recorded_log_file_name',
   MASTER_LOG_POS=recorded_log_position;
   START SLAVE;
   ```

   Replace `'master_ip_address'`, `'replicator_password'`, `'recorded_log_file_name'`, and `recorded_log_position` with the actual values. You can get the log file name and position from the master by examining the master's binary log.

### Verify Slave Status:

1. Check the slave's replication status:

   ```sql
   SHOW SLAVE STATUS\G
   ```

2. Ensure that `Slave_IO_Running` and `Slave_SQL_Running` both say "Yes".