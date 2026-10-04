# 14 — Tests et validation

Checklist de validation **après chaque exécution**, par tier. Les valeurs
« résultat attendu » sont celles observées sur le lab (`README.md` §12).

---

## 1. Avant tout : syntaxe et ciblage (lecture seule)

```bash
# Syntaxe des playbooks (AWS + local)
ansible-playbook -i inventories/hosts.yml playbooks/site.yml --syntax-check
ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml --syntax-check
ansible-playbook -i inventories/local/hosts.yml playbooks/local_db.yml  --syntax-check

# Cibles et tâches (aucune exécution)
ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml --list-hosts
ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml --list-tasks

# Simulation complète (aucune modification)
ansible-playbook -i inventories/local/hosts.yml playbooks/site.yml --check --diff
```

> Artefact connu du `--check` : « Could not find the requested service X »
> quand le paquet n'est pas encore installé (`README.md` §8). Ne pas confondre
> avec une vraie erreur.

---

## 2. Checklist par tier

### Tier cache — `local_cache.yml`

```bash
ansible-playbook -i inventories/local/hosts.yml playbooks/local_cache.yml --vault-password-file .vault_pass
```

| # | Vérification | Commande | Attendu |
|---|---|---|---|
| 1 | Service projet | `systemctl is-active redis-redis01` | `active` (+ `enabled`) |
| 2 | Écoute | `ss -ltnp \| grep 6390` | `127.0.0.1:6390` |
| 3 | Config | `ls -l /etc/redis-redis01/redis.conf` | existe, `0640`, owner `redis01` |
| 4 | Auth | `redis-cli -p 6390 ping` | `NOAUTH Authentication required.` |
| 5 | Auth OK | `REDISCLI_AUTH=<SECRET> redis-cli -p 6390 ping` | `PONG` |
| 6 | **Étrangers intacts** | `redis-cli -p 6379 ping` et `-p 26379 ping` | `PONG` (pids inchangés) |
| 7 | Méta-service | `systemctl is-active redis-server` | **jamais** démarré par le projet |
| 8 | Idempotence | 2e exécution | `changed=0` |

### Tier web — `local_web.yml`

```bash
ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml --vault-password-file .vault_pass
```

| # | Vérification | Commande | Attendu |
|---|---|---|---|
| 1 | Services | `systemctl is-active apache-web01 apache-web02 php-fpm-web01 php-fpm-web02` | `active` ×4 |
| 2 | Écoute | `ss -ltnp \| grep -E '908[12]'` | 9081 + 9082 |
| 3 | Sockets FPM | `ls -l /run/php-web0X/php-fpm.sock` | owner `web0X`, `srw-rw----` |
| 4 | HTTP direct | `curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:9081/` (+ `9082`) | `200` |
| 5 | `.env` | `sudo -u web01 head -c0 /var/www/web01/.env` / `grep DB_PORT` | `0600` owner `web01` ; `DB_PORT=5442` |
| 6 | Câblage DB | `sudo -u web01 php /var/www/web01/artisan db:show` | PostgreSQL 18.x, port **5442** |
| 7 | Câblage Redis | `grep -E 'REDIS_PORT\|SESSION_DRIVER' /var/www/web01/.env` | `6390`, `redis` |
| 8 | phpredis | `sudo -u web01 php --ri redis \| grep 'Redis Support'` | `enabled` |
| 9 | Migrations | `sudo -u postgres psql -p 5442 -c '\dt'` | tables Laravel (9), jouées 1× (`run_once`) |
| 10 | Worker opt-in | `systemctl list-units 'laravel-queue-*'` | aucune unité (défaut `false`) |
| 11 | Idempotence | 2e exécution | `changed=0` sur web01 **et** web02 |

### Tier données — `local_db.yml`

```bash
ansible-playbook -i inventories/local/hosts.yml playbooks/local_db.yml --vault-password-file .vault_pass
```

| # | Vérification | Commande | Attendu |
|---|---|---|---|
| 1 | Clusters | `pg_lsclusters` | `18/main`, `18/db01`, `18/db02` tous **online** |
| 2 | Écoute | `ss -ltnp \| grep -E '544[23]'` | 5442 + 5443 |
| 3 | Réplication | `sudo -u postgres psql -p 5442 -c "SELECT client_addr,state,sent_lsn,replay_lsn FROM pg_stat_replication;"` | `127.0.0.1 \| streaming \| …` (`sent_lsn = replay_lsn`) |
| 4 | Slot | `… "SELECT slot_name,active FROM pg_replication_slots;"` | `db02_slot \| t` |
| 5 | Lecture seule | `sudo -u postgres psql -p 5443 -c 'SHOW transaction_read_only;'` | `on` |
| 6 | Écriture refusée | essai `INSERT` sur 5443 | `ERREUR : read-only transaction` |
| 7 | Bout en bout | INSERT sur 5442 puis SELECT sur 5443 | ligne visible |
| 8 | Applicatif | `psql -h 127.0.0.1 -p 5442 -U laravel -d laravel -c 'select 1;'` | `1` (scram) |
| 9 | **Système intact** | `pg_lsclusters` ligne `18/main` | `online`, port 5432 |
| 10 | Idempotence | 2e exécution | `changed=0` sur db01 **et** db02 |

