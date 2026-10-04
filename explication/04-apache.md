# 04 — Rôle `apache`

## 1. À quoi sert ce rôle ?

Installe Apache HTTP Server, configure son **port d'écoute**, son **vhost
Laravel** (délégation PHP → PHP-FPM via socket Unix), ses **modules**, et — en
mode cloisonné — crée une **instance isolée** par hôte web (répertoires,
utilisateur, unité systemd dédiés). Il pose aussi une page `index.html` de
validation en mode instance.

## 2. Où il vit dans le dépôt

```
roles/apache/
├── defaults/main.yml      # toutes les variables (contrat du rôle)
├── vars/main.yml          # vidé (seul « --- ») — voir explication/02
├── tasks/main.yml         # 260 lignes : mode AWS + mode instance + checks
├── handlers/main.yml      # « Redémarrer Apache »
├── meta/main.yml          # dependencies: []
├── templates/
│   ├── apache2.conf.j2        # [instance] ServerRoot dédié + modules autochargés
│   ├── apache_ports.conf.j2   # « Listen {{ apache_port }} »
│   ├── apache.service.j2      # [instance] unité apache-<instance>
│   ├── apache_vhost.conf.j2   # vhost (SSL conditionnel, proxy_fcgi)
│   ├── envvars.j2             # [instance] APACHE_RUN_* + unset HOME
│   └── index.html.j2          # [instance] page de validation
└── README.md
```

## 3. Comment il est appelé

- **AWS** : `playbooks/applications.yml` → `roles: [apache, php, laravel]` sur `webservers`.
- **Local** : `playbooks/local_web.yml` (même trio) sur `webservers` de
  `inventories/local/hosts.yml`.
- `apache_instance: ""` ⇒ mode AWS ; `apache_instance: web01|web02` ⇒ mode cloisonné.

## 4. Prérequis et dépendances

- Paquet `apache2` (installé par le rôle lui-même).
- En mode instance : modules lus dans `/etc/apache2/mods-available` (paquet
  système, **lecture seule**) — pas de `a2enmod`.
- Le vhost référence `apache_php_fpm_socket` : doit pointer sur le socket géré
  par le rôle `php` (`/run/php-web0X/php-fpm.sock` en local).
