# Roadmap 
#### Download MySQL server for your device:
* https://dev.mysql.com/downloads/mysql/

- Im ssh'ed into a linux server, so Ill demonstrate those commands
```bash
// update apt
sudo apt update
sudo apt upgrade

// Instal MySQL
sudo apt install mysql-server

// Run the security script that comes with MySQL. 
// This script will allow you to remove some insecure default settings and lock down access to your database system.
sudo mysql_secure_installation
```

- Allowing Remote Connections: We wil lallow remote connections to enable MySql Workbench connections from other devices
```bash
// Edit the MySQL Configuration File:
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
// Find the line that contains bind-address and change it to:
bind-address = 0.0.0.0

// Restart MySQL
sudo systemctl restart mysql.service

// Create a MySQL User for Remote Access
// Log into MySQL : Enter the root password you set up during the mysql_secure_installation process.
sudo mysql -u root -p


// Create a New User:
CREATE USER 'remote_user'@'%' IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;

// Configure the Firewall : or other appliance if you have one
sudo ufw allow 3306
sudo ufw reload
```
- Create a New Connection in MySQL Workbench:
* Open MySQL Workbench.
* Click the "+" button to add a new connection.
* Fill in the connection settings:
* Connection Name: Give your connection a name.
* Connection Method: Select "Standard TCP/IP".
* Hostname: Enter the IP address of your remote.
* Port: 3306 (default).
* Username: root or any other user you have configured.
* Click "Store in Vault" to enter and save your password.

- Test and Use the Connection:

* Click "Test Connection" to ensure everything is configured correctly.

Here's the markdown version of the response:

## Securing Remote MySQL Access

After enabling remote access to your MySQL server, it's important to secure it properly. Here are a couple of additional steps you can take:

### 1. Restrict User Access
Currently, `remote_user` can connect from any host. This is convenient but can pose a security risk. Consider restricting access to specific IP addresses or networks where possible:

```sql
ALTER USER 'remote_user'@'%' IDENTIFIED BY 'password' REQUIRE SSL;
```

Then change `'%'` to specific IP addresses or hostnames:

```sql
CREATE USER 'remote_user'@'192.168.1.5' IDENTIFIED BY 'Colorado1!';
GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'192.168.1.5' WITH GRANT OPTION;
```

### 2. Enforce Stronger Authentication and Encryption
If your MySQL server version supports it, consider requiring SSL for remote connections to ensure data is encrypted in transit:

```sql
ALTER USER 'remote_user'@'%' REQUIRE SSL;
```

You may need to set up SSL certificates on your MySQL server for this to work.
