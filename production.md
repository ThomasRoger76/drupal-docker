# Docker Production — Drupal 11

## Vue d'ensemble

Ce guide couvre les optimisations Docker spécifiques à la production : multi-stage builds, OPcache tuning, Traefik pour le multi-projets, configuration reverse proxy, et PHP-FPM.

---

## 1. Multi-stage Dockerfile

```dockerfile
# Dockerfile — Multi-stage pour dev vs prod
# Commande build :
#   dev  : docker build --target development -t monprojet:dev .
#   prod : docker build --target production  -t monprojet:prod .

# ─── STAGE BASE ───────────────────────────────────────────────────────────────
FROM php:8.3-apache AS base

# Extensions PHP requises par Drupal
RUN apt-get update && apt-get install -y \
    libpng-dev \
    libzip-dev \
    libicu-dev \
    libpq-dev \
    libwebp-dev \
    libjpeg-dev \
    && docker-php-ext-configure gd --with-webp --with-jpeg \
    && docker-php-ext-install \
       gd \
       zip \
       intl \
       pdo_mysql \
       pdo_pgsql \
       opcache \
       exif \
    && rm -rf /var/lib/apt/lists/*

# Apache : activer mod_rewrite et headers
RUN a2enmod rewrite headers expires

# Configuration Apache pour Drupal
COPY .docker/services/apache-php-base/conf/drupal.conf /etc/apache2/sites-available/000-default.conf

# OPcache de base (sera surchargé par dev ou prod)
COPY .docker/services/apache-php-base/conf/opcache.ini $PHP_INI_DIR/conf.d/opcache.ini

# ─── STAGE DEVELOPMENT ────────────────────────────────────────────────────────
FROM base AS development

# Xdebug uniquement en dev
RUN pecl install xdebug \
    && docker-php-ext-enable xdebug

COPY .docker/services/apache-php-base/conf/xdebug.ini $PHP_INI_DIR/conf.d/xdebug.ini
COPY .docker/services/apache-php-base/conf/php-dev.ini $PHP_INI_DIR/conf.d/php-dev.ini

# Composer en dev pour l'install/update
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

# ─── STAGE CI ─────────────────────────────────────────────────────────────────
FROM base AS ci

# Outils qualité : Composer pour les dépendances de test
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Créer un utilisateur avec le même UID que le runner CI
ARG HOST_UID=1000
RUN useradd -u $HOST_UID -g www-data -m devuser

WORKDIR /var/www/html

# ─── STAGE PRODUCTION ─────────────────────────────────────────────────────────
FROM base AS production

# Composer uniquement pour l'install --no-dev, puis supprimé
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Copier tout le code source
COPY . /var/www/html

# Installer les dépendances sans les packages dev
RUN cd /var/www/html \
    && composer install \
       --no-dev \
       --optimize-autoloader \
       --classmap-authoritative \
       --no-interaction \
       --no-progress \
    && rm -f /usr/bin/composer

# OPcache production — remplace opcache.ini de base
COPY .docker/services/apache-php-base/conf/opcache-prod.ini $PHP_INI_DIR/conf.d/opcache.ini

# Droits corrects
RUN chown -R www-data:www-data /var/www/html \
    && find /var/www/html -type d -exec chmod 755 {} \; \
    && find /var/www/html -type f -exec chmod 644 {} \;

WORKDIR /var/www/html
```

---

## 2. OPcache Production

```ini
; .docker/services/apache-php-base/conf/opcache-prod.ini
; ⚠ PRODUCTION UNIQUEMENT — ne jamais utiliser ces réglages en dev

opcache.enable=1
opcache.enable_cli=0

; Mémoire allouée au cache (adapter selon la taille de l'application)
opcache.memory_consumption=256

; Buffer pour les chaînes internées (noms de fonctions, constantes)
opcache.interned_strings_buffer=16

; Nombre max de fichiers PHP mis en cache
; Drupal + contrib : compter ~10 000 à 20 000 fichiers
opcache.max_accelerated_files=20000

; ⚠ CRITIQUE : désactiver la vérification des timestamps
; PHP ne vérifie plus les modifications de fichiers = gain de perfs
; CONSÉQUENCE : drush cr est OBLIGATOIRE après un déploiement
opcache.validate_timestamps=0
opcache.revalidate_freq=0

; Shutdown rapide
opcache.fast_shutdown=1

; OBLIGATOIRE pour Drupal — les annotations PHP sont des commentaires
opcache.save_comments=1

; Précharger les classes au démarrage (optionnel, Symfony/Drupal 11+)
; opcache.preload=/var/www/html/vendor/composer/preload.php
; opcache.preload_user=www-data
```

