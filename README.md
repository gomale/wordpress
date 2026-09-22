<a id="top"></a>

# WordPress on Azure VM

WordPress is free. Azure VM hosting may cost money for the VM, disk, public IP, and bandwidth unless covered by your subscription’s free allowance or credits. Check the estimated cost before creating it.

For a small website, use Ubuntu 24.04 LTS + Apache + MariaDB + PHP on one Azure VM. The instructions below assume a fresh VM.

## Table of Contents

- [Deployment Steps](#deployment-steps)
- [Troubleshooting Guide](#troubleshooting-guide)

## Deployment Steps

### 1. Create the Azure VM

| Setting | Suggested value |
|---|---|
| Resource group | `rg-wordpress` |
| VM name | `vm-wordpress` |
| Image | Ubuntu Server 24.04 LTS |
| Size | At least 2 GB RAM for a small site |
| Authentication | SSH public key |
| Username | `azureuser` |
| OS disk | Standard SSD |
| Public IP | Standard, static |

Under the VM’s Networking → Network settings, configure these inbound NSG rules:

| Port | Source | Purpose |
|---|---|---|
| TCP 22 | Your public IP only | SSH administration |
| TCP 80 | Internet | HTTP and certificate validation |
| TCP 443 | Internet | HTTPS |

Keep database port 3306 closed to the Internet.

### 2. Connct from Ubuntu WSL

Replace the key path and public IP:

```bash
chmod 600 ~/.ssh/wordpress-vm.pem

ssh -i ~/.ssh/wordpress-vm.pem azureuser@YOUR_VM_PUBLIC_IP
```

Run all remaining Linux commands inside the Azure VM.

### 3. Install Apache, PHP, and MariaDB

Ubuntu 24.04 provides PHP 8.3 and MariaDB 10.11, matching WordPress’s recommended baseline.

```bash
sudo apt update
sudo apt upgrade -y

sudo apt install -y \
  apache2 mariadb-server \
  php libapache2-mod-php php-mysql php-curl php-gd \
  php-mbstring php-xml php-zip php-intl php-imagick \
  curl ca-certificates unzip

sudo systemctl enable --now apache2 mariadb

php -v
mariadb --version
```

Check the VM firewall:

```bash
sudo ufw status
```

If UFW is active, allow SSH first, followed by web traffic:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Apache Full'
```

### 4. Create the WordPress database

Open MariaDB with interactive command history disabled:

```sh
sudo env MYSQL_HISTFILE=/dev/null mariadb
```

Run the following SQL. Replace REPLACE_WITH_A_LONG_RANDOM_PASSWORD with a unique password; save it for the WordPress installer.

```sh
CREATE DATABASE wordpress
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE USER 'wpuser'@'localhost'
  IDENTIFIED BY 'REPLACE_WITH_A_LONG_RANDOM_PASSWORD';

GRANT ALL PRIVILEGES ON wordpress.*
  TO 'wpuser'@'localhost';

EXIT;
```

### 5. Download WordPress

```sh
curl -fL https://wordpress.org/latest.tar.gz \
  -o /tmp/wordpress.tar.gz

sudo tar -xzf /tmp/wordpress.tar.gz -C /var/www/

sudo chown -R www-data:www-data /var/www/wordpress

sudo find /var/www/wordpress -type d -exec chmod 755 {} \;
sudo find /var/www/wordpress -type f -exec chmod 644 {} \;
```

This ownership lets WordPress install updates, plugins, and themes through its dashboard. The installation follows the standard Apache approach described by Ubuntu.

### 6. Configure Apache

Use your actual domain in place of example.com:

```bash
sudo tee /etc/apache2/sites-available/wordpress.conf > /dev/null <<'EOF'
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/wordpress

    <Directory /var/www/wordpress>
        Options FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/wordpress-error.log
    CustomLog ${APACHE_LOG_DIR}/wordpress-access.log combined
</VirtualHost>
EOF

sudo a2enmod rewrite
sudo a2ensite wordpress
sudo a2dissite 000-default

sudo apache2ctl configtest
```

If the result is Syntax OK, reload Apache:

```bash
sudo systemctl reload apache2
```

### 7. Point your domain to the VM and enable free HTTPS

At your domain's DNS provider, create:

| Type | Name | Value |
|---|---|---|
| A | `@` | Your VM’s public IP |

Once the domain resolves to your VM, install Certbot and request a free Let’s Encrypt certificate:

```bash
sudo apt install -y certbot python3-certbot-apache

sudo certbot --apache -d example.com --redirect

sudo certbot renew --dry-run
```

Replace example.com with the same domain used in Apache. Port 80 must remain reachable for HTTP certificate validation. Certbot configures Apache for HTTPS and supports automatic renewal.

Without a domain, you can initially check the installation at http://YOUR_VM_PUBLIC_IP; set up HTTPS before using it for a public site or entering sensitive credentials.

### 8. Complete the WordPress installer

Open:

```bash
https://example.com
```

Enter:

| Field | Value |
|---|---|
| Database name | `wordpress` |
| Database username | `wpuser` |
| Database password | The password created earlier |
| Database host | `localhost` |
| Table prefix | Keep the default |

Then choose your site title, administrator username, strong password, and email address.

After installation, restrict access to the configuration file:

```bash
sudo chmod 640 /var/www/wordpress/wp-config.php
```

Your administration dashboard is:

```bash
https://example.com/wp-admin
```

Keep Ubuntu, WordPress, plugins, and themes updated, and schedule backups of both the MariaDB database and /var/www/wordpress, stored outside the VM.

[↑ Back to Top](#top)

## Troubleshooting Guide

### Error establishing a database connection

Reset the MariaDB password for wpuser, then use that same password in WordPress. This error can also occur if MariaDB is stopped.

Run these commands on your Azure VM.

1. Check MariaDB

```bash
sudo systemctl status mariadb --no-pager
```

If it is inactive:

```bash
sudo systemctl start mariadb
```

2. Reset the database user's password

Open MariaDB with command history disabled:

```bash
sudo env MYSQL_HISTFILE=/dev/null mariadb
```

Run the following, replacing YOUR_NEW_PASSWORD with a strong, unique password. For easy copying into SQL and PHP, avoid single quotes and backslashes.

```bash
ALTER USER 'wpuser'@'localhost'
  IDENTIFIED BY 'YOUR_NEW_PASSWORD';

GRANT ALL PRIVILEGES ON wordpress.*
  TO 'wpuser'@'localhost';

EXIT;
```

ALTER USER changes the existing account’s password without deleting your database.

3. Verify the new password

```bash
mariadb -u wpuser -p -h localhost wordpress \
  -e "SELECT DATABASE(), CURRENT_USER();"
  ```

Enter the new password at the prompt. Do not put it directly in the command.

A successful result should show wordpress and wpuser@localhost.

4. Enter the matching credentials in WordPress

Your link indicates you are still in the database setup wizard. Click Try Again and enter:

| Field | Value |
|---|---|
| Database name | `wordpress` |
| Username | `wpuser` |
| Password | Your new password |
| Database host | `localhost` |
| Table prefix | Leave unchanged |

If you already have a wp-config.php file, edit it instead:

```bash
sudo nano /var/www/wordpress/wp-config.php
```

Update the existing entries—do not add duplicates:

```bash
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'YOUR_NEW_PASSWORD' );
define( 'DB_HOST', 'localhost' );
```

These settings must match the database credentials. 

Reload the website; no Apache restart is needed.

[↑ Back to Top](#top)
