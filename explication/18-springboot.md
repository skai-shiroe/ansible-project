# 18 — Rôle `springboot` (Backoffice — **non prêt en local**)

> ⚠️ **État du Backoffice** : le rôle `springboot` **existe dans le code**, mais
> le tier Backoffice **n'est pas encore réalisé** : aucun déploiement local, ni
> exécution réelle, ni validation. Même tripartition que
> [17-java.md](17-java.md) : **[CODE]** = présent et vérifiable · **[PRÉVU]**
> = prévu par l'architecture · **[NON IMPLÉMENTÉ]** = absent ou jamais exécuté.
> Ce document sera complété avec l'implémentation réelle (variables retenues,
> services, tests, problèmes) lors de l'étape Backoffice.

---

## 1. À quoi sert ce rôle ?

**[CODE]** Déploie l'application **Spring Boot** (Backoffice) : utilisateur
système dédié, répertoire `/opt/<app>`, copie d'un **JAR statique** (règle
`files/`), fichier d'environnement (datasource → secret), unité systemd,
vérifications (présence du JAR, `systemd-analyze verify`, journal).

## 2. Où il vit dans le dépôt

**[CODE]**
```
roles/springboot/
├── defaults/main.yml      # app_name, port, user, home, jar, env, datasource…
├── vars/main.yml          # 4 constantes (systemd_dir, service_file, env_file)
├── tasks/main.yml         # 114 lignes
├── handlers/main.yml      # « Redémarrer Spring Boot »
├── files/README.md        # règle des fichiers statiques (le JAR s'y dépose)
├── templates/
│   ├── springboot.env.j2      # environnement (JAVA_OPTS, SPRING_*)
│   └── springboot.service.j2  # unité systemd
└── README.md
```

## 3. Comment il est appelé

- **[CODE]** `playbooks/applications.yml`, 2e play, juste après `java` :
  `hosts: backoffice`, `roles: [java, springboot]`.
- **[NON IMPLÉMENTÉ]** Aucun playbook local ne cible `bo01`.

## 4. Prérequis et dépendances

- **[CODE]** `java` exécuté **avant** (fournit `java_home` →
  `springboot_java_bin`), sinon repli sur `java` du `PATH`.
- **[CODE]** Un JAR déposé dans `roles/springboot/files/` **ou** la tâche de
  copie est sautée (`springboot_jar_file: ""` par défaut).
- **[CODE]** `meta/main.yml` : `dependencies: []`.
- **[PRÉVU]** Base PostgreSQL joignable (`springboot_datasource_url`).

## 5. Ports, services et chemins

