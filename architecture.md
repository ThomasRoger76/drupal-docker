# Architecture Docker pour Drupal

## Services Indispensables

```
┌─────────────────────────────────────────────────────────────┐
│  Navigateur → Caddy/Apache (80/443) → PHP (Apache intégré)  │
│                         ↓                                     │
│               MariaDB (3306) ← volumes                       │
│                                                               │
│  Optionnel: Maildev (1080) · Memcached · Node/Webpack        │
└─────────────────────────────────────────────────────────────┘
```

---

## 3 Patterns d'Architecture — Comparatif Réel

### Option A — PHP+Apache tout-en-un (recommandé pour projets simples)

```yaml
services:
  php:
    image: mon-registry/drupal-php-base:8.3
    # OU directement : image: php:8.3-apache
    ports:
      - "80:80"
    volumes:
      - ../:/var/www/html
```

Avantages : 1 seul container à gérer, simpler pour les équipes. Idéal pour les projets où le même container sert le PHP ET expose Apache.

### Option B — PHP-FPM + Apache séparés (projets nécessitant scaling indépendant)

```yaml
services:
  php:    # PHP-FPM uniquement
    image: mon-registry/php-fpm:8.2
    # Pas de port exposé — communication interne via réseau Docker
  httpd:  # Apache ou Nginx — reverse proxy vers php:9000
    image: mon-registry/httpd:2.4
    ports:
      - "80:80"
      - "443:443"
```

Avantages : scaling indépendant, configs séparées, plus proche de la prod.

### Option C — PHP+Apache + Caddy en front (recommandé pour HTTPS local automatique)

```yaml
services:
  php:    # php:VERSION-apache — sert PHP ET Apache sur port 80 interne
    image: mon-registry/drupal-php:8.3
    # Pas de port exposé directement
  caddy:  # TLS terminaison automatique
    image: caddy:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./services/caddy/Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - php
```

```
# Caddyfile — TLS local automatique (certificat auto-signé de confiance)
projet.local www.projet.local {
  tls internal          # Génère un certificat local approuvé par le système
  reverse_proxy php:80  # Forward vers le container PHP
}
```

Pour que le domaine `.local` soit résolu :
```bash
# /etc/hosts (une fois par projet)
127.0.0.1 projet.local www.projet.local
```

---

## Dockerfile PHP+Apache pour Drupal — Template Production

```dockerfile
# .docker/services/apache-php-base/Dockerfile
ARG PHP_VERSION=8.3
FROM php:${PHP_VERSION}-apache

# ── Apache Configuration ────────────────────────────────────
ENV APACHE_DOCUMENT_ROOT=/var/www/html/web

# Pointer le DocumentRoot vers /web (le webroot Drupal)
RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' \
      /etc/apache2/sites-available/*.conf && \
    sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' \
      /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf && \
    sed -ri -e 's!AllowOverride None!AllowOverride All!g' \
      /etc/apache2/apache2.conf  # CRITIQUE pour Drupal .htaccess

# Modules Apache requis
RUN a2enmod rewrite remoteip

# ── Extensions PHP requises par Drupal ──────────────────────
# Installer les dépendances système en un seul layer
RUN apt-get update -y && \
    apt-get install --fix-missing -y \
        libzip-dev \
        libicu-dev \
        libonig-dev \
        libpng-dev \
        libjpeg-dev \
        libwebp-dev \
        libfreetype6-dev \
        default-mysql-client \
    && rm -rf /var/lib/apt/lists/*

# GD (images) — configure avant install
RUN docker-php-ext-configure gd \
        --with-jpeg \
        --with-webp \
        --with-freetype && \
    docker-php-ext-install -j "$(nproc)" gd

# Extensions obligatoires Drupal
RUN docker-php-ext-install -j "$(nproc)" \
    pdo_mysql \
    mysqli \
    zip \
    intl \
    opcache

# APCu (cache PHP user space)
RUN pecl install apcu && \
    docker-php-ext-enable apcu && \
    docker-php-source delete

# Composer (depuis l'image officielle)
COPY --from=composer:2.7 /usr/bin/composer /usr/local/bin/composer

# PHP config de base
COPY php-additionnal.ini /usr/local/etc/php/conf.d/php-additionnal.ini
```

