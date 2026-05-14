# Debugging & Outillage

## Xdebug — La Configuration Complète

### Étape 1 : `extra_hosts` dans docker-compose.yml

```yaml
# .docker/docker-compose.yml ou .docker/docker-compose.dev.yml
services:
  php:
    extra_hosts:
      # Crée un alias 'host.docker.internal' pointant vers la machine hôte
      # Permet à Xdebug (dans le container) de joindre l'IDE (sur l'hôte)
      - "host.docker.internal:host-gateway"
```

Sans `extra_hosts`, Xdebug ne peut pas trouver l'IDE sur la machine hôte.

### Étape 2 : Configuration Xdebug dans `php.ini`

```ini
# .docker/services/php/conf/php-dev.ini — uniquement pour le profil dev

[xdebug]
; Xdebug 3 (depuis PHP 7.3+)
xdebug.mode = debug
xdebug.start_with_request = yes     ; Démarrer à chaque requête (peut être 'trigger' pour performance)
xdebug.client_host = host.docker.internal   ; ← pointe vers l'IDE via extra_hosts
xdebug.client_port = 9003           ; Port par défaut Xdebug 3 (9000 en Xdebug 2)
xdebug.log = /tmp/xdebug.log        ; Logs pour débugger Xdebug lui-même
; xdebug.idekey = PHPSTORM          ; Pour PhpStorm uniquement — VS Code l'ignore
xdebug.max_nesting_level = 512      ; Éviter "maximum nesting level reached"
```

### Étape 3 : Configuration de l'IDE

**VS Code** — `launch.json` :
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Docker Drupal",
      "type": "php",
      "request": "launch",
      "port": 9003,
      "pathMappings": {
        "/var/www/html": "${workspaceFolder}"
      },
      "hostname": "0.0.0.0",
      "ignore": ["**/vendor/**/*.php"]
    }
  ]
}
```

**PhpStorm** :
- Settings → PHP → Debug → Port: `9003`
- Settings → PHP → Servers → Ajouter:
  - Host: `localhost`
  - Port: `80`
  - Debugger: `Xdebug`
  - Cocher "Use path mappings"
  - Mapper le dossier local `/chemin/projet` → `/var/www/html`

### Étape 4 : Vérifier que Xdebug fonctionne

```bash
# Dans le container — vérifier que Xdebug est chargé
docker compose exec php php -m | grep -i xdebug

# Voir la config active
docker compose exec php php -i | grep -A 20 xdebug

# Vérifier la connectivité vers l'hôte
docker compose exec php php -r "
\$sock = @fsockopen('host.docker.internal', 9003, \$errno, \$errstr, 2);
echo \$sock ? 'IDE JOIGNABLE sur port 9003' : 'ECHEC: ' . \$errstr;
"

# Voir les logs Xdebug (si activés)
docker compose exec php tail -f /tmp/xdebug.log
```

---

## Logs Docker — Lire une Page Blanche (Erreur 500)

```bash
# Logs PHP (erreurs PHP, stack traces)
docker compose logs php -f
docker compose logs php --tail=50

# Logs DB (lenteur, erreurs de connexion)
docker compose logs database -f

# Tous les logs simultanément
docker compose logs -f

# Logs d'un container spécifique sur une période
docker compose logs php --since="1h"

# Logs de tous les services avec timestamps
docker compose logs --timestamps

# Filtrer les erreurs dans les logs
docker compose logs php 2>&1 | grep -i "error\|fatal\|warning"

# Accéder aux logs Apache dans le container
docker compose exec php tail -f /var/log/apache2/error.log
docker compose exec php tail -f /var/log/apache2/access.log

# Logs PHP-FPM (si séparé)
docker compose exec php tail -f /var/log/php-fpm/error.log
```

---

## Shell dans les Containers

```bash
# Ouvrir un shell interactif dans PHP
docker compose exec php bash

# Shell dans MariaDB
docker compose exec database bash
docker compose exec database mariadb -u drupal -pdrupal drupal

# Container éphémère (--rm = supprimé après la commande)
docker compose run --rm php bash
docker compose run --rm php php -i | grep memory_limit

# Exécuter une commande directement
docker compose exec php php -r "phpinfo();" | head -30
```

---

## Drush depuis l'Hôte

```bash
# Via docker compose exec (container en cours)
docker compose exec php drush status
docker compose exec php drush cr
docker compose exec php drush cex -y

# Via docker compose run (container éphémère — préférable pour les scripts)
docker compose run --rm php drush site:install --existing-config -y
docker compose run --rm php drush updb -y

