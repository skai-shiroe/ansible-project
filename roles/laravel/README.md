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
| `laravel_db_*` | `defaults` | pointe vers `groups['databases'][0]` | Connexion PostgreSQL |
| `laravel_*_driver` | `defaults` | `redis` (cache, sessions, queues) | Drivers Laravel |
| `laravel_app_key` | `defaults` | `{{ vault_laravel_app_key }}` | **Secret** (vault) |
| `laravel_writable_dirs` | `defaults` | `storage/*`, `bootstrap/cache` | Dossiers créés (sinon HTTP 500) |

## Fichiers de configuration (templates)

| Template | Destination |
|---|---|
| `templates/env.j2` | `{{ laravel_path }}/.env` |

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

## Notes

- `php artisan key:generate` n'est exécuté que **si la clé est vide**
  (`when: laravel_app_key | length == 0`) → idempotent.
- Toute la partie base de données / Redis passe par `group_vars` et les
  secrets par `group_vars/all/vault.yml`.

## Appelé par

`playbooks/applications.yml` — groupe `webservers` (après `apache` et `php`)

