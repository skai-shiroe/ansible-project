# 06 — Rôle `laravel`

## 1. À quoi sert ce rôle ?

Déploie l'application **Laravel** : clonage Git, génération du fichier
`.env` (base de données, Redis, secrets), `composer install`, `artisan
key:generate` (si clé absente), dossiers `storage/*` + `bootstrap/cache`,
permissions, vérifications applicatives (`artisan db:show`), migrations
(gardées par un opt-in) et worker de file d'attente (opt-in).

## 2. Où il vit dans le dépôt

```
roles/laravel/
├── defaults/main.yml      # dépôt, .env, DB, Redis, worker, migrations…
├── vars/main.yml          # laravel_env_file, laravel_storage_dir (2 constantes)
├── tasks/main.yml         # 256 lignes
├── handlers/main.yml      # « Redémarrer le worker de file d'attente »
├── templates/
│   ├── env.j2                 # fichier .env (secrets → no_log)
│   └── queue-worker.service.j2# unité worker (opt-in)
└── README.md
```

## 3. Comment il est appelé

- `playbooks/applications.yml` et `playbooks/local_web.yml` — **3e rôle**
  du trio `[apache, php, laravel]`.
- Variables d'entrée clés (inventaire local) : `laravel_instance: web0X`,
  `laravel_repo: https://github.com/laravel/laravel.git`, `laravel_branch: 13.x`.

## 4. Prérequis et dépendances

- Paquets : `composer`, `git`, `unzip` (installés par le rôle).
- `php` fonctionnel (rôle `php`), extension `pgsql` (+ `redis` pour les drivers).
- Base joignable pour `db:show` / migrations (rôle `postgresql` exécuté séparément).
- `ansible_remote_tmp: /tmp` pour `web01`/`web02` (users nologin sans HOME) —
  déclaré dans l'inventaire local, indispensable aux tâches `become_user`.
- `meta/main.yml` : `dependencies: []`.

## 5. Ports, services et chemins

Ce rôle **ne crée aucun service** (sauf le worker opt-in).

| Élément | Mode AWS | Mode local (web01 / web02) |
|---|---|---|
| Application | `/var/www/laravel` | `/var/www/web01` / `/var/www/web02` |
| Propriétaire | `www-data` | `web01` / `web02` |
| `.env` | `/var/www/laravel/.env` (0600) | `/var/www/web0X/.env` (0600, owner `web0X`) |
| Dépôt | placeholder `TON_COMPTE/TON_PROJET.git` (clonage sauté) | `laravel/laravel`, branche `13.x` |
| COMPOSER_HOME | `(HOME de l'utilisateur SSH)` | `/var/www/web0X/.composer` |
| Worker (opt-in) | `laravel-queue.service` | `laravel-queue-web0X.service` |

## 6. Variables