# Avec DRUSH_LAUNCHER_FALLBACK dans l'env (automatique via Makefile)
# La variable DRUSH_LAUNCHER_FALLBACK est passée dans docker-compose.yml :
# DRUSH_LAUNCHER_FALLBACK: /var/www/html/vendor/bin/drush
```

---

## Troubleshooting — Erreurs Courantes

| Erreur | Cause probable | Diagnostic | Solution |
|--------|---------------|-----------|---------|
| Page blanche (500) | Erreur PHP, permission | `docker compose logs php` | Voir les logs, corriger le code |
| 403 Forbidden | Permissions fichiers | `ls -la web/` | `sudo chown -R 1000:www-data web/sites/default/files` |
| DB connexion refusée | `localhost` dans settings.php | `docker compose exec php php -r "var_dump(getenv('MARIADB_HOSTNAME'));"` | Changer en `database` |
| Container ne démarre pas | Port en conflit | `docker compose ps` | Changer `HTTPD_PORT` dans `.env` |
| `composer install` lent | Cache non monté | Vérifier `composer_home` volume | Vérifier `PERSONAL_GLOBAL_COMPOSER_FOLDER` |
| Xdebug ne se connecte pas | IDE pas en écoute | Vérifier launch.json + port 9003 | Démarrer l'écoute Xdebug dans l'IDE avant la requête |
| Fichiers non mis à jour | Agrégation CSS/JS active | `/admin/config/development/performance` | Désactiver l'agrégation OU `drush cr` |
| `Permission denied` sur `files/` | UID hôte ≠ UID container | `ls -la web/sites/default/files/` | `make install-linux` OU corriger DEV_UID |
| Drupal slow (10s par page) | OPcache désactivé ou bind mount lent | `docker stats` | Voir performance.md |

---

## Utilisateur des Commandes — Context `exec`

Par défaut, `docker compose exec php` lance les commandes en tant que l'utilisateur du container (souvent `root` ou `www-data`). Cela peut créer des fichiers avec de mauvaises permissions.

```bash
# Voir l'utilisateur courant dans le container
docker compose exec php whoami

# Exécuter en tant qu'utilisateur spécifique
docker compose exec --user www-data php drush cr
docker compose exec --user 1000 php drush updb -y

# Pour Drush — toujours cohérent avec l'utilisateur web
docker compose exec --user www-data php drush cr
```

**Règle :** si Drush crée des fichiers dans `web/sites/default/files/` (cache, imports), utiliser `--user www-data` pour éviter que ces fichiers appartiennent à `root`.

---

## DB Readiness — Attendre que MariaDB soit Prête

MariaDB prend 5-30 secondes à démarrer. Lancer `drush site:install` immédiatement après `docker compose up -d` échoue de manière aléatoire.

```bash
# Attendre explicitement dans le Makefile
wait-db: ## Attendre que la DB soit prête
	@echo "→ Attente de la DB..."
	@until docker compose exec database mariadb -u root -p$${BDD_MARIADB_ROOT_PASSWORD} \
	  -e "SELECT 1" > /dev/null 2>&1; do \
	    echo "  DB pas encore prête, attente 2s..."; \
	    sleep 2; \
	done
	@echo "✓ DB prête"

install: wait-db
	@docker compose run --rm php drush site:install ...
```

```yaml
# Alternative : healthcheck dans docker-compose.yml (déjà documenté)
database:
  healthcheck:
    test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
    interval: 5s
    timeout: 5s
    retries: 10
    start_period: 30s   # ← Donner 30s à MariaDB pour démarrer

php:
  depends_on:
    database:
      condition: service_healthy  # ← PHP attend que la DB soit healthy
```

---

## Importer/Exporter la Base de Données

```bash
# Dump depuis le container DB
docker compose exec database mariadb-dump \
  -u root -p${BDD_MARIADB_ROOT_PASSWORD} \
  ${BDD_MARIADB_DATABASE} > .docker/services/database/dumps/dump-$(date +%Y%m%d).sql

# Dump en direct depuis l'hôte via Drush (recommandé)
docker compose exec php drush sql:dump --result-file=/var/www/html/.docker/services/database/dumps/dump.sql

# Importer un dump
docker compose exec -T database mariadb \
  -u ${BDD_MARIADB_USER} -p${BDD_MARIADB_PASSWORD} \
  ${BDD_MARIADB_DATABASE} < .docker/services/database/dumps/dump.sql

# Vider la DB (pour réinstaller depuis zéro)
docker compose exec database mariadb \
  -u root -p${BDD_MARIADB_ROOT_PASSWORD} \
  -e "DROP DATABASE ${BDD_MARIADB_DATABASE}; CREATE DATABASE ${BDD_MARIADB_DATABASE};"
```

---

## Inspections Réseau Docker

```bash
# Voir le réseau du projet
docker network inspect ${COMPOSE_PROJECT_NAME}_default

# Voir les IPs des containers
docker compose exec php cat /etc/hosts

# Tester la connectivité entre containers
# ping peut être absent sur Alpine — préférer nc (netcat)
docker compose exec php nc -zv database 3306
# Ou avec wget (souvent présent)
docker compose exec php wget -q --spider database:3306 2>&1 | head -2

# Voir l'IP de la machine hôte depuis le container
docker compose exec php cat /etc/hosts | grep host.docker.internal
```
