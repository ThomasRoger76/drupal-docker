# Changelog — drupal-docker

---

## v1.3 — 2026-06-09

**Corruptions de texte corrigées (remplacement automatique `ddev` → `docker compose exec php` cassé) :**
- `performance.md` : "Solution 2 — Mutagen (Docker Compose)" contenait un YAML/bash absurde (`.docker compose exec php/config.yaml`, `docker compose exec php start`) — remplacé par une vraie section `docker compose watch` (natif 2.22+)
- `performance.md` : tableau comparatif avec deux lignes "Mutagen + Docker" identiques — corrigé en `docker compose watch` vs `Mutagen + mutagen-compose`
- `services-additionnels.md` : `https://mon-projet.docker compose exec php.site:8983` + "module non nécessaire" — remplacé par une vraie procédure de vérification Solr (`curl .../admin/ping`)
- `dockerignore-build.md` : `.docker compose exec php/` dans le `.dockerignore` — corrigé en `.ddev/`
- `CHANGELOG.md` : référence à un fichier inexistant `docker compose exec php.md` — corrigé en `ddev.md`

**Versions d'images mises à jour (cible Drupal 11) :**
- MariaDB `11.0.2` (patch obsolète de 2023) → `11.4` LTS (support jusqu'en 2029) dans `docker-compose.md`, `architecture.md`, `production.md` (service CI)
- PostgreSQL `16.11` (version inexistante) → `16` (tag majeur stable) dans `docker-compose.md`
- Solr `8.11` → `9` dans `SKILL.md` et `services-additionnels.md` (Search API Solr recommande Solr 9)

**Cohérences corrigées :**
- `performance.md` : config OPcache dev — commentaire trompeur "0 = jamais revalider (prod)" sur un bloc dev — clarifié
- `CHANGELOG.md` : tableau de compatibilité — ligne v1.3 ajoutée (MariaDB 11.4, PHP 8.3/8.4, note exigence D11)
- `README.md` : liste de fichiers incomplète (manquaient `kubernetes.md`, `production.md`, `taskfile.md`) — complétée ; tagline ajustée pour refléter l'approche Docker Compose natif (DDEV reste une alternative non recommandée)

---

## v1.2 — 2026-05-16

**Description frontmatter étendue :**
- Ajout : Kubernetes, Taskfile, Varnish, Solr, Elasticsearch, Traefik, multi-stage production, non-root user, GitLab Registry
- Garantit le déclenchement du skill sur ces sujets déjà couverts dans les fichiers de référence

**architecture.md — Sécurité containers :**
- Nouvelle section "Exécution en Non-Root" : Dockerfile multi-stage avec `groupadd/useradd`, stages dev (root pour Xdebug) vs prod (non-root garanti), compatibilité bind-mount via `USER_UID/USER_GID` args dans docker-compose.yml

---

## v1.1 — 2026-05-14

**Bugs corrigés :**
- `makefile-workflow.md` : `db-dump` et `db-import` utilisaient des credentials hardcodés (`-proot`, `-pdrupal`) — remplacés par lecture depuis `.env`
- `env-config.md` : `dd()` n'existe pas en Drupal natif — remplacé par `var_dump()` + note sur Devel
- `docker-compose.md` : nommage containers Docker Compose v2 (tirets, pas underscores)
- `debugging.md` : `xdebug.idekey = VSCODE` non reconnu par VS Code — corrigé en commentaire PhpStorm
- `performance.md` : `x-mutagen` présenté comme standard Docker Compose — corrigé, `mutagen-compose` requis
- `makefile-workflow.md` : `composer update drupal/*` trop agressif → `composer update drupal/core-recommended`

**Incohérences corrigées :**
- `SKILL.md` : option `cached` Docker supprimée depuis 2022 — retirée
- `architecture.md` : Memcached + Redis — note sur les modules contrib requis ajoutée
- `debugging.md` : `ping` absent Alpine → `nc -zv` recommandé

**Nouveaux fichiers :**
- `dockerignore-build.md` : `.dockerignore` template complet, Dockerfile multi-stage (prod vs dev), `docker compose watch`
- `ddev.md` : comparatif DDEV vs Docker Compose, équivalences de commandes, migration depuis DDEV (alternative non recommandée ici)
- `services-additionnels.md` : Solr Search API, Elasticsearch, Varnish, Redis complet avec prérequis, Mailpit, Portainer

**Ajouts dans les fichiers existants :**
- `debugging.md` : Section `--user www-data` pour les permissions, Section DB readiness (healthcheck + wait-db Makefile)
- `SKILL.md` : 11 nouvelles entrées Quick Decision Table (dockerignore, multi-stage, DDEV, Solr, Varnish, ES, permissions, DB readiness)
- `lessons.md` : 3 nouvelles leçons (MariaDB non-prête, drush root permissions, build lent sans .dockerignore)

---

## v1.0 — 2026-05-14

**Création initiale — basé sur des projets Drupal 10/11 réels**

### Patterns couverts

Les configurations documentées dans ce skill sont extraites de projets Drupal 10/11 containerisés :
- PHP+Apache all-in-one avec MariaDB 11, Maildev, Node webpack, Makefile
- PHP+Apache + Caddy (TLS local), MariaDB, Maildev, Node webpack
- PHP-FPM séparé + Apache httpd, Memcached, Xdebug
- PHP-FPM + Apache + Gulp, MySQL, pipeline CI avec image source-code
- Registre privé GitLab

### Couverture

**`SKILL.md`**
- Quick Decision Table (17 entrées couvrant toutes les situations)
- 9 anti-patterns critiques (localhost DB, AllowOverride, down -v, etc.)
- Versioning Docker Compose v1→v2

**`architecture.md`**
- 3 patterns d'architecture réels (all-in-one, FPM séparé, Caddy en front)
- Dockerfile PHP+Apache complet avec toutes les extensions Drupal
- OPcache, APCu, Composer multi-stage
- Services additionnels : MariaDB, Maildev, Memcached, Node/webpack, PhpMyAdmin
- Réseau Docker — pourquoi `database` et pas `localhost`
- Images privées GitLab + profiles Docker Compose

**`docker-compose.md`**
- Template complet `docker-compose.yml` annoté ligne par ligne
- Stratégie multi-fichiers dev/CI (pattern Digiwin)
- Tableau volumes : named vs bind vs SSH
- Toutes les commandes `docker compose` essentielles
- `depends_on` avec health checks
- Isolation projets via `COMPOSE_PROJECT_NAME`

**`env-config.md`**
- Template `.env.dist` complet (toutes les sections)
- Comment trouver son UID/GID
- Intégration `.env` → `settings.php` via `getenv()`
- `settings.php` complet avec DB, trusted hosts, private path, config sync
- `settings.local.php` avec cache null, verbose errors
- `services.yml` pour Twig debug
- Pattern multi-environnements via `COMPOSE_FILE`
- Gestion des secrets — clés AWS dans `.env.dist` committé (erreur fréquente)

**`makefile-workflow.md`**
- Makefile complet (install, install-linux, up, down, verify, logs, shell, drush, db-dump, db-import, theme-*)
- Workflow d'installation expliqué étape par étape
- Workflow de mise à jour Drupal
- Pipeline GitLab CI complet (build → test unit → test functional → deploy staging)
- Commandes Docker avancées (stats, inspect, volumes, prune)

**`debugging.md`**
- Xdebug 3 — configuration complète en 4 étapes (`extra_hosts` → `php.ini` → IDE → vérification)
- VS Code `launch.json` + PhpStorm path mappings
- Script de vérification connectivité Xdebug
- Logs Docker — comment lire une page blanche
- Shell dans les containers (exec, run --rm)
- Drush depuis l'hôte (exec vs run)
- Tableau troubleshooting des 9 erreurs les plus courantes
- Import/export DB depuis le container et depuis l'hôte
- Inspection réseau Docker

**`performance.md`**
- Problème bind mount Mac/Windows expliqué
- Solution 1 : Mutagen (docker-compose.yml complet)
- Solution 2 : DDEV avec Mutagen automatique
- Solution 3 : VirtioFS Apple Silicon
- OPcache — configuration prod vs dev
- Cache Docker build — ordre COPY optimisé pour Composer
- APCu — configuration + intégration Drupal
- Redis — configuration docker-compose + settings.php
- Profilage avec `docker stats`
- Tableau comparatif des solutions performance

**`lessons.md`**
- 10 leçons pré-remplies basées sur des incidents réels :
  - `localhost` → DB refusée
  - `AllowOverride None` → 404 Drupal
  - `docker compose down -v` → perte DB
  - Clés AWS dans git (credentials dans `.env.dist`)
  - Xdebug non connecté
  - Permission denied sur files/
  - Conflits de ports entre projets
  - Composer lent (pas de cache monté)
  - UUID site après site:install
  - webpack_theming chemin incorrect
  - Page blanche sans erreur

---

## Compatibilité

| Skill version | Docker Compose | MariaDB | PHP | Drupal |
|--------------|---------------|---------|-----|--------|
| v1.0–v1.2 | v2 (plugin) | 11.0.2 | 8.1–8.3 | D10, D11 |
| v1.3 | v2 (plugin) | 11.4 LTS | 8.3, 8.4 | D10.3+, D11 |

> Drupal 11 exige PHP 8.3 minimum. PHP 8.4 est supporté depuis Drupal 10.4 / 11.1.
