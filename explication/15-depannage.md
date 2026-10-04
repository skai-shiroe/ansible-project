# 15 — Dépannage (problèmes réellement rencontrés)

Chaque entrée de ce document correspond à un problème **réellement survenu**
pendant la construction du lab — pas de liste théorique. Format :
**Symptôme → Cause → Correctif (tel qu'il est dans le code)**.

---

## A. Général / exécution

| # | Symptôme | Cause | Correctif |
|---|---|---|---|
| 1 | La commande « credentials » échoue (profils AWS) | credentials AWS/SSO absentes ou session expirée | régénération des credentials ; côté local, utiliser `-i inventories/local/hosts.yml` (aucune crédibilité AWS requise) |
| 2 | `local_db.yml` s'exécute sur **tous** les hôtes (`hosts: all`) | playbook lancé sans inventaire ciblé | **toujours** passer `-i inventories/local/hosts.yml` (sinon l'inventaire par défaut `aws_ec2.yml` de `ansible.cfg` s'applique) |
| 3 | `VARIABLE IS NOT DEFINED : vault_…` | playbook lancé sans `--vault-password-file .vault_pass` | ajouter `--vault-password-file .vault_pass` (`local_web`, `local_db`, `local_cache`) |
| 4 | Répertoire parasite `formation-defops/` apparaît | faute de frappe dans un chemin de commande | répertoire supprimé ; toujours vérifier `git status` avant commit |
| 5 | Erreur « Could not find the requested service X » en `--check` | artefact du mode simulation : le paquet n'est pas installé | normal en `--check` (`README.md` §8) ; en réel le paquet est installé juste avant. Pour cibler : `--limit webservers\|backoffice` |

---

## B. Tier web (Apache / PHP / Laravel)

| # | Symptôme | Cause | Correctif (dans le code) |
|---|---|---|---|
| 6 | PHP-FPM refuse de démarrer — **exit 78** | commentaires `#` dans `php-fpm.conf`/`pool.conf` : PHP-FPM n'accepte que `;` | templates réécrits avec `;` (rappel en tête de `php-fpm.conf.j2`) |
| 7 | HTTP **503** depuis le LB alors que FPM tourne | workers Apache restés sur l'**ancien** socket FPM après reconfiguration | redémarrage des deux services (handlers `Redémarrer Apache` / `Redémarrer PHP-FPM`) |
| 8 | La page Laravel n'apparaît pas : page « Instance Apache : web01 » | `index.html` de validation **masque** `index.php` (ordre du `DirectoryIndex`) | nettoyage avant clone (`laravel_clean_deploy`) + création d'une racine vide ; la page n'existe plus après clonage |
| 9 | `git clone` échoue : la destination n'est pas vide | cible contenant encore l'`index.html` d'Apache | `stat artisan`/`.git` → purge conditionnelle → recréation → clone |
| 10 | Warning/échec « no HOME directory » sur `become_user: web0X` | users `nologin` sans répertoire personnel (Ansible doit créer son répertoire temporaire) | `ansible_remote_tmp: /tmp` sur `web01`/`web02` (inventaire local) + `laravel_composer_home` dédié |
| 11 | Site **vide / HTTP 500** au premier passage | dossiers `storage/*` et `bootstrap/cache` manquants ; extension `pgsql` absente | création des 6 `laravel_writable_dirs` (0775) ; `php8.5-pgsql` dans `php_packages` |
| 12 | Deux `artisan migrate` simultanés → « relation "users" already exists » | 2 instances web partagent **LA même** base | `run_once: true` sur la tâche de migration |
| 13 | `changed` permanent + sessions invalidées à chaque run | `key:generate` régénérait `APP_KEY` à chaque exécution | `APP_KEY` **fixée** dans le vault ; `key:generate` seulement si la clé est vide |
| 14 | Script de test « Permission denied » sur `.env` | `.env` en `0600` appartenant à `web01` | exécuter en `sudo -u web01 php …` (l'utilisateur du service) |
| 15 | Dump des variables d'environnement **vide** | exécution hors du contexte du service / sans `.env` déployé | lire `/var/www/web0X/.env` (déployé par le rôle) ; ou exécuter les tâches après le template `.env` |
| 16 | Laravel ne se connecte pas à la base (`db:show` en échec) | tier web joué **avant** le tier données ; ou mauvais port (5432 système au lieu de 5442) | `failed_when: false` sur `db:show` (non bloquant) ; `laravel_db_host/port` forcés à `127.0.0.1:5442` dans l'inventaire local |
| 17 | Sessions/cache/queues en échec alors que Redis répond | extension `php8.5-redis` absente du pool | ajout à `php_packages` ; diagnostic explicite dans le debug final du rôle `php` |
| 18 | Ansible tente un SSH vers `10.0.0.x` et timeout | inventaire local sans `ansible_connection: local` | `ansible_connection: local` dans `all.vars` de `inventories/local/hosts.yml` |

---

## C. PostgreSQL (étape 5)

| # | Symptôme | Cause | Correctif (dans le code) |
|---|---|---|---|
| 19 | `pg_createcluster` échoue (droits / « must be run as root ») | exécuté en `become_user: postgres` : ne peut pas écrire `/etc/postgresql` ni lancer `initdb` | `become: true` **sans** `become_user` sur cette tâche (commentaire dans les tasks) |
| 20 | `pg_basebackup` : « An error is raised if the slot already exists » | option `-C` (`--create-slot`) alors que le primaire a déjà créé le slot | **`-C` retiré** ; `-S <slot>` seul (le primaire crée le slot idempotamment) |
| 21 | La réplique se connecte au **mauvais port** (elle « se backupe elle-même ») | `-p` utilisait le port local de la réplique (5443) au lieu de celui du primaire | `postgresql_primary_port: 5442` dans `inventories/local/host_vars/db02.yml` |
| 22 | Erreur de module : `port` n'est pas un paramètre de `postgresql_user/query` | renommage dans `community.postgresql` 5.0 : `port` → **`login_port`** | `login_port` + `login_unix_socket` utilisés partout |
| 23 | Réplique « password authentication failed for user replicator » | `-R` ne reporte pas toujours `password=` dans `primary_conninfo` (impossible de le saisir interactivement) | tâche `replace` **idempotente** n'écrivant que si `password=` est absent (constat PG 18.6 : `-R` le fait déjà) |
| 24 | Course au premier run : la réplique échoue tant que le primaire n'est pas prêt | primaire et réplique = 2 clusters d'une **même** machine, exécution parallèle | `serial: 1` dans `local_db.yml` (db01 → handlers → db02) |
| 25 | `pg_hba` refuse la réplication (« no pg_hba.conf entry … ») | ligne générée depuis `ansible_host` (IP privée documentaire `10.0.0.31`) | `postgresql_replication_allowed_addresses: ['127.0.0.1/32']` (host_vars local) |
| 26 | Config non appliquée au moment de la connexion de la réplique | les handlers tournent en **fin de play** | `meta: flush_handlers` placé **avant** les tâches de réplication |
| 27 | La réplique se connectait à « db01 » par DNS (inexistant en local) | `postgresql_primary_host` par défaut = `groups['databases'][0]` (AWS) | `postgresql_primary_host: 127.0.0.1` + `include_vars` (préséance maximale) en `pre_tasks` de `local_db.yml` |
| 28 | Écriture refusée sur la réplique : « read-only transaction » | **normal** : `default_transaction_read_only = on` | c'est le test de succès, pas une erreur (§ tests) |

---

## D. Redis (étape 7)

| # | Symptôme | Cause | Correctif (dans le code) |
|---|---|---|---|
| 29 | Redis du projet **introuvable** : rien sur 6390 | playbook non exécuté / mauvais inventaire | `local_cache.yml` avec `-i inventories/local/hosts.yml` |
| 30 | `NOAUTH Authentication required.` au test | `requirepass` actif — le test **sans** mot de passe est censé échouer | double test du rôle (sans/avec `REDISCLI_AUTH`), tous deux non bloquants |
| 31 | ECONNREFUSED / « Redis server has gone away » côté Laravel | `.env` pointant vers **6379** (Redis étranger, sans le bon mot de passe) — ou sentinel étranger | `laravel_redis_host: 127.0.0.1` + `laravel_redis_port: 6390` (inventaire local) |
| 32 | Le méta-service `redis-server` ne se relance pas / ne doit **jamais** tourner pour le projet | il pilote le Redis d'une **autre application** (6379 + sentinel 26379) | aucune tâche n'appelle `redis-server` : le projet utilise **`redis-redis01`** uniquement |
| 33 | Mot de passe Redis visible dans `ps` | `redis-cli -a <mdp>` | **`REDISCLI_AUTH`** en `environment` (+ `no_log`) |
| 34 | FPM redémarré sans `phpredis` | ordre des playbooks | `local_cache.yml` **avant** `local_web.yml` (documenté `README.md` §8) |

---

## E. Load balancer

| # | Symptôme | Cause | Correctif (dans le code) |
|---|---|---|---|
| 35 | Impossible de lier :80 en local | `nginx.service` étranger occupe le port 80 | instance `nginx-lb01` sur **9080** ; la tâche « site par défaut » est conditionnée au mode AWS |
| 36 | `502 Bad Gateway` depuis le LB | backends injoignables : mauvais port ou noms `web01/web02` non résolubles | upstream lit `hostvars[web0X].apache_port` (9081/9082) ; prérequis `/etc/hosts` côté machine (**non géré par le dépôt**) |
| 37 | Upstream pointé vers 8080 (WAF étranger) | hôte sans `apache_port` → repli `nginx_lb_backend_port: 8080` | chaque web doit porter son `apache_port` dans l'inventaire local |

---

## F. Précedence / variables

| # | Symptôme | Cause | Correctif |
|---|---|---|---|
| 38 | Une variable « ne prend pas » | `roles/<r>/vars/main.yml` (préséance haute) écrase l'inventaire | chemins déplacés vers `defaults/` ; `vars/` vidé des 6 rôles cloisonnés (cf. [02](02-ansible-variables-et-precedence.md)) |
| 39 | La réplique locale lit les valeurs AWS (5432, hôte `db01`) | `playbooks/host_vars` (symlink) est chargé automatiquement | `include_vars` en `pre_tasks` de `local_db.yml` (préséance maximale) |
| 40 | Les `group_vars/` sont ignorées depuis `playbooks/` | absence des liens symboliques | `playbooks/group_vars -> ../group_vars`, `playbooks/host_vars -> ../host_vars` |

---

## G. Méthode de dépannage rapide

```bash
# 1. Où est-ce que ça échoue ? (tâche + hôte)
ansible-playbook … -v 2>&1 | tail -50

# 2. Quelles variables voit l'hôte ?
ansible -i inventories/local/hosts.yml web01 -m debug -a "var=apache_port"
ansible-inventory -i inventories/local/hosts.yml --host web01

# 3. Quel service / port ?
systemctl status <service> ; journalctl -u <service> -n 50 ; ss -ltnp

# 4. Simulation sans impact
ansible-playbook … --check --diff

# 5. Non-régression (les étrangers sont-ils vivants ?)
pg_lsclusters ; redis-cli -p 6379 ping ; redis-cli -p 26379 ping
```

→ Suite : [16 — Commandes utiles](16-commandes-utiles.md)

## H. Références

- [`../README.md`](../README.md) — §8 (artefacts `--check`), §12, §13 (TODO).
- [`../docs/installation-manuelle.md`](../docs/installation-manuelle.md) — équivalents manuels de chaque tâche.
- Les commentaires inline des `tasks/` : chaque correctif y est justifié au moment où il a été appliqué.