```ini
# php-additionnal.ini
memory_limit = 512M
max_execution_time = 300
upload_max_filesize = 64M
post_max_size = 64M

; OPcache (prod)
opcache.enable = 1
opcache.memory_consumption = 256
opcache.max_accelerated_files = 20000
opcache.revalidate_freq = 0   ; 0 = jamais (prod) — 2 en dev

; APCu
apc.enabled = 1
apc.shm_size = 64M
```

---

## Services Additionnels — Templates

### MariaDB 11

```yaml
database:
  image: mariadb:11.4   # LTS (support jusqu'en 2029)
  ports:
    - "${MARIADB_PORT}:3306"
  environment:
    - MARIADB_ROOT_PASSWORD=${BDD_MARIADB_ROOT_PASSWORD}
    - MARIADB_DATABASE=${BDD_MARIADB_DATABASE}
    - MARIADB_USER=${BDD_MARIADB_USER}
    - MARIADB_PASSWORD=${BDD_MARIADB_PASSWORD}
  volumes:
    - database_data:/var/lib/mysql          # Persistance entre redémarrages
    - ./services/database/dumps:/dumps/     # Dumps SQL accessibles depuis le container
```

### Maildev (capture emails en dev)

```yaml
maildev:
  image: maildev/maildev
  ports:
    - "${MAILDEV_PORT}:1080"   # Interface web (ex: 84:1080)
  # Maildev expose aussi SMTP sur le port 1025 (interne au réseau Docker)
```

```php
// settings.local.php — rediriger tous les mails vers Maildev
$config['system.mail']['interface']['default'] = 'test_mail_collector';
// OU via SMTP module :
$config['smtp.settings']['smtp_host'] = 'maildev';  // nom du service Docker
$config['smtp.settings']['smtp_port'] = 1025;        // port SMTP interne
```

### Memcached (cache applicatif)

```yaml
memcached:
  image: memcached:1.6-alpine
  command: ["-m", "512"]   # 512MB de RAM dédiée
  ports:
    - "11211:11211"
```

```bash
# Prérequis : module contrib + extension PHP
composer require drupal/memcache
docker compose exec php php -m | grep memcache  # Vérifier que l'extension est installée
docker compose exec php drush en memcache -y
```

```php
// settings.php — activer Memcached comme backend de cache
// ⚠️ Requiert le module drupal/memcache ET l'extension PHP memcache/memcached
$settings['cache']['default'] = 'cache.backend.memcache';
$settings['memcache']['servers'] = ['memcached:11211' => 'default'];
// 'memcached' = nom du service Docker
```

### Node.js — Compilation du thème en watch mode

```yaml
webpack_theming:
  image: node:22-alpine
  working_dir: /app/web/sites/default/themes/mon_theme
  command: npm run watch
  volumes:
    - ../:/app      # Toute la racine du projet accessible
```

```bash
# Démarrer uniquement le container de theming
docker compose up webpack_theming -d

# Voir les logs de compilation
docker compose logs webpack_theming -f
```

### PhpMyAdmin

```yaml
pma:
  image: phpmyadmin/phpmyadmin:latest
  ports:
    - "${PHPMYADMIN_PORT}:80"   # ex: 82:80
  environment:
    - PMA_HOST=database     # Nom du service MariaDB
    - PMA_USER=root
    - PMA_PASSWORD=root
```

---

## Réseau Docker — Pourquoi `database` et pas `localhost`

Docker crée un réseau virtuel pour chaque projet (nommé `${COMPOSE_PROJECT_NAME}_default`). Dans ce réseau :
- `database` (ou `mysql`) → résolution du service DB par son nom
- `localhost` dans le container PHP = le container PHP lui-même, pas la DB
- `host.docker.internal` → machine hôte (pour Xdebug)

```
container php → réseau Docker → container database
"localhost:3306" ← FAUX
"database:3306" ← CORRECT
```

---

