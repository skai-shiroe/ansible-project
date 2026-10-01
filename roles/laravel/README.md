# Rôle `laravel`

Déploie l'application **Laravel** : dépendances système, clone Git,
fichier `.env` généré, `composer install`, clé d'application et permissions.

## Variables

| Variable | Emplacement | Défaut | Rôle |
|---|---|---|---|
| `laravel_path` | `group_vars/webservers.yml` | `/var/www/laravel` | Racine de l'application |
| `laravel_repo` | `defaults` | **placeholder à remplacer** | Dépôt Git |
| `laravel_repo_configured` | `defaults` (dérivée) | `false` tant que le placeholder est en place | Clonage **sauté** si `false` ; échec **bloquant** si un vrai dépôt échoue |
| `laravel_branch` | `defaults` | `main` | Branche déployée |
| `laravel_owner` / `laravel_group` | `defaults` | `www-data` | Propriétaire des fichiers |
| `laravel_db_*` | `defaults` | pointe vers `groups['databases'][0]`, port `5432` | Connexion PostgreSQL (AWS) |
| `laravel_db_host` / `laravel_db_port` | **`inventories/local/hosts.yml`** | `127.0.0.1` / `5442` | Connexion cloisonnée : primaire `18/db01`, jamais le 5432 du système |
| `laravel_run_migrations` | `defaults` → surchargé en local | `false` (AWS) / `true` (local) | Autorise `php artisan migrate --force` |
| `laravel_*_driver` | `defaults` | `redis` (cache, sessions, queues) | Drivers Laravel |
| `laravel_redis_host` / `laravel_redis_port` | `defaults` → surchargé en local | `groups['redis'][0]` / `6379` (local : **`127.0.0.1`** / **`6390`**) | Connexion Redis de l'application (`REDIS_HOST` / `REDIS_PORT`) |
| `laravel_redis_password` | `defaults` | `{{ vault_redis_password \| default('') }}` | **Secret** — écrit dans le `.env` (`REDIS_PASSWORD`) |
| `laravel_queue_worker_enabled` | `defaults` → `false` en local | `false` | **Opt-in** : l'unité `laravel-queue-*` n'est déployée que si `true` |
| `laravel_queue_worker_service` / `_sleep` / `_tries` / `_max_time` | `defaults` (calculées) | `laravel-queue[-<inst>]`, `3`, `3`, `3600` | Nom de l'unité et réglages du worker |
| `laravel_app_key` | `defaults` | `{{ vault_laravel_app_key }}` | **Secret** (vault) — clé fixe ⇒ idempotence |
| `laravel_writable_dirs` | `defaults` | `storage/*`, `bootstrap/cache` | Dossiers créés (sinon HTTP 500) |

## Fichiers de configuration (templates)

| Template | Destination |
|---|---|
| `templates/env.j2` | `{{ laravel_path }}/.env` (mode `0600`, propriétaire = utilisateur de l'instance) |
| `templates/queue-worker.service.j2` | `{{ laravel_queue_worker_conf_dir }}/{{ laravel_queue_worker_service }}.service` — **uniquement si `laravel_queue_worker_enabled`** |

## Fichiers statiques

Aucun : le code provient du dépôt Git (`ansible.builtin.git`).

## Tâches de vérification (non bloquantes)

- Le clonage est **sauté** tant que `laravel_repo` vaut le placeholder
  (`laravel_repo_configured: false`) — le dry-run local franchit le rôle.
- Si un **vrai** dépôt est configuré, un échec de clonage **bloque** le
  déploiement (pas d'application silencieusement vide en production).
- `stat` sur `artisan` : `composer install`, `key:generate`
  et `artisan --version` sont **sautés** si le dépôt n'est pas cloné ;
  `failed_when: false` sur ces tâches tolère une application incomplète.
- Création des dossiers `storage/*` et `bootstrap/cache` (HTTP 500 sinon).
- `php artisan db:show` (`changed_when: false`, `failed_when: false`) : ouvre une
  **vraie** connexion PDO avec le `.env` de l'instance, sous l'utilisateur de
  PHP-FPM → preuve que l'application joint le bon cluster. La sortie n'expose
  jamais `DB_PASSWORD` (contrairement à un DSN complet).
- `php artisan migrate --force` : seulement si `laravel_run_migrations`
  (`false` par défaut ⇒ AWS inchangé). La tâche porte **`run_once: true`** :
  les instances web partagent une même base, deux `migrate` concurrents
  tenteraient de créer les mêmes tables (`relation "users" already exists`).
  `changed_when: 'Nothing to migrate' not in stdout` ⇒ idempotent.
- **Worker de file d'attente (opt-in)** : `QUEUE_CONNECTION=redis` ne fait que
  *pousser* les jobs dans Redis — sans consommateur, ils s'accumulent. Le rôle
  peut donc déployer une unité systemd dédiée
  (`templates/queue-worker.service.j2` → `laravel-queue[-<instance>].service`,
  exécutant `php artisan queue:work` avec `--sleep` / `--tries` / `--max-time`),
  mais **uniquement** si `laravel_queue_worker_enabled: true`.
  - par défaut `false` : les tâches `[worker]` sont **sautées** et un debug
    rappelle que les jobs poussés restent **sans consommateur** ;
  - activation (une instance à la fois, l'inventaire local force la var) :

  ```bash
  ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml \
    --vault-password-file .vault_pass -e laravel_queue_worker_enabled=true
  ```

  Le démarrage, l'activation et le rechargement (`daemon_reload`) passent par
  le handler `Redémarrer le worker de file d'attente` (tolérant en `--check`
  uniquement).

## Notes

- `php artisan key:generate` n'est exécuté que **si la clé est vide**
  (`when: laravel_app_key | length == 0`) → idempotent. Le rôle écrivant le
  `.env` **avant**, une clé absente du vault serait régénérée à chaque
  exécution (sessions/cookies invalidés, `changed` systématique) : la clé est
  donc **stockée dans le vault** (`vault_laravel_app_key`) et simplement
  réécrite par le template.
- Toute la partie base de données / Redis passe par `group_vars` et les
  secrets par le vault (local : `inventories/local/`).

## Appelé par

- `playbooks/applications.yml` — groupe `webservers` (après `apache` et `php`)
- `playbooks/local_web.yml` — instances cloisonnées `web01` / `web02`

