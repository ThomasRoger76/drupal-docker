# Alternatives à Docker Compose Custom

> **Note :** Ce référentiel utilise exclusivement **Docker Compose custom**. Cette page documente les alternatives pour contexte.

## Comparatif rapide

| Outil | Setup | Flexibilité | CI/CD | Recommandé ici |
|-------|-------|-------------|-------|----------------|
| Docker Compose custom | ⏱️ 2-4h | ✅ Totale | ✅ Natif | ✅ **OUI** |
| DDEV | ✅ 5 min | 🟡 Extensions | 🟡 Possible | ❌ Non |
| Lando | ✅ 10 min | 🟡 Recipes | 🟡 Possible | ❌ Non |

## Équivalences de commandes DDEV → Docker Compose

| Ancien (DDEV) | Docker Compose Custom |
|---------------|----------------------|
| `ddev start` | `docker compose up -d` |
| `ddev stop` | `docker compose down` |
| `ddev drush cr` | `docker compose exec php drush cr` |
| `ddev ssh` | `docker compose exec php bash` |
| `ddev composer install` | `docker compose exec php composer install` |
| `ddev mysql` | `docker compose exec database mysql -u root -p` |
| `ddev describe` | `docker compose ps` |
| `ddev logs` | `docker compose logs -f php` |
| `ddev phpcs` | `docker compose exec php vendor/bin/phpcs --standard=Drupal` |
| `ddev phpstan` | `docker compose exec php vendor/bin/phpstan analyse` |

## Migration depuis DDEV

```bash
# 1. Exporter la DB depuis l'ancien projet DDEV (sur l'autre machine)
# ddev export-db --file=dump.sql  (commande DDEV exécutée sur l'ancienne machine)

# 2. Importer dans Docker Compose
docker compose up -d
docker compose exec database mysql \
  -u "${BDD_MARIADB_USER}" -p"${BDD_MARIADB_PASSWORD}" "${BDD_MARIADB_DATABASE}" \
  < dump.sql

# 3. Adapter settings.php
# DDEV : getenv('DDEV_DATABASE_HOSTNAME') → Docker : getenv('MARIADB_HOSTNAME')
docker compose exec php drush status
```
