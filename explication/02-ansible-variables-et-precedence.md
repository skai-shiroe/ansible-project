# 02 — Variables Ansible et précédence

## Pourquoi ce document ?

C'est la **règle structurante du projet** : chaque variable est placée selon sa
portée. Un déplacement change le comportement — cet ordre est celui réellement
appliqué, avec des **exemples tirés du code**.

---

## 1. Les emplacements (du plus faible au plus fort)

| # | Emplacement | Nature | Exemple réel du dépôt |
|---|---|---|---|
| 1 | `roles/<r>/defaults/main.yml` | Paramètre **surchargeable** (contrat d'entrée) | `apache_port: 80` |
| 2 | `group_vars/<groupe>.yml` (racine, via symlink `playbooks/group_vars`) | Valeur **d'un tier** | `apache_port: 8080` |
| 3 | `host_vars/<hôte>.yml` (racine, via symlink `playbooks/host_vars`) | Valeur **d'une machine** | `postgresql_node_role: primary` |
| 4 | `roles/<r>/vars/main.yml` | **Constante interne** (à ne pas surcharger) | `laravel_env_file: "{{ laravel_path }}/.env"` |
| 5 | `inventories/local/hosts.yml` (variables inline par hôte) | Topologie **locale** (écrase group/host_vars) | `apache_port: 9081` |
| 6 | `include_vars` dans `pre_tasks` (`local_db.yml`) | **Précédence maximale** — écrase aussi `playbooks/host_vars` | `postgresql_instance: db01` |
| 7 | `-e` (extra vars) | Absolu | `-e laravel_queue_worker_enabled=true` |

> **Rappel** : `vars/` (rang 4) **battait** `group_vars`/`host_vars` (rangs 2-3).
> C'est précisément pour cela que `vars/main.yml` des 6 rôles cloisonnés a été
> **vidé** (il ne reste que `---`, sauf 2 constantes laravel non
> conflictuelles) et que les chemins ont été transférés dans `defaults/` :
> rendre le mode AWS surchargeable sans rouvrir le risque.

### État de `vars/main.yml` (vérifié)

| Rôle | Lignes actives | Commentaire |
|---|---|---|
| `apache`, `php`, `postgresql`, `redis`, `nginx_lb` | 0 (seul `---`) | tout est en `defaults/` |
| `laravel` | 2 : `laravel_env_file`, `laravel_storage_dir` | chemins calculés ; `laravel_storage_dir` **n'est référencé nulle part ailleurs** |
| `java` | 10 : `java_jvm_dir`, `java_bin`, `java_apt_lock_timeout`, `java_apt_keyring_dir`, `java_apt_sources_dir`, `java_apt_prerequisites` | constantes internes, jamais surchargées |
| `springboot` | 4 : `springboot_systemd_dir`, `springboot_service_file`, `springboot_env_file` | idem |

---

## 2. Exemples réels de précédence (le « pivot »)

### Exemple A — `apache_port`

```
defaults/main.yml            apache_port: 80      ← contrat du rôle (AWS minimal)
        ↓ écrasé par
group_vars/webservers.yml    apache_port: 8080    ← cible AWS (derrière le LB)
        ↓ écrasé par
inventories/local/hosts.yml  apache_port: 9081    ← inline par hôte (web01 ; 9082 sur web02)
```

Local ⇒ `Listen 9081`. AWS ⇒ `Listen 8080`. Le rôle n'a **pas changé**.

### Exemple B — `postgresql_instance`

```
defaults/main.yml      postgresql_instance: ""      ← AWS : cluster « main »
        ↓ écrasé par
group_vars/databases.yml  postgresql_version: "17", postgresql_port: 5432   ← profil AWS
        ↓ écrasé par (chargé automatiquement à côté de l'inventaire)
inventories/local/host_vars/db01.yml  postgresql_instance: db01,
                                      postgresql_version: "18", postgresql_port: 5442
        ↓ RE-joué en préséance maximale par
playbooks/local_db.yml  pre_tasks: include_vars:
                          inventories/local/host_vars/{{ inventory_hostname }}.yml
```

**Pourquoi le `include_vars` est indispensable** : un playbook lancé depuis
`playbooks/` voit **aussi** `playbooks/host_vars/db01.yml` (symlink →
`host_vars/db01.yml` : topologie AWS). `include_vars` passe **au-dessus** de ce
symlink — sans lui, la réplique locale chercherait l'hôte `db01` par DNS et
viserait le port 5432.

### Exemple C — secrets

```
group_vars/databases.yml   postgresql_password: "{{ vault_postgresql_password | default('') }}"
        ↓ résolu par
inventories/local/group_vars/all/vault.yml   (chiffré, 5 clés)
        ↑ déchiffré via --vault-password-file .vault_pass
```

Le `| default('')` garantit qu'un vault **vide** ne fait jamais échouer le rôle.

## 3. Table des variables principales (valeurs actuelles)

| Variable | Valeur AWS | Valeur local | Source la plus haute | Rôle |
|---|---|---|---|---|
| `apache_port` | `8080` | `9081` / `9082` | `group_vars/webservers.yml` → `inventories/local/hosts.yml` | `apache` |
| `apache_document_root` | `/var/www/laravel/public` | `/var/www/web0X/public` | idem | `apache` |
| `apache_instance` | `""` | `web01` / `web02` | `roles/apache/defaults` → inventaire local | `apache` |
| `php_version` | `"8.5"` | `"8.5"` | `group_vars/webservers.yml` | `php` |
| `php_instance` | `""` | `web01` / `web02` | `roles/php/defaults` → inventaire local | `php` |
| `laravel_path` | `/var/www/laravel` | `/var/www/web0X` | `group_vars/webservers.yml` → inventaire local | `laravel` |
| `laravel_db_host` | `groups['databases'][0]` → `db01` | `127.0.0.1` | `roles/laravel/defaults` → inventaire local | `laravel` |
| `laravel_db_port` | `5432` | `5442` | idem | `laravel` |
| `laravel_run_migrations` | `false` | `true` | `roles/laravel/defaults` → inventaire local | `laravel` |
| `laravel_queue_worker_enabled` | `false` | `false` | `roles/laravel/defaults` (opt-in partout) | `laravel` |
| `nginx_lb_listen_port` | `80` | `9080` | `group_vars/loadbalancers.yml` → inventaire local | `nginx_lb` |
| `nginx_lb_backend_servers` | `groups['webservers']` | idem | `group_vars/loadbalancers.yml` | `nginx_lb` |
| `nginx_lb_lb_method` | `least_conn` | `least_conn` | `group_vars/loadbalancers.yml` | `nginx_lb` |
| `postgresql_version` | `"17"` | `"18"` | `group_vars/databases.yml` → `inventories/local/host_vars/db0X.yml` | `postgresql` |
| `postgresql_port` | `5432` | `5442` / `5443` | idem | `postgresql` |
| `postgresql_instance` | `""` | `db01` / `db02` | `roles/postgresql/defaults` → `inventories/local/host_vars/` | `postgresql` |
| `postgresql_node_role` | `primary` / `replica` | idem | `host_vars/db0X.yml` (racine) | `postgresql` |
| `redis_port` | `6379` | `6390` | `group_vars/redis.yml` → inventaire local | `redis` |
| `redis_instance` | `""` | `redis01` | `roles/redis/defaults` → inventaire local | `redis` |
| `springboot_port` | `8080` | *(non utilisé en local)* | `group_vars/backoffice.yml` | `springboot` |

Les valeurs AWS viennent des `group_vars/` de la racine (accessibles via les
liens symboliques `playbooks/group_vars` et `playbooks/host_vars` — voir
[03 — Inventaires](03-inventaire-et-environnements.md)).

---

## 4. Où modifier une variable — recette rapide

| Je veux changer… | Je modifie… | Je ne touche JAMAIS à… |
|---|---|---|
| Le port d'écoute AWS d'Apache | `group_vars/webservers.yml` | `roles/apache/defaults` (contrat du rôle) |
| Le port de l'instance locale web01 | `inventories/local/hosts.yml` (inline) | `group_vars/` (écrasé de toute façon) |
| Le dimensionnement PG AWS | `group_vars/databases.yml` | `roles/postgresql/defaults` |
| Le primaire vu par la réplique locale | `inventories/local/host_vars/db02.yml` (+ garder le `include_vars` de `local_db.yml`) | `host_vars/db02.yml` racine (AWS) |
| Un secret | le vault du bon environnement | jamais de clair dans un `.yml` |
| Un réglage « pour tout le monde » | `group_vars/all/vars.yml` | — |

---

## 5. Pièges de précédence rencontrés dans ce projet

1. **`vars/` écrase l'inventaire** — d'où le vidage des `vars/main.yml`
   (l'ordre de migration : chemins → `defaults/`, contrôle → inventaire).
2. **`playbooks/host_vars` est un symlink actif** pour les playbooks `local_*` :
   d'où le `include_vars` en `pre_tasks` de `local_db.yml`.
3. **Variables inline > group_vars** : les réglages cloisonnés du tier web sont
   écrits **dans** `inventories/local/hosts.yml` (et non dans un `group_vars`
   local) pour écraser sans ambiguïté `group_vars/webservers.yml`.
4. **`hostvars[...].apache_port` dans le template LB** : le upstream ne connaît
   que cette variable — changer `apache_port` côté web **et** ne pas oublier
   que le repli `nginx_lb_backend_port: 8080` ne s'applique qu'en son absence.

→ Suite : [03 — Inventaires & environnements](03-inventaire-et-environnements.md)

