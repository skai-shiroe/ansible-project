# 01 — Architecture

## Les 6 tiers du projet

| Tier | Rôles | Instance(s) locale(s) | Port(s) local | Port AWS | Service(s) local |
|---|---|---|---|---|---|
| Entrée de trafic | `nginx_lb` | `lb01` | 9080 | 80 | `nginx-lb01` |
| Application web | `apache`, `php`, `laravel` | `web01`, `web02` | 9081 / 9082 (+ sockets FPM) | 8080 | `apache-web0X`, `php-fpm-web0X` |
| Cache / sessions / files | `redis` | `redis01` | 6390 | 6379 | `redis-redis01` |
| Base de données | `postgresql` | `db01` (primaire), `db02` (réplique) | 5442 / 5443 | 5432 | `postgresql@18-db01`, `postgresql@18-db02` |
| Backoffice | `java`, `springboot` | *(non implémenté en local)* | — | 8080 | `backoffice` |
| Hôte | — | machine unique Ubuntu | — | — | services étrangers intacts |

Dépendances : `laravel` → `php` → `apache` (tier web, même playbook) ;
`laravel` → `postgresql` + `redis` (après coup, via `.env`) ; `nginx_lb` →
`apache` (upstream pointe vers les backends) ; `springboot` → `java` +
`postgresql`.

---

## Schéma 1 — Chemin d'une requête HTTP (mode local)

```
client (navigateur)
   |
   |  HTTP http://127.0.0.1:9080/
   v
+-------------------------------+
| nginx-lb01  (lb01)  :9080     |  roles/nginx_lb  — /etc/nginx-lb01
| upstream laravel_web          |
|   least_conn                  |
+-------------------------------+
   |                    |
   | web01:9081         | web02:9082        <- hostvars[web0X].apache_port
   v                    v
+------------------+  +------------------+
| apache-web01     |  | apache-web02     |  roles/apache — /etc/apache2-web0X
| vhost laravel    |  | vhost laravel    |  DocumentRoot /var/www/web0X/public
| SetHandler proxy |  | SetHandler proxy |  (index.php Laravel, pas index.html)
+------------------+  +------------------+
   | socket Unix          | socket Unix
   v                      v
/run/php-web01/       /run/php-web02/
  php-fpm.sock          php-fpm.sock     roles/php — php-fpm-web0X
   |                      |
   +----------+-----------+
              |  le code Laravel s'exécute dans le pool (user web0X)
              v
     /var/www/web01  et  /var/www/web02   roles/laravel
       |                    |
       |  PDO pgsql         |  phpredis
       |  127.0.0.1:5442    |  127.0.0.1:6390
       v                    v
+------------------+   +------------------+
| postgresql@18-db01|   | redis-redis01    |
| PRIMAIRE :5442    |   | :6390 requirepass|
| base `laravel`    |   | sessions/cache/  |
+------------------+   | queues           |
   | WAL streaming      +------------------+
   v                      ^ push jobs (QUEUE_CONNECTION=redis)
+------------------+       | (worker opt-in : laravel_queue_worker_enabled)
| postgresql@18-db02| -----+  # le rôlephp installe php8.5-redis (local_cache)
| REPLIQUE :5443    |
+------------------+
```

## Schéma 2 — Chemin d'une écriture de données

```
artisan migrate / INSERT applicatif
        |
        v
primaire db01 (127.0.0.1:5442)  WAL : wal_level=replica
        |
        |  slot physique `db02_slot` (max_wal_senders=5, max_replication_slots=5)
        |  pg_hba : host replication replicator 127.0.0.1/32 scram-sha-256
        v
réplique db02 (127.0.0.1:5443)  standby.signal + primary_conninfo
        |
        v
lecture seule (default_transaction_read_only=on) — hot_standby=on
écriture refusée : « read-only transaction »
```

## Schéma 3 — Mode AWS (une machine par rôle)

```
 Internet --> lb01:80 (nginx) --:8080--> web01 (apache+php+laravel) --:5432--> db01 (primaire)
                      \----------:8080--> web02 ----------------------'         | streaming
                                                    \--> redis01:6379           v
                                                    `------------------------ db02 (réplique)
 bo01 (java + springboot :8080) --> db01:5432
```

Differences avec le local :

| Aspect | AWS | Local cloisonné |
|---|---|---|
| Exécution | SSH (`ansible_user: ubuntu`) | `ansible_connection: local` (pas de SSH) |
| Noms d'hôtes | tags EC2 (`tag:Name`) | `lb01`, `web01`… résolus par `/etc/hosts` |
| Ports backends | 8080 partout | 9081 / 9082 distincts |
| Base | 5432, cluster `main`, service `postgresql` | 5442/5443, clusters `18/db01`, `18/db02`, services `postgresql@18-db0X` |
| Redis | 6379, `bind 0.0.0.0`, service `redis-server` | 6390, `bind 127.0.0.1`, service `redis-redis01` |
| Migrations Laravel | `laravel_run_migrations: false` | `true` (inventaire local) |
| Secrets | `group_vars/all/vault.yml` (vide, à remplir) | `inventories/local/group_vars/all/vault.yml` (chiffré) |

---

## Points clés retenus

- **Une source unique de vérité par port** : le LB lit
  `hostvars[web0X].apache_port` — ajouter `web03` à l'inventaire suffit.
- **Les deux instances web partagent LA base** `laravel` du primaire → les
  migrations sont jouées **une fois** (`run_once: true` dans le rôle laravel).
- **Le backoffice n'existe qu'en AWS** (code présent, pas de variante locale) :
  voir [17-java.md](17-java.md) et [18-springboot.md](18-springboot.md).

→ Suite : [02 — Variables & précédence](02-ansible-variables-et-precedence.md)