> Après chaque déploiement, `docker compose exec php drush cr` est obligatoire pour vider l'OPcache et les caches Drupal.

```ini
; .docker/services/apache-php-base/conf/opcache.ini
; BASE (dev) — vérification des timestamps activée

opcache.enable=1
opcache.enable_cli=0
opcache.memory_consumption=128
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=10000
opcache.validate_timestamps=1    ; ← Actif en dev pour voir les changements
opcache.revalidate_freq=0
opcache.save_comments=1
```

---

## 3. Traefik — Multi-projets simultanés sur un poste dev

```yaml
# ~/docker/traefik/docker-compose.yml
# Traefik partagé — démarrer une seule fois, tous les projets l'utilisent

services:
  traefik:
    image: traefik:v3.0
    restart: unless-stopped
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --providers.docker.network=traefik_proxy
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      # TLS local avec Let's Encrypt (prod) ou certificats self-signed (dev)
      - --certificatesresolvers.letsencrypt.acme.email=dev@monsite.com
      - --certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web
      # Dashboard Traefik (désactiver en prod)
      - --api.dashboard=true
      - --api.insecure=true
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"    # Dashboard Traefik sur http://localhost:8080
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_letsencrypt:/letsencrypt
    networks:
      - traefik_proxy

volumes:
  traefik_letsencrypt:

networks:
  traefik_proxy:
    name: traefik_proxy
    driver: bridge
```

```bash
# Démarrer Traefik (une seule fois, persistant)
cd ~/docker/traefik
docker compose up -d
```

```yaml
# Dans chaque projet — docker-compose.yml
services:
  php:
    build:
      context: .
      target: development
    networks:
      - default
      - traefik_proxy
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=traefik_proxy"
      - "traefik.http.routers.monprojet.rule=Host(`monprojet.localhost`)"
      - "traefik.http.routers.monprojet.entrypoints=web"
      - "traefik.http.services.monprojet.loadbalancer.server.port=80"
      # HTTPS local (avec certificat self-signed) :
      # - "traefik.http.routers.monprojet-secure.rule=Host(`monprojet.localhost`)"
      # - "traefik.http.routers.monprojet-secure.entrypoints=websecure"
      # - "traefik.http.routers.monprojet-secure.tls=true"

networks:
  default:
    driver: bridge
  traefik_proxy:
    external: true
    name: traefik_proxy
```

> Accès au site : `http://monprojet.localhost` (pas besoin de modifier `/etc/hosts`).
> Chaque projet utilise son propre nom de domaine `.localhost`.

---

## 4. Reverse proxy — trusted_host_patterns OBLIGATOIRE

```php
// settings.php — quand Drupal est derrière Nginx, Caddy, ou Traefik

// ⚠ Sans cette configuration :
// - trusted_host_patterns ne fonctionne pas correctement
// - Le Flood API (brute-force protection) utilise l'IP Docker au lieu de l'IP réelle du client
// - Les URLs générées (absolues) peuvent être incorrectes

$settings['reverse_proxy'] = TRUE;

// IPs des proxies autorisés (réseau Docker)
$settings['reverse_proxy_addresses'] = [
  '172.17.0.0/16',   // Réseau Docker bridge par défaut
  '172.18.0.0/16',   // Réseau Docker custom
  '10.0.0.0/8',      // Réseau Docker overlay (Swarm)
];

// Headers X-Forwarded-* à accepter
$settings['reverse_proxy_trusted_headers'] =
  \Symfony\Component\HttpFoundation\Request::HEADER_X_FORWARDED_FOR |
  \Symfony\Component\HttpFoundation\Request::HEADER_X_FORWARDED_HOST |
  \Symfony\Component\HttpFoundation\Request::HEADER_X_FORWARDED_PORT |
  \Symfony\Component\HttpFoundation\Request::HEADER_X_FORWARDED_PROTO;

// trusted_host_patterns — liste blanche des domaines acceptés
$settings['trusted_host_patterns'] = [
  '^monprojet\.localhost$',
  '^www\.monsite\.com$',
  '^monsite\.com$',
];
```