| Variable | Valeur actuelle (AWS → local) | Fichier | Pourquoi | Exemple de modification | Impact |
|---|---|---|---|---|---|
| `laravel_path` | `/var/www/laravel` → `/var/www/web0X` | `defaults` → `group_vars/webservers.yml` → inventaire local | Racine d'app | inventaire | clone, .env, artisan, permissions |
| `laravel_instance` | `""` → `web01` | `defaults` → inventaire local | Bascule mode | — | gate mode instance |
| `laravel_clean_deploy` | `false` → `true` (local) | `defaults` → inventaire local | Purger la cible de clone (index.html) **seulement si l'app n'est pas déjà là** | — | `file: state: absent` conditionnel |
| `laravel_composer_home` | `""` → `/var/www/web0X/.composer` | `defaults` → inventaire local | users nologin sans HOME | — | env `COMPOSER_HOME` des tâches composer/artisan |
| `laravel_repo` | placeholder `https://github.com/TON_COMPTE/TON_PROJET.git` → `laravel/laravel` | `defaults` → inventaire local | Dépôt réel | renseigner un dépôt privé | clonage activé |
| `laravel_branch` | `main` → `13.x` | idem | Branche du squelette | — | `git` module |
| `laravel_repo_placeholder` | `TON_COMPTE/TON_PROJET.git` | `defaults` | Détecter le placeholder | — | `laravel_repo_configured` |
| `laravel_repo_configured` | calculé | `defaults` | Saute clonage/composer/artisan tant que le dépôt est fictif | — | garde de cohérence |
| `laravel_owner` / `laravel_group` | `www-data` → `web0X` | `defaults` → inventaire local | Propriétaire | — | `become_user`, permissions |
| `laravel_app_name` | `Laravel` → `Laravel-web0X` | idem | Affichage | — | `.env` `APP_NAME` |
| `laravel_app_env` / `laravel_app_debug` | `production` / `false` | `defaults` | Comportement prod | `display_errors`… | `.env` |
| `laravel_app_url` | `http://{{ ansible_host }}` → `http://web0X:908X` | `defaults` → inventaire local | URL canonique | — | `.env` |
| `laravel_app_key` | `{{ vault_laravel_app_key \| default('') }}` | `defaults` ← vault | Clé **fixe** (sinon sessions invalidées à chaque run) | vault | `.env` `APP_KEY` ; `key:generate` sauté si non vide |
| `laravel_db_connection/host/port/database/username/password` | `pgsql` / `db01` / `5432` / `laravel` / `laravel` / vault → local : `127.0.0.1` / `5442` | `defaults` → inventaire local + vault | Câblage BDD | inventaire/vault | `.env` |
| `laravel_run_migrations` | **`false`** → `true` (local) | `defaults` → inventaire local | AWS ne touche jamais le schéma | `-e` ou inventaire | `artisan migrate --force` (`run_once`) |
| `laravel_cache_store` / `laravel_session_driver` / `laravel_queue_connection` | `redis` | `defaults` (surchargeable par hôte) | Sessions/cache/files en Redis | — | `.env` |
| `laravel_redis_host` / `port` / `password` | `groups['redis'][0]` / `6379` → `127.0.0.1` / `6390` / vault | `defaults` → inventaire local | Jamais le 6379 étranger | inventaire | `.env` `REDIS_*` |
| `laravel_queue_worker_enabled` | **`false`** (opt-in partout) | `defaults` | Aucun service supplémentaire par défaut | `-e laravel_queue_worker_enabled=true` | unité worker déployée |
| `laravel_queue_worker_service` | `laravel-queue` → `laravel-queue-web0X` | `defaults` (calculé) | Nom dédié | — | unité + handler |
| `laravel_queue_worker_sleep/tries/max_time` | `3/3/3600` | `defaults` | Réglages `queue:work` | — | `ExecStart` |
| `laravel_system_packages` | `composer, git, unzip` | `defaults` | Outils de déploiement | — | `apt` |
| `laravel_writable_dirs` | 6 répertoires `storage/*` + `bootstrap/cache` | `defaults` | HTTP 500 sinon | — | `file: state: directory` |

## 7. Templates

| Template | Destination | Contenu |
|---|---|---|
| `env.j2` | `{{ laravel_path }}/.env` (`0600`, owner `laravel_owner`) | `APP_*` (dont `APP_KEY`), `DB_*`, `SESSION_DRIVER`, `CACHE_STORE`, `QUEUE_CONNECTION`, `REDIS_CLIENT=phpredis`, `REDIS_HOST/PORT/PASSWORD`, `MAIL_MAILER=log` |
| `queue-worker.service.j2` | `/etc/systemd/system/laravel-queue*.service` | `User=laravel_owner`, `WorkingDirectory`, `ExecStart=php artisan queue:work --sleep=3 --tries=3 --max-time=3600`, `Restart=always`, hardening ; déployée **uniquement** si opt-in |

Tâche de template `.env` : **`no_log: true`** (contient `DB_PASSWORD` + `APP_KEY`).

## 8. Handlers

**`Redémarrer le worker de file d'attente`** : `systemd` +
`daemon_reload: true`, `state: restarted`, tolérant `--check`. Déclenché par
le déploiement de l'unité (opt-in uniquement).

## 9. Parcours des tâches

