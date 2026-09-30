# Installation manuelle — avant l'automatisation

> **Règle du formateur n°1 : « Faire le manuel avant d'automatiser ».**
>
> Ce document décrit **d'abord les commandes shell** à exécuter à la main pour
> obtenir l'architecture cible, puis le **tableau de correspondance**
> *commande manuelle ↔ tâche Ansible*.
>
> **À quoi ça sert :** c'est la **référence de validation**. Quand vous lancez
> `ansible-playbook ... --check`, vous comparez ce que le rôle *va* faire avec
> la procédure manuelle ci-dessous. Si les deux décrivent la même chose,
> l'automatisation est correcte.

**Ordre d'installation (une machine par tier) :**

1. Base de données (PostgreSQL maître → esclave)
2. Cache (Redis)
3. Application web (Apache + PHP-FPM + Laravel)
4. Backoffice (JDK + Spring Boot)
5. Load balancer (Nginx) — **en dernier**

---

## Prérequis communs

```bash
sudo apt update && sudo apt upgrade -y
# Comptes distincts par service (bonne pratique : jamais root pour les apps)
sudo useradd --system --shell /usr/sbin/nologin springboot
# Vérifier que sudo n'exige pas de mot de passe (nécessaire à Ansible)
sudo -v
```

---

## Tier 1 — Load balancer (Nginx)

```bash
sudo apt install -y nginx

sudo tee /etc/nginx/sites-available/laravel_web.conf >/dev/null <<'EOF'
upstream laravel_web {
    least_conn;
    server web01:8080 max_fails=3 fail_timeout=30s;
    server web02:8080 max_fails=3 fail_timeout=30s;
}
server {
    listen 80;
    server_name _;
    location /up { access_log off; return 200 "OK\n"; }
    location / {
        proxy_pass http://laravel_web;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF

sudo rm -f /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/laravel_web.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl enable --now nginx && sudo systemctl reload nginx
```

---

## Tier 2 — Application web (Apache + PHP-FPM + Laravel)

```bash
# 1. Paquets
sudo apt install -y apache2 \
  php8.5-cli php8.5-fpm php8.5-pgsql php8.5-mbstring php8.5-xml \
  php8.5-curl php8.5-zip php8.5-bcmath php8.5-intl \
  composer git unzip

# 2. Modules Apache (PHP-FPM exige proxy_fcgi)
sudo a2enmod rewrite headers proxy proxy_fcgi setenvif

# 3. Port d'écoute
echo 'Listen 8080' | sudo tee /etc/apache2/ports.conf

# 4. Réglages PHP (fichier dédié — on ne touche PAS au php.ini)
sudo tee /etc/php/8.5/fpm/conf.d/99-laravel.ini >/dev/null <<'EOF'
memory_limit = 256M
upload_max_filesize = 64M
post_max_size = 64M
date.timezone = UTC
opcache.enable = 1
EOF

# 5. Application Laravel
sudo mkdir -p /var/www/laravel
sudo git clone -b main <URL_DU_DEPOT_LARAVEL> /var/www/laravel
sudo composer install --no-dev --optimize-autoloader -d /var/www/laravel
sudo -u www-data php /var/www/laravel/artisan key:generate

# 6. Vhost
sudo tee /etc/apache2/sites-available/laravel.conf >/dev/null <<'EOF'
<VirtualHost *:8080>
    ServerName web01
    DocumentRoot /var/www/laravel/public
    <Directory /var/www/laravel/public>
        Options FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.5-fpm.sock|fcgi://localhost/"
    </FilesMatch>
</VirtualHost>
EOF

# 7. Permissions, activation, démarrage
sudo chown -R www-data:www-data /var/www/laravel
sudo a2ensite laravel.conf
sudo systemctl restart php8.5-fpm apache2
```

---

## Tier 3 — Backoffice (JDK + Spring Boot)

