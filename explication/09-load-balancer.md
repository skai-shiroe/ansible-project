# 09 — Rôle `nginx_lb`

## 1. À quoi sert ce rôle ?

Installe Nginx en **équilibreur de charge** devant les instances web : génère
un `upstream` pointant vers chaque backend (port lu par hôtes), un `server`
d'entrée avec health-check `/up` et proxy pass, gère le rechargement à chaud —
et en mode cloisonné, une **instance complète** (répertoires, utilisateur,
unité systemd propres) à côté de Nginx système.

## 2. Où il vit dans le dépôt

```
roles/nginx_lb/
├── defaults/main.yml      # ports, upstream, méthodes, timeouts…
├── vars/main.yml          # vidé (seul « --- »)
├── tasks/main.yml         # 193 lignes
├── handlers/main.yml      # « Recharger Nginx » (state: reloaded)
├── templates/
│   ├── nginx.conf.j2      # [instance] config principale
│   ├── nginx_lb.conf.j2   # upstream + server (partagé aux 2 modes)
│   └── nginx.service.j2   # [instance] unité nginx-<instance>
└── README.md
```

## 3. Comment il est appelé

- **AWS** : `site.yml` (dernier play, `hosts: loadbalancers`) — il ne doit
  pointer que vers des backends existants.
- **Local** : `playbooks/local_lb.yml` (`hosts: loadbalancers`, hôte `lb01`).
- Aucun autre playbook n'appelle ce rôle (contrairement à `applications.yml`).

## 4. Prérequis et dépendances

- Paquet `nginx` (installé par le rôle).
- **Backends résolubles** : l'upstream utilise les **noms d'hôtes**
  (`web01`, `web02`) — leur résolution (DNS ou `/etc/hosts`) est un prérequis
  machine **non géré par le dépôt** (vérifié : aucune tâche ne touche
  `/etc/hosts`).
- `nginx_lb_backend_servers` = `groups['webservers']` (défini dans
  `group_vars/loadbalancers.yml`).
- `meta/main.yml` : `dependencies: []`.

## 5. Ports, services et chemins

| Élément | Mode AWS | Mode local (lb01) |
|---|---|---|
| Port d'écoute | **80** | **9080** |
| Service | `nginx` | `nginx-lb01` |
| Config | `/etc/nginx` (partagée avec `nginx.service` système) | `/etc/nginx-lb01` |
| Site | `sites-available/laravel_web.conf` → `sites-enabled/` | idem sous le répertoire dédié |
| Utilisateur | `www-data` | `lb01` (nologin) |
| Logs / PID / run | `/var/log/nginx`, `/run/nginx` | `/var/log/nginx-lb01`, `/run/nginx-lb01/nginx.pid` |
| Backends | `web01:8080`, `web02:8080` | `web01:9081`, `web02:9082` |
| Health-check | `GET /up` → `200 OK` (répondu par Nginx lui-même) | idem |

## 6. Variables

