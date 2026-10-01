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

# 5 bis. Fichier d'environnement (contient DB_PASSWORD et APP_KEY -> SECRET)
#        La clé APP_KEY est FIXÉE (vault/coffre) : une clé régénérée à chaque
#        déploiement invaliderait sessions et cookies.
sudo -u www-data tee /var/www/laravel/.env >/dev/null <<'EOF'
APP_NAME=Laravel
APP_ENV=production
APP_KEY=<cle_base64>
DB_CONNECTION=pgsql
DB_HOST=<ip_du_primaire>     # variante cloisonnée : 127.0.0.1
DB_PORT=5432                 # variante cloisonnée : 5442 (cluster 18/db01)
DB_DATABASE=laravel
DB_USERNAME=laravel
DB_PASSWORD=<secret_app>
EOF
sudo chmod 600 /var/www/laravel/.env

# 5 ter. Contrôle applicatif puis migrations
#       Une SEULE fois pour tout le tier : les instances web partagent la base.
sudo -u www-data php /var/www/laravel/artisan db:show      # connexion reelle
sudo -u www-data php /var/www/laravel/artisan migrate --force

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

### 4.4 Variante locale cloisonnée (plusieurs clusters sur UNE machine)

Reproduit la même topologie **sans AWS** et **sans jamais toucher** au cluster
`18-main` (port 5432) déjà utilisé par les autres projets de la machine.

```bash
# --- Primaire : cluster dédié 18/db01, port 5442 ---
sudo pg_createcluster 18 db01 --port 5442      # en ROOT + initdb via su

# Mêmes fragments que 4.1, mais dans le cluster db01
sudo tee /etc/postgresql/18/db01/conf.d/10-ansible.conf >/dev/null <<'EOF'
listen_addresses = '*'
port = 5442
max_connections = 100
shared_buffers = 256MB
effective_cache_size = 1GB
wal_level = replica
max_wal_senders = 5
max_replication_slots = 5
hot_standby = on
default_transaction_read_only = off
EOF

# Applications (boucle locale) ET réplication entre les deux clusters
sudo tee -a /etc/postgresql/18/db01/pg_hba.conf >/dev/null <<'EOF'
host    all          all          127.0.0.1/32    scram-sha-256
host    replication  replicator   127.0.0.1/32    scram-sha-256
EOF

sudo systemctl restart postgresql@18-db01       # jamais le méta-service "postgresql"

# Créations : viser LE cluster avec -p 5442
sudo -u postgres psql -p 5442 <<'SQL'
CREATE USER laravel    WITH PASSWORD '<secret_app>';
CREATE USER replicator WITH PASSWORD '<secret_repli>' REPLICATION LOGIN;
CREATE DATABASE laravel OWNER laravel;
SELECT pg_create_physical_replication_slot('db02_slot');
SQL

# --- Réplique : cluster dédié 18/db02, port 5443 ---
sudo pg_createcluster 18 db02 --port 5443
sudo systemctl stop postgresql@18-db02
sudo rm -rf /var/lib/postgresql/18/db02/*

PGPASSWORD='<secret_repli>' pg_basebackup -h 127.0.0.1 -p 5442 -U replicator \
  -D /var/lib/postgresql/18/db02 -X stream -S db02_slot -R
# ↑ pas de -C : le slot est déjà créé sur le primaire, et -C échouerait

sudo systemctl start postgresql@18-db02

# Vérifications
sudo -u postgres psql -p 5442 -c 'SELECT client_addr, state FROM pg_stat_replication;'
sudo -u postgres psql -p 5443 -c 'SELECT pg_is_in_recovery();'   # doit renvoyer "t"
```

> Écarts avec la variante AWS (§4.1) : **un cluster par instance**
> (`pg_createcluster 18 db01|db02`), service `postgresql@<version>-<instance>`
> au lieu du méta-service, ports 5442/5443, et primaire désigné par
> `127.0.0.1` **avec son port** (les deux clusters sont sur la même machine).

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

### 5.1 Variante locale cloisonnée (instance `redis01`, port **6390**)

> ⚠️ Sur une machine où vivent **déjà** un Redis (port **6379**) et un sentinel
> (**26379**), lancés **hors systemd** par une autre application, on ne touche
> **jamais** à ces processus ni au méta-service `redis-server` : on crée une
> instance dédiée, avec son compte, son répertoire, sa configuration et son
> unité systemd propre.

