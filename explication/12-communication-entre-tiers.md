# 12 — Communication entre tiers

Qui parle à qui, sur quel port, avec quel protocole et quel secret.
Tous les tracés ci-dessous sont vérifiables dans les inventaires, les
templates et les `.env` générés.

---

## Schéma A — Application web → PostgreSQL

```
web01 / web02
  processus : php-fpm (user web0X)
  driver    : PDO pgsql (php8.5-pgsql)
  fichier   : /var/www/web0X/.env
              DB_CONNECTION=pgsql
              DB_HOST=127.0.0.1
              DB_PORT=5442          ← laravel_db_port (inventaire local)
              DB_DATABASE=laravel
              DB_USERNAME=laravel
              DB_PASSWORD=<SECRET>  ← vault_laravel_db_password
        |
        |  TCP 127.0.0.1:5442   scram-sha-256
        |  (pg_hba : host all all 127.0.0.1/32 scram-sha-256)
        v
postgresql@18-db01 (PRIMAIRE)   base `laravel`
```

| | AWS | Local |
|---|---|---|
| Hôte | `groups['databases'][0]` → `db01` | `127.0.0.1` |
| Port | 5432 | **5442** |
| Variables | `laravel_db_*` (defaults) | `inventories/local/hosts.yml` (inline) |

Vérification : `sudo -u web01 php /var/www/web01/artisan db:show`
(ou `psql -h 127.0.0.1 -p 5442 -U laravel -d laravel`).

---

## Schéma B — Application web → Redis

```
web01 / web02
  processus : php-fpm (user web0X) via phpredis (php8.5-redis)
  fichier   : /var/www/web0X/.env
              SESSION_DRIVER=redis
              CACHE_STORE=redis
              QUEUE_CONNECTION=redis
              REDIS_CLIENT=phpredis
              REDIS_HOST=127.0.0.1
              REDIS_PORT=6390        ← laravel_redis_port (inventaire local)
              REDIS_PASSWORD=<SECRET>← vault_redis_password
        |
        |  TCP 127.0.0.1:6390   AUTH requirepass
        |  (bind 127.0.0.1 — aucune exposition réseau)
        v
redis-redis01 (instance du projet)
        ^
        |  (optionnel) worker `laravel-queue-*` consomme la file `queues:default`
```

| | AWS | Local |
|---|---|---|
| Hôte | `groups['redis'][0]` → `redis01` | `127.0.0.1` |
| Port | 6379 | **6390** |
| Sécurité | `requirepass` (vault) | idem |

⚠ **Jamais** le 6379 : `redis-cache-svc` (autre application) l'occupe déjà.

---

## Schéma C — Load balancer → instances web

```
client
  |  HTTP 127.0.0.1:9080   (nginx_lb_listen_port)
  v
nginx-lb01  (user lb01, /etc/nginx-lb01)
  upstream laravel_web { least_conn;
      server web01:9081 max_fails=3 fail_timeout=30s;   ← hostvars[web01].apache_port
      server web02:9082 max_fails=3 fail_timeout=30s;   ← hostvars[web02].apache_port
  }
  location /up          → 200 "OK"    (health-check du LB)
  location /            → proxy_pass http://laravel_web
                           + X-Real-IP, X-Forwarded-For, X-Forwarded-Proto
        |                     |
        |  HTTP web01:9081    |  HTTP web02:9082
        v                     v
   apache-web01          apache-web02      (user web0X, /etc/apache2-web0X)
        | socket Unix /run/php-web0X/php-fpm.sock
        v
   php-fpm-web0X (workers web0X) → /var/www/web0X/public/index.php
```

Prérequis machine : `web01` et `web02` **résolubles** (le dépôt ne gère pas
`/etc/hosts` — vérifié par grep).

| | AWS | Local |
|---|---|---|
| Écoute LB | 80 | **9080** |
| Backends | `web0X:8080` | `web01:9081`, `web02:9082` |
| Auth | aucune (réseau interne) | aucune (boucle locale) |

---

## Schéma D — Réplication PostgreSQL (db01 → db02)

```
postgresql@18-db01 :5442 (primaire)
   |  TCP 127.0.0.1:5442   protocole de RÉPLICATION
   |  user replicator + mot de passe <SECRET> (PGPASSWORD / vault)
   |  pg_hba : host replication replicator 127.0.0.1/32 scram-sha-256
   |  slot physique db02_slot (wal_level=replica, max_wal_senders=5)
   v
postgresql@18-db02 :5443 (réplique, standby.signal, lecture seule)
```

Détail complet et commandes de contrôle :
[11 — Réplication PostgreSQL](11-replication-postgresql.md).

---

## Schéma E — Récapitulatif (tableau)

| Source | Destination | Port | Protocole | Auth | Secret |
|---|---|---|---|---|---|
| client | nginx-lb01 | 9080 | HTTP | — | — |
| nginx-lb01 | apache-web01 | 9081 | HTTP (proxy) | — | — |
| nginx-lb01 | apache-web02 | 9082 | HTTP (proxy) | — | — |
| apache-web0X | php-fpm-web0X | socket Unix | FastCGI | permissions socket (`0660`) | — |
| php-fpm-web0X | postgresql@18-db01 | 5442 | TCP PostgreSQL | scram-sha-256 | `DB_PASSWORD` = `<SECRET>` |
| php-fpm-web0X | redis-redis01 | 6390 | TCP Redis | `requirepass` | `REDIS_PASSWORD` = `<SECRET>` |
| laravel-queue (opt-in) | redis-redis01 | 6390 | TCP Redis | `requirepass` | idem |
| postgresql@18-db02 | postgresql@18-db01 | 5442 | réplication | scram + slot | `PGPASSWORD` = `<SECRET>` |
| (AWS) php-fpm | db01 | 5432 | TCP PostgreSQL | scram | idem |
| (AWS) php-fpm | redis01 | 6379 | TCP Redis | `requirepass` | idem |

### Hors périmètre (ne jamais toucher)

| Port | Service étranger |
|---|---|
| 80 | `nginx.service` |
| 8080 | `apache2-waf-svc` (WAF) |
| 8081 | `haproxy-lb-svc` |
| 8082 | `tomcat10` |
| 5432 | `postgresql@18-main` |
| 6379 | `redis-cache-svc` |
| 26379 | `redis-sentinel` |

---

## Diagnostic réseau rapide

```bash
# Qu'écoute la machine (projet vs étranger) ?
ss -ltnp | grep -E ':(9080|9081|9082|5442|5443|6390|80|8080|8081|8082|5432|6379|26379)\b'

# Chaîne HTTP complète
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:9080/up   # LB
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:9081/     # web01 direct
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:9082/     # web02 direct
```

→ Voir aussi [14 — Tests & validation](14-tests-et-validation.md) et
[16 — Commandes utiles](16-commandes-utiles.md).