| Variable | Valeur actuelle (AWS → local) | Fichier | Pourquoi | Exemple de modification | Impact |
|---|---|---|---|---|---|
| `nginx_lb_instance` | `""` → `lb01` | `defaults` → inventaire local | Bascule mode | — | gate de toutes les tâches d'instance |
| `nginx_lb_service` | `nginx` → `nginx-lb01` | `defaults` | **Jamais `nginx` en local** (port 80 étranger) | — | démarrage/handler |
| `nginx_lb_listen_port` | `80` → `9080` | `defaults` → `group_vars/loadbalancers.yml` → inventaire local | Port d'entrée | inventaire | `listen` du `server` |
| `nginx_lb_bind_address` | `""` (toutes interfaces) | `defaults` | Adresse d'écoute explicite | `127.0.0.1` | préfixe du `listen` |
| `nginx_lb_server_name` | `_` | `group_vars/loadbalancers.yml` | Catch-all | — | `server_name` |
| `nginx_lb_upstream_name` | `laravel_web` | `defaults` | Nom de l'upstream | — | bloc `upstream` + `proxy_pass` + nom du fichier site |
| `nginx_lb_backend_servers` | `groups['webservers']` | `group_vars/loadbalancers.yml` | Liste dynamique des backends | filtrer la liste | lignes `server` |
| `nginx_lb_backend_port` | `8080` (repli) | `group_vars/loadbalancers.yml` | Repli si `hostvars[...].apache_port` absent | — | `default(...)` du template |
| `nginx_lb_backend_params` | `max_fails=3 fail_timeout=30s` | `defaults` | Éjection temporaire d'un backend en échec | — | options des lignes `server` |
| `nginx_lb_lb_method` | `least_conn` | `group_vars/loadbalancers.yml` | `round_robin` \| `least_conn` \| `ip_hash` | changer la valeur | directive de l'upstream |
| `nginx_lb_health_check_path` | `/up` | `group_vars/loadbalancers.yml` | Endpoint de santé du LB | — | `location` renvoyant `200 OK` |
| `nginx_lb_proxy_connect_timeout` / `_read_timeout` | `5s` / `60s` | `defaults` | Timeouts proxy | — | `proxy_*` |
| `nginx_lb_conf_dir` / `sites_*` / `default_site` / `site_file` | calculés (`/etc/nginx` → `/etc/nginx-lb01`) | `defaults` | Cloisonnement | — | dépôts + symlink |
| `nginx_lb_user` / `nginx_lb_group` | `www-data` → `lb01` | `defaults` → inventaire local | User dédié | — | `User=` de l'unité |
| `nginx_lb_run_dir` / `log_dir` / `pid_file` | calculés | `defaults` | Isolation runtime/logs | — | config + unité |
| `nginx_lb_worker_connections` | `768` | `defaults` | Performance | — | `events {}` |
| `nginx_lb_package` / `nginx_lb_systemd_dir` | `nginx` / `/etc/systemd/system` | `defaults` | Paquet + destination unité | — | — |

## 7. Templates

| Template | Destination | Contenu clé |
|---|---|---|
| `nginx_lb.conf.j2` | `sites-available/{{ nginx_lb_upstream_name }}.conf` | **upstream** : `least_conn`/`ip_hash` conditionnel + `server {{ hôte }}:{{ hostvars[h].apache_port \| default(nginx_lb_backend_port) }} max_fails=3 fail_timeout=30s` pour chaque backend ; **server** : `listen` (avec `bind_address` optionnel), `server_name`, `location {{ health_check_path }}` → `200 "OK\n"`, `location /` → `proxy_pass http://upstream` + en-têtes `X-Real-IP`/`X-Forwarded-For`/`X-Forwarded-Proto` + timeouts |
| `nginx.conf.j2` | `[instance] /etc/nginx-lb01/nginx.conf` | `worker_processes auto`, `pid`, `error_log`, `events`, `http` avec `include /etc/nginx/mime.types` (table MIME du paquet), `include {{ sites_enabled }}/*.conf` — **pas de directive `user`** (le master tourne déjà sous `lb01`) |
| `nginx.service.j2` | `/etc/systemd/system/nginx-lb01.service` | `User=lb01`, `RuntimeDirectory=nginx-lb01`, `ExecStart=/usr/sbin/nginx -c <conf>/nginx.conf -g "daemon off;"`, `ExecReload=-HUP`, hardening |

## 8. Handlers

**`Recharger Nginx`** : `systemd` + `daemon_reload: true`,
**`state: reloaded`** (rechargement à chaud, pas de coupure),
tolérant `--check`. Déclenché par : suppression du site par défaut (AWS),
tous les dépôts de configuration (les 2 modes) et l'unité (instance).

## 9. Parcours des tâches

