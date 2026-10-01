# Rôle `postgresql`

Installe et configure **PostgreSQL en maître/esclave** : paramètres du cluster,
règles `pg_hba.conf`, utilisateur applicatif, **réplication physique par slots**
et initialisation de la réplique.

## Variables

Deux modes cohabitent dans le même rôle :

| Mode | `postgresql_instance` | Cluster | Service | Port | Préfixe des chemins |
|---|---|---|---|---|---|
| **AWS** (1 base par machine) | `""` (défaut) | `main` | `postgresql` (méta) | 5432 | `/etc/postgresql/<v>/main` |
| **Local cloisonné** (plusieurs bases par machine) | `db01`, `db02`… | `db01` | `postgresql@<v>-db01` | 5442, 5443… | `/etc/postgresql/<v>/db01` |

Tous les chemins (`postgresql_conf_dir`, `postgresql_data_dir`,
`postgresql_hba_file`, `postgresql_conf_include_dir`) sont **calculés** à partir
de ces deux variables : le reste des tâches est identique dans les deux modes.

| Variable | Emplacement | Défaut | Rôle |
|---|---|---|---|
| `postgresql_version` | `group_vars/databases.yml` | `"17"` | Version installée (**AWS** ; le mode local surcharge dans `host_vars`) |
| `postgresql_port` | `group_vars/databases.yml` | `5432` | Port d'écoute |
| `postgresql_database` / `postgresql_user` | `group_vars/databases.yml` | `laravel` | Base et utilisateur applicatif |
| `postgresql_password` | `group_vars/databases.yml` | `{{ vault_postgresql_password }}` | **Secret** |
| `postgresql_node_role` | **`host_vars/db0X.yml`** | `primary` | `primary` \| `replica` |
| `postgresql_primary_host` | `host_vars/db02.yml` | `db01` | Source de la réplication |
| `postgresql_primary_port` | `defaults/main.yml` | `{{ postgresql_port }}` | Port du primaire (≠ port local en mode cloisonné) |
| `postgresql_replication_slot` | `host_vars/db02.yml` | `db02_slot` | Slot dédié à la réplique |
| `postgresql_allowed_networks` | `group_vars/databases.yml` | `0.0.0.0/0` | Réseaux autorisés dans `pg_hba` |
| `postgresql_replication_allowed_addresses` | `defaults/main.yml` | `[]` | Réseaux autorisés pour la **réplication** ; vide = déduit des `ansible_host` des nœuds `replica` |
| `postgresql_instance` | `defaults/main.yml` | `""` | Nom de l'instance cloisonnée (vide = mode AWS) |
| `postgresql_create_cluster` | `defaults/main.yml` | `false` | Autorise `pg_createcluster` (mode cloisonné) |


## Fichiers de configuration (templates — règle du formateur)

| Template | Destination (mode AWS) | Destination (mode instance) |
|---|---|---|
| `templates/pg_hba.conf.j2` | `/etc/postgresql/{{ v }}/main/pg_hba.conf` | `/etc/postgresql/{{ v }}/<instance>/pg_hba.conf` |
| `templates/conf.d/10-ansible.conf.j2` | `/etc/postgresql/{{ v }}/main/conf.d/10-ansible.conf` | `/etc/postgresql/{{ v }}/<instance>/conf.d/10-ansible.conf` |

Le fragment `conf.d` est inclus automatiquement grâce à
`include_dir = 'conf.d'` présent dans `postgresql.conf` : le fichier de la
distribution **n'est jamais écrasé**.

## Mode instance cloisonné (plusieurs clusters sur une machine)

Activé dès que `postgresql_instance` est renseigné (voir
`inventories/local/host_vars/db0X.yml`) :

