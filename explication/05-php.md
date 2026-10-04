# 05 — Rôle `php`

## 1. À quoi sert ce rôle ?

Installe PHP et les extensions nécessaires à Laravel (`pgsql`, `mbstring`,
`redis`…) et configure **PHP-FPM** : soit via le fichier `.ini` partagé du mode
AWS (`conf.d/99-laravel.ini`), soit — en mode cloisonné — via un **master
dédié**, un **pool dédié** et un **socket dédié** par instance web.

## 2. Où il vit dans le dépôt

```
roles/php/
├── defaults/main.yml      # php_version, paquets, tuning pm, OPcache…
├── vars/main.yml          # vidé (seul « --- »)
├── tasks/main.yml         # 169 lignes
├── handlers/main.yml      # « Redémarrer PHP-FPM »
├── templates/
│   ├── 99-laravel.ini.j2     # [AWS] conf.d partagé
│   ├── php-fpm.conf.j2       # [instance] [global] du master
│   ├── pool.conf.j2          # [instance] pool <instance> (réglages Laravel)
│   └── php-fpm.service.j2    # [instance] unité php-fpm-<instance>
└── README.md
```

## 3. Comment il est appelé

- `playbooks/applications.yml` et `playbooks/local_web.yml`, **après** `apache`,
  **avant** `laravel` (ordre `[apache, php, laravel]`).
- **Ordre des playbooks locaux** (documenté dans `README.md` §8 et en tête de
  `playbooks/local_cache.yml`) : `local_cache.yml` **avant** `local_web.yml` —
  c'est au passage du tier web que ce rôle installe `php8.5-redis` (dans
  `php_packages`) et recharge `php-fpm-web01` / `php-fpm-web02`.
- `site.yml` (AWS) place `applications.yml` en premier, puis `database.yml`,
  `cache.yml`, et le LB en dernier.

## 4. Prérequis et dépendances

- Dépôt Ubuntu natif : les 9 paquets `php8.5-*` sont disponibles
  (`8.5.4-0ubuntu1.3`, validé sur Ubuntu 26.04 — `README.md` §13) : **aucun PPA**.
- Binaire `/usr/sbin/php-fpm8.5`.
- En mode instance, le répertoire de scan des extensions reste **compilé dans le
  binaire** (`/etc/php/8.5/fpm/conf.d`) : l'instance cloisonnée **hérite** des
  extensions système (dont `php8.5-redis`) sans configuration propre.

## 5. Ports, services et chemins

| Élément | Mode AWS | Mode local (web01 / web02) |
|---|---|---|
| Service | `php8.5-fpm` | `php-fpm-web01` / `php-fpm-web02` |
| Config | `/etc/php/8.5/fpm` | `/etc/php-web01/fpm` / `/etc/php-web02/fpm` |
| Pool | `pool.d/www.conf` | `pool.d/web01.conf` / `web02.conf` |
| Socket | `/run/php/php8.5-fpm.sock` | `/run/php-web01/php-fpm.sock` / `-web02` |
| Utilisateurs workers | `www-data` | `web01` / `web02` (master = **root**, patron Debian) |
| Logs | `/var/log` (`php-fpm.log`) | `/var/log/php-web01/` / `php-web02/` |
| PID | `/run/php/php8.5-fpm.pid` | `/run/php-web0X/php8.5-fpm.pid` |
| `.ini` réglages | `/etc/php/8.5/fpm/conf.d/99-laravel.ini` | porté par le **pool** (`php_admin_value`) |

## 6. Variables

