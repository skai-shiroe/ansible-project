# Architecture 3-tiers — Automatisation Ansible

Automatisation du déploiement d'une architecture 3-tiers sur AWS :
une application **web publique Laravel**, une application d'administration
**Backoffice Spring Boot** et une couche de données **PostgreSQL en
maître/esclave**, avec **Redis** comme cache et **Nginx** comme équilibreur
de charge.

> **Règles du formateur** → voir [§14 Conventions du projet](#14-conventions-du-projet-règles-du-formateur)
> et le guide d'installation manuelle [`docs/installation-manuelle.md`](docs/installation-manuelle.md).

---

## 1. Architecture cible

```
                    +---------------------------+
   Internet ------->|  Load Balancer (Nginx)    |  loadbalancers : lb01
                    |  repartition de charge    |
                    +-------------+-------------+
                                  |  HTTP :80 -> :8080
                  +---------------+---------------+
                  v                               v
        +-------------------+          +-------------------+
        |  Web #1 (Laravel) |          |  Web #2 (Laravel) |  webservers : web01, web02
        |  Apache + PHP-FPM |          |  Apache + PHP-FPM |
        +---------+---------+          +---------+---------+
                  |        +---------------------+
                  |        |
                  v        v
        +-------------------+          +--------------------+
        |  Redis (cache,    |          |  Backoffice        |  redis : redis01
        |  sessions, queue) |          |  Java + Spring Boot|  backoffice : bo01
        +-------------------+          +---------+----------+
                                                 |
                  +------------------------------+
                  v
        +-------------------+  replication  +-------------------+
        |  PostgreSQL       |<-- streaming -|  PostgreSQL       |  databases : db01, db02
        |  PRIMAIRE (maitre)|   (slots)     |  REPLIQUE (esclave)|
        +-------------------+               +-------------------+
```

| Tier | Rôles | Instances | Résilience |
|---|---|---|---|
| Load balancer | `nginx_lb` | 1 | Point d'entrée unique, répartition de charge |
| Application web | `apache`, `php`, `laravel` | 2 | Tolérance aux pannes, absorption du trafic |
| Backoffice | `java`, `springboot` | 1 | Nombre d'utilisateurs internes maîtrisé |
| Base de données | `postgresql` | 2 | Maître/esclave, réplication, failover |
| Cache | `redis` | 1 | Cache / sessions / queues Laravel |

---

## 2. Prérequis

- **Ansible** : `ansible-core >= 2.15` (testé avec `2.21.3`)
- **Collections** (installées dans `.ansible/collections` du projet) :

  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

  Fournit `amazon.aws`, `community.aws`, `community.general`, `community.postgresql`.
- **AWS** : profil SSO disposant des droits EC2 (`DescribeInstances`) :

  ```bash
  aws sso login --profile formation-sso
  ```

---

## 3. Structure du dépôt

```
.
├── ansible.cfg                  # Configuration Ansible (inventaire, rôles, collections)
├── requirements.yml             # Collections requises
├── inventories/
│   ├── aws_ec2.yml              # Inventaire dynamique AWS (groupes par tags)
│   ├── hosts.yml                # Inventaire statique de validation (sans AWS)
│   ├── gcp_compute.yml
│   └── local/                   # Exécution « cloisonnée » sur UNE machine
│       ├── hosts.yml            # Mêmes groupes que l'AWS, en 127.0.0.1
│       ├── group_vars/all/vault.yml   # Secrets chiffrés de l'environnement local
│       └── host_vars/
│           ├── db01.yml         # Instance dédiée 18/db01 → 5442 (primaire)
│           └── db02.yml         # Instance dédiée 18/db02 → 5443 (réplique)
├── group_vars/                  # Variables partagées par groupe d'hôtes
│   ├── all/vars.yml
│   ├── all/vault.yml            # Secrets (à chiffrer avec ansible-vault)
│   ├── webservers.yml           # Tier web
│   ├── loadbalancers.yml        # Load balancer
│   ├── backoffice.yml           # Backoffice
│   ├── databases.yml            # Cluster PostgreSQL
│   └── redis.yml                # Cache
├── host_vars/                   # Variables propres à une machine
│   ├── db01.yml                 # Nœud primaire
│   └── db02.yml                 # Nœud réplique
├── playbooks/
│   ├── site.yml                 # Orchestration complète
│   ├── applications.yml         # Web + Backoffice
│   ├── database.yml             # PostgreSQL
│   ├── cache.yml                # Redis
│   └── local_*.yml              # Variantes cloisonnées (lb, web, db) — 1 machine
└── roles/                       # 8 rôles autonomes
    ├── nginx_lb/  ├── apache/  ├── php/       ├── laravel/
    ├── java/      ├── springboot/  ├── postgresql/  └── redis/
```

> `playbooks/group_vars` et `playbooks/host_vars` sont des **liens symboliques**
> vers les répertoires de la racine : un playbook lancé depuis `playbooks/`
> charge donc les mêmes variables, **sans duplication de fichiers**. C'est
> **volontaire** : les playbooks `local_*.yml` héritent ainsi du paramétrage de
> groupe (nom de la base, utilisateur applicatif…), dont les valeurs **propres à
> la topologie** sont ensuite surchargées par `inventories/local/` (voir §5).

---

## 4. Inventaires

### 4.1 Inventaire dynamique AWS (`inventories/aws_ec2.yml`)

Les groupes sont construits **automatiquement à partir des tags EC2** :

| Groupe Ansible | Condition (tag) | Contenu |
|---|---|---|
| `loadbalancers` | `Role=lb` | Instance Nginx |
| `webservers` | `Role=web` | 2 instances Laravel |
| `backoffice` | `Role=backoffice` | 1 instance Spring Boot |
| `databases` | `Role=db` | 2 instances PostgreSQL |
| `redis` | `Role=redis` | 1 instance Redis |

- Le **nom d'hôte Ansible** est la valeur du tag **`Name`** :

  ```yaml
  hostnames:
    - tag:Name
    - dns-name
    - private-ip-address
  ```

- La connexion SSH utilise `public_ip_address | default(private_ip_address)`.
- Un **cache** (`/tmp/ansible_aws_ec2_cache`, 600 s) évite de solliciter
  l'API AWS à chaque exécution.

> ⚠️ **Tags requis sur les instances** : `Name`, `Environment` et surtout
> **`Role`** avec les valeurs `lb`, `web`, `backoffice`, `db`, `redis`.
> Sans le tag `Role`, les groupes sont vides et aucun rôle ne s'applique.

### 4.2 Inventaire statique (`inventories/hosts.yml`)

Permet de **valider sans AWS** (syntaxe, câblage des rôles, rendu des templates) :

| Groupe | Hôtes | Adresses |
|---|---|---|
| `loadbalancers` | `lb01` | 10.0.0.10 |
| `webservers` | `web01`, `web02` | 10.0.0.11, 10.0.0.12 |
| `backoffice` | `bo01` | 10.0.0.20 |
| `databases` | `db01` (primaire), `db02` (réplique) | 10.0.0.30, 10.0.0.31 |
| `redis` | `redis01` | 10.0.0.40 |

### 4.3 Inventaire local cloisonné (`inventories/local/`)

Utilisé pour exécuter **le même code de rôle** sur une seule machine (Ubuntu),
avec **plusieurs instances de chaque service** et **sans toucher** aux services
déjà installés par ailleurs.

| Élément | Contenu |
|---|---|
| `inventories/local/hosts.yml` | groupes identiques à l'AWS mais **tous en `127.0.0.1`** |
| `inventories/local/group_vars/all/vault.yml` | secrets **chiffrés** (`ansible-vault encrypt`) |
| `inventories/local/host_vars/db01.yml` | `postgresql_instance: db01`, port **5442**, `node_role: primary` |
| `inventories/local/host_vars/db02.yml` | `postgresql_instance: db02`, port **5443**, `node_role: replica`, `primary_host: 127.0.0.1`, `primary_port: 5442` |
| `inventories/local/hosts.yml` → `redis01` | `redis_instance: redis01`, port **6390**, utilisateur / répertoire / pidfile dédiés, `redis_password: {{ vault_redis_password }}` (jamais 6379/26379) |
| `inventories/local/hosts.yml` → `web01` / `web02` | `laravel_redis_host: 127.0.0.1`, `laravel_redis_port: 6390`, `SESSION_DRIVER` / `CACHE_STORE` / `QUEUE_CONNECTION` = **`redis`**, worker **opt-in** (`laravel_queue_worker_enabled: false`) |

```bash
# Déchiffrer les secrets : fichier de mot de passe hors dépôt (.gitignore)
ansible-playbook -i inventories/local/hosts.yml playbooks/local_db.yml \
  --vault-password-file .vault_pass
```

---

## 5. Convention de placement des variables

C'est la règle structurante du projet : **chaque variable est placée selon sa
portée**.

| Emplacement | Nature | Exemples |
|---|---|---|
| `roles/<r>/defaults/main.yml` | Paramètre **surchargeable** (contrat d'entrée du rôle) | `apache_port: 80`, `php_version: "8.5"` |
| `roles/<r>/vars/main.yml` | **Constante interne** au rôle (à ne pas surcharger) | `/etc/apache2/sites-available`, `postgresql_service` |
| `group_vars/<groupe>.yml` | Valeur **partagée par un tier** (topologie, dimensionnement) | `apache_port: 8080`, `postgresql_max_connections`, liste des backends |
| `host_vars/<hôte>.yml` | Valeur **propre à une machine** | `postgresql_node_role: primary` / `replica` |
| `group_vars/all/vault.yml` | **Secrets** chiffrés | `vault_postgresql_password` |

### Ordre de précédence (du plus faible au plus fort)

```
1. role defaults        (roles/<r>/defaults/main.yml)
2. inventory group_vars (group_vars/<groupe>.yml)
3. inventory host_vars  (host_vars/<hôte>.yml)
4. role vars            (roles/<r>/vars/main.yml)
5. play vars / vars_files / extra-vars
```

### Variables principales

| Variable | Valeur | Emplacement | Rôle |
|---|---|---|---|
| `nginx_lb_backend_servers` | `{{ groups['webservers'] }}` | `group_vars/loadbalancers.yml` | Backends du load balancer |
| `apache_port` | `8080` | `group_vars/webservers.yml` | Port d'écoute Apache |
| `apache_document_root` | `/var/www/laravel/public` | `group_vars/webservers.yml` | Racine web Laravel |
| `php_version` | `"8.5"` | `group_vars/webservers.yml` | Version de PHP |
| `laravel_path` | `/var/www/laravel` | `group_vars/webservers.yml` | Emplacement de l'application |
| `java_version` | `"21"` | `group_vars/backoffice.yml` | Version du JDK |
| `springboot_port` | `8080` | `group_vars/backoffice.yml` | Port de l'application Spring |
| `postgresql_version` | `"17"` | `group_vars/databases.yml` | Version de PostgreSQL |
| `redis_port` | `6379` | `group_vars/redis.yml` | Port de Redis |
| `postgresql_node_role` | `primary` / `replica` | `host_vars/db01.yml`, `db02.yml` | Rôle du nœud dans le cluster |
| `postgresql_primary_host` | `db01` | `host_vars/db02.yml` | Primaire, source de la réplication |
| `postgresql_replication_slot` | `db02_slot` | `host_vars/db02.yml` | Slot de réplication dédié |
| `postgresql_instance` | `db01` / `db02` | `inventories/local/host_vars/db0X.yml` | Cluster Debian dédié (`""` = mode AWS) |
| `postgresql_replication_allowed_addresses` | `['127.0.0.1/32']` | `inventories/local/host_vars/db0X.yml` | Réplication via la boucle locale |
| `laravel_db_host` / `laravel_db_port` | `127.0.0.1` / `5442` | `inventories/local/hosts.yml` | Base partagée par `web01`/`web02` (primaire `18/db01`) |
| `laravel_run_migrations` | `true` (local) / `false` (AWS) | `inventories/local/hosts.yml` + `roles/laravel/defaults` | Migration de schéma, jouée **une fois** pour le tier |

### Cas particulier du tier cloisonné local

Les playbooks `local_*.yml` **réutilisent** les `group_vars` de la racine (via les
liens symboliques de `playbooks/`) et **n'en changent que la topologie** :

```
role defaults                ──►  variante AWS : port 5432, cluster « main », méta-service
group_vars/databases.yml     ──►  nom de la base, utilisateur applicatif, dimensionnement
inventories/local/host_vars  ──►  instance, version 18, port 5442/5443,
                                  service postgresql@<v>-<instance>,
                                  rôle primaire/réplique, primaire = 127.0.0.1
include_vars (local_db.yml)  ──►  rejoue le même fichier en précédence maximale :
                                  indispensable car un playbook lancé depuis
                                  playbooks/ voit AUSSI host_vars/db0X.yml (AWS)
```

C'est ce qui permet de faire basculer un rôle conçu pour AWS vers une exécution
cloisonnée **en n'ajoutant qu'un fichier de variables** — le rôle et le
paramétrage AWS restant intacts.

---

## 6. Les 8 rôles

| Rôle | État | Ce qu'il fait |
|---|---|---|
| `nginx_lb` | **Nouveau** | Installe Nginx, génère l'`upstream` vers les instances web, `least_conn`, health-check `/up`, rechargement à chaud |
| `apache` | **Refactoré** | Installe Apache, active les modules (`rewrite`, `proxy_fcgi`…), configure le port 8080 et le vhost Laravel (délégation PHP à PHP-FPM) |
| `php` | **Refactoré** | Versionné (`php{{ php_version }}`), configure `php.ini`, gère le service PHP-FPM, installe `php{{ php_version }}-redis` (**phpredis**) |
| `laravel` | **Complété** | Déploie le code, génère le `.env` (drivers **Redis** : sessions/cache/queues), `composer install`, `key:generate`, permissions, **worker de file opt-in** |
| `java` | **Complet** | PPA `ondrej/java` (deb822), JDK 21 headless, **détection du vrai `JAVA_HOME`**, `stat` + `assert` (fail-fast), exposition via `/etc/profile.d/java.sh`, `java -version` |
| `springboot` | **Nouveau** | Utilisateur système dédié, déploiement du JAR, unité **systemd**, fichier d'environnement |
| `postgresql` | **Nouveau** | Installation, `postgresql.conf`/`pg_hba.conf`, **utilisateur + slots de réplication** (maître), **`pg_basebackup` + standby** (esclave) |
| `redis` | **Refactoré** | Installe et configure Redis (mémoire, politique d'éviction, `requirepass`) ; **2 modes** : AWS (service `redis-server`) ou **instance cloisonnée** (`redis_instance` ⇒ unité `redis-redis01`, config / utilisateur / répertoires / pidfile dédiés, `Type=notify`) — sans jamais toucher un Redis étranger |
| `local_lb.yml` / `local_web.yml` / `local_db.yml` / `local_cache.yml` | instanciés | déclinent les mêmes rôles sur les instances cloisonnées (`lb01`, `web01`/`web02`, `db01`/`db02`, `redis01`), en exécution **locale** sans SSH (voir §8 — jeu 6) |

Chaque rôle contient : `tasks/main.yml`, `handlers/main.yml`,
`defaults/main.yml`, `vars/main.yml` et, si nécessaire, `templates/`.

---

## 7. Les playbooks

| Playbook | Cibles | Rôles appelés |
|---|---|---|
| `applications.yml` | `webservers` | `apache`, `php`, `laravel` |
| ″ | `backoffice` | `java`, `springboot` |
| `database.yml` | `databases` | `postgresql` |
| `cache.yml` | `redis` | `redis` |
| `site.yml` | orchestration | `import_playbook` des 3 précédents + `nginx_lb` sur `loadbalancers` |

**Ordre de déploiement de `site.yml`** : applications → base de données → cache →
**load balancer en dernier** (il ne doit pointer que vers des backends existants).

---

## 8. Utilisation

```bash
# 1. Vérifier la syntaxe
ansible-playbook -i inventories/hosts.yml playbooks/site.yml --syntax-check

# 2. Visualiser les cibles et les tâches
ansible-playbook -i inventories/hosts.yml playbooks/site.yml --list-hosts
ansible-playbook -i inventories/hosts.yml playbooks/site.yml --list-tasks

# 3. Déployer un tier uniquement
ansible-playbook -i inventories/aws_ec2.yml playbooks/applications.yml
ansible-playbook -i inventories/aws_ec2.yml playbooks/database.yml
ansible-playbook -i inventories/aws_ec2.yml playbooks/cache.yml

# 4. Déploiement complet
AWS_PROFILE=formation-sso ansible-playbook -i inventories/aws_ec2.yml playbooks/site.yml

# 5. Simulation sans AWS (inventaire local, exécution locale sans SSH)
ansible-playbook -i inventories/local/hosts.yml playbooks/site.yml --check --diff

# 6. Tier cloisonné sur une seule machine : un playbook par étage
ansible-playbook -i inventories/local/hosts.yml playbooks/local_lb.yml  --vault-password-file .vault_pass
ansible-playbook -i inventories/local/hosts.yml playbooks/local_cache.yml --vault-password-file .vault_pass
ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml --vault-password-file .vault_pass
ansible-playbook -i inventories/local/hosts.yml playbooks/local_db.yml  --vault-password-file .vault_pass
# ⚠️ local_cache.yml AVANT local_web.yml : le rôle php y installe
#    php8.5-redis et recharge php-fpm-web01 / php-fpm-web02.

# 7. Activer (optionnel) le worker de file d'attente — opt-in, désactivé par défaut
ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml \
  --vault-password-file .vault_pass -e laravel_queue_worker_enabled=true
```

> ℹ️ **Artéfact du mode `--check`** : sur une machine où un service n'est pas
> encore installé (`php8.5-fpm`, `backoffice`…), `--check` s'arrête sur
> `Could not find the requested service <nom>` car il n'installe rien.
> En mode réel, le paquet est installé juste avant et le service existe.
> Pour valider un tier isolément : `--limit webservers|backoffice`.

---

## 9. Réplication PostgreSQL (maître / esclave)

1. Les deux nœuds appliquent les mêmes paramètres `postgresql.conf`
   (`wal_level = replica`, `max_wal_senders = 5`, `max_replication_slots = 5`,
   `hot_standby = on`).
2. `pg_hba.conf` autorise la **réplication** depuis l'IP de chaque nœud
   `replica` (généré dynamiquement via `hostvars`). En cloisonné (une seule
   machine), la liste est fournie explicitement :
   `postgresql_replication_allowed_addresses: ['127.0.0.1/32']`.
3. **Sur le primaire** (`postgresql_node_role: primary`) :
   - création de l'utilisateur applicatif et de la base ;
   - création de l'utilisateur de réplication (`REPLICATION,LOGIN`) ;
   - création d'un **slot de réplication physique** par réplique (idempotent
     grâce à `WHERE NOT EXISTS`).
4. **Sur la réplique** (`postgresql_node_role: replica`) :
   - test de `standby.signal` (la réplique est-elle déjà initialisée ?) — c'est
     le garde d'idempotence : elle n'est **jamais** réécrasée ;
   - arrêt du cluster, purge de son répertoire de données ;
   - `pg_basebackup -X stream -S <slot> -R` : copie des données, écriture de
     `primary_conninfo` et de `standby.signal` ;
   - démarrage en mode standby.

> **`-C` (`--create-slot`) n'est pas utilisé** : le slot est créé par le primaire
> et `pg_basebackup` refuse de créer un slot déjà existant. `-S` exige seulement
> que le primaire soit passé **avant** la réplique — d'où `serial: 1` dans
> `playbooks/local_db.yml`, où primaire et réplique vivent sur la même machine.
>
> La réplique se connecte au port du **primaire** (`postgresql_primary_port`,
> 5442 en cloisonné) et non à son propre port (5443), sinon elle viserait
> son propre cluster.

> **Failover** : la promotion d'une réplique est une action d'exploitation
> (`pg_ctl promote`). Une automatisation complète nécessiterait un outil dédié
> (Patroni, repmgr, keepalived) — **à valider avec le formateur**.

---

## 10. Secrets (Ansible Vault)

`group_vars/all/vault.yml` est **volontairement vide** pour l'instant.
Il devra contenir (chiffré) :

| Variable | Usage |
|---|---|
| `vault_postgresql_password` | Mot de passe de l'utilisateur applicatif |
| `vault_postgresql_replication_password` | Mot de passe de l'utilisateur de réplication |
| `vault_laravel_db_password` | Mot de passe DB du `.env` Laravel |
| `vault_laravel_app_key` | `APP_KEY` Laravel |
| `vault_springboot_db_password` | Mot de passe DB Spring Boot |
| `vault_redis_password` | `requirepass` Redis |

```bash
ansible-vault encrypt group_vars/all/vault.yml
ansible-vault edit    group_vars/all/vault.yml
```

Les rôles les référencent de façon **tolérante** (`{{ vault_x | default('') }}`)
pour ne pas échouer tant que le vault est vide.

---

## 11. Configuration Ansible (`ansible.cfg`)

```ini
[defaults]
inventory = ./inventories/aws_ec2.yml
interpreter_python = auto_silent
stdout_callback = default       # le plugin community.general.yaml a été supprimé
result_format = yaml
remote_user = ansible
private_key_file = ~/.ssh/id_rsa
roles_path = ./roles                       # rôles locaux du projet
collections_path = ./.ansible/collections  # collections du projet

[inventory]
enable_plugins = amazon.aws.aws_ec2, host_list, auto, yaml, ini
```

### Liens symboliques indispensables

Ansible ne cherche `group_vars/` et `host_vars/` qu'**à côté du playbook et de
l'inventaire**. Comme les playbooks sont dans `playbooks/` et les variables à la
racine, deux liens rendent ces dernières visibles :

```
playbooks/group_vars -> ../group_vars
playbooks/host_vars  -> ../host_vars
```

Sans eux, `ansible-playbook` ignore **toutes** les variables de groupe et d'hôte.

---

## 12. Validation effectuée

| Vérification | Résultat |
|---|---|
| `--syntax-check` sur les 4 playbooks | ✅ OK |
| `--list-tasks` | ✅ les 8 rôles sont appelés au bon endroit |
| Précédence `group_vars` > `role defaults` | ✅ `upstream` = `web01`, `web02` |
| `host_vars` PostgreSQL | ✅ `db02 role=replica slot=db02_slot primary=db01` |
| Rendu des templates | ✅ vhost Apache, `pg_hba` (ligne de réplication), `.env` Laravel, unité systemd |
| Exécution réelle du tier DB cloisonné | ✅ `18/db01` (5442, primaire) + `18/db02` (5443, `online,recovery`) ; cluster système `18-main` (5432) **intact** |
| Réplication observée (`local_db.yml`) | ✅ `pg_stat_replication` = `127.0.0.1 \| streaming \| sent_lsn = replay_lsn` ; slot `db02_slot` `active = t` |
| Test de bout en bout | ✅ ligne insérée sur le primaire **visible** sur la réplique ; écriture sur la réplique refusée (`read-only transaction`) |
| Authentification applicative | ✅ `psql -h 127.0.0.1 -p 5442 -U laravel -d laravel` (scram-sha-256) |
| Idempotence | ✅ `local_db.yml` rejoué : `changed=0` sur `db01` **et** `db02` |
| Câblage Laravel ↔ PostgreSQL | ✅ `.env` des 2 instances web : `pgsql` → `127.0.0.1`**`5442`**/`laravel` (jamais le 5432 du système) ; `php artisan db:show` = PostgreSQL 18.6, port 5442 |
| Migrations de schéma | ✅ 3 migrations jouées **une seule fois** (`run_once`, instances partageant la base) ; 9 tables créées et **visibles à l'identique sur la réplique** (5443) |
| `APP_KEY` | ✅ issue du vault (`vault_laravel_app_key`), identique sur `web01`/`web02` et **stable entre deux exécutions** (`key:generate` sauté) |
| Non-régression | ✅ HTTP 200 sur `9081` et `9082` ; `18-main` (5432) toujours `online` |
| Exécution réelle du tier cache cloisonné (`local_cache.yml`) | ✅ `redis-redis01` **active** + **enabled**, port **6390** en écoute, config `/etc/redis-redis01/redis.conf`, utilisateur `redis01` ; méta-service `redis-server` **inactive** (pids d'origine conservés) |
| Cloisonnement du cache | ✅ `redis-cli -p 6390 ping` = `NOAUTH Authentication required.` puis `PONG` avec `REDISCLI_AUTH` ; le Redis étranger **6379** et le sentinel **26379** répondent toujours (`PONG`, pids inchangés) |
| Contrôles du rôle | ✅ debug final : version `8.0.5`, pings `NOAUTH`/`PONG`, `Réplication : role:master connected_slaves:0`, bases logiques listées |
| Idempotence cache | ✅ `local_cache.yml` rejoué : **`changed=0`** |
| `php8.5-redis` | ✅ installé sur `web01` **et** `web02` ; `php --ri redis` → `Redis Support => enabled` (phpredis 6.2.0) |
| Câblage Laravel ↔ Redis | ✅ `.env` des 2 instances : `SESSION_DRIVER` / `CACHE_STORE` / `QUEUE_CONNECTION` = **`redis`**, `REDIS_HOST=127.0.0.1`, `REDIS_PORT=6390`, `REDIS_PASSWORD` = secret du vault |
| Sessions / cache / queue en Redis | ✅ requête HTTP → cookie `laravel-web01-session` **et** clé de session en Redis (payload + TTL ≈ 120 min) ; `Cache::put/get` atterrit en **db1** ; job poussé → `LLEN …queues:default` = **1** (payload JSON) |
| Worker de file **opt-in** | ✅ `laravel_queue_worker_enabled: false` ⇒ tâches `[worker]` **skipped**, aucune unité `laravel-queue-*` ; le job reste dans la file (aucun consommateur) |
| Idempotence web | ✅ `local_web.yml` rejoué : **`changed=0`** sur `web01` **et** `web02` |
| Non-régression (étape 7) | ✅ HTTP 200 sur `9081`/`9082` ; les 3 clusters PostgreSQL (`18-main`, `18/db01`, `18/db02`) toujours `online` ; réplication `db01 → db02` = `streaming`, LSN primaire = LSN de replay |

---

## 13. Points d'attention / TODO

- [ ] **Créer et taguer les instances EC2** (`Role=lb|web|backoffice|db|redis`).
- [ ] **Renommer les `host_vars`** (`db01`/`db02`) selon les tags `Name` réels.
- [ ] **Remplir puis chiffrer `vault.yml`** (voir §10).
- [ ] **Trancher l'utilisateur SSH** : `remote_user = ansible` vs `ansible_user: ubuntu`.
- [ ] **Renseigner `laravel_repo`** avec le vrai dépôt (placeholder `TON_COMPTE/TON_PROJET.git` ; le clonage est sauté automatiquement tant qu'il n'est pas remplacé).
- [x] **`php_version: "8.5"` validé** sur Ubuntu 26.04 — les 9 paquets `php8.5-*` sont natifs (`8.5.4-0ubuntu1.3`), aucun dépôt externe requis ; un PPA type `ondrej/php` ne serait nécessaire que sur Ubuntu < 26.04.
- [ ] **Choisir la stratégie de failover** PostgreSQL (outil dédié ou manuel).
- [ ] Migrer `tags.*` → `ec2_tags.*` (dépréciation `amazon.aws` après 2026-12-01).
- [ ] Chiffrer le vault à partir du gabarit `group_vars/all/vault.yml.example`
      (`ansible-vault encrypt group_vars/all/vault.yml`)
- [ ] Déposer le JAR dans `roles/springboot/files/` et renseigner `springboot_jar_file`

---

## 14. Conventions du projet (règles du formateur)

| # | Règle | Mise en œuvre dans ce dépôt |
|---|---|---|
| **1** | **Faire le manuel avant d'automatiser** | → [`docs/installation-manuelle.md`](docs/installation-manuelle.md) : procédure shell pour chaque tier, **tableau de correspondance *commande manuelle ↔ tâche Ansible*** et checklist de validation. C'est la référence à comparer avec la sortie du `--check`. |
| **2** | **Sécuriser tokens, mots de passe, clés** | • Gabarit non chiffré : `group_vars/all/vault.yml.example`<br>• `no_log: true` sur les 7 tâches traitant des secrets (user PG applicatif, user de réplication, `pg_basebackup`/`PGPASSWORD`, injection du mot de passe dans `primary_conninfo`, environnement Spring Boot, template Redis)<br>• `.gitignore` : `.vault_pass`, `*.vault`, `vault_password*`, `*.key`, `*.pem`, `id_rsa*`<br>• Clé SSH surchargeable par `ANSIBLE_PRIVATE_KEY_FILE` |
| **3** | **Templates `*.conf.j2` pour les fichiers de configuration** | **Aucun `lineinfile` / `blockinfile`** dans les rôles — contrôlé par `grep`. **12 templates Jinja2** dans `roles/*/templates/` (dont `ondrej-java.sources.j2` pour le dépôt APT et `java_env.sh.j2` pour `/etc/profile.d/`), avec des **conditions `{% if %}`** : SSL Apache, OPcache PHP, TLS Redis, `requirepass`, rôle primaire/réplique PostgreSQL, `JAVA_HOME` dans l'environnement Spring Boot. |
| **4** | **Fichiers statiques dans `roles/<rôle>/files/`** | `ansible.builtin.copy` lit dans le dossier `files/` du rôle (ex. `roles/springboot/files/` pour le JAR, avec son propre `README.md` expliquant la règle et ses limites Git). |

### Ce que « utiliser un module `builtin` » veut dire

`builtin` **n'est pas un gros mot** : `apt`, `service`, `user`, `file`
(gestion d'un lien symbolique), `command` restent les bons modules.

La règle vise **l'écriture du contenu d'un fichier de configuration** : ce
travail revient **toujours** à `ansible.builtin.template`, seul outil qui
permet de **variabiliser** et de **conditionner** la configuration.

| À éviter | À utiliser |
|---|---|
| `ansible.builtin.lineinfile` sur un `.conf` / `.ini` / `.properties` | `ansible.builtin.template` + `templates/<fichier>.conf.j2` |
| `ansible.builtin.copy` sur un fichier modifiable | `ansible.builtin.template` + `templates/<fichier>.conf.j2` |
| `ansible.builtin.copy` sur un binaire / PDF / image | `ansible.builtin.copy` + `files/<fichier>` ✅ |