| Étape | Tâche | Détail |
|---|---|---|
| Création | `[instance] Créer le cluster Debian dédié` | `pg_createcluster <version> <instance> --port <port>`, **en root** (l'outil écrit dans `/etc/postgresql` et appelle `initdb` via `su`) et **non idempotent** → gardé par le `stat` du répertoire de configuration |
| Supervision | `[instance] Vérifier l'existence du cluster Debian` | `stat` sur `postgresql_conf_dir` |
| Démarrage | `systemd` sur `postgresql@<version>-<instance>` | Jamais le méta-service `postgresql`, qui démarrerait **tous** les clusters de la machine |
| Ordre | `meta: flush_handlers` avant la réplication | La configuration (port, `wal_level`, `pg_hba`) est appliquée **avant** que la réplique ne se connecte au primaire |

> **Cloisonnement garanti** : aucune tâche ne touche un cluster préexistant
> (ici `18-main`, port 5432) — le rôle n'agit que sur
> `/etc/postgresql/<v>/<instance>`, `/var/lib/postgresql/<v>/<instance>` et
> l'unité `postgresql@<v>-<instance>`.


**Conditionnel Jinja** dans le fragment :

```jinja
{% if postgresql_node_role == 'replica' %}
default_transaction_read_only = on
{% else %}
default_transaction_read_only = off
{% endif %}
```

## Tâches de vérification (non bloquantes)

- `psql --version` et `pg_lsclusters` : état du cluster local.
- `pg_stat_replication` sur le primaire : réplicas connectés et LSN.
- `SHOW transaction_read_only` sur la réplique : confirme le mode standby.
- Toutes ces requêtes portent `failed_when: false` (elles échouent légitimement
  dans l'état opposé : une réplique n'a pas de `pg_stat_replication`).
- Les modules `community.postgresql.*` reçoivent `login_port` +
  `login_unix_socket` : en mode cloisonné, `psql` doit viser **le bon cluster**
  (le socket par défaut pointerait vers `18-main`, port 5432).

## Logique de réplication

| Nœud | Actions |
|---|---|
| **Primaire** | création du user applicatif et de la base, user `REPLICATION`, **slot physique** (`WHERE NOT EXISTS` → idempotent) — le tout via le socket du cluster ciblé (`login_port`) |
| **Ordre** | `serial: 1` dans `playbooks/local_db.yml` : le primaire est entièrement configuré **avant** la réplique (slot + `pg_hba` + redémarrage), les deux clusters vivant sur la même machine |
| **Réplique** | test de `standby.signal` → arrêt du cluster → purge du répertoire de données → `pg_basebackup -X stream -S <slot> -R` → démarrage |
| **Test de `standby.signal`** | C'est le garde d'idempotence : une réplique déjà initialisée n'est **jamais** réécrasée |
| **Mot de passe** | `-R` écrit `standby.signal` + `primary_conninfo` ; selon la version, `password=` peut manquer → tâche `replace` idempotente qui ne complète que s'il est absent (`no_log`) |

> `-S <slot>` fonctionne parce que le slot **est créé par le primaire**.
> L'option `-C` (`--create-slot`) est volontairement **absente** :
> `pg_basebackup` refuse de créer un slot qui existe déjà
> (*« An error is raised if the slot already exists »*), ce qui est toujours le
> cas après le passage du primaire.


## Sécurité

Les tâches manipulant des mots de passe portent `no_log: true`
(user applicatif, user de réplication, `PGPASSWORD` de `pg_basebackup`,
injection du mot de passe dans `primary_conninfo`).

## Handlers

- `Redémarrer PostgreSQL` — piloté par `systemd` (`daemon_reload: true`) afin de
  cibler l'unité `postgresql@<version>-<instance>` en mode cloisonné ;
  tolérant en `--check` (l'unité peut ne pas encore exister en simulation).

## Validation (exécution réelle, mode cloisonné)

| Contrôle | Résultat observé |
|---|---|
| Clusters | `18 db01 5442 online`, `18 db02 5443 online,recovery`, `18 main 5432 online` (intact) |
| Services | `postgresql@18-db01`, `postgresql@18-db02` **et** `postgresql@18-main` actifs ; méta-service jamais appelé |
| `pg_stat_replication` (primaire 5442) | `127.0.0.1 \| streaming \| 18/db02 \| sent_lsn = replay_lsn` (à jour) |
| `pg_replication_slots` | `db02_slot \| physical \| active = t` |
| Réplique (5443) | `pg_is_in_recovery() = t`, `transaction_read_only = on` |
| Test fonctionnel | `CREATE TABLE`/`INSERT` sur le primaire **visibles** sur la réplique ; écriture sur la réplique → `cannot execute INSERT in a read-only transaction` |
| Authentification applicative | `laravel@laravel port=5442` en TCP (scram-sha-256) |
| Idempotence | 2ᵉ exécution : `changed=0` sur `db01` **et** `db02` |

## Appelé par

- `playbooks/database.yml` — groupe `databases` (mode AWS)
- `playbooks/local_db.yml` — groupe `databases`, instances cloisonnées
  `db01`/`db02` (les variables viennent de `inventories/local/host_vars/`)


