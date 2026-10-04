# 08 — Rôle `redis`

## 1. À quoi sert ce rôle ?

Installe et configure Redis pour **les sessions, le cache et les files d'attente
Laravel** : paquet, `redis.conf` (port, bind, mémoire, persistance,
`requirepass`), et — en mode cloisonné — une **instance isolée** (répertoires,
utilisateur, unité systemd `Type=notify` propres) coexistant avec un Redis
préexistant de la machine.

## 2. Où il vit dans le dépôt

```
roles/redis/
├── defaults/main.yml      # redis_instance, port, bind, mémoire, password…
├── vars/main.yml          # vidé (seul « --- »)
├── tasks/main.yml         # 180 lignes
├── handlers/main.yml      # « Redémarrer Redis »
├── templates/
│   ├── redis.conf.j2      # config complète (bind/port/requirepass/…)
│   └── redis.service.j2   # [instance] unité Type=notify
└── README.md
```

## 3. Comment il est appelé

- **AWS** : `playbooks/cache.yml` → `hosts: redis`, `roles: [redis]`.
- **Local** : `playbooks/local_cache.yml` → `hosts: redis`
  (hôte `redis01`, `redis_instance: redis01`).
- **À jouer AVANT `local_web.yml`** (documenté en tête du playbook et dans
  `README.md` §8) : le tier web installe ensuite `php8.5-redis` et recharge
  les FPM.

## 4. Prérequis et dépendances

- Paquet `redis-server` (installé par le rôle).
- Secret : `vault_redis_password` (local) / `vault_redis_password` (AWS,
  `group_vars/redis.yml`).
- Côté applicatif : extension `php8.5-redis` (rôle `php`) — sans elle,
  `SESSION_DRIVER=redis` échoue.
- `meta/main.yml` : `dependencies: []`.

## 5. Ports, services et chemins

| Élément | Mode AWS | Mode local (redis01) |
|---|---|---|
| Port | `6379` (`bind: 0.0.0.0` dans `group_vars`) | **`6390`** (`bind: 127.0.0.1`) |
| Service | `redis-server` (**méta-service**) | `redis-redis01` |
| Config | `/etc/redis/redis.conf` (0640) | `/etc/redis-redis01/redis.conf` (0640) |
| Utilisateur | `redis` | `redis01` |
| Données | `/var/lib/redis` | `/var/lib/redis-redis01` |
| PID | `/run/redis/redis-server.pid` | `/run/redis-redis01/redis-redis01.pid` |
| Logs | journald (pas de `logfile`) | idem → `journalctl -u redis-redis01` |

**Interdits absolu** : le port **6379** et le sentinel **26379** appartiennent
à une autre application ; aucun appel au méta-service `redis-server`.

## 6. Variables

