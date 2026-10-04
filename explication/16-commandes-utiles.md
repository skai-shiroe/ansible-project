# 16 — Commandes utiles

Aide-mémoire du quotidien. Une ligne d'explication par commande. **Les
inventaires et le vault sont obligatoires** pour les jeux locaux.

## 1. Ansible — lecture et simulation

```bash
# Inventaire résolu (groupes + hôtes) — vérifie group_vars/host_vars
ansible-inventory -i inventories/local/hosts.yml --graph

# Variables effectives d'un hôte — vérifie la PRÉCÉDENCE (02)
ansible-inventory -i inventories/local/hosts.yml --host web01

# Syntaxe — détecte les erreurs YAML/Jinja sans rien exécuter
ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml --syntax-check

# Liste des cibles / des tâches — vérifie le câblage des rôles
ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml --list-hosts
ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml --list-tasks

# Simulation complète avec diff — comparer avec docs/installation-manuelle.md
ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml --vault-password-file .vault_pass --check --diff

# Mode verbeux (tâche fautive + hôte) — 1er réflexe de dépannage
ansible-playbook … -v

# Limiter à un hôte / un groupe
ansible-playbook … --limit web01
```

## 2. Ansible — exécution (lab local)

```bash
# Ordre : cache → web → db → lb (cf. README §8)
ansible-playbook -i inventories/local/hosts.yml playbooks/local_cache.yml --vault-password-file .vault_pass
ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml   --vault-password-file .vault_pass
ansible-playbook -i inventories/local/hosts.yml playbooks/local_db.yml    --vault-password-file .vault_pass
ansible-playbook -i inventories/local/hosts.yml playbooks/local_lb.yml

# Activer le worker de file d'attente (opt-in, défaut false)
ansible-playbook -i inventories/local/hosts.yml playbooks/local_web.yml \
  --vault-password-file .vault_pass -e laravel_queue_worker_enabled=true

# AWS (nécessite aws sso login) — inventaire PAR DÉFAUT de ansible.cfg
AWS_PROFILE=formation-sso ansible-playbook playbooks/site.yml
```

## 3. Vault et secrets

```bash
# Lire le vault local (5 clés) — affiche aussi les noms seulement
ansible-vault view inventories/local/group_vars/all/vault.yml --vault-password-file .vault_pass

# Chiffrer / éditer le vault AWS (vide par défaut)
ansible-vault encrypt group_vars/all/vault.yml
ansible-vault edit    group_vars/all/vault.yml

# Vérifier qu'aucun secret n'est en clair avant commit
git status --short        # ne doit montrer que explication/…
```

## 4. Services et journaux (TOUJOURS le nom exact, jamais les méta-services)

```bash
# Services du projet
systemctl is-active nginx-lb01 apache-web01 apache-web02 php-fpm-web01 php-fpm-web02 \
  postgresql@18-db01 postgresql@18-db02 redis-redis01

# Journaux d'un service projet
journalctl -u nginx-lb01 -n 50 --no-pager
journalctl -u postgresql@18-db01 -n 50 --no-pager

# Écoutes TCP : distingue projet (908x/544x/6390) et étrangers (80/808x/5432/6379/26379)
ss -ltnp

# NE PAS FAIRE (méta-services = services étrangers) :
#   systemctl restart postgresql    → redémarrerait 18-main (5432)
#   systemctl restart redis-server  → redémarrerait le Redis 6379
```

## 5. HTTP / load balancer

```bash
# Health-check du LB puis code HTTP des 3 points d'entrée
curl -s http://127.0.0.1:9080/up                          # → OK
for p in 9080 9081 9082; do curl -s -o /dev/null -w "$p %{http_code}\n" http://127.0.0.1:$p/; done

# Voir quel backend le LB utilise (upstream généré)
grep -E 'server web0|listen' /etc/nginx-lb01/sites-enabled/*.conf
```