### Load balancer — `local_lb.yml`

```bash
ansible-playbook -i inventories/local/hosts.yml playbooks/local_lb.yml
```

| # | Vérification | Commande | Attendu |
|---|---|---|---|
| 1 | Service | `systemctl is-active nginx-lb01` | `active` |
| 2 | Écoute | `ss -ltnp \| grep 9080` | `0.0.0.0:9080` |
| 3 | Health-check | `curl -s http://127.0.0.1:9080/up` | `OK` (200) |
| 4 | Proxy | `curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:9080/` | `200` |
| 5 | Upstream | `grep -E 'server web0' /etc/nginx-lb01/sites-enabled/*.conf` | `web01:9081`, `web02:9082` |
| 6 | **Nginx système intact** | `systemctl is-active nginx` | inchangé (port 80) |

---

## 3. Tests fonctionnels applicatifs (en tant que `web01`)

Les scripts sont exécutés **avec l'utilisateur du service** (`.env` en `0600`) :

```bash
# Session en Redis (requête HTTP → clé Redis)
# 1. requête : curl -s -c /tmp/c.txt http://127.0.0.1:9081/
# 2. clé :     REDISCLI_AUTH=<SECRET> redis-cli -p 6390 --scan --pattern '*session*'
#    → cookie laravel-web01-session + clé avec TTL ≈ 120 min

# Cache et file (via tinker, en tant que web01)
sudo -u web01 php /var/www/web01/artisan tinker --execute="Cache::put('k','v',60); dump(Cache::get('k'));"
# → la clé atterrit en db1 de Redis (info keyspace)

# Job poussé dans la file
sudo -u web01 php /var/www/web01/artisan tinker --execute="dispatch(function(){ logger('ok'); });"
REDISCLI_AUTH=<SECRET> redis-cli -p 6390 llen queues:default
# → 1 (payload JSON ; 0 si le worker opt-in est activé et l'a consommé)
```

> **Où vont les preuves** : le tableau des validations réelles (sessions en
> Redis, db1, `LLEN=1`, migrations 3×1, `changed=0`…) est tenu dans
> [`../README.md`](../README.md) §12 — c'est la source faisant foi.

---

## 4. Non-régression (services étrangers)

À exécuter **après chaque playbook** :

```bash
# 3 clusters PG : les 3 online (5432 système inclus)
pg_lsclusters

# Redis étranger + sentinel toujours vivants
redis-cli -p 6379 ping     # PONG
redis-cli -p 26379 ping    # PONG

# Services étrangers toujours actifs
systemctl is-active nginx apache2-waf-svc haproxy-lb-svc tomcat10 \
  postgresql@18-main redis-cache-svc redis-sentinel

# HTTP projet
for p in 9080 9081 9082; do
  printf '%s -> ' $p; curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:$p/;
done
```

---

## 5. Vérification de la couverture « rôles appelés »

```bash
ansible-playbook -i inventories/hosts.yml playbooks/site.yml --list-tasks
# → les 8 rôles apparaissent au bon endroit (applications → database → cache → nginx_lb)

ansible-inventory -i inventories/local/hosts.yml --graph
# → lb01, web01, web02, bo01, db01, db02, redis01
```

---

## 6. Ordre recommandé des validations

```
1. syntax-check        (avant toute exécution)
2. --check --diff      (simulation, comparer avec docs/installation-manuelle.md)
3. local_cache.yml     (Redis + phpredis)
4. local_web.yml       (Apache + FPM + Laravel)
5. local_db.yml        (PG primaire/réplique)
6. local_lb.yml        (LB en dernier)
7. checklist §2 ci-dessus, tier par tier
8. non-régression §4
9. 2e exécution de chaque playbook → changed=0  (idempotence)
```

---

## 7. Points d'attention des tests

- **Toujours** `--vault-password-file .vault_pass` pour `local_web`,
  `local_db`, `local_cache` (sinon : `VARIABLE IS NOT DEFINED` sur les secrets).
- **Toujours** `sudo -u web01` pour les tests applicatifs (`.env` `0600`).
- Ne jamais exécuter de `redis-cli -a` ni de `systemctl restart` d'un service
  étranger.
- Les tests d'insertion sur la réplique **doivent** échouer : c'est le succès.

## 8. Références

- [`../README.md`](../README.md) — §12 (preuves réelles), §8 (commandes d'usage).
- [`../docs/installation-manuelle.md`](../docs/installation-manuelle.md) — checklist finale.
- [13 — Idempotence](13-idempotence-et-handlers.md), [15 — Dépannage](15-depannage.md),
  [16 — Commandes utiles](16-commandes-utiles.md).

## 9. Résumé

Valider = syntaxe → simulation → exécution par tier → vérifications
structurelles (services, ports, configs) → fonctionnelles (HTTP, `artisan`,
réplication, Redis) → **non-régression** des services étrangers → **2e
exécution à `changed=0`**. Les tableaux §2 donnent les commandes exactes et
les résultats attendus observés sur le lab.

