# Makefile & Workflow — Commandes Standard

## Makefile Complet — Template Projet

```makefile
# Makefile à la racine du projet
COMMANDS := $(MAKEFILE_LIST)
.DEFAULT_GOAL := help
.PHONY: help install install-linux up down verify build logs shell drush

help: ## Afficher cette aide
	@grep -E '^[a-zA-Z0-9_-]+:.*?## .*$$' $(MAKEFILE_LIST) \
	  | sed -n 's/^\(.*\): \(.*\)##\(.*\)/\1\3/p' \
	  | column -t -s ' '

# ─── INSTALLATION ─────────────────────────────────────────────────────────────

install: ## Installation complète du projet (première fois)
	@echo "→ Copie de .env.dist..."
	@cp -n .env.dist .env || echo ".env existe déjà"
	@echo "→ Build des images..."
	@docker compose build
	@echo "→ Démarrage des containers..."
	@docker compose up -d
	@echo "→ Installation des dépendances Composer..."
	@docker compose run --rm php composer install
	@echo "→ Installation de Drupal..."
	@docker compose exec php drush site:install \
	  --locale=fr \
	  --no-interaction \
	  --site-name="Mon Site" \
	  --account-name=admin \
	  --account-pass=admin \
	  --existing-config
	@echo "→ Synchronisation UUID du site..."
	@docker compose exec php drush cset system.site uuid \
	  "$$(grep 'uuid' config/sync/system.site.yml | awk '{print $$2}' | tr -d "'")" -y
	@echo "→ Copie des fichiers services locaux..."
	@cp -n web/sites/default/default.services.yml web/sites/default/services.yml || true
	@cp -n web/sites/example.settings.local.php web/sites/default/settings.local.php || true
	@echo "✓ Installation terminée — http://localhost"

install-linux: ## Correction des permissions sur Linux (après install)
	@sudo chown -R 1000:www-data web/sites/default/files
	@sudo chmod -R g+w web/sites/default/files
	@sudo chmod u+w web/sites/default
	@git config core.fileMode false
	@echo "✓ Permissions corrigées"

# ─── CYCLE DE VIE ─────────────────────────────────────────────────────────────

up: ## Démarrer les containers
	@docker compose pull
	@docker compose up -d
	@echo "✓ Containers démarrés"

down: ## Arrêter les containers (sans perdre les données)
	@docker compose down
	@echo "✓ Containers arrêtés"

restart: ## Redémarrer le container PHP
	@docker compose restart php
	@echo "✓ PHP redémarré"

build: ## Reconstruire les images Docker
	@docker compose build --no-cache
	@echo "✓ Images reconstruites"

# ─── DÉVELOPPEMENT ────────────────────────────────────────────────────────────

verify: ## Mettre à jour Drupal core + exporter la config
	@docker compose exec php drush cr
	@docker compose run --rm php composer update drupal/core-recommended drupal/core-dev --with-dependencies
	@docker compose exec php drush updb -y
	@docker compose exec php drush cex -y
	@docker compose exec php drush cr
	@echo "✓ Mise à jour terminée — tester localement puis committer composer.lock + config/sync/"

logs: ## Voir les logs en temps réel
	@docker compose logs php -f

shell: ## Ouvrir un shell dans le container PHP
	@docker compose exec php bash

drush: ## Exécuter une commande Drush (ex: make drush CMD="cr")
	@docker compose exec php drush $(CMD)

# ─── BASE DE DONNÉES ───────────────────────────────────────────────────────────

db-dump: ## Créer un dump de la DB
	$(eval BDD_MARIADB_ROOT_PASSWORD=$(shell grep BDD_MARIADB_ROOT_PASSWORD .env | cut -d '=' -f2))
	$(eval BDD_MARIADB_DATABASE=$(shell grep BDD_MARIADB_DATABASE .env | cut -d '=' -f2))
	@docker compose exec database mariadb-dump \
	  -u root -p$${BDD_MARIADB_ROOT_PASSWORD} $${BDD_MARIADB_DATABASE} \
	  > .docker/services/database/dumps/dump-$$(date +%Y%m%d-%H%M).sql
	@echo "✓ Dump créé dans .docker/services/database/dumps/"

db-import: ## Importer un dump SQL (ex: make db-import FILE=dump-20260514.sql)
	$(eval BDD_MARIADB_USER=$(shell grep BDD_MARIADB_USER .env | cut -d '=' -f2))
	$(eval BDD_MARIADB_PASSWORD=$(shell grep BDD_MARIADB_PASSWORD .env | cut -d '=' -f2))
	$(eval BDD_MARIADB_DATABASE=$(shell grep BDD_MARIADB_DATABASE .env | cut -d '=' -f2))
	@docker compose exec -T database mariadb \
	  -u $${BDD_MARIADB_USER} -p$${BDD_MARIADB_PASSWORD} $${BDD_MARIADB_DATABASE} \
	  < .docker/services/database/dumps/$(FILE)
	@docker compose exec php drush cr
	@echo "✓ Dump importé"

db-reset: ## Réinstaller Drupal depuis zéro (efface la DB)
	@docker compose exec php drush site:install \
	  --locale=fr --no-interaction --existing-config -y
	@echo "✓ DB réinitialisée"

# ─── THÈME ────────────────────────────────────────────────────────────────────

theme-install: ## Installer les dépendances NPM du thème
	@docker compose run --rm webpack_theming npm install

theme-watch: ## Démarrer le watch mode du thème
	@docker compose up webpack_theming -d
	@docker compose logs webpack_theming -f

theme-build: ## Build de production du thème
	@docker compose run --rm webpack_theming npm run build
```