---

## 5. PHP-FPM tuning production

```ini
; /etc/php-fpm.d/www.conf (ou .docker/services/php-fpm/www.conf)

; Processus maître
[www]
user  = www-data
group = www-data

listen = 9000

; Gestionnaire de processus dynamique
; static = nombre fixe de workers (serveurs à forte charge stable)
; dynamic = workers variables entre min/max (recommandé)
; ondemand = workers créés à la demande (faible trafic)
pm = dynamic

; Nombre maximum de workers PHP-FPM
; Calculer : (RAM disponible pour PHP) / (mémoire par process)
; Ex : 1 Go / 20 Mo par process = 50 workers max
pm.max_children = 50

; Workers démarrés au lancement
pm.start_servers = 10

; Workers minimum en veille
pm.min_spare_servers = 5

; Workers maximum en veille
pm.max_spare_servers = 20

; Nombre de requêtes avant de recycler un worker
; Anti-memory-leak : redémarre le process après N requêtes
pm.max_requests = 500

; Timeout d'inactivité avant fermeture d'un worker (ondemand uniquement)
; pm.process_idle_timeout = 10s

; Log des requêtes lentes (> 5s)
slowlog = /var/log/php-fpm/slow.log
request_slowlog_timeout = 5s

; Variables d'environnement — transmettre à PHP-FPM
; Nécessaire si settings.php utilise getenv()
clear_env = no
```

```dockerfile
# Dockerfile PHP-FPM (si séparation Apache/PHP-FPM)
FROM php:8.3-fpm AS base
COPY .docker/services/php-fpm/www.conf /usr/local/etc/php-fpm.d/www.conf
```

---

## 6. Déploiement production — commandes

```bash
# Build image production
docker build --target production --tag monprojet:$(git rev-parse --short HEAD) .

# Démarrer les services
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Après déploiement : mise à jour DB + config + cache
docker compose exec php drush deploy -y
# = drush updb -y + drush cim -y + drush cr

# Vérifier que l'OPcache est vide (validate_timestamps=0 en prod)
docker compose exec php drush php:eval "opcache_reset(); print('OPcache vidé');"
```

---

## 7. PHP 8.3 JIT — Just-In-Time Compilation

> **Contexte :** Le JIT est disponible depuis PHP 8.0. Il améliore les performances des tâches CPU-intensives (migrations, imports, calculs). Pour les requêtes Drupal typiques (I/O-bound : DB + cache), le gain est souvent neutre ou marginal. **Toujours benchmarker avant d'activer en production.**

```ini
; .docker/services/apache-php-base/conf/opcache-prod.ini
; Ajouter à la section OPcache existante

; ─── JIT (PHP 8.0+) ───────────────────────────────────────────────────────────
; Buffer mémoire alloué au JIT — 0 = JIT désactivé
opcache.jit_buffer_size = 64M

; Mode JIT :
; 0    → Désactivé (valeur par défaut — recommandé si incertain)
; 1205 → Fonction (moins agressif, bonne baseline)
; 1235 → Tracing (recommandé si JIT activé)
; 1255 → Tracing + optimisations profondes (le plus agressif)
opcache.jit = 1255

; Pour désactiver le JIT sans toucher au buffer :
; opcache.jit = 0
```

### Quand activer le JIT

| Cas d'usage | JIT utile ? | Mode recommandé |
|-------------|------------|-----------------|
| Migrations Drupal (CSV, D7→D10) | ✅ Oui | `1255` |
| Imports batch (> 10 000 entités) | ✅ Oui | `1235` |
| Requêtes web Drupal standard | ⚠️ Neutre | `0` (désactivé) |
| Génération de sitemaps, exports XML | ✅ Possible | `1235` |
| API REST Drupal (JSON:API) | ⚠️ Neutre | `0` |

### Benchmark avant/après

```bash
# Mesurer une migration sans JIT
docker compose exec php sh -c "php -d opcache.jit=0 vendor/bin/drush migrate:import import_articles --batch-size=500"

# Mesurer avec JIT tracing
docker compose exec php sh -c "php -d opcache.jit=1255 -d opcache.jit_buffer_size=64M vendor/bin/drush migrate:import import_articles --batch-size=500"

# Vérifier que le JIT est bien actif
docker compose exec php php -r "print_r(opcache_get_status()['jit']);"
```

