# 07 — Rôle `postgresql`

## 1. À quoi sert ce rôle ?

Installe PostgreSQL et gère un cluster en **maître/esclave** : création du
cluster Debian dédié (mode cloisonné), configuration réseau/réplication
(fragment `conf.d` + `pg_hba.conf`), utilisateur et base applicatifs,
utilisateur et slots de réplication côté primaire, `pg_basebackup` + standby
côté réplique, vérifications de réplication et de lecture seule.

## 2. Où il vit dans le dépôt

```
roles/postgresql/
├── defaults/main.yml      # version, ports, chemins calculés, réplication…
├── vars/main.yml          # vidé (seul « --- »)
├── tasks/main.yml         # 338 lignes (AWS + instance + primaire + réplique + checks)
├── handlers/main.yml      # « Redémarrer PostgreSQL »
├── templates/
│   ├── conf.d/10-ansible.conf.j2  # listen, port, dimensionnement, wal_level, read_only
│   └── pg_hba.conf.j2             # peer local, scram, réplication
└── README.md
```

## 3. Comment il est appelé

- **AWS** : `playbooks/database.yml` → `hosts: databases`, `roles: [postgresql]`.
- **Local** : `playbooks/local_db.yml` — **`serial: 1`** (db01 puis db02) +
  `pre_tasks: include_vars: inventories/local/host_vars/{{ inventory_hostname }}.yml`
  (préséance maximale, cf. [02](02-ansible-variables-et-precedence.md)).
- Rôle du nœud : `postgresql_node_role` (`primary`/`replica`) dans `host_vars/db0X.yml`.

## 4. Prérequis et dépendances

- Paquets : `postgresql-{{ version }}`, `postgresql-contrib-{{ version }}`,
  `python3-psycopg2` (installés par le rôle).
- **`pg_createcluster` exige root** (écrit dans `/etc/postgresql`, appelle
  `initdb` via `su`) : la tâche utilise `become: true` **sans** `become_user: postgres`.
- Secrets : `vault_postgresql_password`, `vault_postgresql_replication_password`
  (et, pour la réplique, `PGPASSWORD`).
- Collections : `community.postgresql` (`postgresql_user`, `postgresql_db`,
  `postgresql_query`) — **utilisée mais non déclarée** dans `requirements.yml`
  (voir §17).
- `meta/main.yml` : `dependencies: []`.

## 5. Ports, services et chemins

| Élément | Mode AWS | Mode local db01 / db02 |
|---|---|---|
| Port | `5432` | `5442` (primaire) / `5443` (réplique) |
| Service | `postgresql` (**méta-service**) | `postgresql@18-db01` / `postgresql@18-db02` |
| Cluster Debian | `main` (version `17`) | `db01` / `db02` (version `18`) |
| Config | `/etc/postgresql/17/main/` | `/etc/postgresql/18/db01/` / `18/db02/` |
| Données | `/var/lib/postgresql/17/main` | `/var/lib/postgresql/18/db01` / `18/db02` |
| Fragment | `conf.d/10-ansible.conf` (0640, postgres) | idem (0644 en instance) |
| pg_hba | `pg_hba.conf` (0640) | idem (0644 en instance) |
| Socket | `/var/run/postgresql` (partagé par tous les clusters Debian) | idem |
| Binaires | `/usr/lib/postgresql/17/bin` | `/usr/lib/postgresql/18/bin` |

**Interdit absolu** : aucun appel au méta-service `postgresql` en mode
cloisonné (il démarrerait aussi `18-main` du port 5432).

## 6. Variables

