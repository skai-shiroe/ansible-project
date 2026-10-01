# Rôle `redis`

Installe et configure **Redis** pour le cache, les sessions et les files
d'attente de Laravel. Deux modes coexistent, pilotés par **`redis_instance`** :

| Mode | `redis_instance` | Fichier de configuration | Service | Usage |
|---|---|---|---|---|
| **AWS** | `""` (défaut) | `/etc/redis/redis.conf` | méta-service `redis-server` | tier cache unique |
| **Instance** | `"redis01"` | `/etc/redis-redis01/redis.conf` | unité **`redis-redis01`** | cloisonnement local : plusieurs instances sur une machine **et** coexistence avec un Redis étranger |

Tous les chemins (`redis_conf_dir`, `redis_conf_file`, `redis_dir`,
`redis_pid_file`, `redis_service`, `redis_user`…) sont **calculés** depuis
`redis_instance` : c'est ce qui permet de coexister avec le Redis du port
**6379** et le sentinel du **26379**, lancés **hors systemd** par une autre
application de la machine.

> ⚠️ Le rôle n'appelle **jamais** le méta-service `redis-server` en mode
> instance (il piloterait le Redis étranger) : uniquement `redis-redis01`.

## Variables

| Variable | Emplacement | Défaut | Rôle |
|---|---|---|---|
| `redis_instance` | `defaults` (surchargé en local) | `""` | Vide = mode AWS ; `"redis01"` = instance cloisonnée |
| `redis_instance_enabled` | `defaults` (dérivée) | `{{ redis_instance \| length > 0 }}` | Bascule des tâches `[instance]` |
| `redis_service` / `redis_conf_dir` / `redis_conf_file` | `defaults` (calculées) | `redis-server` / `/etc/redis` / `…/redis.conf` | Service, répertoire et fichier **calculés** selon l'instance |
| `redis_user` / `redis_group` / `redis_dir` / `redis_pid_file` | `defaults` (calculées) | `redis` / `/var/lib/redis` / … | Utilisateur, données et pidfile dédiés (`/run/redis-redis01/…`) |
| `redis_port` | `group_vars` / inventaire | `6379` | Port d'écoute (local : **6390**) |
| `redis_bind` | `defaults` | `127.0.0.1` | Interfaces écoutées |
| `redis_maxmemory` | `defaults` | `256mb` | Mémoire maximale |
| `redis_maxmemory_policy` | `defaults` | `allkeys-lru` | Politique d'éviction |
| `redis_password` | inventaire → **vault** | `{{ vault_redis_password }}` | **Secret** (`requirepass`) ; `""` = pas de mot de passe |
| `redis_protected_mode` | `defaults` | `yes` | Mode protégé Redis (valeur du paquet) |
| `redis_socket` | `defaults` | `""` | Socket Unix optionnel (`/run/redis-<inst>/redis.sock`) |
| `redis_databases` | `defaults` | `16` | Bases logiques |
| `redis_tls_enabled` / `redis_tls_port` | `defaults` | `false` / `6381` | **Conditionnel** : bascule TLS |
| `redis_appendonly` | `defaults` | `no` | Persistance AOF |

> **Placement** : les chemins calculés vivent dans `defaults/` et non `vars/` :
> `vars/` est prioritaire sur l'inventaire, un inventaire n'aurait donc jamais
> pu les surcharger (voir le commentaire de `roles/redis/vars/main.yml`).

## Fichiers de configuration (templates — règle du formateur)

| Template | Destination |
|---|---|
| `templates/redis.conf.j2` | `{{ redis_conf_file }}` |
| `templates/redis.service.j2` | `{{ redis_systemd_dir }}/{{ redis_service }}.service` (mode instance uniquement) |

**Conditionnels Jinja présents** :

```jinja
{% if redis_tls_enabled %}    port 0 / tls-port ...        {% endif %}
{% if redis_appendonly %}     appendonly yes               {% endif %}
{% if redis_password | length > 0 %}  requirepass ...      {% endif %}
{% if redis_socket | length > 0 %}     unixsocket ...      {% endif %}
```

L'unité `redis.service.j2` utilise **`Type=notify`** : Redis notifie systemd
dès qu'il est opérationnel (le `PING` du rôle ne part donc jamais trop tôt),
s'exécute sous l'utilisateur de l'instance et référence le bon `redis.conf`.

## Tâches de vérification (non bloquantes)

- `redis-server --version` : version installée.
- `PING` **sans** mot de passe → `NOAUTH Authentication required.` et `PING`
  **avec** mot de passe → `PONG` : le test authentifié passe par la variable
  d'environnement **`REDISCLI_AUTH`** (`no_log: true`) — jamais `-a`, qui
  exposerait le secret dans `ps`.
- `redis-cli info replication` (rôle + `connected_slaves`) et
  `redis-cli info keyspace` (`failed_when: false`, `changed_when: false`) :
  alimentent le debug final d'état.
- `service_facts` + `systemctl is-active` : statut de l'unité **de l'instance**.
- Debug final : version, pings, réplication, bases logiques.

## Sécurité

- Le template de configuration **et** le `PING` authentifié portent
  `no_log: true` (ils écrivent `requirepass` / le secret).
- Fichier déployé en `0640`, propriétaire `redis01:redis01` (mode instance).
- Utilisateur d'exécution dédié `redis01` (compte système `nologin`), jamais
  `root`, jamais l'utilisateur `redis` du système.
- ⚠️ `bind 127.0.0.1` en local : en production, restreindre à l'IP du tier
  (un `bind 0.0.0.0` exposerait Redis avec le réseau).

## Fichiers statiques

Aucun.

## Handlers

- `Redémarrer Redis` — `ansible.builtin.systemd` avec `daemon_reload: true`
  (l'unité `redis-<instance>.service` est nouvelle) ; tolérant **uniquement**
  en `--check` (`failed_when: … and not ansible_check_mode`).

## Appelé par

- `playbooks/cache.yml` — groupe `redis` (mode AWS)
- `playbooks/local_cache.yml` — instance cloisonnée `redis01`
