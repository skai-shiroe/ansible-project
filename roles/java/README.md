# Rôle `java`

Installe le **JDK OpenJDK** requis par l'application Backoffice Spring Boot,
gère son dépôt APT si besoin, détecte le vrai `JAVA_HOME`, l'expose au
système, puis vérifie que la JVM répond correctement.

## Variables

| Variable | Emplacement | Défaut | Rôle |
|---|---|---|---|
| `java_version` | `group_vars/backoffice.yml` | `"21"` | Version du JDK |
| `java_package` | `defaults` | `openjdk-{{ java_version }}-jdk-headless` | Paquet installé |
| `java_architecture` | `defaults` | `amd64` | Nom du répertoire sous `/usr/lib/jvm` |
| `java_home` | `defaults` (+ `set_fact` après détection) | `/usr/lib/jvm/java-{{ java_version }}-openjdk-{{ java_architecture }}` | `JAVA_HOME` (propagé aux rôles suivants) |
| `java_home_autodetect` | `defaults` | `true` | Détecte le vrai chemin via `find` (Option B) |
| `java_apt_repository_enabled` | `defaults` | `true` | Active le PPA `ondrej/java` (requis pour JDK 21) |
| `java_apt_cache_valid_time` | `defaults` | `3600` | Évite un `apt update` systématique |
| `java_profile_file` | `defaults` | `/etc/profile.d/java.sh` | Fichier d'environnement système |

`headless` : variante sans interface graphique, suffisante côté serveur.

## Dépôt APT

`openjdk-21` n'existe pas dans les dépôts Ubuntu par défaut : le rôle déploie
le PPA `ondrej/java` au **format moderne deb822** via le template
`ondrej-java.sources.j2` (clé GPG téléchargée dans `/etc/apt/keyrings/`).
Désactiver avec `java_apt_repository_enabled: false`.

## Constantes internes (`vars/`)

| Variable | Valeur |
|---|---|
| `java_jvm_dir` | `/usr/lib/jvm` |
| `java_bin` | `/usr/bin/java` (repli système) |
| `java_apt_lock_timeout` | `300` |
| `java_apt_keyring_dir` / `java_apt_sources_dir` | `/etc/apt/keyrings` / `/etc/apt/sources.list.d` |

## Fichiers de configuration

| Template | Destination | Rôle |
|---|---|---|
| `ondrej-java.sources.j2` | `/etc/apt/sources.list.d/ondrej-java.sources` | Définition du dépôt (deb822) |
| `java_env.sh.j2` | `{{ java_profile_file }}` | `JAVA_HOME` + `PATH` pour tout le shell |

## Tâches de validation

- `stat` + **`assert`** : échec explicite et immédiat si `JAVA_HOME` est
  introuvable (fail-fast : aucun échec différé au démarrage Spring Boot).
- `{{ java_home }}/bin/java -version` avec `environment: JAVA_HOME`.

## Fichiers statiques

Aucun.

## Appelé par

`playbooks/applications.yml` — groupe `backoffice` (avant `springboot`)

