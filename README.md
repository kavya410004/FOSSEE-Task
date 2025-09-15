# DigitalOcean Droplet Setup with Keycloak SSO + Drupal, Django, PHP

This documentation shows how to set up a DigitalOcean droplet (Rocky Linux 10), configure Keycloak for Single Sign-On (SSO), and deploy three applications: **Drupal 11**, a **Django project**, and a **PHP app**, all integrated with Keycloak.

---

## 1. Droplet Setup & Security

1. Create a droplet on DigitalOcean: Rocky Linux 10, 2GB+ RAM, enable IPv4 & IPv6, add your SSH key.
2. SSH as root and create a user:

   ```bash
   adduser devuser
   passwd devuser
   usermod -aG wheel devuser
   rsync --archive --chown=devuser:devuser ~/.ssh /home/devuser
   ```
3. Disable root login in `/etc/ssh/sshd_config`: set `PermitRootLogin no`, then restart sshd.
4. Enable firewall and allow HTTP, HTTPS, SSH, and Keycloak (8080):

   ```bash
   sudo systemctl enable --now firewalld
   sudo firewall-cmd --permanent --add-service={http,https,ssh}
   sudo firewall-cmd --permanent --add-port=8080/tcp
   sudo firewall-cmd --reload
   ```
5. Update packages and install basics:

   ```bash
   sudo dnf update -y
   sudo dnf install epel-release -y
   sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-10.rpm -y
   sudo dnf module enable php:remi-8.3 -y
   sudo dnf install httpd php mariadb-server python3 python3-pip unzip wget composer -y
   sudo systemctl enable --now httpd php-fpm mariadb
   sudo mysql_secure_installation
   ```

---

## 2. Keycloak Setup

1. Install Java & Keycloak:

   ```bash
   sudo dnf install java-17-openjdk-devel -y
   cd /opt
   sudo wget https://github.com/keycloak/keycloak/releases/download/24.0.4/keycloak-24.0.4.zip
   sudo unzip keycloak-24.0.4.zip && sudo mv keycloak-24.0.4 keycloak
   sudo useradd -r -g keycloak -d /opt/keycloak -s /sbin/nologin keycloak
   sudo chown -R keycloak:keycloak /opt/keycloak
   ```
2. Start Keycloak (dev mode):

   ```bash
   /opt/keycloak/bin/kc.sh start-dev --http-port=8080
   ```

   Go to `http://157.230.55.101:8080`, create admin, log in.
3. Systemd service for production:

   ```ini
   [Unit]
   Description=Keycloak Server
   After=network.target

   [Service]
   User=keycloak
   Group=keycloak
   ExecStart=/opt/keycloak/bin/kc.sh start --optimized --http-port=8080
   Restart=on-failure

   [Install]
   WantedBy=multi-user.target
   ```

   Enable: `sudo systemctl daemon-reload && sudo systemctl enable --now keycloak`

---

## 3. Drupal 11 + SSO

1. Create DB in MariaDB:

   ```sql
   CREATE DATABASE drupaldb;
   CREATE USER 'drupaluser'@'localhost' IDENTIFIED BY 'Drup@lPass123';
   GRANT ALL PRIVILEGES ON drupaldb.* TO 'drupaluser'@'localhost';
   ```
2. Install Drupal:

   ```bash
   cd /var/www
   sudo composer create-project drupal/recommended-project drupal
   sudo chown -R apache:apache /var/www/drupal
   ```
3. Apache vhost:

   ```apache
   <VirtualHost *:80>
     ServerName drupal.example.com
     DocumentRoot /var/www/drupal/web
     <Directory /var/www/drupal/web>
       AllowOverride All
       Require all granted
     </Directory>
   </VirtualHost>
   ```
4. Enable Keycloak module:

   ```bash
   composer require drupal/keycloak
   ```
5. In Keycloak, create client `drupal` with redirect URI `http://drupal.example.com/keycloak/oauth2/callback`. Copy client secret and configure in Drupal.

---

## 4. Django Project + SSO

1. Create DB:

   ```sql
   CREATE DATABASE djangodb;
   CREATE USER 'djangouser'@'localhost' IDENTIFIED BY 'Dj@ngoPass123';
   GRANT ALL PRIVILEGES ON djangodb.* TO 'djangouser'@'localhost';
   ```
2. Set up project:

   ```bash
   mkdir /var/www/django_project && cd /var/www/django_project
   python3 -m venv venv && source venv/bin/activate
   pip install django gunicorn mozilla-django-oidc mysqlclient
   django-admin startproject mysite .
   ```

   Configure DB in `settings.py`.
3. Add Keycloak OIDC settings:

   ```python
   INSTALLED_APPS += ['mozilla_django_oidc']
   AUTHENTICATION_BACKENDS = ['mozilla_django_oidc.auth.OIDCAuthenticationBackend','django.contrib.auth.backends.ModelBackend']
   OIDC_RP_CLIENT_ID = 'django'
   OIDC_RP_CLIENT_SECRET = 'Dj@ngoClientSecret'
   OIDC_OP_AUTHORIZATION_ENDPOINT = 'http://157.230.55.101:8080/realms/master/protocol/openid-connect/auth'
   ```
4. Run Gunicorn on port 8000, proxy via Apache vhost:

   ```apache
   <VirtualHost *:80>
     ServerName django.example.com
     ProxyPass / http://127.0.0.1:8000/
     ProxyPassReverse / http://127.0.0.1:8000/
   </VirtualHost>
   ```
5. In Keycloak, create client `django` with redirect URI `http://django.example.com/oidc/callback/`.

---

## 5. PHP App + SSO

1. Place files in `/var/www/php_app`, set permissions.
2. Install OIDC client:

   ```bash
   cd /var/www/php_app
   composer require jumbojett/openid-connect-php
   ```
3. `login.php`:

   ```php
   <?php
   require 'vendor/autoload.php';
   use Jumbojett\OpenIDConnectClient;
   session_start();
   $oidc = new OpenIDConnectClient('http://157.230.55.101:8080/realms/master','php-app','PhP@ppSecret');
   $oidc->authenticate();
   $_SESSION['user_info'] = $oidc->requestUserInfo();
   header("Location: /profile.php");
   ?>
   ```
4. `profile.php`:

   ```php
   <?php
   session_start();
   $userInfo = $_SESSION['user_info'];
   echo "<h1>Welcome, ".$userInfo->name."</h1>";
   ?>
   ```
5. Apache vhost:

   ```apache
   <VirtualHost *:80>
     ServerName php.example.com
     DocumentRoot /var/www/php_app
     <Directory /var/www/php_app>
       AllowOverride All
       Require all granted
     </Directory>
   </VirtualHost>
   ```
6. In Keycloak, create client `php-app` with redirect `http://php.example.com/callback.php`.

---