## Images — Registre Privé vs Docker Hub

```yaml
# Images officielles Docker Hub
php:
  image: php:8.3-apache       # Officielle, stable, à customiser

# Images d'un registre privé GitLab
php:
  image: registry.gitlab.example.com/drupal/mon-projet/drupal-php-base-dev:${PHP_VERSION}
  build:
    context: services/apache-php-base  # Build local si l'image n'existe pas
    args:
      PHP_VERSION: ${PHP_VERSION}
```

**`profiles:`** — Services construits uniquement sur commande :
```yaml
php-base:  # Image de base sans outils de dev
  image: registry/drupal-php-base:${PHP_VERSION}
  profiles:
    - build    # Uniquement activé avec: docker compose --profile build build php-base
```

---

## Sécurité — Exécution en Non-Root (Bonne Pratique Moderne)

Par défaut, les images `php:*-apache` tournent en `root` — risque de sécurité si le container est compromis.

### Dockerfile multi-stage avec user non-root

```dockerfile
FROM php:8.3-apache AS base

# Créer un utilisateur non-root avec UID/GID fixes
RUN groupadd --gid 1000 drupal && \
    useradd --uid 1000 --gid drupal --shell /bin/bash --create-home drupal

# Donner les droits Apache au user drupal
RUN chown -R drupal:drupal /var/www/html && \
    chown -R drupal:drupal /var/lock/apache2 /var/run/apache2 /var/log/apache2

# Copier les fichiers en tant que user non-root
COPY --chown=drupal:drupal . /var/www/html/

USER drupal

FROM base AS development
# En dev, revenir en root pour installer Xdebug, etc.
USER root
RUN pecl install xdebug && docker-php-ext-enable xdebug
USER drupal

FROM base AS production
# En prod : user non-root garanti
```

### Compatibilité avec les volumes bind-mount

```yaml
# docker-compose.yml — passer les UID/GID de l'hôte
services:
  php:
    build:
      context: .
      args:
        USER_UID: ${DEV_UID:-1000}
        USER_GID: ${DEV_GID:-1000}
    user: "${DEV_UID:-1000}:${DEV_GID:-1000}"
```

```dockerfile
# Dans le Dockerfile — utiliser les args
ARG USER_UID=1000
ARG USER_GID=1000
RUN groupadd --gid ${USER_GID} drupal && \
    useradd --uid ${USER_UID} --gid drupal drupal
```

> **Note :** Sur Linux, faire correspondre `DEV_UID=$(id -u)` et `DEV_GID=$(id -g)` dans `.env` pour éviter les problèmes de permissions sur les fichiers montés.

---

## Docker BuildKit — Builds Accélérés

BuildKit est le nouveau moteur de build Docker (actif par défaut depuis Docker 23+). Il apporte : builds parallèles, cache mount, secrets sécurisés.

```bash
# Activer BuildKit (Docker < 23)
export DOCKER_BUILDKIT=1
docker build .

# OU configurer globalement dans /etc/docker/daemon.json
{
  "features": { "buildkit": true }
}
```

### Dockerfile avec Cache Mount (Composer)

```dockerfile
# syntax=docker/dockerfile:1
FROM php:8.3-apache AS base

# Cache Composer entre les builds — BEAUCOUP plus rapide
RUN --mount=type=cache,target=/root/.composer \
    composer install --no-dev --optimize-autoloader --no-interaction

# Cache APT entre les builds
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && apt-get install -y libzip-dev && rm -rf /var/lib/apt/lists/*
```

### BuildKit avec Docker Compose

```yaml
# docker-compose.yml
services:
  php:
    build:
      context: .
      dockerfile: Dockerfile
      # Cache layers entre builds
      cache_from:
        - registry.gitlab.example.com/drupal/php-base:latest
      args:
        BUILDKIT_INLINE_CACHE: "1"   # Inclure le cache dans l'image pushée
```

```bash
# Build avec docker buildx bake (parallèle multi-target)
docker buildx bake

# Inspecter le cache
docker buildx du
docker buildx prune --filter type=exec.cachemount
```
