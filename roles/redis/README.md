# Rôle `redis`

Installe et configure **Redis** pour le cache, les sessions et les files
d'attente de Laravel.

## Variables

| Variable | Emplacement | Défaut | Rôle |
|---|---|---|---|
| `redis_port` | `group_vars/redis.yml` | `6379` | Port d'écoute |
| `redis_bind` | `group_vars/redis.yml` | `0.0.0.0` | Interfaces écoutées |
| `redis_maxmemory` | `group_vars/redis.yml` | `256mb` | Mémoire maximale |
| `redis_maxmemory_policy` | `group_vars/redis.yml` | `allkeys-lru` | Politique d'éviction |
| `redis_password` | `group_vars/redis.yml` | `{{ vault_redis_password }}` | **Secret** (`requirepass`) |
| `redis_tls_enabled` | `defaults` | `false` | **Conditionnel** : bascule TLS |
| `redis_appendonly` | `defaults` | `no` | Persistance AOF |

## Fichiers de configuration (templates — règle du formateur)

| Template | Destination |
|---|---|
| `templates/redis.conf.j2` | `/etc/redis/redis.conf` |

**Conditionnels Jinja présents** :

```jinja
{% if redis_tls_enabled %}   port 0 / tls-port ...   {% endif %}
{% if redis_appendonly %}    appendonly yes          {% endif %}
{% if redis_password | length > 0 %}  requirepass ... {% endif %}
```

## Tâches de vérification (non bloquantes)

- `redis-server --version` : version installée.
- `redis-cli ping` avec et sans mot de passe (`failed_when: false`,
  le test avec mot de passe porte `no_log: true`).
- `service_facts` + debug : statut du service.

## Sécurité

- La tâche du template porte `no_log: true` (elle écrit `requirepass`).
- Fichier déployé en `0640`, propriétaire `redis:redis`.
- ⚠️ `bind 0.0.0.0` : à restreindre à l'IP du tier en production, sinon Redis
  serait exposé.

## Fichiers statiques

Aucun.

## Handlers

- `Redémarrer Redis`

## Appelé par

`playbooks/cache.yml` — groupe `redis`

