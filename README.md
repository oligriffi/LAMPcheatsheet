# LAMP Stack & Linux Server Configuration — Reference Guide

---

## Task 01 — Network Configuration

### 1. Configure static IP with Netplan

Edit the Netplan config file:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Write this YAML (indentation must be exact — Netplan is strict):

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.1.50/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

Apply it:

```bash
sudo netplan apply
```

> Your NIC might be named `ens33` or `enp0s3` rather than `eth0` — check with `ip link show` first.

### 2. Configure UFW firewall

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
sudo ufw status verbose
```

### 3. Verify connectivity

```bash
ping -c 4 192.168.1.1          # gateway
ping -c 4 8.8.8.8              # internet
nslookup example.com           # DNS resolution
curl -I http://example.com     # HTTP reachability
```

---

## Task 02 — SSH Passwordless Auth

### 1. Generate key pair on your local machine (not the server)

```bash
ssh-keygen -t ed25519 -C "my-server-key"
```

Accept the default path (`~/.ssh/id_ed25519`). Add a passphrase if you want.

### 2. Copy public key to the server

```bash
ssh-copy-id user@192.168.1.50
```

Or manually:

```bash
cat ~/.ssh/id_ed25519.pub | ssh user@192.168.1.50 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### 3. Test key-based login first — critical before the next step

```bash
ssh user@192.168.1.50
```

Confirm you get in without a password prompt.

### 4. Disable password auth on the server

```bash
sudo nano /etc/ssh/sshd_config
```

Set these values:

```
PasswordAuthentication no
PermitRootLogin no
```

```bash
sudo systemctl reload sshd
```

### 5. Verify in a new terminal session

Try the following — it should reject you:

```bash
ssh -o PasswordAuthentication=yes user@192.168.1.50
```

> Always confirm key-based login works in a live session before saving `sshd_config`. One wrong move locks you out.

---

## Task 03 — Apache2 Website Setup

### 1. Install and enable Apache

```bash
sudo apt update && sudo apt install apache2 -y
sudo systemctl enable apache2
sudo systemctl status apache2
```

### 2. Create the document root and a basic page

```bash
sudo mkdir -p /var/www/devtest
echo "<h1>devtest.local is working</h1>" | sudo tee /var/www/devtest/index.html
```

### 3. Create the virtual host config

```bash
sudo nano /etc/apache2/sites-available/devtest.conf
```

```apache
<VirtualHost *:80>
    ServerName devtest.local
    DocumentRoot /var/www/devtest

    <Directory /var/www/devtest>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/devtest_error.log
    CustomLog ${APACHE_LOG_DIR}/devtest_access.log combined
</VirtualHost>
```

### 4. Enable the site and reload

```bash
sudo a2ensite devtest.conf
sudo a2dissite 000-default.conf
sudo systemctl reload apache2
```

**What `a2ensite` does under the hood:** it creates a symlink from `/etc/apache2/sites-enabled/devtest.conf` pointing to your file in `sites-available/`. Apache only loads configs from `sites-enabled`, so this is how you activate and deactivate sites without deleting the config.

### 5. Test

Add `devtest.local` to `/etc/hosts` if no DNS is configured:

```bash
echo "127.0.0.1 devtest.local" | sudo tee -a /etc/hosts
curl http://devtest.local
```

---

## Task 04 — PHP Installation

PHP can be integrated with Apache in two ways. Understanding the difference matters for production decisions.

---

### Option A — Apache module (`mod_php`)

This is the simpler approach. PHP runs as part of the Apache process itself.

```bash
sudo apt install php libapache2-mod-php php-mysql -y
```

Apache automatically enables `mod_php`. Verify it is active:

```bash
apache2ctl -M | grep php
```

Create a test file to confirm PHP is processing:

```bash
echo "<?php phpinfo(); ?>" | sudo tee /var/www/devtest/info.php
curl http://devtest.local/info.php
```

**How it works:** Apache handles the request and passes PHP files directly to the embedded PHP interpreter within the same process. Simple to set up but every Apache worker carries the PHP overhead even for static files.

**Common additional modules:**

```bash
sudo apt install php-curl php-mbstring php-xml php-zip -y
sudo systemctl restart apache2
```

---

### Option B — PHP-FPM (FastCGI Process Manager)

PHP-FPM runs as a separate service. Apache proxies PHP requests to it over a socket. This is the recommended approach for production — it gives you better performance, process isolation, and the ability to run multiple PHP versions.

#### Install PHP-FPM

```bash
sudo apt install php-fpm php-mysql -y
```

