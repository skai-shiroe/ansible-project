# Architecture 3-tiers — Automatisation Ansible

Automatisation du déploiement d'une architecture 3-tiers sur AWS :
une application **web publique Laravel**, une application d'administration
**Backoffice Spring Boot** et une couche de données **PostgreSQL en
maître/esclave**, avec **Redis** comme cache et **Nginx** comme équilibreur
de charge.

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
│   └── gcp_compute.yml
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
│   └── cache.yml                # Redis
└── roles/                       # 8 rôles autonomes
    ├── nginx_lb/  ├── apache/  ├── php/       ├── laravel/
    ├── java/      ├── springboot/  ├── postgresql/  └── redis/
```

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

---

## 5. Convention de placement des variables

C'est la règle structurante du projet : **chaque variable est placée selon sa
portée**.

| Emplacement | Nature | Exemples |
|---|---|---|
| `roles/<r>/defaults/main.yml` | Paramètre **surchargeable** (contrat d'entrée du rôle) | `apache_port: 80`, `php_version: "8.3"` |
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

---

## 6. Les 8 rôles

| Rôle | État | Ce qu'il fait |
|---|---|---|
| `nginx_lb` | **Nouveau** | Installe Nginx, génère l'`upstream` vers les instances web, `least_conn`, health-check `/up`, rechargement à chaud |
| `apache` | **Refactoré** | Installe Apache, active les modules (`rewrite`, `proxy_fcgi`…), configure le port 8080 et le vhost Laravel (délégation PHP à PHP-FPM) |
| `php` | **Refactoré** | Versionné (`php{{ php_version }}`), configure `php.ini`, gère le service PHP-FPM |
| `laravel` | **Complété** | Déploie le code, génère le `.env`, `composer install`, `key:generate`, permissions |
| `java` | **Nouveau** | Installe OpenJDK 21 (headless), vérifie la version |
| `springboot` | **Nouveau** | Utilisateur système dédié, déploiement du JAR, unité **systemd**, fichier d'environnement |
| `postgresql` | **Nouveau** | Installation, `postgresql.conf`/`pg_hba.conf`, **utilisateur + slots de réplication** (maître), **`pg_basebackup` + standby** (esclave) |
| `redis` | **Nouveau** | Installe et configure Redis (mémoire, politique d'éviction, `requirepass`) |

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

# 5. Validation hors AWS (inventaire statique)
ansible-playbook -i inventories/hosts.yml playbooks/site.yml --check --diff
```

---

## 9. Réplication PostgreSQL (maître / esclave)

1. Les deux nœuds appliquent les mêmes paramètres `postgresql.conf`
   (`wal_level = replica`, `max_wal_senders = 5`, `max_replication_slots = 5`,
   `hot_standby = on`).
2. `pg_hba.conf` autorise la **réplication** depuis l'IP de chaque nœud
   `replica` (généré dynamiquement via `hostvars`).
3. **Sur le primaire** (`postgresql_node_role: primary`) :
   - création de l'utilisateur applicatif et de la base ;
   - création de l'utilisateur de réplication (`REPLICATION,LOGIN`) ;
   - création d'un **slot de réplication physique** par réplique (idempotent
     grâce à `WHERE NOT EXISTS`).
4. **Sur la réplique** (`postgresql_node_role: replica`) :
   - test de `standby.signal` (la réplique est-elle déjà initialisée ?) ;
   - `pg_basebackup -X stream -C -S <slot> -R` : copie des données, écriture de
     `primary_conninfo` et de `standby.signal` ;
   - démarrage en mode standby.

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

---

## 13. Points d'attention / TODO

- [ ] **Créer et taguer les instances EC2** (`Role=lb|web|backoffice|db|redis`).
- [ ] **Renommer les `host_vars`** (`db01`/`db02`) selon les tags `Name` réels.
- [ ] **Remplir puis chiffrer `vault.yml`** (voir §10).
- [ ] **Trancher l'utilisateur SSH** : `remote_user = ansible` vs `ansible_user: ubuntu`.
- [ ] **Renseigner `laravel_repo`** (placeholder `TON_COMPTE/TON_PROJET.git`).
- [ ] **Valider `php_version: 8.5`** (nécessite un dépôt externe, type PPA `ondrej`).
- [ ] **Choisir la stratégie de failover** PostgreSQL (outil dédié ou manuel).
- [ ] Migrer `tags.*` → `ec2_tags.*` (dépréciation `amazon.aws` après 2026-12-01).


