# 17 — Rôle `java` (Backoffice — **non prêt en local**)

> ⚠️ **État du Backoffice** : le rôle `java` **existe et est complet dans le
> code**, mais le tier Backoffice **n'est pas encore réalisé** dans ce projet :
> aucun déploiement local n'a été effectué ni validé. Ce document distingue
> strictement : **[CODE]** = présent et vérifiable dans le dépôt ·
> **[PRÉVU]** = prévu par l'architecture · **[NON IMPLÉMENTÉ]** = absent ou
> jamais exécuté. À compléter avec l'implémentation réelle (variables choisies,
> services, templates, playbooks, tests, problèmes) lors de l'étape Backoffice.

---

## 1. À quoi sert ce rôle ?

**[CODE]** Installe un JDK OpenJDK (PPA `ondrej/java`, détection du vrai
`JAVA_HOME`, validation fail-fast) et expose `JAVA_HOME` via `/etc/profile.d/`.
C'est le prérequis du rôle `springboot` (application Backoffice Spring Boot).

## 2. Où il vit dans le dépôt

**[CODE]**
```
roles/java/
├── defaults/main.yml      # java_version, paquet, JAVA_HOME, PPA, profile
├── vars/main.yml          # 10 constantes internes (java_jvm_dir, java_bin, …)
├── tasks/main.yml         # 118 lignes (PPA → JDK → détection → assert → env → test)
├── handlers/main.yml      # « Mettre à jour le cache APT »
├── templates/
│   ├── ondrej-java.sources.j2  # dépôt APT (format deb822)
│   └── java_env.sh.j2          # /etc/profile.d/java.sh
└── README.md
```

## 3. Comment il est appelé

- **[CODE]** `playbooks/applications.yml`, 2e play :
  `hosts: backoffice`, `roles: [java, springboot]` (AWS).
