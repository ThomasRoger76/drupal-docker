# Services Additionnels — Solr, Varnish, Elasticsearch

## Apache Solr — Search API

Apache Solr est le moteur de recherche le plus utilisé avec Drupal Search API.

```yaml
# docker-compose.yml
services:
  solr:
    image: solr:8.11-slim   # OU 9.x selon la compatibilité du module
    ports:
      - "${SOLR_PORT:-8983}:8983"
    environment:
      - SOLR_HEAP=512m
    volumes:
      - solr_data:/var/solr
      - ./services/solr/conf:/opt/solr/server/solr/drupal/conf  # Config Drupal
    command:
      - solr-precreate
      - drupal   # Nom du core Solr
volumes:
  solr_data:
```

### Configurer le core Solr pour Drupal

```bash
# 1. Installer le module Search API Solr
composer require drupal/search_api_solr

# 2. Copier les fichiers de configuration générés par le module
docker compose exec php drush sapi-sc drupal   # Generate Solr config for core 'drupal'
# Les fichiers config apparaissent dans sites/default/files/solr-config-*

# 3. Copier dans le dossier de config du container Solr
cp -r web/sites/default/files/solr-config-*/* .docker/services/solr/conf/
docker compose restart solr
```

### `settings.php` — Connexion Solr

```php
// settings.php — via interface Drupal admin (/admin/config/search/search-api)
// OU directement :
// Host: solr (nom du service Docker)
// Port: 8983
// Core: drupal
```

### Solr avec Docker Compose

```bash
# module non nécessaire avec Docker Compose
docker compose restart php
# Solr accessible sur https://mon-projet.docker compose exec php.site:8983
```

---

## Elasticsearch / OpenSearch

```yaml
services:
  elasticsearch:
    image: elasticsearch:8.12.0
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
      - xpack.security.enabled=false   # Désactiver la sécurité en dev
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data

  # Kibana (interface d'admin)
  kibana:
    image: kibana:8.12.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  elasticsearch_data:
```

```bash
# Tester la connexion
curl http://localhost:9200/_cluster/health

# Installer le module Drupal
composer require drupal/elasticsearch_connector
# OU
composer require drupal/search_api_opensearch   # Pour OpenSearch
```

---

## Varnish — Cache de Pages HTTP

```yaml
services:
  varnish:
    image: varnish:7.4
    ports:
      - "6081:6081"    # Port HTTP via Varnish
    volumes:
      - ./services/varnish/default.vcl:/etc/varnish/default.vcl
    environment:
      - VARNISH_BACKEND_HOST=php   # Nom du service PHP
      - VARNISH_BACKEND_PORT=80
    depends_on:
      - php
```

```vcl
# .docker/services/varnish/default.vcl
vcl 4.0;

backend default {
  .host = "php";      # Nom du service Docker
  .port = "80";
}

sub vcl_recv {
  # Passer les requêtes authentifiées directement (pas de cache)
  if (req.http.Cookie ~ "SESS" || req.http.Cookie ~ "NO_CACHE") {
    return (pass);
  }
  return (hash);
}
```

```php
// settings.php — activer le support Varnish
$config['system.performance']['cache']['page']['max_age'] = 900;
$settings['reverse_proxy'] = TRUE;
$settings['reverse_proxy_addresses'] = ['varnish'];
```

---

## Redis (Service Complet)

```yaml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "${REDIS_PORT:-6379}:6379"
    command:
      - redis-server
      - --maxmemory 256mb
      - --maxmemory-policy allkeys-lru
      - --save ""         # Désactiver la persistance (cache volatil)
    volumes:
      - redis_data:/data
```

```bash
# Prérequis
composer require drupal/redis
# + L'extension PHP redis doit être dans le Dockerfile
RUN pecl install redis && docker-php-ext-enable redis
```

```php
// settings.php
$settings['redis.connection']['interface'] = 'PhpRedis';  // Nécessite ext-redis
$settings['redis.connection']['host']      = 'redis';     // Nom service Docker
$settings['redis.connection']['port']      = 6379;
$settings['cache']['default']              = 'cache.backend.redis';
// Ne pas mettre la config en Redis (risque de deadlock)
$settings['cache']['bins']['config']       = 'cache.backend.database';
// Bootstrap cache en Redis (accélérer le démarrage Drupal)
$settings['cache']['bins']['bootstrap']    = 'cache.backend.redis';

volumes:
  redis_data:
```

---

## Mailpit — Alternative Moderne à Maildev

Mailpit est l'évolution de Maildev avec une interface plus moderne et API REST.

```yaml
services:
  mailpit:
    image: axllent/mailpit:latest
    ports:
      - "8025:8025"   # Interface web
      - "1025:1025"   # SMTP
```

```php
// settings.php
$config['system.mail']['interface']['default'] = 'test_mail_collector';
// OU avec le module Symfony Mailer :
// SMTP host: mailpit, port: 1025
```

---

## Portainer — Interface d'Administration Docker

```yaml
# docker-compose.yml — uniquement en dev, JAMAIS en prod
services:
  portainer:
    image: portainer/portainer-ce:latest
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  # Accès au daemon Docker
      - portainer_data:/data
volumes:
  portainer_data:
```

Interface web accessible sur `http://localhost:9000` pour gérer tous les containers, volumes, et images visuellement.

---

## Variables `.env.dist` pour les Services Additionnels

```bash
# Ajouter dans .env.dist selon les services utilisés

###########################
# Solr
###########################
SOLR_PORT=8983

###########################
# Elasticsearch
###########################
ES_PORT=9200
KIBANA_PORT=5601

###########################
# Redis
###########################
REDIS_PORT=6379

###########################
# Varnish
###########################
VARNISH_PORT=6081

###########################
# Mailpit
###########################
MAILPIT_PORT=8025
```