| Variable | Valeur actuelle (AWS → local) | Fichier | Pourquoi | Exemple de modification | Impact |
|---|---|---|---|---|---|
| `postgresql_instance` | `""` → `db01`/`db02` | `defaults` → `inventories/local/host_vars/db0X.yml` | Bascule mode | — | gate de toutes les tâches d'instance |
| `postgresql_cluster` | `main` → `db01`/`db02` | `defaults` | Nom du cluster Debian | — | chemins, service |
| `postgresql_service` | `postgresql` → `postgresql@18-db0X` | `defaults` → host_vars local | **Jamais le méta-service en local** | — | démarrage/handler |
| `postgresql_create_cluster` | `false` → `true` (local) | `defaults` → host_vars local | `pg_createcluster` au 1er run local | — | création du cluster |
| `postgresql_version` | `"17"` → `"18"` (local) | `group_vars/databases.yml` → host_vars local | Clusters locaux en PG 18 | — | paquets, chemins, binaires |
| `postgresql_port` | `5432` → `5442`/`5443` | `group_vars/databases.yml` → host_vars local | Ports dédiés | — | fragment, `login_port`, basebackup |
| `postgresql_primary_port` | `{{ postgresql_port }}` → `5442` (db02) | `defaults` → host_vars local | La réplique vise le port du **primaire** | — | `pg_basebackup -p` |
| `postgresql_listen_addresses` | `*` | `group_vars/databases.yml` | Écoute réseau | — | fragment |
| `postgresql_conf_dir` / `data_dir` / `hba_file` / `conf_file` | calculés (`…/<ver>/main` → `…/<ver>/<cluster>`) | `defaults` | Cloisonnement | — | tous les dépôts |
| `postgresql_socket_dir` | `/var/run/postgresql` | `defaults` | Socket partagé Debian | — | `login_unix_socket` des modules |
| `postgresql_replication_allowed_addresses` | `[]` (déduit les IPs des répliques via `hostvars`) → `[127.0.0.1/32]` (local) | `defaults` → host_vars local | Même machine = boucle locale | — | ligne `host replication` du `pg_hba` |
| `postgresql_database` / `_user` / `_password` | `laravel` / `laravel` / vault | `group_vars/databases.yml` (+ vault) | Base applicative | vault | utilisateur + base primaire |
| `postgresql_replication_user` / `_password` | `replicator` / vault | idem | Réplication | vault | user `REPLICATION,LOGIN` |
| `postgresql_replication_slot` | `""` → `db02_slot` | `defaults` → `host_vars/db02.yml` (racine, lu par le primaire) | Slot par réplique | — | `pg_create_physical_replication_slot` |
| `postgresql_primary_host` | `groups['databases'][0]` → `db01` (AWS) / `127.0.0.1` (local) | `defaults` → host_vars local | Cible du basebackup | — | `pg_basebackup -h` |
| `postgresql_node_role` | `primary` (defaults) → `primary`/`replica` par hôte | `defaults` → `host_vars/db0X.yml` | Topologie | — | gate primaire/réplique |
| `postgresql_allowed_networks` | `[0.0.0.0/0]` | `group_vars/databases.yml` | Accès applicatifs | restreindre au VPC | `pg_hba` |
| `postgresql_max_connections` / `_shared_buffers` / `_effective_cache_size` | AWS `200/512MB/2GB` → local `100/256MB/1GB` | `group_vars` → host_vars local | Dimensionnement (poste vs serveur) | — | fragment `conf.d` |
| `postgresql_wal_level` / `_max_wal_senders` / `_max_replication_slots` / `_hot_standby` | `replica` / `5` / `5` / `on` | `defaults` | Réplication physique | — | fragment |
| `postgresql_packages` | paquets versionnés `postgresql-<ver>` + `-contrib`, `python3-psycopg2` | `defaults` | Installation + driver Python | — | `apt` |

## 7. Templates

| Template | Destination | Contenu clé |
|---|---|---|
| `conf.d/10-ansible.conf.j2` | `<conf_dir>/conf.d/10-ansible.conf` | `listen_addresses`, `port`, `max_connections`, `shared_buffers`, `effective_cache_size`, `wal_level`, `max_wal_senders`, `max_replication_slots`, `hot_standby`, **`default_transaction_read_only = on/off`** selon le rôle |
| `pg_hba.conf.j2` | `<conf_dir>/pg_hba.conf` | `local … peer`, `127.0.0.1/32` et `::1/128` en `scram-sha-256`, réseaux applicatifs, **lignes `host replication`** (liste explicite si fournie, sinon déduite des hôtes `replica` du groupe `databases`) |