- **[CODE]** Port : `springboot_port: 8080` (`--server.port=` dans l'unité).
  Service : `backoffice` (`springboot_service_name`).
- **[CODE]** Chemins : home `/opt/backoffice`, JAR `backoffice.jar`, env
  `/opt/backoffice/backoffice.env` (**0600**), unité
  `/etc/systemd/system/backoffice.service`, utilisateur `springboot` (`nologin`).
- **[PRÉVU]** Sur AWS : `bo01`, port 8080 (machine dédiée, pas de conflit).
- **[NON IMPLÉMENTÉ]** Choix d'un port local : le **8080 est occupé** par
  `apache2-waf-svc` sur la machine du lab — aucun port alternatif n'a été
  décidé (à trancher lors de l'implémentation ; pas de valeur inventée ici).

## 6. Variables

| Variable | Valeur actuelle | Fichier | Pourquoi | Exemple de modification | Impact |
|---|---|---|---|---|---|
| `springboot_app_name` | `backoffice` | `defaults` + `group_vars/backoffice.yml` | Nom de l'app | — | home, jar, service, env |
| `springboot_port` | `8080` | idem | Port HTTP (`--server.port`) | **à revoir pour le local** (8080 occupé) | `ExecStart` |
| `springboot_user` / `_group` | `springboot` | `defaults` | Compte dédié, jamais root | — | owner fichiers, `User=` |
| `springboot_home` | `/opt/backoffice` | `defaults` (calculé) | Déploiement | — | copie, `WorkingDirectory` |
| `springboot_jar_name` | `backoffice.jar` | `defaults` | Nom à la destination | — | `ExecStart` |
| `springboot_jar_file` | **`""`** | `defaults` | Vide = **ne pas copier** de binaire | `backoffice-1.0.0.jar` (fichier dans `files/`) | condition de la tâche `copy` |
| `springboot_service_name` | `backoffice` | `defaults` | Nom unité systemd | — | template + démarrage + handler |
| `springboot_java_bin` | `{{ java_home }}/bin/java` (ou `java`) | `defaults` | Binaire issu du rôle `java` | — | `ExecStart` |
| `springboot_java_opts` | `-Xms256m -Xmx512m` | `defaults` + `group_vars/backoffice.yml` | Heap JVM | — | `JAVA_OPTS` de l'env |
| `springboot_spring_profiles_active` | `prod` | idem | Profil Spring | — | env |
| `springboot_datasource_url` | `jdbc:postgresql://<databases[0]>:5432/laravel` | `defaults` + `group_vars/backoffice.yml` | Cible PG (AWS : db01:5432) | **à adapter en local (5442)** | env |
| `springboot_datasource_username` | `laravel` | idem | User applicatif partagé | — | env |
| `springboot_datasource_password` | `{{ vault_springboot_db_password \| default('') }}` | `defaults` + vault | Secret | vault | env (**`no_log`**) |
| `springboot_systemd_dir` / `_service_file` / `_env_file` | `/etc/systemd/system`, `backoffice.service`, `<home>/backoffice.env` | **`vars/main.yml`** | constantes internes | — | tâches |

> Le vault local ne contient **pas** `vault_springboot_db_password`
> (5 clés seulement) : tant que le backoffice n'est pas déployé en local, ce
> secret n'est pas provisionné — **non trouvé dans le code actuel côté local**.

## 7. Templates

**[CODE]**
- `springboot.env.j2` → `/opt/backoffice/backoffice.env` (`0600`) :
  `JAVA_OPTS`, `JAVA_HOME` (conditionnel), `SPRING_PROFILES_ACTIVE`,
  `SPRING_DATASOURCE_URL/USERNAME/PASSWORD` — tâche **`no_log: true`**.
- `springboot.service.j2` → `/etc/systemd/system/backoffice.service` :
  `Type=simple`, `User=springboot`, `EnvironmentFile`, `ExecStart=<java>
  $JAVA_OPTS -jar … --server.port={{ springboot_port }}`, `SuccessExitStatus=143`,
  `Restart=on-failure`.

> **[CODE]** Ce dernier template **ne porte pas** le bloc de durcissement
> systemd (`NoNewPrivileges`, `Protect*`) présent dans les autres unités du
> projet (voir [10-securite-et-secrets.md](10-securite-et-secrets.md) §6) :
> écart constaté dans le code, non corrigé à ce jour.

## 8. Handlers

**[CODE]** `Redémarrer Spring Boot` : `systemd` + `daemon_reload: true`,
`state: restarted`. Déclenché par : copie du JAR, `.env`, unité.

## 9. Parcours des tâches

**[CODE]** 1. groupe `springboot` → 2. utilisateur `springboot` (`nologin`,
`create_home: false`) → 3. `/opt/backoffice` → 4. `stat` du JAR → 5. copie du
JAR (`when: springboot_jar_file | length > 0`, `notify`) → 6. template
`.env` (`no_log`, `notify`) → 7. template de l'unité (`notify`) → 8.
démarrage/activation (`daemon_reload`).

**Vérifications** : affichage de l'état du JAR (déposé ou **absent** :
« springboot_jar_file non renseigné : déposez-le dans roles/springboot/files/ »)
→ `systemd-analyze verify` de l'unité (tolérant) → état du service →
5 dernières lignes du `journalctl` — toutes en `changed_when: false` /
`failed_when: false`.

## 10. Sécurité et secrets

- **[CODE]** `.env` `0600` owner `springboot` + **`no_log: true`** sur le
  template (contient `SPRING_DATASOURCE_PASSWORD`).
- **[CODE]** Utilisateur `springboot` `nologin`, jamais root.
- **[PRÉVU]** Secret : `vault_springboot_db_password` (dans le vault
  **AWS**, absent du vault local).
- **[CODE]** Unité **sans** bloc `Protect*`/`NoNewPrivileges` (écart — §7).

## 11. Vérifications intégrées (non bloquantes)

**[CODE]** `stat` du JAR + message d'état ; `systemd-analyze verify` ;
`systemd` (état du service, tolérant) ; `journalctl -u … -n 5` + récapitulatif.
Aucune vérification HTTP applicative (pas de health-endpoint testé).

## 12. Idempotence

**[CODE]** `group/user: state: present`, `file`, `copy` (checksum), `template`,
`state: started` ; redémarrages seulement par `notify` ; vérifications en
`changed_when: false`.

## 13. Tests effectués (preuves)

- **[CODE]** Les contrôles existent dans le rôle (état JAR, `systemd-analyze`,
  état service, journal).
- **[NON IMPLÉMENTÉ]** **Aucune exécution réelle** de ce rôle dans ce lab :
  le tier backoffice n'a pas été déployé localement. Aucun résultat
  « service actif / port 8080 / connexion DB » ne peut être affirmé —
  sera complété lors de l'implémentation.

## 14. Recettes de modification

| Je veux… | Fichier |
|---|---|
| Déployer le JAR | déposer `backoffice-x.y.z.jar` dans `roles/springboot/files/` puis `springboot_jar_file: backoffice-x.y.z.jar` (ex. dans `group_vars/backoffice.yml`) |
| Changer le port (AWS) | `group_vars/backoffice.yml` → `springboot_port` |
| Cibler une autre base | idem → `springboot_datasource_url` (+ vault) |
| Ajuster la JVM | idem → `springboot_java_opts` |

## 15. Intégration avec les autres tiers

- **[CODE]** Amont : `java` (`java_home`).
- **[PRÉVU]** Aval : PostgreSQL (datasource port 5432 en AWS ; **5442 requis en
  local** sur le cluster `18/db01` — pas encore câblé).
- **[NON IMPLÉMENTÉ]** Lien avec le reste du lab local : aucun.

## 16. Points d'attention (pièges)

1. **[CODE]** `springboot_jar_file: ""` par défaut : la copie est **sautée**
   (message explicite) — le service échouerait au démarrage sans JAR.
2. **[CODE]** Le datasource pointe sur `groups['databases'][0]:5432` : en
   local, cela viserait le cluster système **18-main** (interdit) — un câblage
   `127.0.0.1:5442` sera nécessaire à l'implémentation.
3. **[PRÉVU]** Port 8080 : occupé par `apache2-waf-svc` en local (§5).
4. **[CODE]** Unité sans durcissement systemd (§7).

## 17. Non trouvé dans le code actuel

- Aucun `springboot_instance` (pas de mode cloisonné) : **non trouvé dans le code actuel**.
- Aucun playbook `local_backoffice.yml` : **non trouvé dans le code actuel**.
- `vault_springboot_db_password` **absent** du vault local (présent dans le
  gabarit `vault.yml.example`) : **non trouvé dans le code actuel côté local**.
- Le JAR n'est **pas** dans `roles/springboot/files/` (seul un `README.md`
  y est présent) : déploiement binaire **non trouvé dans le code actuel**.