1. `apt` (composer, git, unzip) → 2. création du répertoire d'app →
3. message si dépôt placeholder → **4.** mode instance : `stat` de `artisan`
   et `.git` → nettoyage conditionnel (`laravel_clean_deploy` **et** absence
   d'app existante) → recréation → **5.** `git clone` (`become_user:
   laravel_owner`, `version: laravel_branch`) → **6.** `.env` (`no_log`) →
7. `stat artisan` → 8. `composer install` (`no_dev`, `optimize_autoloader`,
   `COMPOSER_HOME`, `failed_when: false`) → 9. `key:generate --force`
   (**seulement si `laravel_app_key` vide**) → 10. `artisan --version` →
11. `laravel_writable_dirs` (0775) → 12. `chown -R` →
**13.** `artisan db:show` (non bloquant) → **14.** `artisan migrate --force`
(`run_once`, gardé par `laravel_run_migrations`) → **15.** unité worker (opt-in).

## 10. Sécurité et secrets

- `.env` `0600` appartenant à `web0X` — **jamais** lisible par `www-data` ni
  par un autre utilisateur (tests : `sudo -u web01 php …`).
- Secrets injectés via vault (`vault_laravel_db_password`,
  `vault_laravel_app_key`, `vault_redis_password`) — la tâche template est
  masquée (`no_log: true`).
- `APP_KEY` **fixée** dans le vault : régénérée à chaque run, elle invaliderait
  sessions et cookies (`key:generate` donc conditionnel).
- Worker : `NoNewPrivileges`, `ProtectHome`, `ProtectSystem=full`, `PrivateTmp`.

## 11. Vérifications intégrées (non bloquantes)

- `stat artisan` + message « cloné / absent ou incomplet ».
- `composer install` : `failed_when: false` (dépôt placeholder).
- `artisan --version` : tolérant.
- **`artisan db:show`** : vraie connexion PDO via le `.env` courant, sous
  l'utilisateur du pool — la preuve que l'app joint le bon cluster
  (local : `127.0.0.1:5442`) ; la sortie n'expose jamais `DB_PASSWORD`.
  `changed_when: false` + `failed_when: false` (le tier web part avant le tier données).

## 12. Idempotence

- `git` module idempotent, `template` `.env`, `file` directories.
- `key:generate` **conditionné** à une clé vide → jamais rejoué avec un vault.
- Migrations : `changed_when: "'Nothing to migrate' not in stdout"` →
  `changed=0` au 2e passage ; **`run_once: true`** : les instances web
  partagent LA base.
- `laravel_clean_deploy` : ne supprime **que** si ni `artisan` ni `.git`
  n'existe — ne détruit jamais une application déployée.

## 13. Tests effectués (preuves)

- `.env` des 2 instances : `pgsql → 127.0.0.1:5442/laravel`,
  `SESSION_DRIVER/CACHE_STORE/QUEUE_CONNECTION = redis`,
  `REDIS_HOST=127.0.0.1`, `REDIS_PORT=6390` (vérifié — `README.md` §12).
- `artisan db:show` → PostgreSQL 18.6, port 5442.
- 3 migrations jouées **une seule fois** (`run_once`) ; 9 tables visibles à
  l'identique sur la réplique (5443).
- `APP_KEY` identique sur web01/web02 et stable entre deux exécutions.
- Sessions : cookie `laravel-web01-session` + clé Redis (TTL ≈ 120 min) ;
  `Cache::put/get` en db1 ; job poussé → `LLEN …queues:default = 1`.
- Worker **non** déployé (opt-in `false`) : tâches `[worker]` skipped.
- `local_web.yml` rejoué : **`changed=0`** sur les 2 hôtes.

## 14. Recettes de modification

| Je veux… | Fichier |
|---|---|
| Pointer vers mon dépôt | `inventories/local/hosts.yml` → `laravel_repo` + `laravel_branch` |
| Jouer les migrations en AWS | `-e laravel_run_migrations=true` (défaut `false`) |
| Activer le worker | `-e laravel_queue_worker_enabled=true` (voir `roles/laravel/README.md`) |
| Régénérer l'APP_KEY | `ansible-vault edit` du vault → `vault_laravel_app_key` (puis rejouer) |
| Forcer le re-clonage | `laravel_clean_deploy: true` **après** avoir retiré `artisan`/`.git` |

## 15. Intégration avec les autres tiers

- **Aval** : `apache` (sert `laravel_path/public`), `php` (exécute le code).
- **Amont** : `postgresql` (5442 local / 5432 AWS), `redis` (6390 local /
  6379 AWS) — les deux via `.env`.
- Migrations : exécutées **une fois** pour tout le tier (`run_once`), ce qui
  suppose que la base existe déjà (d'où l'ordre des étapes : DB avant web en
  pratique, ou `failed_when: false` de `db:show` au premier passage).

## 16. Points d'attention (pièges)

1. **Une seule migration pour le tier** : deux `migrate` simultanés sur une
   base partagée perdent la course (« relation "users" already exists »).
2. **`.env` 0600** : tout test doit se faire en tant que `web01`/`web02`
   (`sudo -u web01 php …`), pas `www-data`.
3. **`--force` requis** : `APP_ENV=production` interdit `migrate` sans.
4. **Placeholder de dépôt** : tant que `laravel_repo` contient
   `TON_COMPTE/TON_PROJET.git`, clonage/composer/artisan sont sautés (message
   explicite) — aucun faux déploiement silencieux.
5. **`ansible_remote_tmp: /tmp`** : sans lui, `become_user: web0X` échoue
   (nologin, pas de HOME) → warning/erreur Ansible.

## 17. Non trouvé dans le code actuel

- **`laravel_storage_dir`** (`roles/laravel/vars/main.yml`) est défini mais
  **jamais référencé** dans les tasks/templates (vérifié par grep) — les
  dossiers passent par `laravel_writable_dirs`.
- Aucune tâche de **rollback** de migration ni de gestion de seeds.
- Aucun health-check applicatif HTTP dans le rôle (le `/up` appartient au LB).
- Pas de gestion de `queue:restart` lors du redémarrage du worker (le
  redémarrage tue le job courant après `Restart` systemd).
- `laravel_db_host` par défaut vise `groups['databases'][0]` (nom d'hôte) :
  en local ce nom n'est pas résolvable — d'où la surcharge `127.0.0.1` dans
  l'inventaire (pas de valeur de repli native).

## 18. Mode AWS (défaut)

- `laravel_instance: ""` ⇒ déploiement standard dans `/var/www/laravel`,
  propriétaire `www-data`, service **aucun** (le worker est opt-in).
- Dépôt placeholder → **clonage sauté** tant que `laravel_repo` n'est pas
  renseigné (TODO `README.md` §13).
- `laravel_run_migrations: false` : **le schéma AWS n'est jamais modifié**.
- `laravel_clean_deploy: false` : aucun nettoyage destructif.
- DB par défaut : `groups['databases'][0]:5432` ; Redis : `groups['redis'][0]:6379`.
- `COMPOSER_HOME` vide ⇒ HOME de l'utilisateur SSH (comportement inchangé).

## 19. Mode local cloisonné

- `laravel_instance: web0X` ⇒ `/var/www/web0X`, propriétaire `web0X`,
  `laravel_clean_deploy: true`, `laravel_composer_home: /var/www/web0X/.composer`.
- Dépôt réel `laravel/laravel` branche **`13.x`** (cloné, composer, artisan).
- DB `127.0.0.1:5442`, Redis `127.0.0.1:6390`, `laravel_run_migrations: true`.
- `.env` : `SESSION_DRIVER/CACHE_STORE/QUEUE_CONNECTION = redis`.
- Worker **non** déployé par défaut (opt-in `false`, comme en AWS).
- Commande : `ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml --vault-password-file .vault_pass`

## 20. Problèmes rencontrés (réels)

| Symptôme | Cause | Correctif (dans le code) |
|---|---|---|
| Site vide / HTTP 500 au premier démarrage | `index.html` de validation occupait la racine ; dossiers `storage/*` absents ; extension `pgsql` manquante | nettoyage de clone (`laravel_clean_deploy`), création des 6 `laravel_writable_dirs`, `php8.5-pgsql` dans `php_packages` |
| `git clone` refuse la cible non vide | cible contenant l'`index.html` d'Apache | `stat` préalable + purge conditionnelle, puis recréation du répertoire |
| Deux `artisan migrate` simultanés → « relation "users" already exists » | 2 instances web partagent la même base | **`run_once: true`** sur la tâche de migration |
| `key:generate` régénérait la clé à chaque run (sessions invalidées, `changed` permanent) | `.env` réécrit avant la vérification de clé | `APP_KEY` **fixée** dans le vault ; `key:generate` exécuté **seulement si la clé est vide** |
| Warning/échec « no HOME directory » sur `become_user: web0X` | users `nologin` sans répertoire personnel | `ansible_remote_tmp: /tmp` (inventaire local) + `laravel_composer_home` dédié |
| « connection refused » sur Redis/PG au premier run | tiers web déployé **avant** les tiers données/cache | `artisan db:show` en `failed_when: false` ; ordre des playbooks (`local_cache` avant `local_web`) ; migrations gérées séparément |

## 21. Références

- [`../roles/laravel/README.md`](../roles/laravel/README.md) — dont la section
  « Notes » sur `key:generate` et le worker.
- [`../docs/installation-manuelle.md`](../docs/installation-manuelle.md) — §Tier 2, étapes 5/5bis/5ter.
- [`../README.md`](../README.md) — §8 (activation worker), §12 (preuves).
- [05 — PHP](05-php.md), [07 — PostgreSQL](07-postgresql.md), [08 — Redis](08-redis.md).

## 22. Résumé

Rôle de déploiement complet (clone → `.env` → composer → permissions →
vérifications → migrations opt-in → worker opt-in), **jamais destructif** et
**jamais bloquant** sur les tiers non encore déployés. Les deux garde-fous
structurants : `run_once` pour les migrations d'une base partagée et
`APP_KEY` fixée dans le vault. Mode local : `/var/www/web0X` propriétaire
`web0X`, DB 5442, Redis 6390. **Point ouvert** : `laravel_storage_dir` défini
mais inutilisé.