```bash
# 1. JDK
sudo apt install -y openjdk-21-jdk-headless

# 2. Utilisateur système (jamais root) et déploiement du JAR statique
sudo useradd --system --shell /usr/sbin/nologin springboot 2>/dev/null || true
sudo mkdir -p /opt/backoffice
sudo cp backoffice-1.0.0.jar /opt/backoffice/backoffice.jar
sudo chown -R springboot:springboot /opt/backoffice

# 3. Fichier d'environnement (contient la datasource -> SECRET)
sudo tee /opt/backoffice/backoffice.env >/dev/null <<'EOF'
JAVA_OPTS=-Xms256m -Xmx512m
SPRING_PROFILES_ACTIVE=prod
SPRING_DATASOURCE_URL=jdbc:postgresql://db01:5432/laravel
SPRING_DATASOURCE_USERNAME=laravel
SPRING_DATASOURCE_PASSWORD=<secret>
EOF
sudo chmod 600 /opt/backoffice/backoffice.env

# 4. Unité systemd
sudo tee /etc/systemd/system/backoffice.service >/dev/null <<'EOF'
[Unit]
Description=backoffice (Spring Boot)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=springboot
Group=springboot
WorkingDirectory=/opt/backoffice
EnvironmentFile=/opt/backoffice/backoffice.env
ExecStart=/usr/bin/java $JAVA_OPTS -jar /opt/backoffice/backoffice.jar --server.port=8080
SuccessExitStatus=143
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload && sudo systemctl enable --now backoffice
```

---

## Tier 4 — Base de données (PostgreSQL maître / esclave)

### 4.1 Les DEUX nœuds

```bash
sudo apt install -y postgresql-17 python3-psycopg2

# Paramètres : fragment séparé, inclu par include_dir = 'conf.d'
sudo tee /etc/postgresql/17/main/conf.d/10-ansible.conf >/dev/null <<'EOF'
listen_addresses = '*'
port = 5432
max_connections = 200
shared_buffers = 512MB
effective_cache_size = 2GB
wal_level = replica
max_wal_senders = 5
max_replication_slots = 5
hot_standby = on
default_transaction_read_only = off
EOF

# Autoriser les applications ET la réplication
sudo tee -a /etc/postgresql/17/main/pg_hba.conf >/dev/null <<'EOF'
host    all          all          0.0.0.0/0       scram-sha-256
host    replication  replicator   10.0.0.31/32    scram-sha-256
EOF

sudo systemctl restart postgresql
```

### 4.2 Sur le MAÎTRE

```bash
sudo -u postgres psql <<'SQL'
CREATE USER replicator WITH PASSWORD '<secret_repli>' REPLICATION LOGIN;
CREATE USER laravel    WITH PASSWORD '<secret_app>';
CREATE DATABASE laravel OWNER laravel;
SQL

sudo -u postgres psql -c "SELECT pg_create_physical_replication_slot('db02_slot');"
```

### 4.3 Sur l'ESCLAVE (initialisation)

```bash
sudo systemctl stop postgresql
sudo rm -rf /var/lib/postgresql/17/main/*

PGPASSWORD='<secret_repli>' pg_basebackup \
  -h db01 -p 5432 -U replicator \
  -D /var/lib/postgresql/17/main -X stream -C -S db02_slot -R

sudo chown -R postgres:postgres /var/lib/postgresql/17/main
sudo systemctl start postgresql

# Vérification : doit renvoyer "t"
psql -U postgres -c 'SELECT pg_is_in_recovery();'
```

---

## Tier 5 — Cache (Redis)

```bash
sudo apt install -y redis-server

sudo tee /etc/redis/redis.conf >/dev/null <<'EOF'
bind 0.0.0.0
port 6379
daemonize no
supervised no
loglevel notice
databases 16
appendonly no
requirepass <secret>
maxmemory 256mb
maxmemory-policy allkeys-lru
EOF

sudo systemctl enable --now redis-server
redis-cli -a <secret> ping   # doit répondre PONG
```

---

## Correspondance : commande manuelle ↔ tâche Ansible

