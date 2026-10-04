# 00 — Vue d'ensemble

## Objectif du projet

Déployer une architecture 3-tiers **aussi bien sur AWS que localement**, avec le
même code Ansible :

- **AWS** : une machine par rôle, inventaire dynamique (`inventories/aws_ec2.yml`).
- **Local** : **tous les rôles sur UNE seule machine**, en *mode cloisonné*
  (`inventories/local/hosts.yml`) : chaque instance a son propre répertoire de
  configuration, son propre service systemd, son propre utilisateur et son propre
  port — sans jamais perturber les services déjà présents sur la machine.

Le levier central : une variable `*_instance` par rôle. **Vide = mode AWS**
( comportement d'origine inchangé ), **non vide = mode cloisonné**.

---

## Ports et services : le projet vs la machine hôte

La machine hôte fait déjà tourner d'autres applications. Le projet leur
**survit** : il choisit des ports libres et n'appelle **jamais** les
« méta-services » qui piloteraient les instances étrangères.

### Ports du projet (mode local, vérifiés sur la machine)

| Port | Service systemd | Rôle | Utilisateur | Config |
|---|---|---|---|---|
| **9080** | `nginx-lb01` | `nginx_lb` (load balancer) | `lb01` | `/etc/nginx-lb01/` |
| **9081** | `apache-web01` | `apache` (instance web01) | `web01` | `/etc/apache2-web01/` |
| **9082** | `apache-web02` | `apache` (instance web02) | `web02` | `/etc/apache2-web02/` |
| **5442** | `postgresql@18-db01` | `postgresql` (primaire) | `postgres` (cluster Debian) | `/etc/postgresql/18/db01/` |
| **5443** | `postgresql@18-db02` | `postgresql` (réplique) | `postgres` (cluster Debian) | `/etc/postgresql/18/db02/` |
| **6390** | `redis-redis01` | `redis` (cache du projet) | `redis01` | `/etc/redis-redis01/` |

Sockets Unix (PHP-FPM, pas de port TCP) :
`/run/php-web01/php-fpm.sock` et `/run/php-web02/php-fpm.sock`, propriétaires
`web01:web01` et `web02:web02`.

### Services et ports ÉTRANGERS — jamais touchés

| Port | Service | Propriétaire réel | Règle |
|---|---|---|---|
| **80** | `nginx.service` | un autre projet | aucun playbook ne vise `nginx` |
| **8080** | `apache2-waf-svc` | WAF Apache2 + ModSecurity | aucun playbook ne vise ce service |
| **8081** | `haproxy-lb-svc` | HAProxy d'un autre projet | hors périmètre |
| **8082** | `tomcat10` | Tomcat 10 | hors périmètre |
| **5432** | `postgresql@18-main` | cluster système partagé | **jamais** le méta-service `postgresql` |
| **6379** | `redis-cache-svc` (`/etc/cache-svc/redis.conf`) | autre application Redis | **jamais** le méta-service `redis-server` |
| **26379** | `redis-sentinel` | Sentinel Redis | hors périmètre |

**Deux interdits absolus** (présents dans tous les commentaires `local_*`) :

```
1. Ne JAMAIS appeler le méta-service « postgresql »  → il redémarrerait 18-main (5432).
2. Ne JAMAIS appeler le méta-service « redis-server » → il redémarrerait le Redis 6379 / sentinel 26379.
```

Le mode cloisonné utilise uniquement `postgresql@18-db01`,
`postgresql@18-db02`, `redis-redis01`, `nginx-lb01`, `apache-web0X`,
`php-fpm-web0X`.

### Ports AWS (mode défaut, issus de `group_vars/`)

| Tier | Port AWS | Source |
|---|---|---|
| Load balancer Nginx | 80 | `group_vars/loadbalancers.yml` → `nginx_lb_listen_port` |
| Apache (backends) | 8080 | `group_vars/webservers.yml` → `apache_port` |
| PostgreSQL | 5432 | `group_vars/databases.yml` → `postgresql_port` |
| Redis | 6379 (`bind: 0.0.0.0`) | `group_vars/redis.yml` |
| Spring Boot (backoffice) | 8080 | `group_vars/backoffice.yml` → `springboot_port` |

Les valeurs AWS ne sont **jamais** utilisées localement : elles sont surchargées
par l'inventaire local (voir [02 — Variables & précédence](02-ansible-variables-et-precedence.md)).

---

## Ordre de déploiement (mode local)

```bash
# 1. Inventaire / syntaxe
ansible-playbook -i inventories/local/hosts.yml playbooks/local_lb.yml --syntax-check

# 2. Étages, dans cet ordre :
ansible-playbook -i inventories/local/hosts.yml playbooks/local_cache.yml --vault-password-file .vault_pass  # AVANT local_web !
ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml   --vault-password-file .vault_pass
ansible-playbook -i inventories/local/hosts.yml playbooks/local_db.yml    --vault-password-file .vault_pass
ansible-playbook -i inventories/local/hosts.yml playbooks/local_lb.yml    --vault-password-file .vault_pass
```

- `local_cache.yml` **avant** `local_web.yml` : le rôle `php` y installe
  `php8.5-redis` et recharge `php-fpm-web01/02`.
- `local_db.yml` contient `serial: 1` : primaire **puis** réplique (la réplique
  a besoin du primaire déjà configuré).
- Le load balancer est déployé en dernier : il ne pointe que vers des backends existants.

---

## Invariants de non-régression

À **tous** les stades du projet, ces éléments doivent rester intacts :

```bash
# Les 3 clusters PostgreSQL restent online (le système inclus)
pg_lsclusters
# → 18/main online (5432) | 18/db01 online (5442) | 18/db02 online (5443)

# Redis étranger + sentinel répondent toujours
redis-cli -p 6379 ping                 # → PONG
redis-cli -p 26379 ping                # → PONG (sentinel)

# HTTP du projet
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:9080/   # LB
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:9081/   # web01
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:9082/   # web02
```

---

## Suite

→ [01 — Architecture](01-architecture.md) : qui parle à qui, avec quel port.
