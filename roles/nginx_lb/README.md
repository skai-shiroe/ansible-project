# Rôle `nginx_lb`

Installe **Nginx en équilibreur de charge** : génère un bloc `upstream` avec
une entrée par instance web, répartit le trafic et expose un health-check.

## Variables

| Variable | Emplacement | Défaut | Rôle |
|---|---|---|---|
| `nginx_lb_listen_port` | `group_vars/loadbalancers.yml` | `80` | Port public |
| `nginx_lb_backend_servers` | `group_vars/loadbalancers.yml` | `{{ groups['webservers'] }}` | Backends découverts dynamiquement |
| `nginx_lb_backend_port` | `group_vars/loadbalancers.yml` | `8080` | Port des backends |
| `nginx_lb_lb_method` | `defaults` | `least_conn` | `round_robin` \| `least_conn` \| `ip_hash` |
| `nginx_lb_health_check_path` | `defaults` | `/up` | Endpoint de santé |
| `nginx_lb_upstream_name` | `defaults` | `laravel_web` | Nom du bloc `upstream` |

## Fichiers de configuration (templates)

| Template | Destination |
|---|---|
| `templates/nginx_lb.conf.j2` | `/etc/nginx/sites-available/{{ nginx_lb_site_file }}` |

Contenu généré :

```nginx
upstream laravel_web {
    least_conn;
    server web01:8080 max_fails=3 fail_timeout=30s;
    server web02:8080 max_fails=3 fail_timeout=30s;
}
```

`max_fails` / `fail_timeout` apportent la **tolérance aux pannes** (le backend
coupé est écarté temporairement).

## Tâches de vérification (non bloquantes)

- `nginx -t` : test de syntaxe de la configuration (`failed_when: false`,
  résultat affiché).
- `nginx -v` : version installée.
- `service_facts` + debug : statut du service et résumé de l'`upstream`
  (nombre de backends, port).

## Fichiers statiques

Aucun.

## Handlers

- `Recharger Nginx` (reload, pas de coupure)

## Appelé par

`playbooks/site.yml` — groupe `loadbalancers`, **en dernier** (les backends
doivent exister avant le LB)

