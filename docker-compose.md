# Docker Compose — Structure Complète

## Fichier Complet — Template Projet Drupal (Pattern Réel)

```yaml
# .docker/docker-compose.yml

services:

  # ── PHP + Apache (all-in-one) ────────────────────────────────────────
  php:
    image: ${REGISTRY:-mon-registry}/drupal-php-base-dev:${PHP_VERSION:-8.3}
    build:
      context: services/apache-php-base
      args:
        PHP_VERSION: ${PHP_VERSION:-8.3}
    ports:
      - "${HTTPD_PORT:-80}:80"
    volumes:
      # Racine du projet montée dans le container
      - ../:/var/www/html
      # Config PHP custom (override du php.ini par défaut)
      - ./services/php/conf/php.ini:/usr/local/etc/php/conf.d/php-additionnal.ini
      # SSH de l'hôte → dans le container (pour composer avec dépôts privés)
      - ${PERSONAL_SSH_FOLDER:-~/.ssh}:/home/devuser/.ssh:ro
      # Cache Composer de l'hôte → pas de re-download à chaque container
      - ${PERSONAL_GLOBAL_COMPOSER_FOLDER:-~/.composer}:/home/devuser/.composer
      # Named volume pour composer tmp (builds plus rapides)
      - composer_home:/tmp
    environment:
      # Alignement UID/GID avec l'utilisateur hôte (anti-Permission denied)
      HOST_UID: ${DEV_UID:-1000}
      HOST_GID: ${DEV_GID:-1000}
      COMPOSER_MEMORY_LIMIT: 2G
      COMPOSER_PROCESS_TIMEOUT: 10000
      DRUSH_LAUNCHER_FALLBACK: /var/www/html/vendor/bin/drush
      # DB — nom du service Docker, pas localhost !
      MARIADB_DATABASE: ${BDD_MARIADB_DATABASE}
      MARIADB_USER: ${BDD_MARIADB_USER}
      MARIADB_PASSWORD: ${BDD_MARIADB_PASSWORD}
      MARIADB_HOSTNAME: database   # ← Nom du service, résolu par DNS Docker
      MARIADB_PORT: ${MARIADB_PORT:-3306}
    extra_hosts:
      # Permet à Xdebug dans le container de joindre l'IDE sur la machine hôte
      - "host.docker.internal:host-gateway"
    depends_on:
      - database

  # ── MariaDB ──────────────────────────────────────────────────────────
  database:
    image: mariadb:11.4   # LTS (support jusqu'en 2029) — épingler un tag mineur, pas 'latest'
    ports:
      - "${MARIADB_PORT:-3306}:3306"
    environment:
      - MARIADB_ROOT_PASSWORD=${BDD_MARIADB_ROOT_PASSWORD}
      - MARIADB_DATABASE=${BDD_MARIADB_DATABASE}
      - MARIADB_USER=${BDD_MARIADB_USER}
      - MARIADB_PASSWORD=${BDD_MARIADB_PASSWORD}
    volumes:
      # Named volume = persistance des données entre redémarrages
      - database_data:/var/lib/mysql
      # Bind mount = accès aux dumps SQL depuis l'hôte ET le container
      - ./services/database/dumps:/dumps/
    # Santé de la DB pour depends_on
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── PhpMyAdmin ───────────────────────────────────────────────────────
  pma:
    image: phpmyadmin/phpmyadmin:latest
    ports:
      - "${PHPMYADMIN_PORT:-82}:80"
    environment:
      - PMA_HOST=database
      - PMA_USER=root
      - PMA_PASSWORD=${BDD_MARIADB_ROOT_PASSWORD}

  # ── Maildev (capture mails en dev) ───────────────────────────────────
  maildev:
    image: maildev/maildev
    ports:
      - "${MAILDEV_PORT:-84}:1080"   # Interface web
    # SMTP interne : maildev:1025 (depuis PHP/Drupal dans le réseau Docker)

  # ── Node.js — Compilation du thème ───────────────────────────────────
  webpack_theming:
    image: node:22-alpine
    working_dir: /app/web/sites/default/themes/mon_theme
    command: npm run watch
    volumes:
      - ../:/app   # Toute la racine montée pour accéder à package.json

# ── Volumes nommés ────────────────────────────────────────────────────
volumes:
  composer_home:    # Cache Composer partagé entre runs
  database_data:    # Données MariaDB persistantes
```

---

## Stratégie Multi-Fichiers — dev / CI

### Pattern Digiwin (réel)

```
.docker/
├── docker-compose.yml         # Services de base (sans ports)
├── docker-compose.dev.yml     # Override dev (ports exposés, volumes SSH)
└── ci/
    └── docker-compose.yml     # Services CI (pas de ports, image source-code)
```

```yaml
# .env — choisir les fichiers Compose à utiliser
COMPOSE_FILE=.docker/docker-compose.yml:.docker/docker-compose.dev.yml
```

