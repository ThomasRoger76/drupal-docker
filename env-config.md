# .env / settings.php — Configuration et Variables d'Environnement

## Le Pattern `.env.dist` → `.env`

```
.env.dist   ← Committé dans git — valeurs de référence, placeholders
.env        ← Gitignored — valeurs réelles par développeur
```

**Règle absolue :** Jamais de secrets (mots de passe, clés API AWS) dans `.env.dist`.

```bash
# .gitignore
.env
# ⚠️ .env.dist est dans git — ne mettre que des valeurs non sensibles
```

---

## `.env.dist` Complet — Template Projet

```bash
###########################
# Compose
###########################
COMPOSE_PROJECT_NAME=mon_projet          # Préfixe des containers → isolation entre projets
COMPOSE_FILE=.docker/docker-compose.yml  # OU multi-fichiers :
# COMPOSE_FILE=.docker/docker-compose.yml:.docker/docker-compose.dev.yml

###########################
# Ports exposés (adapter par projet pour éviter les conflits)
###########################
HTTPD_PORT=80
HTTPD_SECURE_PORT=443
MAILDEV_PORT=84
PHPMYADMIN_PORT=82
MKDOCS_PORT=8050

###########################
# MariaDB
###########################
BDD_MARIADB_ROOT_PASSWORD=root
BDD_MARIADB_DATABASE=drupal
BDD_MARIADB_USER=drupal
BDD_MARIADB_PASSWORD=drupal
MARIADB_PORT=3306

###########################
# Permissions fichiers (DOIT correspondre à l'UID/GID de l'hôte)
###########################
DEV_UID=1000
DEV_GID=1000

###########################
# Outils
###########################
PHP_VERSION=8.3
PERSONAL_SSH_FOLDER=~/.ssh
PERSONAL_GLOBAL_COMPOSER_FOLDER=~/.composer

###########################
# Registre Docker (optionnel si images sur registre privé)
###########################
REGISTRY=registry.gitlab.example.com/mon-projet
```

---

## Trouver son UID/GID

```bash
# Sur Linux/macOS
id -u    # → 1000 (DEV_UID)
id -g    # → 1000 (DEV_GID)

# Sur WSL (Windows)
id -u    # → 1000 généralement

# Si UID ≠ 1000 → mettre à jour .env local
DEV_UID=$(id -u)
DEV_GID=$(id -g)
```

---

## Intégration `.env` → `settings.php` — Le Pont Critique

**PHP lit les variables d'environnement passées par Docker Compose via `getenv()`.**

```php
// web/sites/default/settings.php

// ─── BASE DE DONNÉES ─────────────────────────────────────────────────────
// ⚠️ host = nom du service Docker, pas 'localhost'
$databases['default']['default'] = [
  'driver'    => 'mysql',
  'namespace' => 'Drupal\\mysql\\Driver\\Database\\mysql',
  'autoload'  => 'core/modules/mysql/src/Driver/Database/mysql/',
  'host'      => getenv('MARIADB_HOSTNAME') ?: 'database',
  'port'      => getenv('MARIADB_PORT') ?: '3306',
  'database'  => getenv('MARIADB_DATABASE') ?: 'drupal',
  'username'  => getenv('MARIADB_USER') ?: 'drupal',
  'password'  => getenv('MARIADB_PASSWORD') ?: 'drupal',
  'prefix'    => '',
  'charset'   => 'utf8mb4',
  'collation' => 'utf8mb4_general_ci',
];

// ─── HASH SALT ───────────────────────────────────────────────────────────
$settings['hash_salt'] = getenv('HASH_SALT') ?: 'valeur-par-defaut-pour-le-dev';

// ─── TRUSTED HOSTS ───────────────────────────────────────────────────────
$settings['trusted_host_patterns'] = [
  '^localhost$',
  '^' . preg_quote(getenv('TRUSTED_HOSTS') ?: 'localhost') . '$',
  '\.local$',   // ← pour les domaines *.local avec Caddy
];

// ─── FICHIERS PRIVÉS ─────────────────────────────────────────────────────
$settings['file_private_path'] = '/var/www/html/private';
$settings['config_sync_directory'] = '/var/www/html/config/sync';
```