| Variable | Valeur actuelle (AWS → local) | Fichier | Pourquoi | Exemple de modification | Impact |
|---|---|---|---|---|---|
| `redis_instance` | `""` → `redis01` | `defaults` → inventaire local | Bascule mode | — | gate de toutes les tâches d'instance |
| `redis_service` | `redis-server` → `redis-redis01` | `defaults` | **Jamais le méta-service en local** | — | démarrage/handler |
| `redis_conf_dir` / `redis_conf_file` | `/etc/redis{,/redis.conf}` → `/etc/redis-redis01{/redis.conf}` | `defaults` | Cloisonnement | — | dépôt du template |
| `redis_user` / `redis_group` | `redis` → `redis01` | `defaults` | Utilisateur dédié | — | propriétaire config/données, `User=` de l'unité |
| `redis_dir` | `/var/lib/redis` → `/var/lib/redis-redis01` | `defaults` → inventaire local | Données isolées | — | `dir` de `redis.conf` |
| `redis_pid_file` | `/run/redis/redis-server.pid` → `/run/redis-redis01/redis-redis01.pid` | `defaults` → inventaire local | PID isolé | — | `pidfile` + `PIDFile=` unité |
| `redis_port` | `6379` (defaults + `group_vars`) → `6390` | `group_vars/redis.yml` → inventaire local | Port libre | inventaire | `port` de `redis.conf` + tests + `.env` cible |
| `redis_bind` | `127.0.0.1` (defaults) / `0.0.0.0` (group_vars AWS) → `127.0.0.1` (local) | idem | Exposition minimale | — | `bind` |
| `redis_password` | `""` (defaults) → `{{ vault_redis_password }}` | `group_vars` + vault, inventaire local | `requirepass` | vault | `requirepass`/`masterauth` + auth des tests |
| `redis_maxmemory` | `256mb` → `128mb` (local) | `group_vars` → inventaire local | Machine partagée | — | `maxmemory` |
| `redis_maxmemory_policy` | `allkeys-lru` | `group_vars/redis.yml` | Politique d'éviction cache | — | `maxmemory-policy` |
| `redis_appendonly` | `no` | `group_vars/redis.yml` | Cache : pas de persistance | `yes` si durable | `appendonly` |
| `redis_databases` | `16` | `defaults` | Bases logiques | — | `databases` |
| `redis_loglevel` | `notice` | `defaults` | Journalisation journald | — | `loglevel` |
| `redis_protected_mode` | `yes` | `defaults` | Directive neutre (défaut Debian + défaut Redis) | — | `protected-mode` |
| `redis_socket` | `""` | `defaults` | Socket Unix optionnel (`unixsocket`) | renseigner pour clients locaux | directive optionnelle |
| `redis_tls_enabled` / `redis_tls_port` | `false` / `6381` | `defaults` | TLS conditionnel | `true` | `port 0` + `tls-port` |
| `redis_dbfilename` / `redis_appendfilename` | `dump.rdb` / `appendonly.aof` | `defaults` | Fichiers de persistance | — | — |
| `redis_systemd_dir` | `/etc/systemd/system` | `defaults` | Destination unité | — | — |
| `redis_package` | `redis-server` | `defaults` | Paquet Debian | — | `apt` |

## 7. Templates

| Template | Destination | Contenu clé |
|---|---|---|
| `redis.conf.j2` | `<conf_dir>/redis.conf` (0640) | `bind` + `port` (ou `port 0` + `tls-port` si TLS), `daemonize no`, `supervised no`, `pidfile`, `dir`, `dbfilename`, `protected-mode`, `loglevel` (pas de `logfile` → journald), `databases`, `unixsocket` optionnel, `appendonly` conditionnel, **`requirepass` + `masterauth` si mot de passe**, `maxmemory` + `maxmemory-policy` |
| `redis.service.j2` | `/etc/systemd/system/redis-redis01.service` | `Type=notify`, `User=redis01`, `RuntimeDirectory=redis-redis01`, `ExecStart=redis-server <conf> --supervised systemd --daemonize no`, `PIDFile`, `UMask=0007`, hardening |

Tâches déployant `redis.conf` : **`no_log: true`** (le template contient
`requirepass`).

## 8. Handlers

**`Redémarrer Redis`** : `systemd` + `daemon_reload: true`,
`state: restarted`, tolérant `--check`. Déclenché par : config (les deux
modes) et unité (instance).

## 9. Parcours des tâches

**AWS** : `apt redis-server` → template `redis.conf` (`no_log`) →
démarrage `redis-server`.

**Instance** : `apt` → groupe + user `redis01` (`nologin`) → répertoires
`/etc/redis-redis01` + `/var/lib/redis-redis01` (0750) → `redis.conf` (`no_log`)
→ unité `redis-redis01` → démarrage (`daemon_reload`).

**Vérifications** : `redis-server --version` → `redis-cli -p <port> ping`
(sans mot de passe — `NOAUTH` attendu si `requirepass`) → idem **avec**
`REDISCLI_AUTH` dans l'environnement (`no_log`) → `info replication` →
`info keyspace` → `service_facts` + récapitulatif.

## 10. Sécurité et secrets

- `requirepass` issu du vault (`redis_password`).
- **`REDISCLI_AUTH` en variable d'environnement, jamais `redis-cli -a`** :
  `-a` expose le mot de passe dans la table des processus (`ps`), lisible par
  tout utilisateur ; `no_log: true` sur la tâche.
