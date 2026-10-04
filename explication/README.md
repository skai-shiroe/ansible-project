# `explication/` — Documentation pédagogique du projet

Cette documentation **explique** l'architecture, le code et les décisions du
dépôt `ansible-project`. Elle est **strictement documentaire** : aucun rôle,
task, template, inventaire, variable ou secret n'est modifié par ces fichiers.

> **Règle d'or** : tout ce qui est affirmé ici est vérifiable dans le code du
> dépôt. Ce qui n'existe pas est explicitement marqué
> **« Non trouvé dans le code actuel »** — jamais comblé par invention.

**Lectures complémentaires (à ne pas dupliquer)**
- [`../README.md`](../README.md) — vue projet, validation, TODO, conventions.
- [`../docs/installation-manuelle.md`](../docs/installation-manuelle.md) — le manuel avant l'automatisation.
- [`../roles/<rôle>/README.md`](../roles/) — référence de chaque rôle.

---

## Sommaire

| # | Document | Contenu | Quand le lire |
|---|---|---|---|
| 00 | [Vue d'ensemble](00-vue-ensemble.md) | Objectif, ports du projet, services étrangers à ne jamais toucher | Première lecture |
| 01 | [Architecture](01-architecture.md) | Tiers, composants, schémas ASCII des requêtes | Première lecture |
| 02 | [Variables & précédence](02-ansible-variables-et-precedence.md) | Placement, ordre de précédence, pivot `defaults`/`vars` | Avant de toucher à une variable |
| 03 | [Inventaires & environnements](03-inventaire-et-environnements.md) | AWS vs local, groupes, liens symboliques | Avant de lancer un playbook |
| 04 | [Apache](04-apache.md) | Rôle `apache` (22 sections) | Tier web |
| 05 | [PHP](05-php.md) | Rôle `php` (22 sections) | Tier web |
| 06 | [Laravel](06-laravel.md) | Rôle `laravel` (22 sections) | Tier web |
| 07 | [PostgreSQL](07-postgresql.md) | Rôle `postgresql` (22 sections) | Tier données |
| 08 | [Redis](08-redis.md) | Rôle `redis` (22 sections) | Tier cache |
| 09 | [Load balancer](09-load-balancer.md) | Rôle `nginx_lb` (22 sections) | Entrée de trafic |
| 10 | [Sécurité & secrets](10-securite-et-secrets.md) | Vault, `no_log`, permissions, utilisateurs | Avant tout déploiement réel |
| 11 | [Réplication PostgreSQL](11-replication-postgresql.md) | Étapes pédagogiques, schémas, limites | Étape 5 |
| 12 | [Communication entre tiers](12-communication-entre-tiers.md) | 4 schémas source/destination/port/auth | Débogage réseau |
| 13 | [Idempotence & handlers](13-idempotence-et-handlers.md) | `changed=0`, `notify`, `flush_handlers` | Écriture de tâches |
| 14 | [Tests & validation](14-tests-et-validation.md) | Checklist par tier, preuves obtenues | Après chaque exécution |
| 15 | [Dépannage](15-depannage.md) | Problèmes réellement rencontrés → cause → correctif | Symptôme en cours |
| 16 | [Commandes utiles](16-commandes-utiles.md) | Aide-mémoire Ansible / services / bases | Quotidien |
| 17 | [Java](17-java.md) | Rôle `java` — état réel (code / prévu / non implémenté) | Backoffice (non prêt) |
| 18 | [Spring Boot](18-springboot.md) | Rôle `springboot` — état réel (code / prévu / non implémenté) | Backoffice (non prêt) |

---

## Structure imposée pour une documentation de rôle (sections 1 → 22)

Chaque document `04` → `09`, `17`, `18` suit les mêmes 22 sections :

1. À quoi sert ce rôle
2. Où il vit dans le dépôt
3. Comment il est appelé
4. Prérequis et dépendances
5. Ports, services et chemins
6. Variables *(tableau : Variable · Valeur actuelle · Fichier · Pourquoi · Exemple de modification · Impact)*
7. Templates
8. Handlers
9. Parcours des tâches
10. Sécurité et secrets
11. Vérifications intégrées (non bloquantes)
12. Idempotence
13. Tests effectués (preuves)
14. Recettes de modification
15. Intégration avec les autres tiers
16. Points d'attention (pièges)
17. Non trouvé dans le code actuel
18. **Mode AWS (défaut)**
19. **Mode local cloisonné**
20. **Problèmes rencontrés (réels)**
21. Références
22. Résumé

## État de la documentation

| Point | État |
|---|---|
| Rôles documentés | **8 / 8** : `apache`, `php`, `laravel`, `postgresql`, `redis`, `nginx_lb`, `java`, `springboot` |
| Playbooks documentés | 8 / 8 (`site`, `applications`, `database`, `cache`, `local_lb`, `local_cache`, `local_web`, `local_db`) |
| Inventaires documentés | `aws_ec2.yml`, `hosts.yml`, `local/` (+ `gcp_compute.yml`, fichier vide) |
| Secrets | Jamais de valeur : uniquement `<SECRET>` et les **noms** `vault_*` |
| Fichiers du dépôt modifiés par cette documentation | **Aucun** (`explication/` est un ajout pur) |
| Points non trouvés dans le code | Récapitulés ci-dessous et dans chaque §17 / §20 |

### Points « Non trouvé dans le code actuel » (récapitulatif)

1. **`php_conf_d_dir` / `php_conf_file`** — référencés dans
   `roles/php/tasks/main.yml` (tâches du **mode AWS** uniquement) mais absents
   de `defaults/`, `vars/`, `group_vars/` et de tous les inventaires. En mode
   local ces tâches sont sautées (`when: not php_instance_enabled`) ; en mode
   AWS elles échoueraient sur une variable indéfinie.
2. **`requirements.yml`** ne déclare que `amazon.aws` et `community.aws`,
   alors que les rôles utilisent aussi `community.general` (`apache2_module`,
   `composer`) et `community.postgresql` (`postgresql_user`, `postgresql_db`,
   `postgresql_query`). Ces collections sont présentes dans
   `.ansible/collections/`, mais pas déclarées.
3. **`group_vars/customer.yml`** et **`group_vars/postgresql.yml`** — fichiers vides (0 octet).
4. **`inventories/gcp_compute.yml`** — fichier vide (0 octet) : l'inventaire GCP est prévu
   (commentaire dans `ansible.cfg`) mais non implémenté.
5. **`laravel_storage_dir`** — défini dans `roles/laravel/vars/main.yml` mais jamais référencé ailleurs.
6. **Failover PostgreSQL automatisé** — non implémenté (promotion manuelle ;
   `README.md` §9 : outil dédié « à valider avec le formateur »).
7. **Tier Backoffice en local** — aucun playbook `local_backoffice.yml`, aucun mode
   `*_instance` pour `java`/`springboot`, `bo01` sans variables dans l'inventaire
   local → voir [17-java.md](17-java.md) et [18-springboot.md](18-springboot.md).
8. **`springboot_jar_file`** vide → la tâche de copie du JAR est sautée par conception.
9. Le `README.md` §14 mentionne **7** tâches `no_log` ; un `grep` sur les
   `tasks/` en compte **9** (ajouts ultérieurs : template Redis mode instance,
   test `redis-cli` authentifié).

*(Détail dans chaque document, sections 17 et 20.)*

