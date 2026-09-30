# Rôle `php`

Installe **PHP-FPM** et ses extensions, puis configure PHP via un
**fichier `.ini` dédié** (sans toucher au `php.ini` de la distribution).

## Variables

| Variable | Emplacement | Défaut | Rôle |
|---|---|---|---|
| `php_version` | `group_vars/webservers.yml` (défaut `defaults` aligné) | `"8.5"` | Version de PHP (native sur Ubuntu 26.04, aucun PPA) |
| `php_fpm_service` | `defaults` | `php{{ php_version }}-fpm` | Service systemd |
| `php_packages` | `defaults` | cli, fpm, pgsql, mbstring, xml, curl, zip, bcmath, intl | Paquets |
| `php_ini_settings` | `defaults` | memory_limit, upload_max_filesize, post_max_size, date.timezone | Valeurs du `.ini` |
| `php_display_errors` | `defaults` | `false` | **Conditionnel** : affichage des erreurs |
| `php_opcache_enabled` | `defaults` | `true` | **Conditionnel** : bloc OPcache |

## Fichiers de configuration (templates — règle du formateur)

| Template | Destination |
|---|---|
| `templates/99-laravel.ini.j2` | `/etc/php/{{ php_version }}/fpm/conf.d/99-laravel.ini` |

PHP est compilé avec un répertoire de scan
(`Scan this dir for additional .ini files => /etc/php/<version>/fpm/conf.d`),
donc le fichier est chargé automatiquement et le `php.ini` reste intact.

**Conditionnels Jinja présents** : gestion des erreurs et bloc OPcache.

> ℹ️ Sur **Ubuntu 26.04**, OPcache est **compilé dans le binaire** `php8.5`
> (aucun paquet `php8.5-opcache` dans les dépôts) : le bloc OPcache du
> template s'applique sans paquet supplémentaire.

## Tâches de vérification (non bloquantes)

- `php -v` : version installée (le rôle **ne force aucune version** :
  `php_version` est surchargeable en `group_vars`).
- `php -m` (`failed_when: false`) : confirme la présence de `pdo_pgsql`
  (indispensable à la connexion PostgreSQL de Laravel).

## Fichiers statiques

Aucun.

## Handlers

- `Redémarrer PHP-FPM`

## Appelé par

`playbooks/applications.yml` — groupe `webservers` (après `apache`)