- Config `0640` appartenant à `redis01`; répertoires `0750`.
- `bind: 127.0.0.1` en local (pas d'exposition réseau) ; `protected-mode yes`.
- Unité systemd durcie + `UMask=0007`.

## 11. Vérifications intégrées (non bloquantes)

- Version du serveur.
- Ping **sans** mot de passe (constate `NOAUTH` = `requirepass` actif) puis
  ping **avec** authentification — les deux en `failed_when: false`.
- `info replication` → `role:master connected_slaves:0` ; `info keyspace` →
  bases logiques (« aucune clé : normal sur une instance neuve »).
- `service_facts` → état du service.
- Aucune de ces tâches ne bloque un déploiement.

## 12. Idempotence

- `apt`, `template`, `file`, `state: started` idempotents.
- Redémarrages uniquement par `notify` (config réellement changée).
- Preuve : `local_cache.yml` rejoué → **`changed=0`**.

## 13. Tests effectués (preuves)

- `redis-redis01` : `active` + `enabled`, port **6390** en écoute sur
  `127.0.0.1`, config `/etc/redis-redis01/redis.conf`, user `redis01`.
- `redis-cli -p 6390 ping` → `NOAUTH Authentication required.` puis `PONG`
  avec `REDISCLI_AUTH`.
- **Services étrangers intacts** : `redis-server` (6379) et `redis-sentinel`
  (26379) répondent toujours (`PONG`, pids inchangés) ; méta-service
  `redis-server` **inactive**.
- Debug du rôle : version `8.0.5`, pings, `role:master`, keyspace listé.
- Câblage applicatif : `.env` des 2 web → `SESSION_DRIVER/CACHE_STORE/
  QUEUE_CONNECTION=redis`, `REDIS_HOST=127.0.0.1`, `REDIS_PORT=6390`,
  `REDIS_PASSWORD=<SECRET>`.
- Session HTTP → clé Redis (TTL ≈ 120 min) ; `Cache::put/get` en db1 ;
  `LLEN …queues:default = 1` après poussée d'un job.

## 14. Recettes de modification

| Je veux… | Fichier |
|---|---|
| Changer le port local | `inventories/local/hosts.yml` → `redis01: redis_port` **et** `laravel_redis_port` (des web) |
| Activer la persistance AOF | `group_vars/redis.yml` → `redis_appendonly: true` |
| Régénérer le mot de passe | vault → `vault_redis_password` (puis rejouer `local_cache` + `local_web`) |
| Limiter la mémoire | inventaire local → `redis_maxmemory` |
| Exposer le Redis AWS en interne | `group_vars/redis.yml` → `redis_bind` (à restreindre) |

## 15. Intégration avec les autres tiers

- **Clients** : `laravel` via phpredis (`REDIS_HOST/PORT/PASSWORD` du `.env`).
- **Amont requis** : paquet PHP `php8.5-redis` (rôle `php` sur les web).
- **Worker opt-in** (`queue-worker.service.j2`) consomme les jobs de cette
  instance ; aucune dépendance d'ordonnancement déclarée (le rôle laravel
  ignore le nom de l'unité Redis — Laravel réessaie de lui-même).
- **Ordre de déploiement local** : `local_cache` → `local_web`.

## 16. Points d'attention (pièges)

1. **Jamais `redis-server` (méta)** : il pilote le Redis étranger du 6379.
2. **Jamais `-a`** dans `redis-cli` : utiliser `REDISCLI_AUTH`.
3. `php8.5-redis` hérite de `conf.d` compilé dans le binaire PHP : le rang
   `local_cache` avant `local_web` évite un FPM démarré sans extension à jour.
4. `bind 0.0.0.0` (AWS) est un réglage **de groupe** : ne pas le propager au local.
5. Pas de `logfile` : les logs passent par journald
   (`journalctl -u redis-redis01`).

## 17. Non trouvé dans le code actuel

- **Pas de réplication Redis** (master/slave) ni de Sentinel gérés par le
  rôle : `masterauth` est écrit mais aucun pair n'est configuré
  (`connected_slaves:0` observé). Le sentinel `26379` de la machine est étranger.
- **Pas de TLS vérifié en production** : `redis_tls_enabled: false` (bloc
  conditionnel présent dans le template, jamais activé).
- Pas de `unixsocket` actif (`redis_socket: ""`).
- Aucune tâche de purge/sauvegarde (`BGSAVE`, `LASTSAVE`) ni d'optimisation.
- `group_vars/all/vault.yml.example` contient `vault_springboot_db_password`
  mais le vault local n'a que **5 clés** (le backoffice local n'existe pas).

