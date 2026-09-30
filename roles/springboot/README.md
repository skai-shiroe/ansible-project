# Rôle `springboot`

Déploie l'application **Backoffice Spring Boot** : utilisateur système dédié,
JAR statique, fichier d'environnement et **unité systemd** gérée de bout en bout.

## Variables

| Variable | Emplacement | Défaut | Rôle |
|---|---|---|---|
| `springboot_app_name` | `group_vars/backoffice.yml` | `backoffice` | Nom de l'application |
| `springboot_port` | `group_vars/backoffice.yml` | `8080` | Port d'écoute |
| `springboot_home` | `group_vars/backoffice.yml` | `/opt/backoffice` | Racine de déploiement |
| `springboot_jar_file` | `defaults` | `""` | **Fichier statique** situé dans `files/` |
| `springboot_jar_name` | `defaults` | `backoffice.jar` | Nom à la destination |
| `springboot_java_opts` | `defaults` | `-Xms256m -Xmx512m` | Options JVM |
| `springboot_java_bin` | `defaults` | `{{ java_home }}/bin/java` (ou `java` du PATH) | Binaire Java de l'unité systemd |
| `springboot_datasource_*` | `defaults` | pointe vers `groups['databases'][0]` | Connexion PostgreSQL |

## Fichiers de configuration (templates)

| Template | Destination |
|---|---|
| `templates/springboot.service.j2` | `/etc/systemd/system/{{ springboot_app_name }}.service` |
| `templates/springboot.env.j2` | `{{ springboot_home }}/{{ springboot_app_name }}.env` |

L'unité référence `EnvironmentFile=` (contenant la datasource **et `JAVA_HOME`**
si le rôle `java` a été exécuté) et le binaire issu de `java_home`
(`springboot_java_bin`). Redémarrage automatique en cas d'échec
(`Restart=on-failure`).

## Tâches de vérification (non bloquantes)

- `stat` du JAR + debug : indique s'il est déployé ou absent.
- `systemd-analyze verify` sur l'unité (`failed_when: false`).
- État du service + `journalctl -u … -n 5` (`failed_when: false`).

## Fichiers statiques — règle du formateur

> Les binaires (`*.jar`) se déposent dans **`roles/springboot/files/`** et sont
> déployés par `ansible.builtin.copy`.

| Variable | Signification |
|---|---|
| `springboot_jar_file` | nom du fichier **dans `files/`** (vide = tâche sautée) |
| `springboot_jar_name` | nom **sur le serveur** |

Voir `files/README.md`.

## Sécurité

- Utilisateur système `springboot` en `nologin` (jamais root).
- La tâche du fichier d'environnement porte `no_log: true` (contient un
  mot de passe).

## Handlers

- `Redémarrer Spring Boot`

## Appelé par

`playbooks/applications.yml` — groupe `backoffice` (après `java`)