### Override dev — Ajouter des ports et volumes locaux

```yaml
# .docker/docker-compose.dev.yml
services:
  php:
    volumes:
      - ./services/php/conf/php-dev.ini:/usr/local/etc/php/conf.d/php-additionnal.ini
    environment:
      - XDEBUG_MODE=debug   # Activer Xdebug seulement en dev
  database:
    command: --max_allowed_packet=32505856
    ports:
      - "${MARIADB_PORT}:3306"
  httpd:
    ports:
      - "${HTTPD_PORT}:80"
      - "${HTTPD_SECURE_PORT}:443"
```

### CI docker-compose — Sans ports, avec image source-code

```yaml
# .docker/ci/docker-compose.yml
services:
  # Image contenant uniquement le code source (sans DB, sans ports exposés)
  source-code:
    image: ${CI_REGISTRY_IMAGE}/source-code:${CI_COMMIT_SHA}
    build:
      dockerfile: .docker/ci/source-code/Dockerfile
      context: ../

  # PHP sans ports exposés (communication interne uniquement)
  drupal-apache-php:
    image: ${CI_REGISTRY_IMAGE}/drupal-apache-php:11-8.3
    build:
      context: drupal-apache-php
    depends_on:
      - memcached

  memcached:
    image: memcached:1.6-alpine
    command: ["-m", "512"]
```

---

## Volumes — Stratégie Complète

