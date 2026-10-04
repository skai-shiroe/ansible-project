# 10 — Sécurité et secrets

## 1. Les deux coffres (Ansible Vault)

| | AWS | Local cloisonné |
|---|---|---|
| Fichier | `group_vars/all/vault.yml` | `inventories/local/group_vars/all/vault.yml` |
| État | **vide** (0 octet) — le déploiement AWS n'exige aucun vault | **chiffré** (ansible-vault) |
| Gabarit non chiffré | `group_vars/all/vault.yml.example` (6 clés, valeurs `CHANGE_ME_*`) | — |
| Mot de passe | `ansible-vault encrypt/edit` (interactif) | `.vault_pass` à la racine (mode `0600`, **gitignoré**) |
| Utilisation | `ansible-vault …` | `--vault-password-file .vault_pass` |

### Clés

| Clé `vault_*` | Utilisée par | Dans le vault local ? |
|---|---|---|
| `vault_postgresql_password` | `group_vars/databases.yml` → `postgresql_password` | ✅ |
| `vault_postgresql_replication_password` | `group_vars/databases.yml` → `postgresql_replication_password` | ✅ |
| `vault_laravel_db_password` | `roles/laravel/defaults` → `laravel_db_password` | ✅ |
| `vault_laravel_app_key` | `roles/laravel/defaults` → `laravel_app_key` | ✅ |
| `vault_redis_password` | `group_vars/redis.yml` + inventaire local → `redis_password` | ✅ |
| `vault_springboot_db_password` | `group_vars/backoffice.yml` → `springboot_datasource_password` | ❌ absent (backoffice non déployé en local) |

> Dans cette documentation, toute valeur de secret est notée **`<SECRET>`**.
> Les rôles lisent les secrets de façon tolérante
> (`{{ vault_x | default('') }}`) : un vault vide ne fait jamais échouer.

### `.gitignore` (extrait — secrets jamais commités en clair)

```
.vault_pass
*.vault
vault_password*
*.pem
*.key
id_rsa*
```

Seuls sont versionnés : `group_vars/all/vault.yml` (vide), le gabarit
`.example`, et `inventories/local/group_vars/all/vault.yml` (**chiffré**).

---

## 2. Les 9 tâches `no_log: true` (vérifié par grep)

| # | Rôle | Tâche | Ce qui est masqué |
|---|---|---|---|
| 1 | `laravel` | template `env.j2` | `DB_PASSWORD`, `APP_KEY`, `REDIS_PASSWORD` |
| 2 | `postgresql` | création utilisateur applicatif | mot de passe PG |
| 3 | `postgresql` | création utilisateur de réplication | mot de passe `replicator` |
| 4 | `postgresql` | `pg_basebackup` | `PGPASSWORD` en environnement |
| 5 | `postgresql` | injection `password=` dans `primary_conninfo` | mot de passe en clair dans la ligne |
| 6 | `redis` | template `redis.conf` (mode AWS) | `requirepass` |
| 7 | `redis` | template `redis.conf` (mode instance) | `requirepass` |
| 8 | `redis` | `redis-cli ping` authentifié | `REDISCLI_AUTH` |
| 9 | `springboot` | template `springboot.env.j2` | `SPRING_DATASOURCE_PASSWORD` |

> Le `README.md` §14 en compte **7** : les tâches 7 et 8 ont été ajoutées
> lors du mode cloisonné (écart documenté, pas de modification du README).

---

## 3. `REDISCLI_AUTH` plutôt que `redis-cli -a`

```bash
# ❌ JAMAIS — le mot de passe apparaît dans « ps », lisible par tout utilisateur
redis-cli -p 6390 -a '<SECRET>' ping

# ✅ — variable d'environnement (ce que fait le rôle Ansible)
REDISCLI_AUTH='<SECRET>' redis-cli -p 6390 ping
```

C'est la raison pour laquelle la tâche du rôle `redis` porte `no_log`.

---

## 4. Permissions des fichiers sensibles

| Fichier | Mode | Propriétaire | Contenu |
|---|---|---|---|
| `/var/www/web0X/.env` | `0600` | `web0X` | `DB_PASSWORD`, `APP_KEY`, `REDIS_PASSWORD` |
| `inventories/local/group_vars/all/vault.yml` | chiffré | — | 5 secrets |
| `.vault_pass` | `0600` | vous | mot de passe du vault |
| `/etc/redis-redis01/redis.conf` | `0640` | `redis01` | `requirepass` |
| `/etc/postgresql/18/db0X/pg_hba.conf` + `conf.d/10-ansible.conf` | `0640` (`0644` instance) | `postgres` | règles d'accès |
| `/opt/backoffice/backoffice.env` (prévu) | `0600` | `springboot` | datasource password |
| `/var/www/laravel/.env` (AWS) | `0600` | `www-data` | idem |

**Conséquence pratique** : un script test doit s'exécuter **en tant que
l'utilisateur du service** :

```bash
sudo -u web01 php /var/www/web01/artisan db:show   # ✅ (lui lit .env 0600)
php /var/www/web01/artisan db:show                  # ❌ permission denied si root/autre
```

---

## 5. Utilisateurs système (jamais root pour les apps)

