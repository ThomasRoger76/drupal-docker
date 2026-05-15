# Taskfile.yml — Alternative Moderne aux Makefiles

## Pourquoi Taskfile plutôt que Makefile ?

| Critère | Makefile | Taskfile.yml |
|---------|----------|--------------|
| Syntaxe | Indentation par tabulation (source d'erreurs) | YAML — lisible et standard |
| Variables | Syntaxe cryptique (`$(shell ...)`, `$$var`) | Syntaxe claire (`{{.VAR}}`, `.env` natif) |
| Cross-platform | ❌ Tab obligatoire, `sh` Unix-only | ✅ Fonctionne sur macOS, Linux, Windows |
| Dépendances entre tâches | Limité (`prerequisite:`) | `deps:` avec exécution parallèle ou séquentielle |
| Variables d'env | `$(shell grep ...)` | `.env` chargé automatiquement |
| Watch mode | ❌ Non natif | ✅ `--watch` intégré |
| Affichage | Commandes brutes affichées | `silent: true` ou `desc:` pour aide intégrée |
| Conditions `if` | ❌ Verbose | ✅ `preconditions:` et `status:` |
| Usage typique Drupal | projets avec Makefile simple | projets avancés multi-équipe |

---

## Installation de Task

```bash
# Linux / macOS — via script officiel
sh -c "$(curl --location https://taskfile.dev/install.sh)" -- -d -b /usr/local/bin

# macOS — via Homebrew
brew install go-task/tap/go-task

# Windows — via winget
winget install Task.Task

# Via Go
go install github.com/go-task/task/v3/cmd/task@latest

# Vérifier l'installation
task --version
# task version 3.x.x (...)
```

---

## Syntaxe Taskfile.yml — Comparaison avec Makefile

### Makefile (avant)

```makefile
install: ## Installation complète
	@echo "→ Copie de .env.dist..."
	@cp -n .env.dist .env || echo ".env existe déjà"
	@docker compose build
	@docker compose up -d
	@docker compose run --rm php composer install

db-dump: ## Dump DB
	$(eval BDD_MARIADB_ROOT_PASSWORD=$(shell grep BDD_MARIADB_ROOT_PASSWORD .env | cut -d '=' -f2))
	@docker compose exec database mariadb-dump \
	  -u root -p$${BDD_MARIADB_ROOT_PASSWORD} drupal \
	  > .docker/services/database/dumps/dump-$$(date +%Y%m%d-%H%M).sql
```

### Taskfile.yml (après)

```yaml
version: '3'

dotenv: ['.env', '.env.dist']   # Chargement automatique des variables

tasks:
  install:
    desc: "Installation complète du projet (première fois)"
    cmds:
      - cp -n .env.dist .env || echo ".env existe déjà"
      - docker compose build
      - docker compose up -d
      - docker compose run --rm php composer install

  db-dump:
    desc: "Créer un dump de la base de données"
    cmds:
      - |
        docker compose exec database mariadb-dump \
          -u root -p{{.BDD_MARIADB_ROOT_PASSWORD}} {{.BDD_MARIADB_DATABASE}} \
          > .docker/services/database/dumps/dump-$(date +%Y%m%d-%H%M).sql
    # Les variables .env sont disponibles via {{.NOM_VARIABLE}}
```

---

## Taskfile.yml Complet pour un Projet Drupal

```yaml
# Taskfile.yml — Racine du projet
version: '3'

dotenv: ['.env', '.env.dist']

vars:
  COMPOSE_CMD: docker compose
  PHP_CONTAINER: php
  DB_CONTAINER: database

tasks:

  # ── Aide ──────────────────────────────────────────────────────────────
  default:
    desc: "Afficher toutes les tâches disponibles"
    cmds:
      - task --list
    silent: true

  # ── Installation ──────────────────────────────────────────────────────
  install:
    desc: "Installation complète (première fois)"
    cmds:
      - task: env
      - task: build
      - task: up
      - task: composer
      - task: drupal:install
      - task: permissions
    silent: false

  env:
    desc: "Copier .env.dist → .env si absent"
    cmds:
      - cp -n .env.dist .env || echo ".env existe déjà, skip"
    status:
      - test -f .env   # Tâche sautée si .env existe déjà

  # ── Cycle de vie des containers ───────────────────────────────────────
  up:
    desc: "Démarrer les containers"
    cmds:
      - "{{.COMPOSE_CMD}} pull --quiet"
      - "{{.COMPOSE_CMD}} up -d"
      - echo "✓ Containers démarrés"

  down:
    desc: "Arrêter les containers (sans perdre les données)"
    cmds:
      - "{{.COMPOSE_CMD}} down"

  restart:
    desc: "Redémarrer le container PHP"
    cmds:
      - "{{.COMPOSE_CMD}} restart {{.PHP_CONTAINER}}"

  build:
    desc: "Reconstruire les images Docker"
    cmds:
      - "{{.COMPOSE_CMD}} build --no-cache"

  logs:
    desc: "Voir les logs PHP en temps réel"
    cmds:
      - "{{.COMPOSE_CMD}} logs {{.PHP_CONTAINER}} -f"

  shell:
    desc: "Ouvrir un shell dans le container PHP"
    cmds:
      - "{{.COMPOSE_CMD}} exec {{.PHP_CONTAINER}} bash"

  # ── Composer ──────────────────────────────────────────────────────────
  composer:
    desc: "Installer les dépendances Composer"
    cmds:
      - "{{.COMPOSE_CMD}} run --rm {{.PHP_CONTAINER}} composer install"

  composer:update:
    desc: "Mettre à jour Drupal core (usage: task composer:update)"
    cmds:
      - |
        {{.COMPOSE_CMD}} run --rm {{.PHP_CONTAINER}} composer update \
          drupal/core-recommended drupal/core-dev --with-dependencies

  # ── Drupal ────────────────────────────────────────────────────────────
  drupal:install:
    desc: "Installer Drupal depuis la config existante"
    cmds:
      - |
        {{.COMPOSE_CMD}} exec {{.PHP_CONTAINER}} drush site:install \
          --locale=fr \
          --no-interaction \
          --site-name="{{.SITE_NAME | default "Mon Site"}}" \
          --account-name=admin \
          --account-pass=admin \
          --existing-config -y

  drupal:update:
    desc: "Appliquer les updates DB et exporter la config"
    deps: [cache:clear]
    cmds:
      - "{{.COMPOSE_CMD}} exec {{.PHP_CONTAINER}} drush updb -y"
      - "{{.COMPOSE_CMD}} exec {{.PHP_CONTAINER}} drush cex -y"
      - task: cache:clear

  drush:
    desc: "Exécuter une commande Drush (usage: task drush -- cr)"
    cmds:
      - "{{.COMPOSE_CMD}} exec {{.PHP_CONTAINER}} drush {{.CLI_ARGS}}"

  # ── Cache ─────────────────────────────────────────────────────────────
  cache:clear:
    desc: "Vider le cache Drupal"
    cmds:
      - "{{.COMPOSE_CMD}} exec {{.PHP_CONTAINER}} drush cr"
    aliases: [cr]

  # ── Base de données ───────────────────────────────────────────────────
  db:dump:
    desc: "Créer un dump de la DB MariaDB"
    cmds:
      - |
        docker compose exec {{.DB_CONTAINER}} mariadb-dump \
          -u root -p{{.BDD_MARIADB_ROOT_PASSWORD}} {{.BDD_MARIADB_DATABASE}} \
          > .docker/services/database/dumps/dump-$(date +%Y%m%d-%H%M).sql
        echo "✓ Dump créé dans .docker/services/database/dumps/"

  db:import:
    desc: "Importer un dump SQL (usage: task db:import FILE=dump.sql)"
    cmds:
      - |
        docker compose exec -T {{.DB_CONTAINER}} mariadb \
          -u {{.BDD_MARIADB_USER}} -p{{.BDD_MARIADB_PASSWORD}} {{.BDD_MARIADB_DATABASE}} \
          < .docker/services/database/dumps/{{.FILE}}
      - task: cache:clear

  db:reset:
    desc: "Réinstaller Drupal depuis zéro (efface la DB)"
    prompt: "⚠️  Ceci va effacer toutes les données. Continuer ?"
    cmds:
      - |
        {{.COMPOSE_CMD}} exec {{.PHP_CONTAINER}} drush site:install \
          --locale=fr --no-interaction --existing-config -y

  # ── Permissions ───────────────────────────────────────────────────────
  permissions:
    desc: "Corriger les permissions des fichiers Drupal (Linux)"
    platforms: [linux]
    cmds:
      - sudo chown -R 1000:www-data web/sites/default/files
      - sudo chmod -R g+w web/sites/default/files
      - sudo chmod u+w web/sites/default
      - git config core.fileMode false
      - echo "✓ Permissions corrigées"

  # ── Thème ─────────────────────────────────────────────────────────────
  theme:install:
    desc: "Installer les dépendances NPM du thème"
    cmds:
      - "{{.COMPOSE_CMD}} run --rm webpack_theming npm install"

  theme:watch:
    desc: "Lancer le watch mode du thème"
    cmds:
      - "{{.COMPOSE_CMD}} up webpack_theming -d"
      - "{{.COMPOSE_CMD}} logs webpack_theming -f"

  theme:build:
    desc: "Build de production du thème"
    cmds:
      - "{{.COMPOSE_CMD}} run --rm webpack_theming npm run build"

  # ── Vérification ──────────────────────────────────────────────────────
  verify:
    desc: "Vérifier l'état du projet (composer + drush)"
    cmds:
      - task: cache:clear
      - task: composer:update
      - task: drupal:update
      - echo "✓ Vérification terminée — committer composer.lock + config/sync/"
```

---

## Fonctionnalités Avancées de Taskfile

### Variables dynamiques

```yaml
vars:
  GIT_COMMIT:
    sh: git rev-parse --short HEAD
  DATE:
    sh: date +%Y%m%d

tasks:
  tag-image:
    cmds:
      - docker tag drupal:latest drupal:{{.GIT_COMMIT}}-{{.DATE}}
```

### Conditions avec `status:` (skip si déjà fait)

```yaml
tasks:
  composer:
    desc: "Installer Composer uniquement si vendor/ absent"
    status:
      - test -d vendor   # Skip si vendor/ existe
    cmds:
      - docker compose run --rm php composer install
```

### Dépendances parallèles

```yaml
tasks:
  ci:
    desc: "Lancer tous les checks en parallèle"
    deps:
      - task: phpcs
      - task: phpstan
      - task: phpunit
    # Les 3 tâches s'exécutent en parallèle !
```

### Prompt de confirmation

```yaml
tasks:
  db:reset:
    prompt: "⚠️  Ceci va effacer toutes les données. Continuer ?"
    cmds:
      - docker compose exec php drush site:install --existing-config -y
```

### Tâches spécifiques à une plateforme

```yaml
tasks:
  permissions:
    platforms: [linux]   # Ignorée sur macOS/Windows
    cmds:
      - sudo chown -R 1000:www-data web/sites/default/files
```

### Watch mode intégré

```yaml
tasks:
  watch-theme:
    watch: true
    sources:
      - web/themes/mon_theme/src/**/*.scss
    cmds:
      - docker compose run --rm webpack_theming npm run build
```

---

## Commandes Task Essentielles

```bash
# Lister toutes les tâches avec description
task --list
task -l     # Raccourci

# Exécuter une tâche
task install
task up
task db:dump

# Passer des arguments
task drush -- cr
task db:import FILE=dump-20260514.sql

# Tâche par défaut (définie comme `default:`)
task

# Mode dry-run (afficher sans exécuter)
task --dry install

# Forcer l'exécution même si status: dit skip
task --force composer

# Watch mode sur une tâche
task --watch theme:build

# Voir les tâches disponibles avec leur source
task --list-all
```

---

## Cohabitation Makefile + Taskfile

Durant la migration, les deux peuvent coexister. Pour rediriger les habitués du Makefile :

```makefile
# Makefile — déléguer vers Task
.PHONY: install up down verify
install up down verify:
	task $@

# Ou ajouter un alias global dans le Makefile
%:
	task $@
```

---

## Structure de Répertoire avec Taskfile

```
projet/
├── Taskfile.yml          # Tâches globales
├── .env.dist
├── .env
├── web/
│   └── themes/
│       └── mon-theme/
│           └── Taskfile.yml   # Tâches spécifiques au thème
└── .docker/
    └── Taskfile.yml           # Tâches Docker uniquement (optionnel)
```

```yaml
# Taskfile.yml racine — inclure des sous-Taskfiles
version: '3'

includes:
  theme:
    taskfile: ./web/themes/mon-theme/Taskfile.yml
    dir: ./web/themes/mon-theme
  docker:
    taskfile: ./.docker/Taskfile.yml

# Usage : task theme:watch, task docker:prune, etc.
```
