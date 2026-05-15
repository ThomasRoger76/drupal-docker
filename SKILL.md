---
name: drupal-docker
description: Use when setting up Docker environments for Drupal projects, writing docker-compose.yml with PHP/MariaDB/Caddy services, configuring .env and settings.php integration, managing file permissions with DEV_UID/GID, connecting Xdebug from container to IDE via extra_hosts, setting up Makefile workflows for install/up/verify, handling bind mount performance issues, or configuring CI docker-compose pipelines for Drupal in Drupal 8-11+
---

# Drupal Docker & Environment Management — Référence Complète

## Overview

Référentiel complet de l'environnement Docker pour Drupal : architecture services, Dockerfile PHP+Apache, Caddy TLS local, connexion DB via nom de service, volumes, `.env` ↔ `settings.php`, Xdebug, Makefile, CI/CD. Basé sur des projets Drupal 10/11 réels.

## Quick Decision Table

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Créer l'environnement Docker d'un projet Drupal | `docker-compose.yml` + `.env.dist` | [docker-compose.md](docker-compose.md) |
| Dockerfile PHP+Apache pour Drupal | `php:VERSION-apache` + extensions | [architecture.md](architecture.md) |
| TLS local automatique sans config SSL | Caddy avec `tls internal` | [architecture.md](architecture.md) |
| Relier PHP à la DB dans settings.php | `getenv('MARIADB_HOSTNAME')` | [env-config.md](env-config.md) |
| Partager `.env` entre devs sans exposer de secrets | `.env.dist` → `.env` (gitignored) | [env-config.md](env-config.md) |
| Éviter `Permission denied` sur les fichiers montés | `DEV_UID/DEV_GID` dans le container | [env-config.md](env-config.md) |
| Commandes standard (install, up, verify) | `Makefile` | [makefile-workflow.md](makefile-workflow.md) |
| Connecter Xdebug du container à l'IDE | `extra_hosts: host.docker.internal` | [debugging.md](debugging.md) |
| Lire les logs PHP-FPM / Apache | `docker compose logs php -f` | [debugging.md](debugging.md) |
| Exécuter Drush dans le container | `docker compose run --rm php drush` | [debugging.md](debugging.md) |
| Monter le cache Composer local | Volume SSH + Composer host | [docker-compose.md](docker-compose.md) |
| Persistance DB entre redémarrages | Named volume `database_data` | [docker-compose.md](docker-compose.md) |
| Séparer dev/prod/CI | `docker-compose.dev.yml`, `docker-compose.ci.yml` | [docker-compose.md](docker-compose.md) |
| Thème Node.js dans le container | Service `webpack_theming` | [architecture.md](architecture.md) |
| Capture emails en local | `maildev/maildev` | [architecture.md](architecture.md) |
| Cache Memcached | Service `memcached:alpine` | [architecture.md](architecture.md) |
| Performance bind mounts (Mac/Windows) | Mutagen, VirtioFS, ou volume nommé | [performance.md](performance.md) |
| Pipeline CI Docker | `docker-compose.ci.yml` + image source-code | [makefile-workflow.md](makefile-workflow.md) |
| `.dockerignore` pour accélérer les builds | Exclure vendor, node_modules, .git | [dockerignore-build.md](dockerignore-build.md) |
| Image prod sans Xdebug/Composer | Dockerfile multi-stage (`target: production`) | [dockerignore-build.md](dockerignore-build.md) |
| Hot reload sans bind mount lent | `docker compose watch` (Compose 2.22+) | [dockerignore-build.md](dockerignore-build.md) |
| PostgreSQL à la place de MariaDB | Service `postgres:16.x` + `pgsql` driver | [docker-compose.md](docker-compose.md) |
| Taskfile.yml — alternative moderne au Makefile | `task install`, `task up`, `task verify` | [taskfile.md](taskfile.md) |
| Kubernetes — Drupal en production | Deployment, PVC, Ingress, kubectl drush | [kubernetes.md](kubernetes.md) |
| GitLab Registry — images custom dans CI | `registry.gitlab.example.com/projet/image:sha` | [docker-compose.md](docker-compose.md) |
| Solr / Search API | Service `solr:8.11` + core Drupal | [services-additionnels.md](services-additionnels.md) |
| Elasticsearch / OpenSearch | Service `elasticsearch:8.x` | [services-additionnels.md](services-additionnels.md) |
| Varnish — cache de pages HTTP | Service `varnish:7.4` + VCL Drupal | [services-additionnels.md](services-additionnels.md) |
| MariaDB pas encore prête à l'install | `healthcheck` + `start_period` + wait-db | [debugging.md](debugging.md) |
| Permissions fichiers selon user du container | `docker compose exec --user www-data` | [debugging.md](debugging.md) |
| Image Docker optimisée pour la production | Multi-stage Dockerfile (`target: production`) | [production.md](production.md) |
| OPcache production (validate_timestamps=0) | `opcache.ini` dédié prod | [production.md](production.md) |
| Multi-projets simultanés sur un poste dev | Traefik reverse proxy partagé | [production.md](production.md) |
| PHP derrière reverse proxy (IP réelle du client) | `$settings['reverse_proxy']` | [production.md](production.md) |
| PHP-FPM tuning production | `pm.max_children`, `pm.max_requests` | [production.md](production.md) |
| JIT PHP 8.3 pour migrations/imports lourds | `opcache.jit = 1255` avec benchmark avant activation | [production.md](production.md) |
| Log aggregation légère (Loki + Grafana) | Services Loki + Promtail + Grafana dans docker-compose | [production.md](production.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Raison |
|---------------------|------------------|--------|
| `host: localhost` dans settings.php | `host: getenv('MARIADB_HOSTNAME')` | `localhost` = le container PHP, pas la DB |
| Committer `.env` avec des secrets réels | `.env.dist` committé, `.env` gitignored | Fuite de credentials dans git |
| `AllowOverride None` dans Apache | `AllowOverride All` obligatoire | Drupal `.htaccess` ne fonctionne pas |
| Volume sans nom (données dans le container) | Named volume `database_data:/var/lib/mysql` | Perte des données au `docker compose down -v` |
| RUN apt-get en plusieurs layers | `apt-get update && apt-get install -y ...` en un seul RUN | Cache Docker — chaque RUN = un layer |
| `docker compose down -v` en routune | `docker compose down` (sans -v) | `-v` supprime les volumes = perte de la DB |
| PHP+DB sur le même port `localhost:3306` | Chaque service expose son port unique | Conflits entre projets |
| Xdebug `client_host=localhost` dans le container | `client_host=host.docker.internal` | `localhost` dans le container = le container |
| Secrets (clés API) dans `.env.dist` | Placeholders vides dans `.env.dist`, valeurs réelles dans `.env` | `.env.dist` est dans git |

## Évolution par Version

| Feature | Docker CE 20 | Docker CE 24+ | Docker CE 27+ |
|---------|-------------|---------------|---------------|
| `docker-compose` (v1) | ✅ | ⚠️ deprecated | ❌ supprimé |
| `docker compose` (v2, plugin) | ✅ | ✅ standard | ✅ |
| `profiles:` dans compose | ✅ | ✅ | ✅ |
| `depends_on: condition:` | ✅ | ✅ | ✅ |
| `include:` (compose modulaire) | ❌ | ✅ 2.20+ | ✅ |
| `watch:` (hot reload) | ❌ | ✅ 2.22+ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Problèmes rencontrés en projet réel.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions (v1.0 courante).

## See Also

- `drupal-core` — settings.php, Config API
- `drupal-config` — `drush cex/cim` dans les containers
- `drupal-testing` — tests dans le CI Docker
- `drush` — Drush CLI, aliases, déploiement