> **Anti-pattern :** activer `opcache.jit = 1255` en production sans benchmark. Sur certaines configurations, le JIT peut augmenter la consommation mémoire sans gain de performance notable sur les requêtes HTTP Drupal.

---

## 8. Loki + Promtail — Log Aggregation légère

Alternative à ELK, bien plus légère, idéale pour les stacks Docker de taille moyenne.

```yaml
# docker-compose.yml — ajout des services de log aggregation
# Ajouter dans la section services: existante

services:
  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - loki_data:/loki
    networks:
      - default

  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      # Logs du host (Apache, Nginx, PHP-FPM)
      - /var/log:/var/log:ro
      # Socket Docker pour la découverte automatique des containers
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./services/promtail/config.yml:/etc/promtail/config.yml:ro
    command: -config.file=/etc/promtail/config.yml
    depends_on:
      - loki
    networks:
      - default

  grafana:
    image: grafana/grafana:latest
    ports:
      - "${GRAFANA_PORT:-3000}:3000"
    volumes:
      - grafana_data:/var/grafana
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: "Viewer"
    depends_on:
      - loki
    networks:
      - default

volumes:
  loki_data:
  grafana_data:
```

```yaml
# services/promtail/config.yml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Scraper les logs des containers Docker dont le nom contient "php"
  - job_name: drupal-php
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: [__meta_docker_container_name]
        regex: '.*php.*'
        action: keep
      - source_labels: [__meta_docker_container_name]
        target_label: container
    pipeline_stages:
      # Parser les logs PHP (format : [DD-Mon-YYYY HH:MM:SS UTC] PHP Error...)
      - regex:
          expression: '^\[(?P<timestamp>[^\]]+)\] PHP (?P<level>\w+): (?P<message>.+)'
      - labels:
          level:

  # Scraper les logs Nginx/Apache (accès + erreurs)
  - job_name: drupal-nginx
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: [__meta_docker_container_name]
        regex: '.*(nginx|apache|caddy).*'
        action: keep
      - source_labels: [__meta_docker_container_name]
        target_label: container
```

### Accéder à Grafana et configurer la source Loki

```bash
# Lancer la stack complète
docker compose up -d loki promtail grafana

# Accéder à Grafana
# URL : http://localhost:3000 (ou ${GRAFANA_PORT})
# Datasource : Loki → URL = http://loki:3100

# Requête LogQL pour les erreurs PHP Drupal
# Dans Grafana → Explore → Loki :
# {container=~".*php.*"} |= "PHP Fatal" or "PHP Error"

# Voir les 100 dernières erreurs Drupal via CLI
docker compose exec loki logcli query '{container=~".*php.*"} |= "error"' --limit=100
```

> **Ressources :** Loki + Promtail consomment ~100-300 Mo RAM. Pour un poste de dev, utiliser `grafana/loki:2.9.0-amd64` et limiter la rétention dans `local-config.yaml` à 72h.

---

## 9. Pipeline GitLab CI — Build Image + Push Registry

Build de l'image Docker multi-stage dans GitLab CI, push vers le GitLab Container Registry, puis utilisation de l'image buildée pour les tests.