| Variable | Valeur actuelle (AWS → local) | Fichier | Pourquoi | Exemple de modification | Impact |
|---|---|---|---|---|---|
| `php_version` | `"8.5"` (defaults + `group_vars/webservers.yml`) | `defaults` / `group_vars` | Version unique | `group_vars`: `php_version: "8.4"` | paquets, service, chemins |
| `php_instance` | `""` → `web01` | `defaults` → inventaire local | Bascule AWS/instance | — | gate des tâches `[instance]` |
| `php_fpm_service` | `php8.5-fpm` → `php-fpm-web0X` | `defaults` | Service dédié | — | démarrage + handler |
| `php_fpm_binary` | `/usr/sbin/php-fpm8.5` | `defaults` | Binaire | — | `ExecStart`, test `-t` |
| `php_fpm_conf_dir` / `php_fpm_pool_dir` | `/etc/php/8.5/fpm{,/pool.d}` → `/etc/php-web0X/fpm{,/pool.d}` | `defaults` (calculé) | Cloisonnement | — | emplacements des templates |
| `php_fpm_pool_name` | `www` → `web0X` | `defaults` | Nom du pool | — | fichier `<pool>.conf`, section `[web0X]` |
| `php_fpm_socket` | `/run/php/php8.5-fpm.sock` → `/run/php-web0X/php-fpm.sock` | `defaults` → inventaire local (`apache_php_fpm_socket` doit suivre) | Socket dédié | **coordonner** avec `apache_php_fpm_socket` | vhost Apache (503 sinon) |
| `php_fpm_user` / `php_fpm_group` | `www-data` → `web0X` | `defaults` → inventaire local | Workers sous le propriétaire de l'app | — | `user=`, `listen.owner=` du pool |
| `php_fpm_pm*` | `dynamic`, `5/2/1/3`, `max_requests: 500` | `defaults` | Dimensionnement | — | pool |
| `php_packages` | 9 paquets `php8.5-*` dont `php8.5-redis`, `php8.5-pgsql` | `defaults` | Dépendances Laravel + phpredis | ajouter une extension | `apt` + `notify: Redémarrer PHP-FPM` |
| `php_ini_settings` | `256M/64M/64M/UTC` | `defaults` | Limites Laravel | — | pool (instance) ou `.ini` (AWS) |
| `php_display_errors` | `false` | `defaults` | Production | `true` en dev | `display_errors` |
| `php_opcache_*` | `true/256/10000/false/60` | `defaults` | Performances | — | OPcache conditionnel |
| `php_fpm_daemonize` | `false` | `defaults` | systemd gère le processus | — | `[global]` |
| `php_fpm_systemd_dir` | `/etc/systemd/system` | `defaults` | Destination unité | — | — |
| `php_fpm_scan_dir` | calculé | `defaults` | Répertoire de scan (vérifications) | — | debug extensions |
| `php_conf_d_dir` / `php_conf_file` | **NON DÉFINIS** (référencés en mode AWS) | **absents du code** | — | à définir | tâches AWS en échec (voir §17) |

## 7. Templates

| Template | Destination | Remarque |
|---|---|---|
| `99-laravel.ini.j2` | `[AWS] /etc/php/8.5/fpm/conf.d/99-laravel.ini` | réglages Laravel **sans** toucher au `php.ini` du paquet |
| `php-fpm.conf.j2` | `[instance] /etc/php-web0X/fpm/php-fpm.conf` | `[global]` : pid, error_log, `daemonize no`, `include pool.d/*.conf` — **commentaires `;` uniquement** (cf. §20) |
| `pool.conf.j2` | `[instance] pool.d/web0X.conf` | `[web0X]`, `user/group`, `listen = /run/php-web0X/php-fpm.sock` (`listen.mode 0660`), pm dynamique, `php_admin_value` (memory/upload/timezone/OPcache) |
| `php-fpm.service.j2` | `/etc/systemd/system/php-fpm-web0X.service` | `Type=notify`, **master root** / workers `web0X` (patron Debian), `RuntimeDirectory=php-web0X`, hardening |

## 8. Handlers

**`Redémarrer PHP-FPM`** : `systemd` + `daemon_reload: true`,
`state: restarted`, tolérant `--check` uniquement. Déclenché par : paquets,
`.ini` (AWS), `php-fpm.conf`, `pool.conf`, unité (instance).

