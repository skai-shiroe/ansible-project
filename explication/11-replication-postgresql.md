# 11 — Réplication PostgreSQL (maître / esclave)

Document pédagogique : comment la réplication **physique streaming** fonctionne
dans ce projet, étape par étape, avec les schémas réels et les limites.

## 0. Le principe en 30 secondes

PostgreSQL conserve son historique d'écritures dans le **WAL** (*Write-Ahead
Log*). Une **réplique physique** copie les données du maître (`pg_basebackup`),
puis **suit** le WAL en continu : chaque écriture du maître est rejouée sur la
réplique. La réplique reste **ouverte en lecture** (`hot_standby`) mais refuse
toute écriture (`default_transaction_read_only = on`).

```
PRIMAIRE (db01 :5442)                    RÉPLIQUE (db02 :5443)
+-----------------------+                +------------------------+
| écritures clients     |                | lecture seule          |
|   ↓                   |    WAL         |   ↓                    |
| WAL ──────────────────┼─── streaming ─→│ standby.signal         |
| slot db02_slot        |   (scram)      │ primary_conninfo       |
| pg_stat_replication   |                │ replay_lsn             |
+-----------------------+                +------------------------+
```

---

## 1. Les 4 briques PostgreSQL utilisées

| Brique | Rôle dans ce projet | Où c'est configuré |
|---|---|---|
| **WAL** | journal des écritures répliquables | `wal_level = replica` (fragment `conf.d/10-ansible.conf.j2`) |
| **Slot de réplication physique** | mémorise le WAL tant que la réplique ne l'a pas lu (anti-perte) | tâche `pg_create_physical_replication_slot` (primaire) + `-S` (basebackup) |
| **`primary_conninfo` + `standby.signal`** | dit à la réplique : « connecte-toi au primaire et reste en standby » | écrits par `pg_basebackup -R` (filet de sécurité `replace` pour le mot de passe) |
| **`pg_hba` réplication** | autorise l'utilisateur `replicator` depuis la réplique | `pg_hba.conf.j2` : `host replication replicator <adr> scram-sha-256` |

---

## 2. Schéma complet des flux (mode local)

```
        web01/web02 (Laravel)
              |  PDO pgsql 127.0.0.1:5442
              v
 ┌──────────────────────────────┐
 │ db01  cluster 18/db01 :5442  │  PRIMAIRE
 │ service postgresql@18-db01   │
 │  - base `laravel`            │
 │  - user `laravel`            │
 │  - user `replicator`         │
 │  - slot `db02_slot`          │
 │  - wal_level=replica         │
 │  - default_transaction_      │
 │    read_only = off           │
 └──────────────┬───────────────┘
                │  streaming (127.0.0.1), scram-sha-256
                │  pg_hba : host replication replicator 127.0.0.1/32
                v
 ┌──────────────────────────────┐
 │ db02  cluster 18/db02 :5443  │  RÉPLIQUE
 │ service postgresql@18-db02   │
 │  - standby.signal            │
 │  - primary_conninfo =        │
 │    host=127.0.0.1 port=5442  │
 │    user=replicator …         │
 │  - hot_standby=on            │
 │  - default_transaction_      │
 │    read_only = on            │
 └──────────────────────────────┘

 18-main :5432 (système) reste INTACT — hors flux, jamais touché.
```

---

## 3. Étape par étape — ce que fait `local_db.yml`

`serial: 1` impose l'ordre : **db01 d'abord, db02 ensuite**.

### Sur le PRIMAIRE (db01)

1. `pg_createcluster 18 db01 --port 5442` (si absent, en **root**).
2. Dépôt du fragment `conf.d/10-ansible.conf` :
   `wal_level = replica`, `max_wal_senders = 5`, `max_replication_slots = 5`,
   `hot_standby = on`, `port = 5442`.
3. Dépôt du `pg_hba.conf` : ligne
   `host replication replicator 127.0.0.1/32 scram-sha-256`
   (`postgresql_replication_allowed_addresses`).