| # | Commande manuelle | Rôle | Tâche / module Ansible |
|---|---|---|---|
| 1 | `apt install -y nginx` | `nginx_lb` | `ansible.builtin.apt` |
| 2 | `tee .../laravel_web.conf` | `nginx_lb` | **`template`** → `nginx_lb.conf.j2` |
| 3 | `ln -s ... sites-enabled/` | `nginx_lb` | `ansible.builtin.file` (`state: link`, `force: true`) |
| 4 | `systemctl reload nginx` | `nginx_lb` | **Handler** `Recharger Nginx` |
| 5 | `a2enmod rewrite headers ...` | `apache` | `community.general.apache2_module` (boucle) |
| 6 | `echo 'Listen 8080' > ports.conf` | `apache` | **`template`** → `apache_ports.conf.j2` |
| 7 | `tee laravel.conf` + `a2ensite` | `apache` | **`template`** → `apache_vhost.conf.j2` + `file state=link` |
| 8 | `tee conf.d/99-laravel.ini` | `php` | **`template`** → `99-laravel.ini.j2` |
| 9 | `apt install -y php8.5-*` | `php` | `ansible.builtin.apt` |
| 10 | `git clone -b main <url>` | `laravel` | `ansible.builtin.git` |
| 11 | `composer install --no-dev` | `laravel` | `community.general.composer` |
| 12 | `php artisan key:generate` | `laravel` | `ansible.builtin.command` (`when`: clé vide) |
| 13 | `tee /var/www/laravel/.env` | `laravel` | **`template`** → `env.j2` |
| 14 | `apt install openjdk-21-jdk-headless` | `java` | `ansible.builtin.apt` |
| 15 | `useradd springboot` + `cp jar` | `springboot` | `user` + **`copy`** (de `roles/springboot/files/`) |
| 16 | `tee backoffice.service` + `daemon-reload` | `springboot` | **`template`** → `springboot.service.j2` + `systemd` |
| 17 | `tee conf.d/10-ansible.conf` | `postgresql` | **`template`** → `conf.d/10-ansible.conf.j2` |
| 18 | `tee -a pg_hba.conf` | `postgresql` | **`template`** → `pg_hba.conf.j2` |
| 19 | `psql CREATE USER / DATABASE` | `postgresql` | `community.postgresql.postgresql_user` / `postgresql_db` (**`no_log`**) |
| 20 | `pg_create_physical_replication_slot()` | `postgresql` | `community.postgresql.postgresql_query` (**`no_log`**) |
| 21 | `pg_basebackup -X stream -C -S -R` | `postgresql` | `ansible.builtin.command` (**`no_log`**, `PGPASSWORD`) |
| 22 | `tee /etc/redis/redis.conf` | `redis` | **`template`** → `redis.conf.j2` (**`no_log`**) |

> **Les 3 premiers rôles de la colonne « Tâche » montrent bien la règle du
> formateur** : dès qu'un fichier de configuration est concerné, c'est un
> **`template` Jinja2**, jamais un module `builtin` d'écriture de ligne.

---

## Checklist de validation

```bash
# 1. Syntaxe
ansible-playbook -i inventories/local/hosts.yml playbooks/site.yml --syntax-check

# 2. Câblage des rôles
ansible-playbook -i inventories/local/hosts.yml playbooks/site.yml --list-tasks

# 3. Simulation (aucune modification) : comparer avec CE document
ansible-playbook -i inventories/local/hosts.yml playbooks/site.yml --check --diff

# 4. Chiffrement des secrets
ansible-vault encrypt group_vars/all/vault.yml
```

- [ ] Le résultat du `--check` correspond **à la lettre** à la procédure manuelle
      de ce document
- [ ] Aucun `lineinfile` / `blockinfile` n'écrit dans un fichier de configuration
- [ ] Tous les secrets passent par `vault.yml` + `no_log: true`
- [ ] Tout fichier statique provient de `roles/<rôle>/files/`