```bash
# 1. Compte et répertoires dédiés (jamais root, jamais l'utilisateur « redis »)
sudo groupadd --system redis01
sudo useradd --system --gid redis01 --home /var/lib/redis-redis01 \
  --shell /usr/sbin/nologin redis01
sudo install -d -o redis01 -g redis01 -m 0750 /etc/redis-redis01 /var/lib/redis-redis01
sudo install -d -o redis01 -g redis01 -m 0755 /run/redis-redis01

# 2. Configuration de l'instance (port 6390, dir et pidfile dédiés)
sudo tee /etc/redis-redis01/redis.conf >/dev/null <<'EOF'
bind 127.0.0.1
port 6390
daemonize no
supervised systemd
loglevel notice
databases 16
protected-mode yes
dir /var/lib/redis-redis01
dbfilename dump.rdb
pidfile /run/redis-redis01/redis-redis01.pid
logfile ""
appendonly no
requirepass <secret>
maxmemory 128mb
maxmemory-policy allkeys-lru
EOF
sudo chown redis01:redis01 /etc/redis-redis01/redis.conf
sudo chmod 0640 /etc/redis-redis01/redis.conf

# 3. Unité systemd dédiée (Type=notify : Redis notifie systemd quand il est prêt)
sudo tee /etc/systemd/system/redis-redis01.service >/dev/null <<'EOF'
[Unit]
Description=Redis cloisonné (instance redis01, port 6390)
After=network.target

[Service]
Type=notify
User=redis01
Group=redis01
ExecStart=/usr/bin/redis-server /etc/redis-redis01/redis.conf
ExecStop=/bin/kill -s TERM $MAINPID
Restart=on-failure
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now redis-redis01

# 4. Vérifications (aucune commande ne vise « redis-server » ni le 6379)
systemctl is-active redis-redis01                      # active
systemctl is-active redis-server                       # inactive (méta-service intouché)
redis-cli -p 6390 ping                                 # NOAUTH Authentication required.
REDISCLI_AUTH=<secret> redis-cli -p 6390 ping          # PONG
REDISCLI_AUTH=<secret> redis-cli -p 6390 info replication | head -3
redis-cli -p 6379 ping                                 # PONG — le Redis étranger répond toujours
redis-cli -p 26379 ping                                # PONG — le sentinel est intact
```

```bash
# 5. Côté application : PHP a besoin de l'extension phpredis
sudo apt install -y php8.5-redis && sudo systemctl reload php-fpm-web01 php-fpm-web02
php --ri redis | head -3        # Redis Support => enabled

# 6. Drivers Laravel (fichier .env de chaque instance)
#    SESSION_DRIVER=redis
#    CACHE_STORE=redis
#    QUEUE_CONNECTION=redis
#    REDIS_CLIENT=phpredis
#    REDIS_HOST=127.0.0.1
#    REDIS_PORT=6390
#    REDIS_PASSWORD=<secret>
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
| 23 | `pg_createcluster 18 db01 --port 5442` (variante cloisonnée) | `postgresql` | `ansible.builtin.command` **en root** (jamais `become_user: postgres`), gardé par un `stat` du répertoire de config (tâche non idempotente) |
| 24 | `systemctl restart postgresql@18-db01` | `postgresql` | `ansible.builtin.systemd` (unité `postgresql@<v>-<instance>`) + **Handler** `Redémarrer PostgreSQL` |
| 25 | `psql -p 5442 …` (viser **le bon** cluster) | `postgresql` | `login_port` + `login_unix_socket` sur les modules `community.postgresql.*` |
| 26 | `pg_basebackup -h 127.0.0.1 -p 5442 -S db02_slot -R` | `postgresql` | `ansible.builtin.command` (**`no_log`**, `PGPASSWORD`) — **sans `-C`**, le slot étant créé par le primaire |
| 27 | `sudo systemctl start postgresql@18-main` (à **ne pas** faire) | `postgresql` | — : le rôle n'appelle **jamais** le méta-service `postgresql`, qui démarrerait tous les clusters |
| 28 | `php artisan db:show` (utilisateur de PHP-FPM) | `laravel` | `ansible.builtin.command` — vérification **non bloquante** (`failed_when: false`), sortie sans `DB_PASSWORD` |
| 29 | `php artisan migrate --force` | `laravel` | `ansible.builtin.command` gardé par `laravel_run_migrations` (**`run_once`** : 2 web ↔ 1 base partagée) |
| 30 | `useradd --system redis01` + `install -d` (répertoires de l'instance) | `redis` | `ansible.builtin.group` / `user` / `file` (tâches `[instance]`) |
| 31 | `tee /etc/redis-redis01/redis.conf` | `redis` | **`template`** → `redis.conf.j2` (**`no_log`**) |
| 32 | `tee /etc/systemd/system/redis-redis01.service` | `redis` | **`template`** → `redis.service.j2` (`Type=notify`, utilisateur de l'instance) |
| 33 | `systemctl enable --now redis-redis01` | `redis` | `ansible.builtin.systemd` (`daemon_reload: true`) + **Handler** `Redémarrer Redis` |
| 34 | `REDISCLI_AUTH=… redis-cli -p 6390 ping` / `info replication` | `redis` | `ansible.builtin.command` (`no_log` sur le ping, `failed_when: false`, `changed_when: false`) |
| 35 | `apt install php8.5-redis` | `php` | `ansible.builtin.apt` (paquet ajouté à `php_packages`) + `php --ri redis` en contrôle |
| 36 | `tee …/laravel-queue-web01.service` (**opt-in**) | `laravel` | **`template`** → `queue-worker.service.j2` — sauté si `laravel_queue_worker_enabled: false` |

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
- [ ] En variante cloisonnée, le cluster système voisin (`18-main`) n'est
      **jamais** arrêté ni redémarré : aucune tâche ne vise le méta-service
      `postgresql`, et `pg_lsclusters` doit montrer `18-main` toujours `online`
- [ ] En variante cloisonnée Redis, `systemctl is-active redis-server` reste
      `inactive` (le méta-service n'est **jamais** appelé) et le Redis étranger
      du port 6379 comme le sentinel du 26379 répondent toujours ; seules
      l'unité `redis-redis01`, le port 6390 et le compte `redis01` appartiennent
      au projet


