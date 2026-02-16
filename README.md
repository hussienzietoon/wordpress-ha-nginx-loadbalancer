## 🔒 Security Group Configuration for Instances

- **HTTP (Port 80):** Allow inbound traffic from the load balancer's security group.
- **SSH (Port 22):** Allow inbound traffic from your IP address for management purposes.
- **MySQL (Port 3306):** Allow inbound traffic from the database server's security group.

---

# WordPress High Availability Setup with Nginx Load Balancer

This setup deploys two WordPress instances behind an Nginx load balancer on Ubuntu servers, using PHP-FPM and a shared MariaDB database.

---

## 🛠️ 1. Configure the Database Server

1. **Connect to the Database Server**:

   ```bash
   ssh -i aws_key.pem ubuntu@13.51.55.145
   ```

2. **Install MariaDB Server**:

   ```bash
   sudo apt update && sudo apt upgrade -y
   sudo apt install mariadb-server -y
   ```

3. **Create WordPress Database and User**:

   ```sql
   CREATE USER 'wp_user'@'172.31.30.105' IDENTIFIED BY 'wp_pass'; -- WP1
   GRANT ALL PRIVILEGES ON wp1_db.* TO 'wp_user'@'172.31.30.105';

   CREATE USER 'wp_user'@'172.31.21.147' IDENTIFIED BY 'wp_pass'; -- WP2
   GRANT ALL PRIVILEGES ON wp2_db.* TO 'wp_user'@'172.31.21.147';

   FLUSH PRIVILEGES;
   ```

4. **Edit MariaDB config to allow external connections**:

   - File: `/etc/mysql/mariadb.conf.d/50-server.cnf`
   - Change `bind-address` to:
     ```
     bind-address = 0.0.0.0
     ```

5. **Restart MariaDB**:
   ```bash
   sudo systemctl restart mariadb
   ```

---

## 🖥️ 2. Configure wp1 and wp2 Instances

1. **Connect to the WordPress Instance**:

   ```bash
   ssh -i aws_key.pem ubuntu@16.16.28.209
   ```

2. **Install necessary packages**:

   ```bash
   sudo apt update
   sudo apt install nginx -y
   sudo systemctl start nginx
   sudo systemctl enable nginx

   sudo apt install php-fpm php-mysql php-curl php-gd php-xml php-mbstring php-zip php-soap -y
   ```

3. **Download and setup WordPress**:

   ```bash
   cd /tmp
   curl -O https://wordpress.org/latest.tar.gz
   tar -xvzf latest.tar.gz
   sudo mkdir -p /var/www/html/wp1
   sudo cp -R wordpress/* /var/www/html/wp1/

   sudo chown -R www-data:www-data /var/www/html/wp1
   sudo chmod -R 755 /var/www/html/wp1

   cd /var/www/html/wp1
   sudo cp wp-config-sample.php wp-config.php
   sudo nano wp-config.php
   # Edit the following lines:
   define('DB_NAME', 'wp1_db');
   define('DB_USER', 'wp_user');
   define('DB_PASSWORD', 'wp_pass');
   define('DB_HOST', '172.31.25.176');
   ```

4. **Nginx Configuration (`/etc/nginx/sites-available/wp1` or `wp2`)**:

   ```nginx
   server {
       listen 80;
       server_name 51.21.129.181 _;

       root /var/www/html/wp1;
       index index.php index.html;

       location / {
           try_files $uri $uri/ /index.php?$args;
       }

       location ~ \.php$ {
           include snippets/fastcgi-php.conf;
           fastcgi_pass unix:/run/php/php8.3-fpm.sock;
       }

       location ~ /\.ht {
           deny all;
       }
   }
   ```

5. **Enable and restart Nginx**:

   ```bash
   sudo ln -s /etc/nginx/sites-available/wp1 /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl reload nginx
   ```

6. **Test Connection from WP1 to DB**:

To test the connection from WP1 to the database, follow these steps:

1. **Open a terminal on WP1**: Connect to your WP1 instance using SSH.

2. **Run the MySQL command**: Use the following command to connect to the database server. Replace `172.31.25.176` with the actual IP address of your database server if it's different.

   ```bash
   mysql -h 172.31.25.176 -u wp_user -p
   ```

   - `-h` specifies the host (IP address of the database server).
   - `-u` specifies the username (`wp_user` in this case).
   - `-p` prompts for the password. Enter the password for `wp_user` when prompted.

3. **Select the database**: Once connected, select the `wp1_db` database by running:

   ```sql
   USE wp1_db;
   ```

This will allow you to interact with the `wp1_db` database. If you encounter any connection issues, ensure that the database server's security group allows inbound traffic on port 3306 from the WP1 instance.

---

---

## ⚖️ 3. Configure the Load Balancer

1. **Install Nginx**:

   ```bash
   sudo apt update && sudo apt install nginx -y
   ```

2. **Create Load Balancer Config (`/etc/nginx/sites-available/loadbalancer`)**:

   ```nginx
   upstream wordpress_backend {
       server 172.31.30.105;  # wp1
       server 172.31.21.147;  # wp2
   }

   server {
       listen 80 default_server;
       server_name  13.60.70.137 _;

       location / {
           proxy_pass http://wordpress_backend;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       }
   }
   ```

3. **Enable and restart Nginx**:
   ```bash
   sudo ln -s /etc/nginx/sites-available/loadbalancer /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl reload nginx
   ```

---

## 🧪 4. Test the Setup

- Access `http://13.60.70.137`
- You should see the WordPress install screen.
- Set up once, and you'll be load-balanced between both backends.

---

## ✅ Done!

## You now have a fully working WordPress High Availability setup using Nginx.

## ⚠️ Troubleshooting

### Issue: Default Nginx Configuration Interference

**Problem**: After setting up the Nginx load balancer and WordPress instances, you might encounter issues where the default Nginx configuration file (`/etc/nginx/sites-enabled/default` and `/etc/nginx/sites-available/default`) interferes with your custom configurations.

**Solution**:

1. **Remove the Default Configuration**:

   - Delete the default configuration file from both `sites-enabled` and `sites-available` directories:
     ```bash
     sudo rm /etc/nginx/sites-enabled/default
     sudo rm /etc/nginx/sites-available/default
     ```

2. **Restart Nginx**:
   - After removing the default files, restart Nginx to apply the changes:
     ```bash
     sudo systemctl restart nginx
     ```

**Note**: Always ensure that your custom configuration files are correctly set up and enabled before removing the default configuration to avoid downtime.

---