- **[PRÉVU]** Un équivalent local (`local_backoffice.yml` ?) reste à définir.
- **[NON IMPLÉMENTÉ]** Aucun playbook `local_*.yml` pour le tier backoffice ;
  l'hôte `bo01` de l'inventaire local ne porte **que** `ansible_host`
  (aucune variable d'instance).

## 4. Prérequis et dépendances

- **[CODE]** Réseau pour le PPA et le keyserver Ubuntu ; paquets prérequis
  `ca-certificates`, `curl`, `gnupg` (`java_apt_prerequisites`, `vars/`).
- **[CODE]** `meta/main.yml` : `dependencies: []` (l'ordre vient du playbook).
- **[PRÉVU]** `java` doit précéder `springboot` (déjà le cas dans le play).

## 5. Ports, services et chemins

- **[CODE]** **Aucun port, aucun service** : ce rôle n'installe que le JDK.
- **[CODE]** Chemins : `JAVA_HOME` = `/usr/lib/jvm/java-21-openjdk-amd64`
  (repli) ; profil `/etc/profile.d/java.sh` ; trousseaux
  `/etc/apt/keyrings/ondrej-java.asc` ; sources
  `/etc/apt/sources.list.d/ondrej-java.sources`.
- **[PRÉVU]** L'application écoutera sur `springboot_port: 8080`
  (`group_vars/backoffice.yml`) — port **réservé au tier backoffice en AWS** ;
  en local, le 8080 est occupé par `apache2-waf-svc` → un port distinct devra
  être choisi lors de l'implémentation locale (à décider, pas de valeur
  inventée ici).
- **[NON IMPLÉMENTÉ]** Service systemd, utilisateur système, déploiement JAR :
  voir [18-springboot.md](18-springboot.md).

## 6. Variables

| Variable | Valeur actuelle | Fichier | Pourquoi | Exemple de modification | Impact |
|---|---|---|---|---|---|
| `java_version` | `"21"` | `defaults` + `group_vars/backoffice.yml` | Version du JDK | `group_vars` → `"21.0.x"`… | paquet + chemin `JAVA_HOME` |
| `java_package` | `openjdk-21-jdk-headless` | `defaults` | Headless = suffisant pour Spring Boot | — | `apt` |
| `java_architecture` | `amd64` | `defaults` | Nom du répertoire JVM | — | chemin |
| `java_home` | `/usr/lib/jvm/java-21-openjdk-amd64` | `defaults` | Repli si détection absente | — | assert + environnement |
| `java_home_autodetect` | `true` | `defaults` | « Option B » : détecter le vrai `JAVA_HOME` | `false` | `find` + `set_fact` |
| `java_apt_repository_enabled` | `true` | `defaults` | openjdk-21 absent des dépôts Ubuntu par défaut | `false` (si déjà présent) | tout le bloc PPA |
| `java_apt_repository_name` / `_uri` / `_key_url` | `ondrej-java` / PPA launchpad / clé keyserver | `defaults` | Dépôt officiel pour JDK 21 | — | `.sources` + `.asc` |
| `java_apt_cache_valid_time` | `3600` | `defaults` | Éviter un `apt update` à chaque run | — | `apt: cache_valid_time` |
| `java_profile_file` | `/etc/profile.d/java.sh` | `defaults` | `JAVA_HOME` global | — | template `java_env.sh.j2` |
| `java_jvm_dir`, `java_bin`, `java_apt_keyring_dir`, `java_apt_sources_dir`, `java_apt_lock_timeout`, `java_apt_prerequisites` | constantes (chemins `/usr/lib/jvm`, `/usr/bin/java`, `300`, 3 paquets) | **`vars/main.yml`** | internes, jamais surchargées | — | tâches |

> **Note de précédent** : contrairement aux 6 rôles cloisonnés, `vars/main.yml`
> de `java` reste **actif** (10 lignes) — ses constantes ne conflictualisent
> avec aucun réglage d'inventaire (voir [02](02-ansible-variables-et-precedence.md)).

## 7. Templates

**[CODE]** `ondrej-java.sources.j2` (dépôt deb822 : `Signed-By`, `URIs`,
`Suites`) et `java_env.sh.j2` (`export JAVA_HOME=…`).

## 8. Handlers

**[CODE]** `Mettre à jour le cache APT` (`apt: update_cache`,
`cache_valid_time: 0`) — déclenché par le téléchargement de la clé et le
dépôt du `.sources`.

## 9. Parcours des tâches

**[CODE]** 1. `apt` prérequis (`cache_valid_time`, `lock_timeout`) →
2. `/etc/apt/keyrings` → 3. `get_url` de la clé de signature → 4. template du
`.sources` (deb822) → `notify: Mettre à jour le cache APT` → 5. `apt` du JDK →
6. `find /usr/lib/jvm/java-21-openjdk-*` → `set_fact java_home` →
7. `stat` + **`assert`** (fail-fast : échoue immédiatement si `JAVA_HOME`
introuvable, pas au démarrage de l'app) → 8. template `/etc/profile.d/java.sh`
→ 9. `java -version` (`changed_when: false`).

## 10. Sécurité et secrets

- **[CODE]** Aucun secret. Clé APT téléchargée depuis le keyserver Ubuntu
  (`B8DC7E53946656EFBCE4C1DD71DAEAAB4AD4CAB6`), fichier `0644` root.
- **[PRÉVU]** Le secret du backoffice (`vault_springboot_db_password`)
  appartient à `springboot`, pas à `java`.

## 11. Vérifications intégrées (non bloquantes)

**[CODE]** `java -version` (via `{{ java_home }}/bin/java`) en
`changed_when: false` + affichage de `stderr_lines` (le banner Java sort sur
stderr).

## 12. Idempotence

**[CODE]** `apt state: present` + `cache_valid_time`, `get_url` (hash),
`file`, `template`, `find/changed_when: false`. Le cache APT n'est rafraîchi
que par `notify`.

## 13. Tests effectués (preuves)

- **[CODE]** Les commandes de vérification existent dans le rôle
  (`java -version`, `assert` de `JAVA_HOME`).
- **[NON IMPLÉMENTÉ]** Aucune exécution réelle de ce rôle n'a été menée dans
  ce lab (le tier backoffice n'a pas été déployé localement) : **aucune
  validation « exécution réelle » à affirmer ici**. Ce sera à compléter lors
  de l'implémentation.

## 14. Recettes de modification

| Je veux… | Fichier |
|---|---|
| Changer la version du JDK (AWS) | `group_vars/backoffice.yml` → `java_version` |
| Désactiver le PPA (JDK déjà présent) | `java_apt_repository_enabled: false` |
| Forcer un `JAVA_HOME` fixe | `java_home_autodetect: false` + `java_home` |

## 15. Intégration avec les autres tiers

- **[CODE]** Aval : `springboot` (`springboot_java_bin` lit `java_home`).
- **[PRÉVU]** `backoffice` → base PostgreSQL (`springboot_datasource_url`,
  port 5432 en AWS).
- **[NON IMPLÉMENTÉ]** Aucun lien local : `bo01` n'a aucune variable dans
  `inventories/local/hosts.yml` et aucun playbook local ne le cible.

## 16. Points d'attention (pièges)

1. **[CODE]** openjdk-21 n'est **pas** dans les dépôts Ubuntu par défaut
   (seulement 17) → PPA requis (`java_apt_repository_enabled`).
2. **[CODE]** La détection (`find` + `set_fact`) prime sur le repli `java_home`
   : en cas d'échec de détection, l'`assert` bloque **avant** tout démarrage.
3. **[PRÉVU]** Le port `springboot_port: 8080` (AWS) **conflit** avec le WAF
   étranger sur la machine locale : un choix de port local devra être fait au
   moment de l'implémentation (non décidé à ce jour).

## 17. Non trouvé dans le code actuel

- Aucun `java_instance` (pas de mode cloisonné — contrairement aux 6 rôles
  du lab) : **non trouvé dans le code actuel**.
- Aucun playbook `local_backoffice.yml` : **non trouvé dans le code actuel**.
- Aucune variable d'instance pour `bo01` dans l'inventaire local
  (**non trouvé dans le code actuel**).
- Aucun test d'exécution réelle du rôle dans ce lab (voir §13).

## 18. Mode AWS (défaut)

- **[CODE]** C'est le **seul** mode opératoire à ce jour : `applications.yml`
  → `hosts: backoffice` → `roles: [java, springboot]`, via l'inventaire
  dynamique EC2 (tag `Role=backoffice` → groupe `backoffice`).
- **[PRÉVU]** Déploiement sur `bo01` (1 instance, `10.0.0.20` en statique).

## 19. Mode local cloisonné

- **[NON IMPLÉMENTÉ]** Non prévu à ce jour : ni `java_instance`, ni playbook,
  ni variables d'inventaire local. **Ne rien supposer** : lors de
  l'implémentation, ce sera l'objet d'une mise à jour de ce document.

## 20. Problèmes rencontrés (réels)

- **Aucun problème d'exécution réelle** : le rôle n'a pas encore été exécuté
  dans ce lab — il n'y a donc **aucun retour d'expérience** à décrire.
  (Un problème de conception est noté §16.3 : conflit de port 8080 prévu en local.)

## 21. Références

- [`../roles/java/README.md`](../roles/java/README.md) — référence du rôle
  (dont « Dépôt APT », « Constantes internes », « Tâches de validation »).
- [`../docs/installation-manuelle.md`](../docs/installation-manuelle.md) — §Tier 3, étape 1 (manuel).
- [`../README.md`](../README.md) — §6 (état des rôles), §7 (`applications.yml`).
- [18-springboot.md](18-springboot.md) (rôle aval).

## 22. Résumé

**[CODE]** Rôle Java **complet dans le code** : PPA `ondrej/java` (deb822),
JDK 21 headless, détection + `assert` fail-fast du `JAVA_HOME`,
`/etc/profile.d/java.sh`, `java -version`. **[PRÉVU]** Prérequis du Backoffice
(`bo01`, tag `Role=backoffice`). **[NON IMPLÉMENTÉ]** Rien n'en local
(pas de `java_instance`, pas de playbook, pas de test réel) : ce document sera
complété avec les variables, services, tests et problèmes **réels** lors de
l'étape Backoffice.