**AWS** : `apt nginx` → suppression de `sites-enabled/default`
(occupé par `nginx.service` :80) → template du site → symlink `sites-enabled`
(`force: true`) → démarrage `nginx`.

**Instance** : `apt` → groupe + user `lb01` (`nologin`) →
`/etc/nginx-lb01{,/sites-available,/sites-enabled}` → `/var/log/nginx-lb01` →
`nginx.conf` → template du site → symlink → unité `nginx-lb01` → démarrage
(`daemon_reload`).

**Vérifications** : `nginx -t` (AWS) / `nginx -t -c <conf instance>`,
`nginx -v`, `service_facts`, récapitulatif « backends … (ports lus depuis
hostvars[].apache_port, repli …) » — toutes en `changed_when: false` /
`failed_when: false`.

## 10. Sécurité et secrets

- Aucun secret manipulé.
- Utilisateur `lb01` (`nologin`) en mode instance, config `0644` root,
  logs/dossier appartenant à `lb01`.
- Unité systemd durcie ; proxy ne transmet que les en-têtes HTTP standards.

## 11. Vérifications intégrées (non bloquantes)

- `nginx -t` (syntaxe) + `nginx -v` (version).
- `service_facts` → état du service + récapitulatif des backends.
- Health-check applicatif **intégré au template** : `GET /up` → `200 OK`
  (vérifiable en `curl`).

## 12. Idempotence

- `apt`, `template`, `file: state: link/absent`, `state: started`.
- Rechargements **uniquement** par `notify` (`state: reloaded` : zéro coupure).
- Preuve : le rôle rejoué sans changement de config reste à `changed=0`.

## 13. Tests effectués (preuves)

- `nginx-lb01` : `active`, écoute **9080** (vue `ss`).
- `curl http://127.0.0.1:9080/up` → `200 OK`.
- `curl -o /dev/null -w '%{http_code}' http://127.0.0.1:9080/` → `200`
  (requête répartie vers `web01:9081` / `web02:9082`).
- Configuration : `/etc/nginx-lb01/nginx.conf` + site `laravel_web.conf`.
- Le `nginx` système (port 80) n'est **jamais** rechargé/redémarré par ce rôle.

## 14. Recettes de modification