4. **`flush_handlers`** : redémarrage du cluster **avant** toute réplication.
5. Création de l'utilisateur applicatif (`laravel`, mot de passe `<SECRET>`).
6. Création de la base `laravel` (owner `laravel`).
7. Création de `replicator` avec `REPLICATION,LOGIN`.
8. Création **idempotente** du slot :
   `SELECT pg_create_physical_replication_slot('db02_slot') WHERE NOT EXISTS (…)`.
9. Vérification : `pg_stat_replication` → `client_addr/state/sent_lsn/replay_lsn`.

### Sur la RÉPLIQUE (db02)

10. `pg_createcluster 18 db02 --port 5443` (si absent, en root).
11. Dépôt du fragment (même contenu, avec
    `default_transaction_read_only = on`) et du `pg_hba.conf`.
12. **`stat standby.signal`** → le garde d'idempotence : s'il existe, la
    réplique est déjà initialisée et **rien** n'est refait.
13. Si absent : arrêt du cluster, **purge** du répertoire de données.
14. `pg_basebackup -h 127.0.0.1 -p 5442 -D /var/lib/postgresql/18/db02
    -U replicator -X stream -S db02_slot -R` avec `PGPASSWORD=<SECRET>`.
    - `-X stream` : le WAL est streamé pendant la copie ;
    - `-S db02_slot` : utilise (sans le créer) le slot du primaire ;
    - `-R` : écrit `standby.signal` + `primary_conninfo`.
15. Filet de sécurité idempotent : si `primary_conninfo` n'a pas `password=`,
    le mot de passe y est injecté (sinon la réplication échouerait en
    `scram-sha-256` — le processus ne peut pas le saisir interactivement).
16. Démarrage du cluster → la réplique rejoint le primaire.
    Vérification : `SHOW transaction_read_only` → `on`.

### Commandes de contrôle (lecture seule)

```bash
# Sur le primaire : l'état de la réplication
sudo -u postgres psql -p 5442 -c "SELECT client_addr, state, sent_lsn, replay_lsn FROM pg_stat_replication;"
# → 127.0.0.1 | streaming | <lsn> | <lsn>   (sent_lsn = replay_lsn : à jour)

# Slot actif ?
sudo -u postgres psql -p 5442 -c "SELECT slot_name, active FROM pg_replication_slots;"
# → db02_slot | t

# Sur la réplique : lecture seule + LSN de replay
sudo -u postgres psql -p 5443 -c "SHOW transaction_read_only;"     # → on
sudo -u postgres psql -p 5443 -c "SELECT pg_last_wal_replay_lsn();"

# Test de bout en bout
sudo -u postgres psql -p 5442 -c "CREATE TABLE _t(id int); INSERT INTO _t VALUES (1);"
sudo -u postgres psql -p 5443 -c "SELECT * FROM _t;"               # → 1 (vu)
sudo -u postgres psql -p 5443 -c "INSERT INTO _t VALUES (2);"      # → ERREUR : read-only transaction
sudo -u postgres psql -p 5442 -c "DROP TABLE _t;"
```

---

## 4. Pourquoi chaque garde-fou existe

