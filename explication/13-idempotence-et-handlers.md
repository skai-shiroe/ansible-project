# 13 — Idempotence et handlers

## 1. Ce qu'est l'idempotence

Une tâche est **idempotente** si son exécution **répétée** produit le même
état final que son exécution unique : le 2e passage signale `changed=0` et ne
modifie rien. C'est le critère de qualité central du projet :

> Chaque playbook local a été rejoué : **`changed=0`** sur tous les hôtes
> (`README.md` §12 : `local_db`, `local_cache`, `local_web`).

```bash
# Le test d'idempotence
ansible-playbook -i inventories/local/hosts.yml playbooks/local_cache.yml \
  --vault-password-file .vault_pass | grep -E 'ok=|changed='
# → PLAY RECAP : … changed=0 …
```

---

## 2. Les mécanismes utilisés dans le code

### 2.1 Modules déclaratifs (état voulu, pas d'action)

| Tâche | Module | Idempotence |
|---|---|---|
| Installer un paquet | `apt: state: present` | déjà installé → `ok` |
| Créer répertoire/utilisateur/groupe | `file: state: directory/present` | existe → `ok` |
| Lier un site | `file: state: link` | bon lien → `ok` |
| Déployer une config | `template` | hash identique → `ok` |
| Démarrer un service | `service`/`systemd: state: started` | déjà démarré → `ok` |
| Bascule AWS/instance | `when:` | condition fausse → `skipped` |

### 2.2 `changed_when` — dire la vérité sur une commande

```yaml
# laravel : « Nothing to migrate » = rien à faire → ok, pas changed
changed_when: "'Nothing to migrate' not in laravel_migrate.stdout"

# vérifications : jamais de changement
changed_when: false        # php -v, nginx -t, pg_lsclusters, redis-cli ping…

# commande non détectable : marquée honnêtement
changed_when: true         # pg_createcluster (création d'un cluster)
```

### 2.3 `failed_when: false` — diagnostiquer sans bloquer

Les **vérifications non bloquantes** des 8 rôles (`configtest`, `php -v`,
`artisan db:show`, `pg_stat_replication`, pings Redis…) diagnostiquent sans
interrompre le déploiement. C'est volontaire : le tier web part parfois avant
le tier données.

### 2.4 Tolérance `--check` (simulation)

```yaml
failed_when: <reg> is failed and not ansible_check_mode
```

En `--check`, les services « viennent d'être installés en simulation » et
peuvent ne pas exister : tolérance **uniquement** en simulation. En mode réel,
tout échec reste **bloquant** (artefact documenté `README.md` §8).

### 2.5 `notify` + handlers — ne redémarrer que si nécessaire

```yaml
- name: Déployer la configuration
  ansible.builtin.template: …
  notify: Redémarrer Apache      # ← déclenché SEULEMENT si la tâche est changed
```

| Rôle | Handler | Action |
|---|---|---|
| `apache` | Redémarrer Apache | `state: restarted` + `daemon_reload` |
| `php` | Redémarrer PHP-FPM | idem |
| `nginx_lb` | **Recharger** Nginx | `state: reloaded` (à chaud, zéro coupure) |
| `postgresql` | Redémarrer PostgreSQL | `restarted` + `daemon_reload` |
| `redis` | Redémarrer Redis | idem |
| `laravel` | Redémarrer le worker de file d'attente | idem (opt-in) |
| `springboot` | Redémarrer Spring Boot | idem |
| `java` | Mettre à jour le cache APT | `apt: update_cache` |

`daemon_reload: true` est présent partout : les unités (`apache-web0X`,
`nginx-lb01`, `php-fpm-web0X`, `redis-redis01`, `laravel-queue-*`) sont
**nouvelles** — systemd doit les recharger avant le restart.

### 2.6 `meta: flush_handlers` — forcer l'exécution immédiate

Un handler tournait en **fin de play**. Or la réplication démarre **pendant**
le play : le primaire doit avoir appliqué `wal_level`, le port et `pg_hba`
**avant** que la réplique ne se connecte.

```yaml
# roles/postgresql/tasks/main.yml
- name: Appliquer la configuration PostgreSQL avant la réplication
  ansible.builtin.meta: flush_handlers
```

### 2.7 `run_once` — une opération pour tout le tier

```yaml
# roles/laravel : les 2 instances web partagent LA base
- name: Jouer les migrations de base de données
  ansible.builtin.command: php artisan migrate --force
  run_once: true            # ← seulement sur le premier hôte du groupe
```

Sans `run_once`, deux `migrate` simultanés se disputent les mêmes tables
(« relation "users" already exists »).

---

## 3. Les gardes d'idempotence par rôle

| Rôle | Garde | Effet |
|---|---|---|
| `postgresql` | `stat standby.signal` | la réplique n'est **jamais** réinitialisée |
| `postgresql` | `stat` du répertoire de config | `pg_createcluster` uniquement au 1er run |
| `postgresql` | `WHERE NOT EXISTS` (slot) | slot créé une fois |
| `postgresql` | look-ahead `password=` | `primary_conninfo` modifié 1 fois |
| `laravel` | `stat artisan` / `.git` | `laravel_clean_deploy` ne détruit jamais une app |
| `laravel` | `laravel_app_key \| length == 0` | `key:generate` jamais rejoué (vault) |
| `laravel` | `laravel_run_migrations` (défaut `false`) | AWS : schéma intouché |
| `laravel` | `laravel_repo_configured` | pas de clonage sur un dépôt fictif |
| `laravel` | `run_once` + `changed_when` | migrations : 1 seule fois, `changed=0` après |
| `apache`/`nginx_lb` | `force: true` sur le symlink | pas d'échec en `--check` |
| tous | `when:` sur `*_instance_enabled` | un mode n'exécute jamais les tâches de l'autre |

## 4. Vérifications non bloquantes — pourquoi `changed_when: false`

Chaque rôle finit par des tâches de **diagnostic** (`configtest`, versions,
`service_facts`, `pg_stat_replication`, pings Redis, `artisan db:show`) :

- `changed_when: false` → elles ne **truffent jamais** le récapitulatif ;
- `failed_when: false` → elles ne **cassent jamais** un déploiement (l'état
  peut être légitimement incomplet au 1er passage).

Résultat : un run stable affiche `changed=0` **et** reste lisible.

---

## 5. Références

- [`../README.md`](../README.md) — §12 (preuves `changed=0`), §8 (artefacts `--check`).
- Les `handlers/main.yml` des 8 rôles ; les commentaires `# Tolérant en --check`.
- [14 — Tests & validation](14-tests-et-validation.md) (checklist complète).

## 6. Résumé

L'idempotence repose sur 6 mécanismes du code : modules déclaratifs,
`changed_when` honnêtes, `failed_when: false` pour les diagnostics, tolérance
`--check` stricte, `notify`/handlers (restart **uniquement** si changement,
`daemon_reload` pour les unités neuves, `reloaded` pour Nginx) et
`flush_handlers` quand l'ordre compte. Les gardes par rôle
(`standby.signal`, `run_once`, `stat`…) protègent les opérations
irréversibles. Preuve : `changed=0` sur les playbooks locaux rejoués.