---

## Workflow d'Installation Expliqué

```bash
# 1. Cloner le dépôt
git clone git@gitlab.example.com:projet.git
cd projet

# 2. Copier le .env (jamais committer .env)
cp .env.dist .env
# → Adapter les ports si conflit avec d'autres projets

# 3. Lancer l'installation complète
make install
# OU étape par étape :
docker compose up -d
docker compose run --rm php composer install
docker compose exec php drush site:install --existing-config -y

# 4. Sur Linux uniquement — corriger les permissions
make install-linux
# OU manuellement :
sudo chown -R 1000:www-data web/sites/default/files
sudo chmod -R g+w web/sites/default/files
```

---

## Workflow de Mise à Jour Drupal

```bash
# Mettre à jour via Composer (depuis le container)
docker compose run --rm php composer update drupal/core-recommended --with-dependencies

# Appliquer les updates DB + config
docker compose exec php drush updb -y
docker compose exec php drush cex -y
docker compose exec php drush cr

# Committer
git add composer.lock config/sync/
git commit -m "chore: upgrade drupal core to X.Y.Z"
```

---

## CI/CD — Pipeline GitLab CI Complet

```yaml
# .gitlab-ci.yml
variables:
  COMPOSE_PROJECT_NAME: "${CI_PROJECT_NAME}-${CI_PIPELINE_ID}"
  COMPOSE_FILE: ".docker/ci/docker-compose.yml"

stages:
  - build
  - test
  - deploy

# ─── BUILD ────────────────────────────────────────────────────────────────────
build:source-code:
  stage: build
  script:
    - docker compose build source-code
    - docker compose push source-code
  only:
    - merge_requests
    - main

# ─── TESTS ────────────────────────────────────────────────────────────────────
test:unit:
  stage: test
  script:
    - docker compose up -d
    - docker compose run --rm drupal-apache-php vendor/bin/phpunit --testsuite unit
  after_script:
    - docker compose down -v   # Nettoyer les containers CI

test:functional:
  stage: test
  script:
    - docker compose up -d
    - docker compose exec drupal-apache-php drush site:install --existing-config -y
    - docker compose run --rm drupal-apache-php vendor/bin/phpunit --testsuite functional
  after_script:
    - docker compose down -v

# ─── DÉPLOIEMENT ──────────────────────────────────────────────────────────────
deploy:staging:
  stage: deploy
  environment:
    name: staging
  script:
    - # Déployer sur le serveur staging
    - ssh deploy@staging.example.com "cd /var/www/projet && git pull && make verify"
  only:
    - main
```

---

## Commandes Docker Compose Avancées

```bash
# Voir l'utilisation des ressources (CPU, RAM)
docker compose stats

# Inspecter un container
docker compose exec php env | sort   # Variables d'environnement
docker compose exec php df -h        # Espace disque dans le container

# Copier un fichier hôte → container
docker cp local-file.sql $(docker compose ps -q database):/dumps/

# Voir les volumes et leur taille
docker volume ls | grep COMPOSE_PROJECT_NAME
docker volume inspect COMPOSE_PROJECT_NAME_database_data

# Purger les images non utilisées (libérer de l'espace)
docker system prune -f
docker system prune -a -f   # ⚠️ Supprime TOUTES les images non utilisées

# Reconstruire proprement une image (sans cache)
docker compose build --no-cache --pull php
```
