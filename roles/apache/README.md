# Rôle `apache`

Installe et configure le serveur web **Apache** pour l'application Laravel :
modules, port d'écoute, vhost, délégation PHP à PHP-FPM et bascule HTTP/HTTPS.

## Variables

| Variable | Emplacement | Défaut | Rôle |
|---|---|---|---|
| `apache_package` / `apache_service` | `defaults` + `group_vars` | `apache2` | Paquet et service |
| `apache_port` | `group_vars/webservers.yml` | `8080` | Port d'écoute |
| `apache_document_root` | `group_vars/webservers.yml` | `/var/www/laravel/public` | Racine web |
| `apache_modules` | `group_vars/webservers.yml` | rewrite, headers, proxy, proxy_fcgi, setenvif | Modules activés |
| `apache_modules_effective` | `defaults` | ajoute `ssl` si HTTPS actif | Liste réellement appliquée |
| `apache_ssl_enabled` | `group_vars/webservers.yml` | `false` | Bascule HTTP/HTTPS (conditionnel) |
| `apache_vhost_file` | `defaults` | `laravel.conf` | Nom du vhost |
| `apache_php_fpm_socket` | `defaults` | `/run/php/php{{ php_version }}-fpm.sock` | Socket PHP-FPM |

## Fichiers de configuration (templates — règle du formateur)

| Template | Destination |
|---|---|
| `templates/apache_vhost.conf.j2` | `/etc/apache2/sites-available/{{ apache_vhost_file }}` |
| `templates/apache_ports.conf.j2` | `/etc/apache2/ports.conf` |

Le vhost contient un **bloc conditionnel Jinja** :

```jinja
{% if apache_ssl_enabled %}
<VirtualHost *:{{ apache_ssl_port }}>   ... SSLCertificateFile ...
{% else %}
<VirtualHost *:{{ apache_port }}>
{% endif %}
```

## Tâches de vérification (non bloquantes)

- `apache2ctl configtest` : test de syntaxe (`failed_when: false`).
- `apache2_module` `state: query` sur chaque module attendu : confirme que les
  modules sont actifs, sans modifier quoi que ce soit.
- `service_facts` + debug : statut du service.

## Fichiers statiques

Aucun — voir `roles/<rôle>/files/` pour cette règle.

## Handlers

- `Redémarrer Apache`

## Appelé par

`playbooks/applications.yml` — groupe `webservers`