## 6. PHP / Laravel (en tant que l'utilisateur du service)

```bash
# .env est 0600 owner web01 → TOUJOURS sudo -u web01
sudo -u web01 php /var/www/web01/artisan db:show          # connexion PDO réelle → PG 18.x:5442
sudo -u web01 php /var/www/web01/artisan --version
sudo -u web01 php --ri redis | grep 'Redis Support'       # phpredis chargé ?
sudo -u web01 php /var/www/web01/artisan migrate:status    # migrations

# Tâche artisan ponctuelle (migrations = run_once côté Ansible, pas ici)
sudo -u web01 php /var/www/web01/artisan cache:clear
```

## 7. PostgreSQL (clusters dédiés ; jamais le méta « postgresql »)

```bash
# Les 3 clusters et leurs ports (18-main 5432 intact | db01 5442 | db02 5443)
pg_lsclusters

# SQL en superuser local (peer) sur LE cluster visé
sudo -u postgres psql -p 5442 -c "SELECT version();"
sudo -u postgres psql -p 5443 -c "SHOW transaction_read_only;"   # → on (réplique)

# État de la réplication (primaire)
sudo -u postgres psql -p 5442 -c "SELECT client_addr, state, sent_lsn, replay_lsn FROM pg_stat_replication;"
sudo -u postgres psql -p 5442 -c "SELECT slot_name, active FROM pg_replication_slots;"

# En tant qu'utilisateur applicatif (scram, via TCP)
psql -h 127.0.0.1 -p 5442 -U laravel -d laravel -c 'select 1;'

# Administration d'un cluster dédié
pg_lsclusters ; sudo pg_ctlcluster 18 db01 status
```

## 8. Redis (instance projet 6390 ; jamais `-a` ni le 6379 étranger)

```bash
# Auth : variable d'environnement (le rôle fait pareil), JAMAIS redis-cli -a
export REDISCLI_AUTH='<SECRET>'

# Instance du projet
redis-cli -p 6390 ping                    # sans mdp : NOAUTH (attendu si requirepass)
redis-cli -p 6390 info replication        # role:master connected_slaves:0
redis-cli -p 6390 info keyspace           # bases logiques utilisées (sessions db0…, cache db1)
redis-cli -p 6390 llen queues:default     # jobs en attente dans la file

# Clés d'une session Laravel
redis-cli -p 6390 --scan --pattern '*session*'

# Étrangers : intacts = PONG (hors périmètre du projet)
redis-cli -p 6379 ping
redis-cli -p 26379 ping
```

## 9. Contrôle de non-régression (après chaque playbook)

```bash
pg_lsclusters                                              # 3 clusters online (5432 inclus)
systemctl is-active nginx apache2-waf-svc haproxy-lb-svc tomcat10 \
  postgresql@18-main redis-cache-svc redis-sentinel        # tous inchangés
redis-cli -p 6379 ping && redis-cli -p 26379 ping          # PONG
git status --short                                         # aucune modification hors explication/
```

## 10. Collections et versions

```bash
ansible --version                                  # ansible-core (2.21.3 testé)
ansible-galaxy collection list | grep -E 'community|amazon'
# community.general (apache2_module, composer), community.postgresql (postgresql_user…),
# amazon.aws + community.aws (déclarées dans requirements.yml)
ansible-galaxy collection install -r requirements.yml   # installation
```

## 11. Git (documentation)

```bash
git status --short          # la phase documentaire ne doit ajouter que explication/
git add explication/        # puis commit (aucun fichier fonctionnel modifié)
```

---

## Récapitulatif des invariants (à tester avant de conclure)

```bash
# Projet OK  : 9080/9081/9082 → 200 ; 5442/5443/6390 en écoute ; réplication streaming
# Étrangers OK : 80, 8080, 8081, 8082, 5432, 6379, 26379 toujours actifs
# Idempotence : chaque playbook rejoué → changed=0
```

→ Retour à [l'index](README.md)