| Type | Exemple | Usage | Perte si `down -v` ? |
|------|---------|-------|---------------------|
| Named volume | `database_data:/var/lib/mysql` | DB, données persistantes | ✅ Oui (attention !) |
| Bind mount absolu | `../:/var/www/html` | Code source (dev) | ❌ Non (sur l'hôte) |
| Bind mount relatif | `./services/database/dumps:/dumps` | Dumps SQL | ❌ Non |
| Named volume partagé | `composer_home:/tmp` | Cache Composer | ✅ Oui (recréé auto) |
| Volume SSH en readonly | `~/.ssh:/home/devuser/.ssh:ro` | Clés SSH | ❌ Non |

**Règle :** `docker compose down` = OK. `docker compose down -v` = supprime les named volumes = perte DB.

---

## Commandes Essentielles

```bash
# Démarrer tous les services
docker compose up -d

# Démarrer un service spécifique
docker compose up -d php database

# Stopper (sans perdre les données)
docker compose down

# ⚠️ Stopper ET supprimer les volumes (perte DB)
docker compose down -v

# Reconstruire les images
docker compose build --no-cache php
docker compose up -d --build php

# Voir les containers en cours
docker compose ps

# Redémarrer un service
docker compose restart php

# Exécuter une commande dans le container en cours
docker compose exec php bash
docker compose exec php drush cr

# Exécuter dans un container éphémère (--rm = supprimé après)
docker compose run --rm php drush site:install
docker compose run --rm php composer install

# Voir les logs en temps réel
docker compose logs php -f
docker compose logs database -f

# Voir les logs des 100 dernières lignes
docker compose logs --tail=100 php
```

---

## `depends_on` — Ordre de Démarrage

```yaml
services:
  php:
    depends_on:
      database:
        condition: service_healthy   # Attendre que la DB soit vraiment prête
      maildev:
        condition: service_started   # Attendre juste que le container soit démarré
```

Sans `depends_on`, PHP peut démarrer avant que la DB soit prête → erreur de connexion.

---

## PostgreSQL — Alternative à MariaDB

### Service PostgreSQL

```yaml
# .docker/docker-compose.yml
services:

  php:
    image: ${REGISTRY}/drupal-php-base-dev:${PHP_VERSION:-8.3}
    environment:
      # Variables PostgreSQL — préfixe différent de MariaDB
      POSTGRES_DB: ${BDD_POSTGRES_DATABASE}
      POSTGRES_USER: ${BDD_POSTGRES_USER}
      POSTGRES_PASSWORD: ${BDD_POSTGRES_PASSWORD}
      POSTGRES_HOSTNAME: postgres   # ← Nom du service, résolu par DNS Docker
      POSTGRES_PORT: ${POSTGRES_PORT:-5432}
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16   # Tag majeur stable — Drupal 11 supporte PostgreSQL 16
    ports:
      - "${POSTGRES_PORT:-5432}:5432"
    volumes:
      - ./services/postgres/dump:/dump      # Dumps SQL accessibles depuis le container
      - pgdata:/var/lib/postgresql/data     # Données persistantes
    environment:
      - PGDATA=/var/lib/postgresql/data
      - POSTGRES_PASSWORD=${BDD_POSTGRES_PASSWORD}
      - POSTGRES_USER=${BDD_POSTGRES_USER}
      - POSTGRES_DB=${BDD_POSTGRES_DATABASE}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${BDD_POSTGRES_USER} -d ${BDD_POSTGRES_DATABASE}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Adminer — interface web pour PostgreSQL ET MariaDB
  adminer:
    image: adminer:latest
    ports:
      - "${ADMINER_PORT:-8080}:8080"
    environment:
      - ADMINER_DEFAULT_SERVER=postgres   # Pointer vers PostgreSQL par défaut

volumes:
  pgdata:   # Données PostgreSQL persistantes
```

### .env.dist avec variables PostgreSQL

```bash
###########################
# PostgreSQL
###########################
BDD_POSTGRES_DATABASE=drupal
BDD_POSTGRES_USER=drupal
BDD_POSTGRES_PASSWORD=drupal
POSTGRES_PORT=5432

###########################
# Interface DB (Adminer supporte PostgreSQL, contrairement à PhpMyAdmin)
###########################
ADMINER_PORT=8080
```

### settings.php pour PostgreSQL

```php
// web/sites/default/settings.php — driver pgsql
$databases['default']['default'] = [
  'driver'    => 'pgsql',
  'namespace' => 'Drupal\\pgsql\\Driver\\Database\\pgsql',
  'autoload'  => 'core/modules/pgsql/src/Driver/Database/pgsql/',
  'host'      => getenv('POSTGRES_HOSTNAME') ?: 'postgres',
  'port'      => getenv('POSTGRES_PORT') ?: '5432',
  'database'  => getenv('POSTGRES_DB') ?: 'drupal',
  'username'  => getenv('POSTGRES_USER') ?: 'drupal',
  'password'  => getenv('POSTGRES_PASSWORD') ?: 'drupal',
  'prefix'    => '',
  // Pas de charset/collation ici — PostgreSQL les gère à la création de la DB
];
```

### Différences PostgreSQL vs MariaDB dans Drupal

| Aspect | MariaDB | PostgreSQL |
|--------|---------|------------|
| Driver Drupal | `mysql` | `pgsql` |
| Namespace | `Drupal\\mysql\\...` | `Drupal\\pgsql\\...` |
| Variable hôte | `MARIADB_HOSTNAME` | `POSTGRES_HOSTNAME` |
| Port par défaut | 3306 | 5432 |
| Interface web | PhpMyAdmin ou Adminer | **Adminer uniquement** (PMA ne supporte pas PSQL) |
| Format dump | `.sql` via `mariadb-dump` | `.sql` via `pg_dump` |
| Import dump | `mariadb < dump.sql` | `psql < dump.sql` |
| Commande dump container | `docker compose exec postgres pg_dump -U user db > dump.sql` | idem |
| Commande import container | `docker compose exec -T postgres psql -U user db < dump.sql` | idem |
| `--` commentaires SQL | ✅ compatible | ✅ compatible |
| Extensions Drupal core | `mysql` module | `pgsql` module (inclus dans core) |

### Dump et restauration PostgreSQL

```bash
# Créer un dump
docker compose exec postgres \
  pg_dump -U ${BDD_POSTGRES_USER} ${BDD_POSTGRES_DATABASE} \
  > .docker/services/postgres/dump/dump-$(date +%Y%m%d-%H%M).sql

# Restaurer un dump
docker compose exec -T postgres \
  psql -U ${BDD_POSTGRES_USER} ${BDD_POSTGRES_DATABASE} \
  < .docker/services/postgres/dump/dump-20260514-1200.sql

# Vérifier la connexion
docker compose exec postgres \
  psql -U ${BDD_POSTGRES_USER} -d ${BDD_POSTGRES_DATABASE} -c "\dt"
```

### Extension PHP pdo_pgsql dans le Dockerfile

```dockerfile
# Ajouter dans le Dockerfile PHP
RUN apt-get update -y && \
    apt-get install -y libpq-dev && \
    rm -rf /var/lib/apt/lists/*

RUN docker-php-ext-install -j "$(nproc)" pdo_pgsql pgsql
```

---

## Isolation entre Projets — `COMPOSE_PROJECT_NAME`

```bash
# Sans COMPOSE_PROJECT_NAME (Docker Compose v2 — tirets, pas underscores)
# Containers nommés : drupal-php-1, drupal-database-1 → conflits entre projets !

# Avec COMPOSE_PROJECT_NAME dans .env
COMPOSE_PROJECT_NAME=ccilemans
# Containers : ccilemans-php-1, ccilemans-database-1

# Afficher les containers avec leur nom
docker compose ps   # ← affiche les noms au format PROJECT-SERVICE-INDEX

# Ports différents par projet pour éviter les conflits :
# projet-a: HTTPD_PORT=80, MARIADB_PORT=3306
# projet-b: HTTPD_PORT=81, MARIADB_PORT=3307
# projet-c: HTTPD_PORT=82, MARIADB_PORT=3308
```