| Je veux… | Fichier |
|---|---|
| Changer le port d'entrée local | `inventories/local/hosts.yml` → `lb01: nginx_lb_listen_port` |
| Changer la politique de répartition | `group_vars/loadbalancers.yml` → `nginx_lb_lb_method` (`round_robin`/`least_conn`/`ip_hash`) |
| Ajouter `web03` au LB | rien côté LB : ajouter `web03` au groupe `webservers` (l'upstream est généré depuis `groups['webservers']`) |
| Renommer le health-check | `group_vars/loadbalancers.yml` → `nginx_lb_health_check_path` |
| Sécuriser par IP | `nginx_lb_bind_address: 127.0.0.1` |

## 15. Intégration avec les autres tiers

- **Amont** : clients → `9080` (local) / `80` (AWS).
- **Aval** : `webservers` — le port backend est lu **par hôte**
  (`hostvars[web0X].apache_port`) : 8080 en AWS, 9081/9082 en local.
- **Ordre** : déployé **en dernier** (`site.yml` et `local_lb.yml`) — il ne
  doit pointer que vers des backends existants.

## 16. Points d'attention (pièges)

1. **Résolution des noms de backends** : `server web01:9081` exige que
   `web01` soit résoluble — le dépôt ne crée pas l'entrée `/etc/hosts`.
2. **Ne jamais supprimer le site par défaut en mode local** : la tâche est
   conditionnée (`when: not nginx_lb_instance_enabled`) car
   `/etc/nginx/sites-enabled/default` appartient au `nginx` système du port 80.
3. **`state: reloaded`** (pas `restarted`) : pas de coupure du trafic.
4. `nginx_lb_backend_port: 8080` est un **repli** : si un hôte web n'a pas
   `apache_port`, il serait envoyé sur 8080 (port WAF étranger en local) —
   d'où l'importance que chaque web porte bien son `apache_port`.
5. Pas de directive `user` dans `nginx.conf.j2` en mode instance : elle
   provoquerait un avertissement (master non-root).

## 17. Non trouvé dans le code actuel

- **Pas de health-check actif des backends** : `max_fails=3 fail_timeout=30s`
  est la seule mécanique (éjection passive). Aucun `check interval…` (onglet
  `upstream` commercial/plus récent) ni test HTTP périodique configuré.
  `nginx_lb_health_check_path: /up` est le health-check **du LB lui-même**.
- **Pas de TLS côté LB** : aucun bloc `listen 443 ssl` ni gestion de certificats.
- **Pas de gestion de `/etc/hosts`** (résolution des backends) — prérequis manuel.
- Pas de rate-limiting, WAF ni cache HTTP dans ce rôle.

## 18. Mode AWS (défaut)

- `nginx_lb_instance: ""` ⇒ `/etc/nginx`, service **`nginx`**, port **80**,
  user `www-data`.
- Suppression de `sites-enabled/default` (partagé avec le `nginx` système —
  en AWS, ce rôle **est** le Nginx de la machine).
- Backends : `web01:8080`, `web02:8080` (via `group_vars/webservers.yml`).
- Rechargement à chaud (`state: reloaded`).

## 19. Mode local cloisonné

- `nginx_lb_instance: lb01` ⇒ `/etc/nginx-lb01`, service **`nginx-lb01`**,
  port **9080**, user **`lb01`**, logs `/var/log/nginx-lb01`,
  pid `/run/nginx-lb01/nginx.pid`.
- `nginx.conf` dédié (include des sites de l'instance, table MIME du paquet).
- Backends : `web01:9081`, `web02:9082` (lus dans `hostvars`).
- Commande : `ansible-playbook -i inventories/local/hosts.yml playbooks/local_lb.yml`

## 20. Problèmes rencontrés (réels)

| Symptôme | Cause | Correctif (dans le code) |
|---|---|---|
| LB impossible sur le port 80 en local | `nginx.service` étranger occupe déjà :80 | port dédié **9080** + service `nginx-lb01` + conf `/etc/nginx-lb01` |
| Tâche de suppression du site par défaut inapplicable en local | `/etc/nginx/sites-enabled/default` appartient au nginx système | `when: not nginx_lb_instance_enabled` sur la tâche |
| Backends envoyés vers le mauvais port (8080 = WAF étranger) | port backend codé en dur | port lu **par hôte** : `hostvars[server].apache_port` avec repli `nginx_lb_backend_port` |
| `--check` échoue sur le symlink du site | la source n'existe pas encore en simulation | `force: true` sur `file: state: link` |
| Avertissement « the 'user' directive makes sense only if … » | directive `user` avec master non-root | absente de `nginx.conf.j2` (commentée) |

## 21. Références

- [`../roles/nginx_lb/README.md`](../roles/nginx_lb/README.md)
- [`../docs/installation-manuelle.md`](../docs/installation-manuelle.md) — §Tier 1.
- [`../README.md`](../README.md) — §7 (ordre des playbooks), §12.
- [01 — Architecture](01-architecture.md) (chemin de la requête),
  [14 — Tests & validation](14-tests-et-validation.md).

## 22. Résumé

Point d'entrée unique, déployé en dernier. Génère l'upstream à partir de
l'inventaire (port backend par hôtes) : ajouter un web = l'ajouter au groupe.
Double mode : AWS = `nginx` sur 80 ; local = `nginx-lb01` sur 9080 à côté du
nginx système (jamais touché). Rechargement à chaud, health-check `/up` intégré,
éjection passive `max_fails`. **Points ouverts** : pas de health-check actif
des backends, pas de TLS au LB, résolution DNS des noms hors périmètre.