```yaml
# .gitlab-ci.yml — Build et push de l'image Docker vers GitLab Registry
variables:
  DOCKER_DRIVER: overlay2
  IMAGE_BASE: ${CI_REGISTRY_IMAGE}/drupal-php
  PHP_VERSION: "8.3"

stages:
  - build
  - test
  - deploy

# ─────────────────────────────────────────────────────────────────────────────
# STAGE BUILD — Construire et pousser les images vers le GitLab Registry
# ─────────────────────────────────────────────────────────────────────────────

# Build l'image de production et la push dans le registry
build:production-image:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    # Build multi-stage — target production uniquement
    - |
      docker build \
        --target production \
        --tag ${IMAGE_BASE}:${CI_COMMIT_SHA} \
        --tag ${IMAGE_BASE}:latest \
        --cache-from ${IMAGE_BASE}:latest \
        --build-arg PHP_VERSION=${PHP_VERSION} \
        --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
        --build-arg VCS_REF=${CI_COMMIT_SHA} \
        -f .docker/services/apache-php-base/Dockerfile \
        .
    - docker push ${IMAGE_BASE}:${CI_COMMIT_SHA}
    - docker push ${IMAGE_BASE}:latest
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_TAG'

# Build l'image de développement (avec Xdebug, PCOV, Composer)
build:dev-image:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - |
      docker build \
        --target development \
        --tag ${IMAGE_BASE}-dev:${CI_COMMIT_SHA} \
        --tag ${IMAGE_BASE}-dev:latest \
        --cache-from ${IMAGE_BASE}-dev:latest \
        --build-arg PHP_VERSION=${PHP_VERSION} \
        -f .docker/services/apache-php-base/Dockerfile \
        .
    - docker push ${IMAGE_BASE}-dev:${CI_COMMIT_SHA}
    - docker push ${IMAGE_BASE}-dev:latest
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_MERGE_REQUEST_ID'

# ─────────────────────────────────────────────────────────────────────────────
# STAGE TEST — Utiliser l'image buildée pour les tests
# ─────────────────────────────────────────────────────────────────────────────

# Utiliser l'image dev buildée pour PHPUnit
test:phpunit:
  stage: test
  image: ${IMAGE_BASE}-dev:${CI_COMMIT_SHA}
  needs:
    - job: build:dev-image
  services:
    - name: mariadb:11.0
      alias: database
  variables:
    MARIADB_ROOT_PASSWORD: root
    MARIADB_DATABASE: drupal_test
    MARIADB_USER: drupal
    MARIADB_PASSWORD: drupal
    SIMPLETEST_DB: mysql://drupal:drupal@database/drupal_test
    SIMPLETEST_BASE_URL: http://localhost
  before_script:
    # Les dépendances sont déjà dans l'image — uniquement les dépendances de test
    - composer install --no-progress --prefer-dist --optimize-autoloader
    - cp phpunit.xml.dist phpunit.xml
    # Attendre que MariaDB soit prête
    - until mysqladmin ping -h database --silent; do sleep 2; done
  script:
    - vendor/bin/phpunit web/modules/custom --no-coverage --log-junit phpunit.xml
  artifacts:
    reports:
      junit: phpunit.xml
    when: always
  rules:
    - if: '$CI_MERGE_REQUEST_ID'
    - if: '$CI_COMMIT_BRANCH == "main"'

# ─────────────────────────────────────────────────────────────────────────────
# STAGE DEPLOY — Déployer l'image production
# ─────────────────────────────────────────────────────────────────────────────

deploy:production:
  stage: deploy
  image: docker:latest
  needs:
    - job: build:production-image
    - job: test:phpunit
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    # Tirer l'image buildée (pas de rebuild)
    - docker pull ${IMAGE_BASE}:${CI_COMMIT_SHA}
    # Déployer via SSH sur le serveur de production
    - |
      ssh -o StrictHostKeyChecking=no deploy@${PROD_SERVER} "
        docker pull ${IMAGE_BASE}:${CI_COMMIT_SHA}
        docker compose -f /srv/drupal/docker-compose.prod.yml up -d
        docker compose -f /srv/drupal/docker-compose.prod.yml exec -T php drush deploy -y
      "
  rules:
    - if: '$CI_COMMIT_TAG'
      when: manual
  environment:
    name: production
    url: https://${PROD_DOMAIN}
```

### Variables GitLab CI requises

```bash
# À configurer dans GitLab → Settings → CI/CD → Variables
CI_REGISTRY_USER   # Automatique (GitLab CI)
CI_REGISTRY_PASSWORD  # Automatique (GitLab CI)
CI_REGISTRY        # Automatique = registry.gitlab.com
PROD_SERVER        # IP ou hostname du serveur de prod
PROD_DOMAIN        # Domaine de production (ex: monsite.com)
```

### Utilisation locale — reproduire le build CI

```bash
# Construire l'image production localement (identique au CI)
docker build \
  --target production \
  --tag drupal-php:local \
  -f .docker/services/apache-php-base/Dockerfile \
  .

# Vérifier l'image construite
docker run --rm drupal-php:local php -v

# Nettoyer les anciennes images du registry local
docker image prune -f --filter "dangling=true"
```
