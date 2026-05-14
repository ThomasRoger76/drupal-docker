# DDEV — Alternative Clé en Main

## DDEV vs Docker Compose Custom — Quand Choisir

| Critère | DDEV | Docker Compose Custom |
|---------|------|-----------------------|
| Setup initial | ✅ 5 minutes | ⏱️ 2-4h |
| Flexibilité | 🟡 Bonne (extensible) | ✅ Totale |
| HTTPS local | ✅ Automatique | 🟡 Caddy manuel |
| Xdebug | ✅ `ddev xdebug on` | 🟡 Config manuelle |
| Mutagen (Mac/Windows) | ✅ Automatique | 🟡 mutagen-compose séparé |
| Multi-projet | ✅ Isolation native | 🟡 Gestion des ports |
| CI/CD | 🟡 Possible | ✅ Plus flexible |
| Contrôle des images | 🟡 Limité | ✅ Total |
| Utilisé dans les projets Digiwin | ❌ Pas observé | ✅ Oui (docker-compose.yml) |

**Recommandation :** DDEV pour les projets solo/équipes petites ou les nouveaux projets. Docker Compose custom pour les projets avec besoins spécifiques (Memcached, CI custom, images sur registre privé).

---

## Installation et Configuration DDEV

```bash
# Installation (macOS)
brew install ddev/ddev/ddev

# Installation (Linux)
curl -fsSL https://ddev.com/install.sh | bash

# Vérifier l'installation
ddev version
```

---

## Initialiser un Projet Drupal avec DDEV

```bash
# Depuis la racine du projet Drupal
cd /home/thomasroger/Projets/Drupal/mon-projet

# Configurer DDEV pour Drupal
ddev config \
  --project-name=mon-projet \
  --project-type=drupal10 \
  --docroot=web \
  --php-version=8.3 \
  --database=mariadb:11.0 \
  --webserver-type=apache-fpm

# Démarrer
ddev start

# Installer les dépendances
ddev composer install

# Installer Drupal
ddev drush site:install --existing-config -y

# Accéder au site (HTTPS automatique !)
ddev launch
```

---

## Fichier `.ddev/config.yaml` — Configuration Complète

```yaml
# .ddev/config.yaml
name: mon-projet
type: drupal10
docroot: web
php_version: "8.3"
webserver_type: apache-fpm
database:
  type: mariadb
  version: "11.0"

# Performance (Mac/Windows) — Mutagen automatique
mutagen_enabled: true

# Services additionnels
# (voir ddev get pour les add-ons disponibles)

# Ports (optionnel — DDEV gère automatiquement)
router_http_port: "80"
router_https_port: "443"

# Variables PHP custom
web_environment:
  - COMPOSER_MEMORY_LIMIT=2G

# Hooks
hooks:
  post-start:
    - exec: "drush cr"

  post-import-db:
    - exec: "drush cr"
    - exec: "drush updb -y"
    - exec: "drush cim -y"
```

---

## Commandes DDEV Essentielles

```bash
# Cycle de vie
ddev start          # Démarrer
ddev stop           # Arrêter
ddev restart        # Redémarrer
ddev poweroff       # Arrêter TOUS les projets DDEV
ddev delete -y      # Supprimer le projet (volumes compris)

# Développement
ddev drush cr                    # Cache rebuild
ddev drush cex -y                # Export config
ddev drush cim -y                # Import config
ddev drush sql:dump > dump.sql   # Dump DB
ddev import-db --file=dump.sql   # Import DB

# Composer
ddev composer install
ddev composer require drupal/paragraphs
ddev composer update drupal/core-recommended --with-dependencies

# Shell dans le container web
ddev ssh                     # Shell interactif
ddev exec "php -i | grep memory"

# Xdebug
ddev xdebug on               # Activer Xdebug
ddev xdebug off              # Désactiver
ddev xdebug status           # Voir l'état

# Base de données
ddev mysql                   # Shell MySQL interactif
ddev import-db --file=dump.sql
ddev export-db --file=dump.sql

# Logs
ddev logs                    # Logs en temps réel
ddev logs -s web             # Logs du container web
ddev logs -s db              # Logs de la DB

# Informations
ddev describe                # Résumé du projet (URLs, services)
ddev list                    # Tous les projets DDEV actifs
```

---

## Add-ons DDEV — Services Additionnels

```bash
# Voir les add-ons disponibles
ddev get --list

# Solr (Search API)
ddev get ddev/ddev-solr
ddev restart
ddev drush en search_api_solr -y

# Redis
ddev get ddev/ddev-redis
ddev restart

# PhpMyAdmin
ddev get ddev/ddev-phpmyadmin
ddev restart
ddev launch -p   # Ouvrir phpMyAdmin

# MailHog (capture emails)
# Inclus par défaut dans DDEV — accessible sur port 8025
ddev launch -m   # Ouvrir MailHog

# ChromeDriver (tests FunctionalJavascript)
ddev get ddev/ddev-selenium-standalone-chrome
ddev restart

# Elasticsearch / OpenSearch
ddev get ddev/ddev-opensearch
```

---

## Intégration settings.php avec DDEV

DDEV injecte automatiquement la configuration DB. Mais pour les projets qui utilisent `getenv()` dans settings.php :

```php
// web/sites/default/settings.php — compatible DDEV et Docker Compose custom
$databases['default']['default'] = [
  'driver'   => 'mysql',
  'host'     => getenv('MARIADB_HOSTNAME') ?: getenv('DDEV_DATABASE_HOSTNAME') ?: 'database',
  'port'     => getenv('MARIADB_PORT') ?: '3306',
  'database' => getenv('MARIADB_DATABASE') ?: getenv('DDEV_DATABASE') ?: 'drupal',
  'username' => getenv('MARIADB_USER') ?: getenv('DDEV_DATABASE_USER') ?: 'drupal',
  'password' => getenv('MARIADB_PASSWORD') ?: getenv('DDEV_DATABASE_PASSWORD') ?: 'drupal',
  'prefix'   => '',
  'charset'  => 'utf8mb4',
];
```

---

## DDEV — Xdebug Configuration IDE

### VS Code

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "DDEV Xdebug",
      "type": "php",
      "request": "launch",
      "port": 9003,
      "pathMappings": {
        "/var/www/html": "${workspaceFolder}"
      }
    }
  ]
}
```

```bash
# Activer et démarrer le debug
ddev xdebug on
# Démarrer l'écoute dans VS Code
# Ouvrir le site → breakpoints actifs
ddev xdebug off  # Désactiver après usage (impact sur les performances)
```

### PhpStorm

DDEV configure automatiquement les path mappings avec PhpStorm via le plugin "DDEV Integration".

```bash
# Installer le plugin PhpStorm DDEV Integration depuis JetBrains Marketplace
# Puis : outils → DDEV → Attach Debugger
```

---

## Migrer de Docker Compose Custom vers DDEV

Si tu as déjà un projet avec `docker-compose.yml` custom :

```bash
# 1. Arrêter les containers custom
docker compose down

# 2. Initialiser DDEV (depuis la racine du projet)
ddev config --project-name=mon-projet --project-type=drupal10 --docroot=web

# 3. Copier les variables importantes dans .ddev/config.yaml
# (DB credentials, version PHP, etc.)

# 4. Démarrer DDEV
ddev start

# 5. Importer la DB existante
ddev import-db --file=.docker/services/database/dumps/dump.sql

# 6. Vérifier
ddev drush status
ddev launch
```
