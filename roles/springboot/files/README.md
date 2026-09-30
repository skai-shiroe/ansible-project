# `roles/springboot/files/` — fichiers statiques

## Règle (formateur)

Les fichiers **statiques et non générés** (binaires `.jar`, images, PDF, scripts
fixes, etc.) se déposent **ici**, et non dans `templates/` :

```
votre_projet/roles/votre_role/files/<fichier>
```

- **`files/`** → contenu **copié tel quel** (`ansible.builtin.copy`)
- **`templates/`** → contenu **généré par Jinja2** (`ansible.builtin.template`),
  à utiliser dès qu'il faut variabiliser ou rendre la configuration conditionnelle

## Usage dans ce rôle

| Variable | Valeur attendue |
|---|---|
| `springboot_jar_file` | nom du fichier **présent dans ce répertoire** (ex. `backoffice-1.0.0.jar`) |
| `springboot_jar_name` | nom **à la destination** sur le serveur (ex. `backoffice.jar`) |

Exemple dans `group_vars/backoffice.yml` :

```yaml
springboot_jar_file: backoffice-1.0.0.jar
```

Si `springboot_jar_file` reste vide (valeur par défaut), la tâche de déploiement
du JAR est **sautée** : le reste du rôle s'exécute sans erreur.

> ⚠️ **Sécurité** : les binaires volumineux ne doivent pas être committés en
> clair. Préférez un artefact externe (artefactory, S3, Git LFS) et renseignez
> uniquement son nom ici.