Aucun `lineinfile` : tout passe par des templates (règle du formateur).

## 8. Handlers

**`Redémarrer PostgreSQL`** : `systemd` + `daemon_reload: true`,
`state: restarted`, tolérant `--check`. Déclenché par : fragment, `pg_hba`,
création de cluster, `pg_basebackup`, `primary_conninfo`.
**`meta: flush_handlers`** est appelé **avant** les opérations de réplication :
la configuration doit être appliquée quand la réplique se connecte.

## 9. Parcours des tâches

**Commun** : `apt` → (AWS : fragment + pg_hba + démarrage | instance : stat
cluster → `pg_createcluster` si absent → fragment + pg_hba → démarrage) →
**`flush_handlers`**.

**Primaire** (`node_role == 'primary'`) : créer l'utilisateur applicatif
(`no_log`) → créer la base → créer `replicator` (`REPLICATION,LOGIN`, `no_log`)
→ créer les slots (`pg_create_physical_replication_slot` avec `WHERE NOT
EXISTS`, en boucle sur les hôtes `replica` du groupe `databases`).

**Réplique** (`node_role == 'replica'`) : `stat standby.signal` (garde
d'idempotence) → **si absent** : arrêt du cluster → purge du répertoire de
données → `pg_basebackup -h <primaire> -p <port du primaire> -D <data_dir>
-U replicator -X stream -S <slot> -R` (`no_log`, `PGPASSWORD`) → filet de
sécurité `replace` du mot de passe dans `primary_conninfo` (idempotent,
`no_log`) → démarrage.

**Vérifications** : `psql --version`, `pg_lsclusters`,
`pg_stat_replication` (primaire), `SHOW transaction_read_only` (réplique).

## 10. Sécurité et secrets

- `no_log: true` sur 4 tâches : utilisateur applicatif, utilisateur de
  réplication, `pg_basebackup` (`PGPASSWORD`), injection dans `primary_conninfo`.
- Connexions applicatives et réplication en **`scram-sha-256`**.
- `pg_hba.conf` / fragment : `0640` (postgres) en AWS, `0644` en instance.
- `.env` applicatif contient `DB_PASSWORD` (vault) — hors périmètre de ce rôle.
- Réplication locale limitée à `127.0.0.1/32` (pas d'exposition réseau).

## 11. Vérifications intégrées (non bloquantes)

- `psql --version` + `pg_lsclusters` (affiche les 3 clusters : `18-main`,
  `18-db01`, `18-db02` en local).
- **Primaire** : `SELECT client_addr, state, sent_lsn, replay_lsn FROM
  pg_stat_replication` (`login_port` + `login_unix_socket`) → debug « Réplicas
  connectés ».
- **Réplique** : `SHOW transaction_read_only` → debug « lecture seule = … ».
- Toutes en `changed_when: false` + `failed_when: false` (échec légitime
  selon le nœud/état).

## 12. Idempotence

- `pg_createcluster` gardé par `stat` du répertoire de config
  (`changed_when: true` uniquement quand il crée).
- Slots : `WHERE NOT EXISTS` dans la requête SQL.
- Réplique : **`standby.signal` est le garde** — basebackup/purge jamais rejoués.
- `primary_conninfo` : look-ahead négatif `(? !.*password=)` → n'écrit que si
  `password=` est absent.
- Modules `postgresql_user/db` : `state: present`.
- Preuve : `local_db.yml` rejoué → **`changed=0`** sur db01 et db02.

## 13. Tests effectués (preuves)

- `pg_lsclusters` : `18/db01 online (5442)`, `18/db02 online (5443, recovery)`,
  `18-main online (5432)` **intact**.
- Réplication : `pg_stat_replication` = `127.0.0.1 | streaming | sent_lsn =
  replay_lsn` ; slot `db02_slot` `active = t`.
- Bout en bout : ligne insérée sur le primaire **visible** sur la réplique ;
  écriture sur la réplique refusée (`read-only transaction`).
- `psql -h 127.0.0.1 -p 5442 -U laravel -d laravel` en `scram-sha-256`.
- Idempotence : `changed=0` sur les 2 hôtes.

## 14. Recettes de modification

| Je veux… | Fichier |
|---|---|
| Ajouter un réglage PG | `roles/postgresql/templates/conf.d/10-ansible.conf.j2` **+** variable dans `defaults` |
| Restreindre les réseaux applicatifs | `group_vars/databases.yml` → `postgresql_allowed_networks` |
| Changer le dimensionnement local | `inventories/local/host_vars/db0X.yml` |
| Renommer le slot | `host_vars/db02.yml` **et** `inventories/local/host_vars/db02.yml` (mêmes noms — cf. §16) |
| Changer le user applicatif | `group_vars/databases.yml` + vault |

## 15. Intégration avec les autres tiers

- **Consommateurs** : `laravel` (connexion sur 5442 local / 5432 AWS via
  `.env`), `springboot` (`springboot_datasource_url`) — voir [18-springboot.md](18-springboot.md).
- **Réplique** : lit le primaire ; les migrations jouées par `laravel`
  apparaissent sur la réplique (tables identiques vérifiées).
- **Playbook local** : `local_db.yml` (`serial: 1`). `artisan db:show` étant
  non bloquant, l'ordre web→db ou db→web est toléré ; les migrations exigent
  cependant une base existante.

## 16. Points d'attention (pièges)

1. **Nom de slot cohérent** : le primaire (db01, joué en 1er) lit
   `hostvars[db02].postgresql_replication_slot` dans **`playbooks/host_vars/db02.yml`**
   (seule source visible avant le passage de db02) — le fichier local doit
   porter **le même nom** (`db02_slot`).
2. **`login_port` / `login_unix_socket`** : la collection `community.postgresql`
   5.0 a renommé `port` → `login_port` (l'ancien nom échouait).
3. **`pg_basebackup -p` = port du PRIMAIRE** (`postgresql_primary_port`) : viser
   son propre port (5443) ferait un backup de soi-même.
4. **Pas de `-C`** : le slot est créé par le primaire ; `-C` échoue si le slot
   existe déjà (« An error is raised if the slot already exists »).
5. **`pg_createcluster` en root** : `become_user: postgres` échouerait (droits).
6. **`flush_handlers` obligatoire** avant réplication : la réplique se connecte
   pendant l'exécution du rôle — la config doit être déjà appliquée.
7. **Jamais le méta-service `postgresql`** en mode local (cf. §5).
8. **`serial: 1`** : sans lui, primaire et réplique partent en parallèle et la
   réplique échoue (primaire pas encore configuré).

## 17. Non trouvé dans le code actuel

- **Failover automatisé** : aucune tâche de promotion (`pg_ctl promote`) ni
  outil dédié (Patroni / repmgr / keepalived) — `README.md` §9 : « à valider
  avec le formateur ».
- **`requirements.yml` ne déclare pas `community.postgresql`** alors que 3
  modules de la collection sont utilisés (§4) — la collection est présente
  dans `.ansible/collections/` (5.0.0) mais non listée.
- Pas de sauvegarde (`pg_dump`), pas d'archivage WAL (`archive_command`),
  pas de monitoring de réplication entre deux exécutions.
- Pas de gestion de réinitialisation de réplique (re-init) ni de nettoyage de
  slot orphelin.
- `postgresql_instance` n'existe qu'à travers l'inventaire local (aucun
  `group_vars` ne l'expose).

## 18. Mode AWS (défaut)

- `postgresql_instance: ""` ⇒ cluster **`main`**, service méta **`postgresql`**,
  version **`17`**, port **5432**.
- Dimensionnement serveur (`200` / `512MB` / `2GB`), réseaux applicatifs
  `[0.0.0.0/0]` (à restreindre en production — `group_vars/databases.yml`).
- `pg_hba` de réplication : adresses déduites des nœuds `replica`
  (`ansible_host/32`) de l'inventaire.
- `postgresql_create_cluster: false` : le paquet fournit `main`.
- Création du fragment en `0640` (postgres).

## 19. Mode local cloisonné

- `postgresql_instance: db01|db02` via `inventories/local/host_vars/` +
  `include_vars` de `local_db.yml`.
- Version **18**, clusters `18/db01` (5442, primaire) et `18/db02` (5443,
  réplique), services `postgresql@18-db01/02`, `postgresql_create_cluster: true`.
- Réplication via la **boucle locale** : `postgresql_replication_allowed_addresses:
  [127.0.0.1/32]`, `postgresql_primary_host: 127.0.0.1`, `postgresql_primary_port: 5442`.
- Dimensionnement réduit (`100` / `256MB` / `1GB`).
- Commande :
  `ansible-playbook -i inventories/local/hosts.yml playbooks/local_db.yml --vault-password-file .vault_pass`

## 20. Problèmes rencontrés (réels)

| Symptôme | Cause | Correctif (dans le code) |
|---|---|---|
| `pg_createcluster` échoue (droits) | appelé en `become_user: postgres` : ne peut ni écrire `/etc/postgresql` ni lancer `initdb` | `become: true` **sans** `become_user` sur cette tâche |
| `pg_basebackup` : « slot already exists » | option `-C` recrée un slot déjà créé par le primaire | **`-C` retiré** — `-S` seul, le slot étant créé idempotamment par le primaire |
| La réplique se backupe elle-même / se connecte au mauvais endpoint | `-p` pointait sur le port local de la réplique (5443) | `postgresql_primary_port: 5442` (host_vars local) : le basebackup vise le **primaire** |
| Erreur de module : paramètre `port` inconnu | renommage `port` → `login_port` dans `community.postgresql` 5.0 | `login_port` + `login_unix_socket` partout |
| Réplique « password authentication failed » (scram) | `-R` ne reporte pas toujours `password=` dans `primary_conninfo` | tâche `replace` **idempotente** (n'écrit que si `password=` absent) ; constat PG 18.6 : `-R` le fait déjà, la tâche ne modifie rien |
| Course au premier run local | primaire et réplique sur la même machine, exécution parallèle | `serial: 1` dans `local_db.yml` |
| `pg_hba` refusait la réplication en boucle locale | ligne générée depuis `ansible_host` (IP privée documentaire) | `postgresql_replication_allowed_addresses: [127.0.0.1/32]` |
| Config non appliquée au moment de la connexion de la réplique | handlers exécutés en fin de play | `meta: flush_handlers` **avant** les tâches de réplication |

## 21. Références

- [`../roles/postgresql/README.md`](../roles/postgresql/README.md) — dont
  « Mode instance cloisonné », « Logique de réplication », « Validation ».
- [`../docs/installation-manuelle.md`](../docs/installation-manuelle.md) — §Tier 4 (+ §4.4 variante locale).
- [`../README.md`](../README.md) — §9 (réplication), §12 (preuves).
- [11 — Réplication PostgreSQL](11-replication-postgresql.md),
  [12 — Communication entre tiers](12-communication-entre-tiers.md),
  [10 — Sécurité & secrets](10-securite-et-secrets.md).

## 22. Résumé

Rôle maître/esclave complet, double mode : AWS = cluster `main` / 5432 /
service méta ; local = clusters Debian `18/db0X` dédiés (5442/5443), création
par `pg_createcluster`, réplication par la boucle locale (`serial: 1`,
`include_vars`). Les 3 invariants : `standby.signal` (jamais de re-init), slot
créé par le primaire (jamais `-C`), `pg_basebackup -p` sur le port du
**primaire**. Secrets masqués (`no_log`), connexions `scram-sha-256`.
**Points ouverts** : failover non automatisé, `community.postgresql` absente
de `requirements.yml`.






