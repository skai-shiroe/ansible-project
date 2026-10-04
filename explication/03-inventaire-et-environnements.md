# 03 — Inventaires et environnements

## Les 4 inventaires du dépôt

| Fichier | Type | Usage | État |
|---|---|---|---|
| `inventories/aws_ec2.yml` | **dynamique** (`amazon.aws.aws_ec2`) | déploiement AWS réel | complet |
| `inventories/hosts.yml` | statique | validation syntaxe/list-tasks **sans AWS** | complet |
| `inventories/local/hosts.yml` | statique, `ansible_connection: local` | **lab cloisonné** (une machine) | complet |
| `inventories/gcp_compute.yml` | — | prévu (commentaire `ansible.cfg`) | **fichier vide (0 octet) — non implémenté** |

`ansible.cfg` définit l'inventaire par défaut :
`inventory = ./inventories/aws_ec2.yml` — tout lancer **sans `-i`** part sur
AWS. Les jeux locaux passent **systématiquement** par `-i inventories/local/hosts.yml`.

---

## 1. Inventaire dynamique AWS (`aws_ec2.yml`)

- **Plugin** : `amazon.aws.aws_ec2`, région `eu-north-1`, filtre
  `instance-state-name: running`.
- **Groupes** :
  - `keyed_groups` : `env_*` (tag `Environment`), `role_*` (tag `Role`),
    `type_*`, `region_*`.
  - `groups:` : `webservers` ← `'web' in tags.Role`, `databases` ← `'db'`,
    `loadbalancers` ← `'lb'`, `backoffice` ← `'backoffice'`, `redis` ← `'redis'`,
    `all_ec2` ← `'ec2' in group_names`.
- **Nom d'hôte** : `tag:Name` → `dns-name` → `private-ip-address`.
- **SSH** : `ansible_host = public_ip_address | default(private_ip_address)`.
- **Cache** : `jsonfile` sur `/tmp/ansible_aws_ec2_cache`, 600 s.

Prérequis (`README.md` §2) : `aws sso login --profile formation-sso`.

> **TODO connu** (`README.md` §13) : migrer `tags.*` → `ec2_tags.*`
> (dépréciation `amazon.aws` postérieure au 2026-12-01).

## 2. Inventaire statique de validation (`hosts.yml`)

Mêmes noms d'hôtes que les tags EC2 (`lb01`, `web01`, `web02`, `bo01`,
`db01`, `db02`, `redis01`), adresses `10.0.0.x` **documentaires** :

```bash
ansible-inventory -i inventories/hosts.yml --graph
ansible-playbook -i inventories/hosts.yml playbooks/site.yml --syntax-check
```

## 3. Inventaire local cloisonné (`local/hosts.yml`)

```yaml
all:
  vars:
    ansible_connection: local          # pas de SSH
    ansible_python_interpreter: /usr/bin/python3
  children:
    loadbalancers: { lb01 : nginx_lb_instance: lb01, nginx_lb_listen_port: 9080, ... }
    webservers:    { web01: apache_instance: web01, apache_port: 9081, ... }
                   { web02: apache_instance: web02, apache_port: 9082, ... }
    backoffice:    { bo01:  ansible_host: 10.0.0.20 }   # ⚠ aucune autre variable
    databases:     { db01, db02 }                       # variables dans host_vars local
    redis:         { redis01: redis_instance: redis01, redis_port: 6390, ... }
```

- Les `ansible_host: 10.0.0.x` ne servent que de **documentation de topologie**.
- Les noms `web01`/`web02` doivent être résolubles (le LB y pointe son
  upstream) : le dépôt ne gère **pas** `/etc/hosts` (aucune tâche ne le modifie
  — vérifié par grep ; l'entrée doit exister côté machine).
- Les variables **inline** par hôte écrasent sans ambiguïté les `group_vars/` racine.

### Variables cloisonnées par hôte (aperçu ; détail dans chaque doc de rôle)

| Hôte | Variables inline principales |
|---|---|
| `lb01` | `nginx_lb_instance: lb01`, `nginx_lb_service: nginx-lb01`, `nginx_lb_listen_port: 9080`, `nginx_lb_user/group: lb01` |
| `web01`/`web02` | `ansible_remote_tmp: /tmp`, `apache_instance`, `apache_service`, `apache_port` (9081/9082), `apache_user/group`, `apache_document_root`, `apache_php_fpm_socket`, `laravel_path`, `php_instance`, `php_fpm_service`, `php_fpm_user/group`, `laravel_*` (repo `laravel/laravel` branche `13.x`, `laravel_clean_deploy: true`, `laravel_run_migrations: true`, DB `127.0.0.1:5442`, Redis `127.0.0.1:6390`) |
| `redis01` | `redis_instance: redis01`, `redis_service: redis-redis01`, `redis_port: 6390`, `redis_bind: 127.0.0.1`, `redis_user/group: redis01`, `redis_dir`, `redis_pid_file`, `redis_maxmemory: 128mb`, `redis_password: "{{ vault_redis_password }}"` |
| `db01`/`db02` | **aucune inline** → `inventories/local/host_vars/db0X.yml` + `include_vars` |

## 4. Les deux `host_vars` PostgreSQL

| Fichier | Rôle | Contenu (racine = **AWS**) | Contenu (**local**) |
|---|---|---|---|
| `host_vars/db01.yml` (racine, servi par `playbooks/host_vars/`) | primaire | `postgresql_node_role: primary` | — |
| `host_vars/db02.yml` (racine) | réplique | `postgresql_node_role: replica`, `postgresql_primary_host: db01`, `postgresql_replication_slot: db02_slot` | — |
| `inventories/local/host_vars/db01.yml` | primaire | — | `postgresql_instance: db01`, `version: "18"`, `port: 5442`, `service: postgresql@18-db01`, `create_cluster: true`, `node_role: primary`, `replication_allowed_addresses: [127.0.0.1/32]`, dimensionnement réduit (`100` / `256MB` / `1GB`) |
| `inventories/local/host_vars/db02.yml` | réplique | — | `postgresql_instance: db02`, `version: "18"`, `port: 5443`, `service: postgresql@18-db02`, `create_cluster: true`, `node_role: replica`, `primary_host: 127.0.0.1`, `primary_port: 5442`, `replication_slot: db02_slot`, idem dimensionnement |

⚠ **Le nom de slot doit être identique dans les deux fichiers** : le primaire
(db01) est configuré **avant** db02 (`serial: 1`) et lit le nom du slot dans
`playbooks/host_vars/db02.yml` — seule source visible à ce moment-là.

## 5. Liens symboliques indispensables

```
playbooks/group_vars -> ../group_vars
playbooks/host_vars  -> ../host_vars
```

Ansible ne cherche `group_vars/`/`host_vars` qu'à côté du **playbook** et de
l'**inventaire**. Sans ces liens, un playbook lancé depuis `playbooks/`
ignorerait **toutes** les variables de groupe et d'hôte (`README.md` §11).

## 6. Environnements et vault

| | AWS | Local |
|---|---|---|
| Fichier de secrets | `group_vars/all/vault.yml` (**vide**, 0 octet) | `inventories/local/group_vars/all/vault.yml` (**chiffré**, 5 clés) |
| Clés | à remplir d'après `vault.yml.example` (6 clés) | `vault_postgresql_password`, `vault_postgresql_replication_password`, `vault_laravel_db_password`, `vault_laravel_app_key`, `vault_redis_password` |
| Déverrouillage | `ansible-vault encrypt/edit` | `--vault-password-file .vault_pass` (`.gitignore`, mode `0600`) |

→ Suite : [04 — Apache](04-apache.md)