- Aucune health-check HTTP, aucun test de connexion DB dans le rôle.
- Aucune exécution réelle dans ce lab (§13).

## 18. Mode AWS (défaut)

- **[CODE]** C'est le seul mode prévu à ce jour : `applications.yml` →
  `hosts: backoffice` → `roles: [java, springboot]`.
- **[PRÉVU]** `bo01` (tag `Role=backoffice`), port **8080**, base `db01:5432`,
  profil `prod`, heap `-Xms256m -Xmx512m`.
- **[PRÉVU]** Prérequis manuel : remplir `vault_springboot_db_password` et
  déposer le JAR (`README.md` §13, TODO « Déposer le JAR … »).

## 19. Mode local cloisonné

- **[NON IMPLÉMENTÉ]** : pas de `springboot_instance`, pas de playbook local,
  `bo01` sans variables dans l'inventaire local, port/datasource/slot de
  secrets non adaptés. **Rien n'est supposé ici** : variables, services,
  tests et problèmes seront documentés **lors de l'implémentation réelle**.

## 20. Problèmes rencontrés (réels)

- **Aucun problème d'exécution réelle** (le rôle n'a pas tourné dans ce lab).
- Points de conception **constatés dans le code** et à traiter à
  l'implémentation :
  1. port `8080` déjà occupé en local (WAF étranger) ;
  2. datasource AWS `5432` inadaptée au cluster local `18/db01:5442` ;
  3. unité `backoffice.service` sans durcissement systemd ;
  4. JAR absent de `files/` (déploiement sauté par conception).

## 21. Références

- [`../roles/springboot/README.md`](../roles/springboot/README.md) — dont
  « Fichiers statiques » et « Sécurité ».
- [`../roles/springboot/files/README.md`](../roles/springboot/files/README.md) — règle du formateur.
- [`../docs/installation-manuelle.md`](../docs/installation-manuelle.md) — §Tier 3, étapes 2 à 4 (manuel).
- [`../README.md`](../README.md) — §6, §13 (TODO JAR).
- [17-java.md](17-java.md) (rôle amont), [10-securite-et-secrets.md](10-securite-et-secrets.md).

## 22. Résumé

**[CODE]** Rôle Spring Boot fonctionnel « par construction » : user dédié,
`/opt/backoffice`, JAR statique opt-in (`springboot_jar_file`), `.env` `0600`
(`no_log`), unité systemd `backoffice` sur `--server.port=8080`, vérifications
non bloquantes. **[PRÉVU]** Tier Backoffice sur `bo01` (AWS), base `db01:5432`,
secret `vault_springboot_db_password`. **[NON IMPLÉMENTÉ]** Tout le volet
local (playbook, instance, port libre, datasource 5442, JAR, tests réels) :
à compléter avec l'implémentation réelle, sans rien inventer d'ici là.