| Garde | Sans lui | Problème observé/attendu |
|---|---|---|
| `serial: 1` | db02 démarre en parallèle de db01 | la réplique échoue : slot/`pg_hba`/user `replicator` pas encore créés |
| `flush_handlers` avant réplication | les handlers tournent en fin de play | la réplique se connecte avec l'**ancienne** config (pas de `wal_level`, port faux) |
| **Pas de `-C`** | `pg_basebackup -C` recrée le slot | « An error is raised if the slot already exists » — le primaire l'a déjà créé |
| `-p 5442` (`postgresql_primary_port`) | `-p 5443` (son propre port) | la réplique se « basebackup » elle-même |
| `standby.signal` = garde | purge à chaque exécution | la réplique serait **réinitialisée** à chaque run (perte de données) |
| `WHERE NOT EXISTS` sur le slot | appel `pg_create_…` direct | erreur à chaque run (slot existant) |
| `replace` de `primary_conninfo` | `-R` ne reporte pas `password=` selon la version | réplication en échec `password authentication failed` (silencieux jusqu'à `pg_stat_replication`) |
| Mêmes noms de slot dans `host_vars/db02.yml` **et** `inventories/local/host_vars/db02.yml` | le primaire lit le fichier racine (AWS) avant le passage de db02 | slot orphelin / `pg_basebackup -S` introuvable |

---

## 5. Réplication AWS vs local

| | AWS | Local cloisonné |
|---|---|---|
| Nœuds | `db01`, `db02` (machines distinctes) | 2 clusters Debian sur **une** machine |
| Primaire joignable par la réplique | `db01` (DNS/tag) | `127.0.0.1` |
| Port du primaire | `5432` | **`5442`** |
| Autorisation `pg_hba` réplication | déduite : `ansible_host/32` de db02 | explicite : `['127.0.0.1/32']` |
| Sérialisation | ordre naturel (machines distinctes) | `serial: 1` (ordre de l'inventaire) |
| Ordre de jeu | `database.yml` (les deux d'un coup) | `local_db.yml` |

## 6. Failover : état actuel et limites

**Non implémenté** (dans le code : aucune tâche de promotion ;
`README.md` §9 : « à valider avec le formateur »).

- La promotion manuelle d'une réplique s'effectuerait par :

```bash
# ⚠ Procédure MANUELLE — non automatisée par le projet
sudo -u postgres pg_ctlcluster 18 db02 promote
# puis pointer les applications vers 5443 (ou un alias) et rejouer la topologie
```

- Une automatisation de production nécessiterait un outil dédié
  (**Patroni**, **repmgr**, **keepalived**) : détection de panne, promotion,
  bascule des clients, réintégration de l'ancien maître.
- Le projet se limite à : réplication streaming vérifiée + lecture seule
  garantie. C'est un **TODO explicite** du dépôt.

## 7. Diagnostic rapide des pannes

| Symptôme | Vérifier | Cause fréquente |
|---|---|---|
| `pg_stat_replication` vide côté primaire | `systemctl status postgresql@18-db02` ; `journalctl -u postgresql@18-db02` | primaire pas redémarré (`flush_handlers` absent), `pg_hba` refusant, port faux |
| « password authentication failed » | `primary_conninfo` dans `/var/lib/postgresql/18/db02/postgresql.auto.conf` | `password=` absent (filet `replace` désactivé ou échoué) |
| Réplique en `no-connection` | connectivité `127.0.0.1:5442`, `listen_addresses` du primaire | primaire à l'arrêt / port changé |
| `read-only transaction` inattendu **sur la réplique** | — | **normal** : c'est le comportement voulu |
| `read-only transaction` **sur le primaire** | rôle du nœud (`postgresql_node_role`) | db01 paramétré en `replica` par erreur |
| Slot `active = f` | état de la réplique | réplique déconnectée : le WAL s'accumule (risque de saturation disque) |

## 8. Références

- [`../roles/postgresql/README.md`](../roles/postgresql/README.md) — « Logique de réplication ».
- [`../docs/installation-manuelle.md`](../docs/installation-manuelle.md) — §4.1 à §4.4 (procédure manuelle complète).
- [`../README.md`](../README.md) — §9, §12 (résultats observés).
- [07 — PostgreSQL](07-postgresql.md) (rôles/variables), [12 — Communication entre tiers](12-communication-entre-tiers.md).

## 9. Résumé

Réplication **physique streaming** en 4 briques (`wal_level`, slot, `-R` +
`standby.signal`, `pg_hba` scram). Le primaire crée tout (user, base, slot) ;
la réplique se copie une **seule fois** (`standby.signal` = garde) et suit le
WAL. Trois commandes `psql` suffisent à tout vérifier (`pg_stat_replication`,
`pg_replication_slots`, `SHOW transaction_read_only`). **Failover non
automatisé** : promotion manuelle, outil dédié à choisir (TODO projet).