| Utilisateur | Shell | Créé par | Sert à |
|---|---|---|---|
| `web01`, `web02` | `/usr/sbin/nologin` | rôle `apache` (puis réutilisé par `php`, `laravel`) | workers Apache, workers FPM, propriétaire de l'app |
| `lb01` | `/usr/sbin/nologin` | rôle `nginx_lb` | master Nginx du LB |
| `redis01` | `/usr/sbin/nologin` | rôle `redis` | serveur Redis du projet |
| `postgres` | système | paquet PostgreSQL | clusters PG (via `become_user: postgres`) |
| `springboot` (prévu) | `/usr/sbin/nologin` | rôle `springboot` | application backoffice |
| `www-data` | système | paquet | mode AWS uniquement |

`nologin` + `create_home: false` ⇒ deux conséquences gérées dans le code :
`ansible_remote_tmp: /tmp` (inventaire local) et `unset HOME` dans
`envvars.j2` (Apache).

## 6. Durcissement systemd (unités du projet)

Toutes les unités déployées par le projet portent :

```ini
NoNewPrivileges=true
ProtectHome=true
ProtectSystem=full
PrivateTmp=true
Restart=on-failure
```

- `apache.service.j2`, `nginx.service.j2`, `php-fpm.service.j2`,
  `redis.service.j2`, `queue-worker.service.j2` : + `RuntimeDirectory` dédié.
- `redis.service.j2` : + `UMask=0007`, `Type=notify`.
- `springboot.service.j2` : hardening **absent** (unité d'origine, non
  refactorée — voir [18-springboot.md](18-springboot.md)).

## 7. Réseau : ce qui est exposé

| Service | Bind local | Exposition |
|---|---|---|
| Redis projet (6390) | `127.0.0.1` | boucle locale uniquement |
| Redis étranger (6379) | `127.0.0.1` + `::1` | hors périmètre (ne pas toucher) |
| PostgreSQL 5442/5443 | `*` (fragment `listen_addresses`) | interfaces — filtré par `pg_hba` (scram) |
| PostgreSQL système 5432 | `127.0.0.1` | hors périmètre |
| Apache web01/02 (9081/9082) | toutes interfaces | derrière le LB |
| Nginx LB (9080) | toutes interfaces (ou `nginx_lb_bind_address`) | point d'entrée |
| Réplication PG | — | `pg_hba` : `127.0.0.1/32` uniquement |

`postgresql_allowed_networks: [0.0.0.0/0]` (AWS) est un réglage de
développement — **à restreindre au VPC en production** (commentaire du
`group_vars/databases.yml`).

## 8. Authentifications résumées

| Lien | Mécanisme | Secret |
|---|---|---|
| App → PostgreSQL | `scram-sha-256` | `DB_PASSWORD` = `<SECRET>` (vault) |
| Réplique → primaire (réplication) | `scram-sha-256` + slot physique | `PGPASSWORD` = `<SECRET>` |
| App → Redis | `requirepass` | `REDIS_PASSWORD` = `<SECRET>` |
| `psql` local (superuser) | `peer` (unix socket, user `postgres`) | aucun |
| Laravel (sessions) | cookie signé | `APP_KEY` = `<SECRET>` (vault) |
| SSH (AWS) | clé privée | `ANSIBLE_PRIVATE_KEY_FILE` / `~/.ssh/id_rsa` |

## 9. Recettes

| Je veux… | Commande / fichier |
|---|---|
| Ajouter un secret | 1. le nommer `vault_<usage>` dans `vault.yml.example` ; 2. l'ajouter au vault de l'environnement ; 3. le consommer via `{{ vault_x \| default('') }}` dans `defaults` ou `group_vars` |
| Chiffrer le vault AWS | `ansible-vault encrypt group_vars/all/vault.yml` |
| Vérifier qu'aucun secret n'est en clair | `git grep -iE 'password.*: .[^{<]' -- '*.yml'` (hors vault) ; vérifier `git status` avant commit |
| Lire le vault local | `ansible-vault view inventories/local/group_vars/all/vault.yml --vault-password-file .vault_pass` |
| Tester l'app | `sudo -u web01 php /var/www/web01/artisan …` |

## 10. Non trouvé dans le code actuel

- Pas de rotation automatique des secrets ni de date d'expiration.
- Pas de gestion de certificats TLS (ACME/Let's Encrypt) : snakeoil en AWS,
  aucun TLS au LB local.
- Pas de `vault_springboot_db_password` dans le vault local (backoffice absent).
- Pas de policy IAM / Security Group dans le dépôt (hors périmètre Ansible ici).
- Aucune tâche ne chiffre/déchiffre un fichier hors `group_vars` (pas de
  `ansible.builtin.copy` avec `vault` embarqué).

## 11. Références

- [`../README.md`](../README.md) — §10 (secrets), §14 (règle « Sécuriser tokens… »).
- [`../docs/installation-manuelle.md`](../docs/installation-manuelle.md) — étapes « SECRET ».
- [`../group_vars/all/vault.yml.example`](../group_vars/all/vault.yml.example)
- [15 — Dépannage](15-depannage.md) (symptômes liés aux permissions/vault).

## 12. Résumé

Un vault par environnement (AWS vide à remplir, local chiffré + `.vault_pass`
gitignoré), **9 tâches masquées** par `no_log`, `REDISCLI_AUTH` au lieu de
`-a`, `.env` en `0600` lu par l'utilisateur du service, users `nologin`
partout, hardening systemd sur toutes les unités projet. Aucun secret n'apparaît
dans cette documentation : uniquement `<SECRET>` et les noms `vault_*`.

