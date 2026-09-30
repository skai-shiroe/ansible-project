# Rôle `postgresql`

Installe et configure **PostgreSQL en maître/esclave** : paramètres du cluster,
règles `pg_hba.conf`, utilisateur applicatif, **réplication physique par slots**
et initialisation de la réplique.

## Variables

| Variable | Emplacement | Défaut | Rôle |
|---|---|---|---|
| `postgresql_version` | `group_vars/databases.yml` | `"17"` | Version installée |
| `postgresql_port` | `group_vars/databases.yml` | `5432` | Port d'écoute |
| `postgresql_database` / `postgresql_user` | `group_vars/databases.yml` | `laravel` | Base et utilisateur applicatif |
| `postgresql_password` | `group_vars/databases.yml` | `{{ vault_postgresql_password }}` | **Secret** |
| `postgresql_node_role` | **`host_vars/db0X.yml`** | `primary` | `primary` \| `replica` |
| `postgresql_primary_host` | `host_vars/db02.yml` | `db01` | Source de la réplication |
| `postgresql_replication_slot` | `host_vars/db02.yml` | `db02_slot` | Slot dédié à la réplique |
| `postgresql_allowed_networks` | `group_vars/databases.yml` | `0.0.0.0/0` | Réseaux autorisés dans `pg_hba` |

## Fichiers de configuration (templates — règle du formateur)

| Template | Destination |
|---|---|
| `templates/pg_hba.conf.j2` | `/etc/postgresql/{{ v }}/main/pg_hba.conf` |
| `templates/conf.d/10-ansible.conf.j2` | `/etc/postgresql/{{ v }}/main/conf.d/10-ansible.conf` |

Le fragment `conf.d` est inclus automatiquement grâce à
`include_dir = 'conf.d'` présent dans `postgresql.conf` : le fichier de la
distribution **n'est jamais écrasé**.

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

## Logique de réplication

| Nœud | Actions |
|---|---|
| **Primaire** | création du user applicatif et de la base, user `REPLICATION`, **slot physique** (`WHERE NOT EXISTS` → idempotent) |
| **Réplique** | test de `standby.signal` → `pg_basebackup -X stream -C -S <slot> -R` → démarrage en standby |

> `-R` génère `primary_conninfo` + `standby.signal` : aucune étape manuelle.

## Sécurité

Les tâches manipulant des mots de passe portent `no_log: true`
(user applicatif, user de réplication, `PGPASSWORD` de `pg_basebackup`).

## Handlers

- `Redémarrer PostgreSQL`

## Appelé par

`playbooks/database.yml` — groupe `databases`