## 9. Parcours des tâches

**AWS** : `apt php_packages` → création `conf.d` → template `99-laravel.ini`
(`when: not php_instance_enabled`) → démarrage `php8.5-fpm`.

**Instance** : groupe + user `web0X` (`nologin`) → `/etc/php-web0X/fpm` +
`pool.d` → `/var/log/php-web0X` → `php-fpm.conf` → `pool.conf` → unité →
démarrage (`daemon_reload`).

**Vérifications** : `php -v`, test syntaxe `php-fpm8.5 -t -y …` (instance),
`php -m` + debug « pdo_pgsql / redis présents ou ABSENTS ».

## 10. Sécurité et secrets

- Aucun secret (le `.env` appartient au rôle `laravel`).
- Workers sous `web0X` (mode instance) ; socket `0660` appartenant à `web0X`.
- Unité systemd durcie ; `expose_php = off`.

## 11. Vérifications intégrées (non bloquantes)

- `php -v` / `php -m` (`failed_when: false`, `changed_when: false`).
- Diagnostic explicite : `pdo_pgsql` → « ABSENTE (la connexion PostgreSQL
  échouera) » ; `redis` → « ABSENTE (sessions, cache et files Redis échoueront) ».
- `php-fpm -t` en mode instance.

## 12. Idempotence

- `apt state: present`, templates, `state: started`.
- Les redémarrages ne partent que par `notify` (changement réel).
- Tâches de contrôle : `changed_when: false`.

## 13. Tests effectués (preuves)

- `local_web.yml` rejoué : **`changed=0`** sur `web01`/`web02`.
- `systemctl is-active php-fpm-web01 php-fpm-web02` → `active`.
- Sockets : `/run/php-web01/php-fpm.sock` et `/run/php-web02/php-fpm.sock`
  (propriétaires `web0X`, `srw-rw----`).
- `php --ri redis` → `Redis Support => enabled` (phpredis 6.2.0) installé sur
  les 2 hôtes ; `pdo_pgsql` présent (`artisan db:show` fonctionne).

## 14. Recettes de modification

| Je veux… | Fichier |
|---|---|
| Monter `memory_limit` | `roles/php/defaults/main.yml` → `php_ini_settings.memory_limit` |
| Activer `display_errors` (dev) | idem → `php_display_errors: true` |
| Changer le dimensionnement pm | idem → `php_fpm_pm_max_children`… |
| Changer le socket de web01 | `inventories/local/hosts.yml` → `php_fpm_socket` **et** `apache_php_fpm_socket` (coordonnés) |
| Ajouter une extension | `php_packages` (defaults) |

## 15. Intégration avec les autres tiers

- **Aval d'`apache`** : le vhost Apache pointe sur le socket de ce rôle.
- **Amont de `laravel`** : le code PHP s'exécute dans le pool.
- **Redis** : l'extension `php8.5-redis` (liste `php_packages`) est indispensable
  aux drivers `redis` de Laravel — d'où l'ordre `local_cache → local_web`.

## 16. Points d'attention (pièges)

1. **Les fichiers `php-fpm.conf` n'acceptent PAS les commentaires `#`** :
   uniquement `;` (contrairement aux `.ini`). Un `#` provoque l'erreur FPM
   **exit 78** (cf. §20).
2. **Le répertoire de scan des extensions est compilé dans le binaire** :
   l'instance cloisonnée lit `/etc/php/8.5/fpm/conf.d` du système — c'est
   **voulu** (héritage de `php8.5-redis` sans configuration).
3. Le master tourne en **root**, les workers sous `web0X` (patron Debian) :
   ne pas tenter de lancer le master sous `web0X`.
4. `php_fpm_socket` et `apache_php_fpm_socket` doivent rester **identiques** —
   ils sont définis dans deux rôles différents.

## 17. Non trouvé dans le code actuel