Check the version installed (you'll need this for the socket path):

```bash
php -v
```

The FPM socket will be at something like `/run/php/php8.1-fpm.sock` — substitute your version number throughout.

#### Enable required Apache modules

```bash
sudo a2enmod proxy_fcgi setenvif
sudo a2enconf php8.1-fpm        # substitute your version
sudo systemctl restart apache2
```

#### Update the virtual host to use FPM

```bash
sudo nano /etc/apache2/sites-available/devtest.conf
```

```apache
<VirtualHost *:80>
    ServerName devtest.local
    DocumentRoot /var/www/devtest

    <Directory /var/www/devtest>
        AllowOverride All
        Require all granted
    </Directory>

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.1-fpm.sock|fcgi://localhost"
    </FilesMatch>

    ErrorLog ${APACHE_LOG_DIR}/devtest_error.log
    CustomLog ${APACHE_LOG_DIR}/devtest_access.log combined
</VirtualHost>
```

```bash
sudo systemctl reload apache2
sudo systemctl enable php8.1-fpm
sudo systemctl start php8.1-fpm
```

Verify FPM is running:

```bash
sudo systemctl status php8.1-fpm
```

Test with the same `info.php` file:

```bash
curl http://devtest.local/info.php
```

---

### mod_php vs PHP-FPM — key differences

| | mod_php | PHP-FPM |
|---|---|---|
| PHP runs as | Part of Apache process | Separate service |
| Performance | Lower — every worker loads PHP | Higher — workers are dedicated |
| Process isolation | No | Yes |
| Multiple PHP versions | No | Yes (one FPM pool per version) |
| Recommended for | Dev/simple setups | Production |
| Config file | `php.ini` | `php.ini` + `/etc/php/x.x/fpm/pool.d/www.conf` |

> After installing PHP either way, remove `info.php` from production servers — it exposes version and config information.

```bash
sudo rm /var/www/devtest/info.php
```

---

## Task 05 — Self-Signed SSL

### 1. Generate the certificate

```bash
sudo mkdir -p /etc/ssl/devtest
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/devtest/devtest.key \
  -out /etc/ssl/devtest/devtest.crt
```

Fill in the prompts — for `Common Name` use `devtest.local`.

### 2. Enable SSL and rewrite modules

```bash
sudo a2enmod ssl
sudo a2enmod rewrite
```

### 3. Update the virtual host config

```bash
sudo nano /etc/apache2/sites-available/devtest.conf
```

Replace the contents with:

```apache
<VirtualHost *:80>
    ServerName devtest.local
    RewriteEngine On
    RewriteRule ^(.*)$ https://%{HTTP_HOST}$1 [R=301,L]
</VirtualHost>

<VirtualHost *:443>
    ServerName devtest.local
    DocumentRoot /var/www/devtest

    SSLEngine on
    SSLCertificateFile    /etc/ssl/devtest/devtest.crt
    SSLCertificateKeyFile /etc/ssl/devtest/devtest.key

    <Directory /var/www/devtest>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

```bash
sudo systemctl reload apache2
curl -k https://devtest.local
```

**Why `-k`?** It tells curl to skip certificate verification. Because the cert is self-signed, no trusted CA has vouched for it, so without `-k` curl refuses the connection. In production this is unacceptable — skipping verification removes protection against MITM attacks. Use a cert from Let's Encrypt or a trusted internal CA instead.

---

## Task 06 — MariaDB

### 1. Install and secure

```bash
sudo apt install mariadb-server -y
sudo systemctl enable mariadb
sudo mysql_secure_installation
```

At the prompts: set a root password → remove anonymous users → disallow root remote login → remove test database → reload privileges. Answer yes to all.

### 2. Create the database and user

```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE devapp;
CREATE USER 'devuser'@'localhost' IDENTIFIED BY 'StrongPass123!';
GRANT ALL PRIVILEGES ON devapp.* TO 'devuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 3. Create the table

```bash
mysql -u devuser -p devapp
```

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 4. Insert records and query

```sql
INSERT INTO users (username, email) VALUES
    ('alice', 'alice@example.com'),
    ('bob', 'bob@example.com'),
    ('charlie', 'charlie@example.com');

SELECT * FROM users ORDER BY created_at DESC;
```

> `FLUSH PRIVILEGES` is required after manual `GRANT` statements to reload the in-memory privilege tables.

---

## Task 07 — Display Database on the Website (index.php)

Now that Apache, PHP, and MariaDB are all running, replace the placeholder `index.html` with a PHP page that connects to the `devapp` database and renders the `users` table in the browser.

### 1. Remove the old index.html

```bash
sudo rm /var/www/devtest/index.html
```

### 2. Create index.php

```bash
sudo nano /var/www/devtest/index.php
```

Paste the following:

```php
<?php

// --- Database credentials ---
$host   = '127.0.0.1';
$dbname = 'devapp';
$user   = 'devuser';
$pass   = 'StrongPass123!';

// --- Connect via PDO ---
try {
    $pdo = new PDO(
        "mysql:host=$host;dbname=$dbname;charset=utf8",
        $user,
        $pass,
        [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
    );
} catch (PDOException $e) {
    die('<p style="color:red;">Connection failed: ' . htmlspecialchars($e->getMessage()) . '</p>');
}

// --- Fetch all users ordered by newest first ---
$stmt = $pdo->query('SELECT id, username, email, created_at FROM users ORDER BY created_at DESC');
$users = $stmt->fetchAll(PDO::FETCH_ASSOC);

?>
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Users — devapp</title>
    <style>
        body {
            font-family: sans-serif;
            max-width: 800px;
            margin: 40px auto;
            padding: 0 1rem;
            background: #f9f9f9;
            color: #222;
        }
        h1 { margin-bottom: 1rem; }
        table {
            width: 100%;
            border-collapse: collapse;
            background: #fff;
            box-shadow: 0 1px 4px rgba(0,0,0,0.1);
        }
        th, td {
            text-align: left;
            padding: 10px 14px;
            border-bottom: 1px solid #e0e0e0;
            font-size: 14px;
        }
        th {
            background: #333;
            color: #fff;
            font-weight: 500;
        }
        tr:last-child td { border-bottom: none; }
        tr:hover td { background: #f1f1f1; }
        .count { margin-top: 0.75rem; font-size: 13px; color: #666; }
    </style>
</head>
<body>

<h1>Registered Users</h1>

<?php if (empty($users)): ?>
    <p>No users found in the database.</p>
<?php else: ?>
    <table>
        <thead>
            <tr>
                <th>ID</th>
                <th>Username</th>
                <th>Email</th>
                <th>Created At</th>
            </tr>
        </thead>
        <tbody>
            <?php foreach ($users as $row): ?>
            <tr>
                <td><?= htmlspecialchars($row['id']) ?></td>
                <td><?= htmlspecialchars($row['username']) ?></td>
                <td><?= htmlspecialchars($row['email']) ?></td>
                <td><?= htmlspecialchars($row['created_at']) ?></td>
            </tr>
            <?php endforeach; ?>
        </tbody>
    </table>
    <p class="count"><?= count($users) ?> user(s) found.</p>
<?php endif; ?>

</body>
</html>
```

### 3. Set correct file permissions

```bash
sudo chown www-data:www-data /var/www/devtest/index.php
sudo chmod 640 /var/www/devtest/index.php
```

### 4. Test in the browser or with curl

```bash
curl -k https://devtest.local
```

You should see an HTML table with Alice, Bob, and Charlie listed.

### Notes on this script

- **PDO** is used instead of the older `mysqli_*` functions — it supports prepared statements and works across different database drivers.
- **`htmlspecialchars()`** is used on every output value to prevent XSS — never echo database content directly.
- **`PDO::ERRMODE_EXCEPTION`** means any database error throws a PHP exception rather than silently failing.
- The credentials are hardcoded here for simplicity. In a real application you would move them into a config file outside the document root (e.g. `/etc/devapp/db.conf`) and read them in with `parse_ini_file()` or environment variables.

---

## Task 08 — Cron Job & Bash Script



### 1. Write the backup script

```bash
sudo nano /usr/local/bin/db_backup.sh
```

```bash
#!/bin/bash

DB_NAME="devapp"
DB_USER="devuser"
DB_PASS="StrongPass123!"
BACKUP_DIR="/var/backups"
DATE=$(date +%F)

mkdir -p "$BACKUP_DIR"

mysqldump -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" > "$BACKUP_DIR/${DB_NAME}_${DATE}.sql"

echo "[$(date)] Backup completed: ${DB_NAME}_${DATE}.sql"
```

### 2. Make it executable and test manually

```bash
sudo chmod +x /usr/local/bin/db_backup.sh
sudo /usr/local/bin/db_backup.sh
ls /var/backups/
```

### 3. Schedule with cron

```bash
sudo crontab -e
```

Add this line:

```
0 2 * * * /usr/local/bin/db_backup.sh >> /var/log/db_backup.log 2>&1
```

### Cron expression breakdown

| Field | Value | Meaning |
|---|---|---|
| minute | `0` | at minute 0 |
| hour | `2` | at 2am |
| day of month | `*` | every day |
| month | `*` | every month |
| day of week | `*` | any day |

**The `>> /var/log/db_backup.log 2>&1` part:** `>>` appends stdout to the log file. `2>&1` redirects stderr into stdout, so both normal output and errors land in the same log. Check runs with:

```bash
cat /var/log/db_backup.log
```

---

## Quick Reference — Key Concepts

- **`a2ensite`** creates a symlink in `sites-enabled` — it does not copy the file.
- **Self-signed vs CA cert** — self-signed is fine for internal/test use; production requires a trusted CA (Let's Encrypt or internal PKI) to prevent MITM.
- **mod_php vs PHP-FPM** — use mod_php for simplicity in dev; use PHP-FPM in production for performance and isolation.
- **Disabling password SSH** — always confirm key login works in a live session before saving `sshd_config`.
- **Cron field order** — minute, hour, DOM, month, DOW.
- **`FLUSH PRIVILEGES`** — required after manual `GRANT` statements to reload the in-memory privilege tables.