- `meta/main.yml` : `dependencies: []` (aucune dépendance déclarée —
  l'ordre vient des playbooks).

## 5. Ports, services et chemins

| Élément | Mode AWS | Mode local (web01 / web02) |
|---|---|---|
| Port | `8080` (`group_vars/webservers.yml`) | `9081` / `9082` (inline inventaire) |
| Service | `apache2` | `apache-web01` / `apache-web02` |
| Config | `/etc/apache2` | `/etc/apache2-web01` / `-web02` |
| Logs | `/var/log/apache2` | `/var/log/apache2-web01` / `-web02` |
| Runtime | `/run/apache2` | `/run/apache2-web01` / `-web02` |
| Utilisateur | `www-data` | `web01` / `web02` (nologin) |
| DocumentRoot | `/var/www/laravel/public` | `/var/www/web01/public` / `/var/www/web02/public` |
| Vhost | `sites-available/laravel.conf` | idem, sous le répertoire dédié |

## 6. Variables

| Variable | Valeur actuelle (AWS → local) | Fichier | Pourquoi | Exemple de modification | Impact |
|---|---|---|---|---|---|
| `apache_instance` | `""` → `web01` | `defaults/main.yml` → `inventories/local/hosts.yml` | Bascule AWS/instance | `-e apache_instance=web03` | active tout le bloc `[instance]` |
| `apache_instance_enabled` | `{{ apache_instance \| length > 0 }}` | `defaults` | Dériver les `when` | — (calculé) | gate de toutes les tâches d'instance |
| `apache_service` | `apache2` → `apache-web0X` | `defaults` | Ne jamais appeler un service étranger | — | handler + démarrage |
| `apache_port` | `80` (defaults) / `8080` (group_vars) → `9081`/`9082` | `defaults` → `group_vars/webservers.yml` → inventaire local | Port d'écoute | éditer `inventories/local/hosts.yml` | `ports.conf`, vhost, upstream du LB (via `hostvars`) |
| `apache_document_root` | `/var/www/html` → `/var/www/laravel/public` → `/var/www/web0X/public` | idem | Racine web | idem | répertoire créé + vhost + `<Directory>` |
| `apache_user` / `apache_group` | `www-data` → `web0X` | idem | Isolation des workers | idem | propriétaire des logs/racine, `User=` de l'unité |
| `apache_server_name` | `{{ inventory_hostname }}` | `defaults` | ServerName propre | `-e apache_server_name=exemple.test` | vhost |
| `apache_conf_dir` | `/etc/apache2` → `/etc/apache2-web0X` | `defaults` (calculé) | Cloisonnement | — | tous les chemins de config |
| `apache_log_dir` / `apache_run_dir` | calculés | `defaults` | idem | — | envvars, unité, tests `-t` |
| `apache_modules` | `rewrite, headers, proxy, proxy_fcgi, setenvif` | `defaults` (+ surcharge `group_vars/webservers.yml`) | Modules AWS via `a2enmod` | ajouter une entrée dans `group_vars` | activés en mode AWS seulement |
| `apache_instance_modules` | 24 modules (`mpm_prefork`, `proxy_fcgi`…) | `defaults` | Jeu **autonome** pour l'instance (sans mod_php/mod_security) | — | inclus dans `apache2.conf.j2` |
| `apache_ssl_enabled` | `false` | `defaults` (piloté par group_vars) | Bascule HTTPS | `true` + cert | vhost en 443, module `ssl` ajouté |
| `apache_ssl_port/cert/key` | `443`, snakeoil | `defaults` | TLS | — | vhost |
| `apache_vhost_file` | `laravel.conf` | `defaults` | Nom du vhost | — | fichiers available/enabled |
| `apache_php_fpm_socket` | `/run/php/php8.5-fpm.sock` → `/run/php-web0X/php-fpm.sock` | `defaults` → inventaire local | Délégation PHP | — | `SetHandler proxy:unix:...` (503 si faux) |
| `apache_systemd_dir` | `/etc/systemd/system` | `defaults` | Destination de l'unité | — | — |

## 7. Templates

| Template | Destination (mode instance) | Rôle |
|---|---|---|
| `apache2.conf.j2` | `/etc/apache2-web0X/apache2.conf` | `ServerRoot` dédié, modules autochargés depuis `/etc/apache2/mods-available` (chemins absolus, `IncludeOptional`), `<Directory>` de la racine |
| `apache_ports.conf.j2` | `.../ports.conf` | `Listen {{ apache_port }}` (unique directive) |
| `apache_vhost.conf.j2` | `sites-available/laravel.conf` | VirtualHost (SSL conditionnel), `DocumentRoot`, `SetHandler proxy:unix:{{ apache_php_fpm_socket }}\|fcgi://localhost/` |
| `envvars.j2` | `.../envvars` | `APACHE_RUN_USER/GROUP`, `APACHE_RUN_DIR`, `unset HOME` (users nologin) |
| `apache.service.j2` | `/etc/systemd/system/apache-web0X.service` | `User=web0X`, `RuntimeDirectory=apache2-web0X`, source via `-d {{ apache_conf_dir }}`, hardening (`NoNewPrivileges`, `ProtectHome`, `ProtectSystem=full`, `PrivateTmp`) |
| `index.html.j2` | `{{ apache_document_root }}/index.html` | page de validation (`<h1>Instance Apache : web0X</h1>`) |

Aucun `lineinfile`/`blockinfile` (règle du formateur — `README.md` §14).

## 8. Handlers

**`Redémarrer Apache`** (`handlers/main.yml`) : `ansible.builtin.systemd` avec
`daemon_reload: true` (l'unité est nouvelle en mode instance), `state: restarted`,
`failed_when` tolérant **uniquement** en `--check`. Déclenché par toutes les
tâches de configuration (ports, vhost, unité, envvars…).

## 9. Parcours des tâches

**Mode AWS** (`when: not apache_instance_enabled`) :
1. `apt: apache2` → 2. `apache2_module` (`a2enmod`, `apache_modules_effective`)
→ 3. `ports.conf` → 4. vhost → 5. symlink `sites-enabled` (`force: true` pour
`--check`) → 6. démarrage `apache2`.

**Mode instance** (`when: apache_instance_enabled`) :
1. groupe + utilisateur `web0X` (`nologin`, `create_home: false`) →
2. arborescence `/etc/apache2-web0X/{,sites-available,sites-enabled}` →
3. répertoire de logs (propriétaire `web0X`) → 4. racine web →
5. démarrage provisoire (`daemon_reload`) → 6. `apache2.conf` → 7. `envvars` →
8. `ports.conf` → 9. vhost → 10. symlink → 11. unité systemd → 12. redémarrage final.

**Vérifications non bloquantes** (les deux modes) : `apache2ctl configtest`
(AWS) ou `/usr/sbin/apache2 -t -d ...` avec variables d'env (instance),
`changed_when: false` + `failed_when: false`, affichage du résultat ; requête
des modules (`state: query`) ; `service_facts` + récapitulatif.

## 10. Sécurité et secrets

- Aucun secret dans ce rôle.
- Utilisateur système dédié `nologin` (mode instance), logs appartenant à `web0X`.
- Unité systemd avec hardening complet.
- Port > 1024 : aucun besoin de root pour le worker.

## 11. Vérifications intégrées (non bloquantes)

- `configtest` syntaxe (tolérant : `failed_when: false`).
- État des modules (`query`) avec récapitulatif actifs/en échec.
- `service_facts` → affiche l'état du service + port.
- **Ne bloquent jamais** le déploiement : elles diagnostiquent.

## 12. Idempotence

- `apt` `state: present`, `file: state: link/directory`, `template` (hash).
- Le démarrage utilise `state: started` (pas de restart systématique) ;
  les redémarrages passent par le handler, donc **uniquement** en cas de
  changement de config.
- Tâches de vérification : `changed_when: false`.
- `failed_when: <reg> is failed and not ansible_check_mode` : tolérance
  **uniquement** en simulation.

## 13. Tests effectués (preuves)

(Valide pour le tier web complet ; voir aussi [14 — Tests & validation](14-tests-et-validation.md))

- `local_web.yml` rejoué : **`changed=0`** sur `web01` et `web02`.
- HTTP 200 sur `http://127.0.0.1:9081/` et `:9082/`.
- Les services `apache-web01/02` : `active`.
- `18-main` (5432) et les services étrangers intacts après exécution.

## 14. Recettes de modification

| Je veux… | Fichier à modifier |
|---|---|
| Changer le port de web01 | `inventories/local/hosts.yml` → `web01: apache_port:` |
| Ajouter un module (AWS) | `group_vars/webservers.yml` → `apache_modules` |
| Passer en HTTPS | `group_vars/webservers.yml` → `apache_ssl_enabled: true` (+ cert) |
| Changer la racine web locale | `inventories/local/hosts.yml` → `apache_document_root` |
| Pointer vers un autre socket FPM | idem → `apache_php_fpm_socket` |

## 15. Intégration avec les autres tiers

- **Amont** : `nginx_lb` (`upstream` → `web0X:{{ hostvars[web0X].apache_port }}`).
- **Aval** : `php` (socket `/run/php-web0X/php-fpm.sock`), `laravel`
  (code dans `apache_document_root`).
- **Playbook** : `local_web.yml` / `applications.yml`, dans l'ordre
  `apache → php → laravel`.

## 16. Points d'attention (pièges)

1. **`index.html` masque `index.php`** : Apache sert `DirectoryIndex index.html`
   avant `index.php` → la page Laravel n'apparaît plus. La page de validation
   est volontairement déployée **avant** le clone Laravel ; elle disparaît au
   nettoyage (`laravel_clean_deploy`).
2. **Ancien socket** : après changement de `apache_php_fpm_socket`, un worker
   peut rester sur l'ancien socket (503) → redémarrage requis (le handler le fait).
3. **`a2enmod` en mode AWS uniquement** : il écrirait dans `/etc/apache2` partagé.
4. **`envvars` avec `unset HOME`** : nécessaire car `web0X` est `nologin` sans HOME.
5. Le paquet système fournit les `.so` : l'instance reste **dépendante du paquet
   `apache2`**, mais pas de sa configuration.

## 17. Non trouvé dans le code actuel

- Aucun health-check d'Apache côté LB au-delà du `/up` renvoyé par Nginx lui-même.
- Pas de gestion de rotation des logs (logrotate) dans le rôle.
- Pas de `ServerName` SSL ni ACME : `apache_ssl_cert/key` pointent sur le
  certificat snakeoil du paquet (`/etc/ssl/certs/ssl-cert-snakeoil.pem`).
- Le dépôt ne crée **pas** l'entrée `/etc/hosts` (`web01`, `web02`) dont
  dépend l'upstream Nginx — aucune tâche ne la gère (vérifié par grep).

## 18. Mode AWS (défaut)

- `apache_instance: ""` ⇒ **toutes** les tâches `when: not apache_instance_enabled`.
- Service système `apache2`, config `/etc/apache2`, modules via `a2enmod`
  (`apache_modules_effective` ajoute `ssl` si `apache_ssl_enabled`).
- Port `8080` (`group_vars/webservers.yml`), utilisateur `www-data`,
  DocumentRoot `/var/www/laravel/public`.
- Vérification via `apache2ctl configtest`.
- Aucune unité système créée, aucun utilisateur créé.

## 19. Mode local cloisonné

- `apache_instance: web01|web02` (inline dans `inventories/local/hosts.yml`).
- Ports **9081/9082**, services `apache-web01/02`, users `web01/02`,
  configs `/etc/apache2-web0X`, logs `/var/log/apache2-web0X`.
- Modules : jeu autonome `apache_instance_modules` (24 entrées, sans mod_php
  ni mod_security — indépendance vis-à-vis de l'état système).
- Page `index.html` de validation déployée dans la racine web.
- Tests de syntaxe : `/usr/sbin/apache2 -t -d {{ apache_conf_dir }}` + env.
- Exécution : `ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml --vault-password-file .vault_pass`

## 20. Problèmes rencontrés (réels)

| Symptôme | Cause | Correctif (dans le code) |
|---|---|---|
| HTTP 503 depuis le LB alors que PHP tourne | workers Apache restés sur l'**ancien** socket FPM après changement de configuration | redémarrage complet du service (handler `Redémarrer Apache`) ; le `SetHandler` du vhost lit le socket courant |
| La page Laravel n'apparaît pas (page blanche « Instance Apache ») | `index.html` (validation) **écrase** `index.php` dans l'ordre de résolution | page déployée seulement en mode instance, remplacée par le clone Laravel (`laravel_clean_deploy` crée une racine vide) |
| En `--check`, échec « No such file or directory » sur le symlink `sites-enabled` | la source n'existe pas encore en simulation | `force: true` sur le `file: state: link` |
| En `--check`, « Could not find the requested service apache2 » | le paquet n'est pas installé en simulation | `failed_when: <reg> is failed and not ansible_check_mode` |

## 21. Références

- [`../roles/apache/README.md`](../roles/apache/README.md) — référence du rôle.
- [`../docs/installation-manuelle.md`](../docs/installation-manuelle.md) — §Tier 2 (manuel avant automatisation).
- [`../README.md`](../README.md) — §6 (rôles), §12 (validation).
- [02 — Variables & précédence](02-ansible-variables-et-precedence.md),
  [05 — PHP](05-php.md), [09 — Load balancer](09-load-balancer.md).

## 22. Résumé

Rôle **double mode** : AWS = `apache2` classique sur 8080 ; local = une instance
complète par hôte (`/etc/apache2-web0X`, `apache-web0X`, `web0X`, port 908X).
Il ne touche jamais la configuration système partagée hors paquet. Pièges
majeurs : `index.html` vs `index.php`, socket FPM après reconfiguration,
`a2enmod` réservé au mode AWS. Aucun secret manipulé.