---

## `settings.local.php` — Surcharges par Développeur

```bash
# Copier depuis l'exemple après installation
cp web/sites/example.settings.local.php web/sites/default/settings.local.php
# Ce fichier est gitignored
```

```php
// web/sites/default/settings.local.php
<?php

// ─── CACHE DÉSACTIVÉ (dev uniquement) ────────────────────────────────────
$settings['cache']['bins']['render'] = 'cache.backend.null';
$settings['cache']['bins']['page']   = 'cache.backend.null';
$settings['cache']['bins']['dynamic_page_cache'] = 'cache.backend.null';

// ─── CSS/JS PREPROCESS DÉSACTIVÉ (voir les changements immédiatement) ────
$config['system.performance']['css']['preprocess'] = FALSE;
$config['system.performance']['js']['preprocess']  = FALSE;

// ─── ERREURS AFFICHÉES EN DEV ─────────────────────────────────────────────
$config['system.logging']['error_level'] = 'verbose';
assert_options(ASSERT_ACTIVE, TRUE);

// ─── SYNC DIRECTORY (override si chemin différent en local) ───────────────
// $settings['config_sync_directory'] = '../config/sync';
```

---

## `services.yml` — Twig Debug

```bash
# Copier le fichier services par défaut
cp web/sites/default/default.services.yml web/sites/default/services.yml
```

```yaml
# web/sites/default/services.yml (gitignored)
parameters:
  twig.config:
    debug: true
    auto_reload: true
    cache: false
```

```bash
# OBLIGATOIRE après modification de services.yml
docker compose exec php drush cr
```

---

## Variables d'Environnement Utiles dans les Containers

```bash
# Lister toutes les variables d'environnement dans le container PHP
docker compose exec php env | sort

# Vérifier une variable spécifique
docker compose exec php printenv MARIADB_HOSTNAME

# Depuis PHP — débugger dans un hook ou controller
var_dump(getenv('MARIADB_HOSTNAME')); exit;  // PHP natif (toujours disponible)
// OU avec le module Devel installé :
// dump(getenv('MARIADB_HOSTNAME'));           // Affichage formaté
// dd(getenv('MARIADB_HOSTNAME'));             // ← Symfony/Laravel uniquement, PAS Drupal natif
```

---

## Pattern Multi-Environnements via `COMPOSE_FILE`

```bash
# Local dev (avec overrides locaux)
COMPOSE_FILE=.docker/docker-compose.yml:.docker/docker-compose.dev.yml

# Staging (sans ports exposés sauf HTTP)
COMPOSE_FILE=.docker/docker-compose.yml:.docker/docker-compose.staging.yml

# CI (sans volumes hôte, avec image source-code)
COMPOSE_FILE=.docker/ci/docker-compose.yml
```

---

## Gestion des Secrets en Équipe

```bash
# Pattern recommandé pour les secrets d'équipe
.env.dist     → Valeurs de développement (git) — pas de secrets réels
.env.prod     → Secrets de production (hors git, déployé via CI/CD vault)
.env.secret   → Secrets locaux additionnels (gitignored)

# Dans .env.dist — clés AWS par exemple
AWS_ACCESS_KEY_ID=VOTRE_CLÉ_ICI         # ← placeholder visible dans git
AWS_SECRET_ACCESS_KEY=VOTRE_SECRET_ICI  # ← placeholder visible dans git

# Dans .env local réel (gitignored)
AWS_ACCESS_KEY_ID=AKIAQZFBVWEDY2KM3ZWN
AWS_SECRET_ACCESS_KEY=UkA3meHBHaHI5sLnKGdShppU...
```

**⚠️ Cas réel detecté :** Le projet `francetvpro.fr` avait des clés AWS réelles dans `.env.dist` — celles-ci étaient dans git. C'est une fuite de credentials. Toujours des placeholders dans `.env.dist`.
