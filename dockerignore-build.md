# `.dockerignore` & Optimisation des Builds

## Pourquoi `.dockerignore` est Critique

Sans `.dockerignore`, `docker compose build` envoie le **contexte de build entier** au daemon Docker avant de construire l'image. Sur un projet Drupal avec vendor et node_modules, ça représente souvent **500MB à 2GB** de données transférées à chaque build.

```bash
# Voir la taille du contexte de build sans .dockerignore
docker build --no-cache . 2>&1 | head -3
# → Sending build context to Docker daemon  847.3MB   ← PROBLÈME
```

---

## `.dockerignore` — Template Projet Drupal

```dockerignore
# .dockerignore — à la racine du projet (même niveau que docker-compose.yml)

# Contrôle de version
.git
.gitignore
.gitattributes

# Dépendances gérées dans le container
vendor/
node_modules/

# Fichiers Docker (pas besoin dans l'image)
.docker/

# Config environment (ne jamais dans une image)
.env
.env.dist
.env.*

# Base de données
*.sql
*.dump

# Fichiers générés / cache
web/sites/default/files/
private/
config/

# Outils de développement
.docker compose exec php/
.idea/
.vscode/

# Fichiers de documentation
docs/
*.md
CHANGELOG*
README*

# Tests (pour les images de prod uniquement — retirer si image de test)
tests/
web/modules/custom/*/tests/

# Logs
*.log

# OS artifacts
.DS_Store
Thumbs.db
```

**Vérifier l'effet :**
```bash
# Avec .dockerignore
docker build . --no-cache 2>&1 | head -3
# → Sending build context to Docker daemon  12.34MB   ← BEAUCOUP MIEUX
```

---

## Dockerfile Multi-Stage — Production vs Développement

### Le Problème des Images All-in-One

L'image de dev contient Composer, SSH, Xdebug, outils de debug — **jamais en production**. Le pattern multi-stage résout ça.

```dockerfile
# .docker/services/apache-php-base/Dockerfile

# ──────────────────────────────────────────────────────────────────────────────
# STAGE 1 : base — extensions PHP + Apache (commun prod et dev)
# ──────────────────────────────────────────────────────────────────────────────
ARG PHP_VERSION=8.3
FROM php:${PHP_VERSION}-apache AS base

# Apache — DocRoot vers /web
ENV APACHE_DOCUMENT_ROOT=/var/www/html/web
RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' \
      /etc/apache2/sites-available/*.conf \
      /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf && \
    sed -ri -e 's!AllowOverride None!AllowOverride All!g' \
      /etc/apache2/apache2.conf && \
    a2enmod rewrite remoteip

# Extensions PHP Drupal
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends \
        libzip-dev libicu-dev libpng-dev libjpeg-dev libwebp-dev \
        libfreetype6-dev default-mysql-client && \
    rm -rf /var/lib/apt/lists/*

RUN docker-php-ext-configure gd --with-jpeg --with-webp --with-freetype && \
    docker-php-ext-install -j "$(nproc)" gd pdo_mysql mysqli zip intl opcache

RUN pecl install apcu && docker-php-ext-enable apcu && docker-php-source delete

COPY php-additionnal.ini /usr/local/etc/php/conf.d/

# ──────────────────────────────────────────────────────────────────────────────
# STAGE 2 : prod — Composer install des dépendances, OPcache agressif
# ──────────────────────────────────────────────────────────────────────────────
FROM base AS production

# Composer pour installer les dépendances (seulement pendant le build)
COPY --from=composer:2.7 /usr/bin/composer /usr/local/bin/composer

WORKDIR /var/www/html

# 1. Copier les manifestes Composer d'abord (cache Docker optimisé)
COPY composer.json composer.lock ./

# 2. Installer sans les dépendances de dev
RUN composer install \
    --no-dev \
    --no-scripts \
    --no-plugins \
    --optimize-autoloader \
    --prefer-dist

# 3. Copier le code source APRÈS l'install Composer
COPY web/ ./web/
COPY config/ ./config/

# Retirer Composer de l'image finale
RUN rm /usr/local/bin/composer

# OPcache production — ne jamais revalider
COPY php-production.ini /usr/local/etc/php/conf.d/php-production.ini

# ──────────────────────────────────────────────────────────────────────────────
# STAGE 3 : dev — Ajouter outils de développement
# ──────────────────────────────────────────────────────────────────────────────
FROM base AS development

# Composer global disponible en dev
COPY --from=composer:2.7 /usr/bin/composer /usr/local/bin/composer

# Xdebug uniquement en dev
RUN pecl install xdebug && docker-php-ext-enable xdebug

# OPcache dev — revalider à chaque changement
COPY php-dev.ini /usr/local/etc/php/conf.d/php-dev.ini

WORKDIR /var/www/html
# Code source monté via volume (bind mount) — pas copié dans l'image dev
```

### `php-production.ini` — Pas de revalidation

```ini
; php-production.ini
opcache.enable = 1
opcache.memory_consumption = 256
opcache.max_accelerated_files = 20000
opcache.revalidate_freq = 0
opcache.validate_timestamps = 0   ; ← JAMAIS en prod (gain majeur de perf)
opcache.save_comments = 1
```

### `php-dev.ini` — Revalidation à chaque requête

```ini
; php-dev.ini
opcache.enable = 1
opcache.validate_timestamps = 1   ; ← Revalider (voir les changements sans drush cr)
opcache.revalidate_freq = 0       ; ← À chaque requête

[xdebug]
xdebug.mode = debug
xdebug.start_with_request = yes
xdebug.client_host = host.docker.internal
xdebug.client_port = 9003
xdebug.log = /tmp/xdebug.log
```

### Utiliser le bon stage dans docker-compose

```yaml
# .docker/docker-compose.yml (dev)
services:
  php:
    build:
      context: services/apache-php-base
      target: development   # ← Stage dev avec Xdebug + Composer
      args:
        PHP_VERSION: ${PHP_VERSION:-8.3}
    volumes:
      - ../:/var/www/html   # Code source en bind mount

# .docker/docker-compose.prod.yml (prod)
services:
  php:
    build:
      context: services/apache-php-base
      target: production    # ← Stage prod — pas d'outils dev
      args:
        PHP_VERSION: ${PHP_VERSION:-8.3}
    # Pas de bind mount — code copié dans l'image
```

---

## `docker compose watch` — Alternative Moderne à Mutagen (D2.22+)

Depuis Docker Compose 2.22, le mode `watch` synchronise les fichiers automatiquement sans bind mount lent ni outil externe.

```yaml
# docker-compose.yml
services:
  php:
    develop:
      watch:
        # Synchroniser le code source PHP (pas vendor ni web/core)
        - action: sync
          path: ./web/modules/custom
          target: /var/www/html/web/modules/custom
          ignore:
            - tests/
        - action: sync
          path: ./web/themes/custom
          target: /var/www/html/web/themes/custom
        # Rebuild de l'image si Dockerfile ou composer.json change
        - action: rebuild
          path: ./composer.json
        - action: rebuild
          path: .docker/services/apache-php-base/Dockerfile
```

```bash
# Démarrer avec le watch actif
docker compose watch

# Ou en arrière-plan
docker compose up --watch -d
```

**Avantages vs Mutagen :**
- Natif Docker Compose — aucun outil externe
- Plus sélectif (sync uniquement ce qui a changé)
- Rebuild automatique si Dockerfile/composer.json change

**Limitations :**
- Sync unidirectionnel (hôte → container uniquement)
- Pas adapté si on crée des fichiers dans le container (drush, composer install)