## 18. Mode AWS (défaut)

- `redis_instance: ""` ⇒ config `/etc/redis/redis.conf`, service méta
  **`redis-server`**, user `redis`, port **6379**.
- `group_vars/redis.yml` : `bind: 0.0.0.0` (réseau interne VPC),
  `maxmemory: 256mb`, `allkeys-lru`, `appendonly: no`,
  `redis_password: {{ vault_redis_password | default('') }}`.
- Tolérant en `--check` (artefact `README.md` §8).

## 19. Mode local cloisonné

- `redis_instance: redis01` ⇒ `/etc/redis-redis01/redis.conf`,
  service **`redis-redis01`**, user `redis01`, `dir /var/lib/redis-redis01`,
  `pidfile /run/redis-redis01/redis-redis01.pid`, port **6390**,
  `bind 127.0.0.1`, `maxmemory 128mb`, `requirepass` = vault.
- Unité `Type=notify` avec `--supervised systemd --daemonize no`.
- Commande :
  `ansible-playbook -i inventories/local/hosts.yml playbooks/local_cache.yml --vault-password-file .vault_pass`

## 20. Problèmes rencontrés (réels)

| Symptôme | Cause | Correctif (dans le code) |
|---|---|---|
| Risque de casser le Redis d'une autre application | méta-service `redis-server` / port 6379 partagés avec `redis-cache-svc` | instance dédiée : port **6390**, répertoires/utilisateur/`pidfile` propres ; **aucune tâche n'appelle `redis-server`** (commentaires `local_cache.yml`, `redis.conf.j2`, tasks) |
| Exposition du mot de passe Redis dans `ps` | `redis-cli -a <mdp>` | **`REDISCLI_AUTH`** en `environment` (+ `no_log`) |
| `NOAUTH` inattendu au test | `requirepass` actif — le test sans mdp est censé échouer | double test (sans/avec auth), les deux en `failed_when: false`, debug explicite |
| Laravel « Redis server has gone away » / ECONNREFUSED | `.env` pointant vers un mauvais port (6379 étranger sans le bon mot de passe) | `laravel_redis_host: 127.0.0.1` + `laravel_redis_port: 6390` dans l'inventaire local |
| FPM démarré sans `phpredis` | ordre des playbooks | `local_cache.yml` **avant** `local_web.yml` (documenté `README.md` §8) |
| Défauts AWS (`6379`, `0.0.0.0`) hérités par le local | `group_vars/redis.yml` visible depuis les playbooks locaux (symlink) | variables inline du `redis01` dans l'inventaire local (préséance supérieure) |

## 21. Références

- [`../roles/redis/README.md`](../roles/redis/README.md) — dont « Sécurité ».
- [`../docs/installation-manuelle.md`](../docs/installation-manuelle.md) — §Tier 5 (+ §5.1 variante locale).
- [`../README.md`](../README.md) — §6, §12 (preuves étape 7).
- [06 — Laravel](06-laravel.md) (drivers), [10 — Sécurité & secrets](10-securite-et-secrets.md),
  [16 — Commandes utiles](16-commandes-utiles.md).

## 22. Résumé

Rôle double mode : AWS = `/etc/redis/redis.conf` + méta `redis-server` (6379) ;
local = instance `redis01` isolée (6390, répertoires, user, unité `Type=notify`)
coexistant avec le Redis étranger. Les 3 règles : **jamais** le méta-service,
**jamais** `redis-cli -a`, `local_cache` avant `local_web`. Secrets (`requirepass`)
masqués par `no_log`, liens applicatifs `.env` → `127.0.0.1:6390`.
**Points ouverts** : pas de réplication/sentinel gérés, TLS inactif.