- **`php_conf_d_dir` et `php_conf_file`** : référencés dans `roles/php/tasks/main.yml`
  (tâches « Créer le répertoire de configuration PHP-FPM » et « Déployer la
  configuration PHP-FPM », **mode AWS** : `when: not php_instance_enabled`) mais
  **absents** de `defaults/main.yml`, `vars/main.yml`, de tous les
  `group_vars/` et de tous les inventaires (vérifié par grep +
  `ansible-inventory --host web01`). En mode local, ces tâches sont sautées ;
  en mode AWS, elles échoueraient sur une variable indéfinie. **Aucune valeur
  n'est supposée ici.**
- Pas de gestion de `opcache.revalidate_paths` ni de `php.ini` CLI séparé.
- Aucun health-check applicatif PHP côté LB.

## 18. Mode AWS (défaut)

- `php_instance: ""` ⇒ service `php8.5-fpm`, config `/etc/php/8.5/fpm`,
  pool `www`, socket `/run/php/php8.5-fpm.sock`, workers `www-data`.
- Réglages Laravel déposés en `conf.d/99-laravel.ini` (ne jamais écraser le
  `php.ini` du paquet).
- Paquets : 9 × `php8.5-*` dont `php8.5-redis` (validé Ubuntu 26.04, `README.md` §13).
- Le démarrage est tolérant en `--check` (artefact documenté `README.md` §8).

## 19. Mode local cloisonné

- `php_instance: web01|web02` ⇒ `/etc/php-web0X/fpm`, service `php-fpm-web0X`,
  pool `web0X`, socket `/run/php-web0X/php-fpm.sock`, workers `web0X`,
  logs `/var/log/php-web0X/`.
- Master root + workers dédiés (patron Debian conservé).
- Réglages Laravel portés par le **pool** (`php_admin_value`), pas par `.ini`.
- Commande : `ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml --vault-password-file .vault_pass`

## 20. Problèmes rencontrés (réels)

| Symptôme | Cause | Correctif (dans le code) |
|---|---|---|
| PHP-FPM refuse de démarrer, **exit 78** (config invalide) | commentaires `#` dans `php-fpm.conf` / `pool.conf` — PHP-FPM n'accepte que `;` | templates réécrits avec `;` ; le constat est rappelé en tête de `php-fpm.conf.j2` |
| Sessions/cache Redis en échec malgré Redis OK | extension `php8.5-redis` absente | ajout de `php8.5-redis` à `php_packages` + ordre `local_cache` avant `local_web` ; diagnostic explicite dans le debug final |
| Après un changement de socket, Apache répond 503 | workers / vhost désynchronisés | redémarrage des deux services (handlers) |
| Utilisateur `web0X` nologin → warning Ansible « no HOME » | `become_user: web0X` sans répertoire personnel | `ansible_remote_tmp: /tmp` déclaré pour `web01`/`web02` dans l'inventaire local (affecte aussi les tâches du rôle `laravel`) |

## 21. Références

- [`../roles/php/README.md`](../roles/php/README.md)
- [`../docs/installation-manuelle.md`](../docs/installation-manuelle.md) — §Tier 2, étapes 1/4/7.
- [`../README.md`](../README.md) — §6, §12, §13 (`php 8.5` validé).
- [04 — Apache](04-apache.md) (socket), [06 — Laravel](06-laravel.md), [08 — Redis](08-redis.md) (phpredis).

## 22. Résumé

Rôle double mode : AWS = `.ini` partagé + service `php8.5-fpm` ; local = master
et pool par instance, socket dédié, réglages Laravel dans le pool. Le binaire
partage le répertoire de scan (héritage des extensions, dont phpredis). Pièges :
commentaires `;` obligatoires (exit 78), coordonnation socket Apache/FPM,
`ansible_remote_tmp: /tmp` pour les users nologin. **Point ouvert** : variables
`php_conf_d_dir`/`php_conf_file` référencées mais non définies (mode AWS).


