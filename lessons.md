# Leçons — drupal-docker

Problèmes réels rencontrés sur les projets Drupal containerisés. Mis à jour après chaque incident.

---

## 2026-05-14 — Création du skill (basé sur projets réels)

### `localhost` dans settings.php → DB connexion refusée
- **Symptôme :** "SQLSTATE[HY000] [2002] Connection refused" au démarrage
- **Cause :** `'host' => 'localhost'` — dans le container PHP, `localhost` = le container lui-même, pas la DB
- **Correct :** `'host' => getenv('MARIADB_HOSTNAME') ?: 'database'` — `database` = nom du service Docker
- **Prévention :** Toujours utiliser le nom du service Docker comme hostname DB. Jamais `localhost` ni `127.0.0.1`

### `AllowOverride None` dans Apache → Drupal 404 sur tout sauf la home
- **Symptôme :** La page d'accueil charge mais toutes les autres routes retournent 404
- **Cause :** Apache ignore le `.htaccess` de Drupal qui gère le routing Drupal
- **Correct :** Dans le Dockerfile ou la config Apache : `AllowOverride All` + `a2enmod rewrite`
- **Prévention :** Toujours inclure dans le Dockerfile : `sed -ri -e 's!AllowOverride None!AllowOverride All!g' /etc/apache2/apache2.conf` + `RUN a2enmod rewrite`

### `docker compose down -v` → Perte de la base de données
- **Symptôme :** Après `down -v`, la DB est vide au prochain `up`
- **Cause :** `-v` supprime les named volumes incluant `database_data` où MariaDB stocke ses données
- **Correct :** `docker compose down` (sans `-v`) pour arrêter sans perdre les données
- **Prévention :** Créer des dumps régulièrement avec `make db-dump`. Documenter dans README que `-v` = perte DB

### Clés AWS réelles dans `.env.dist` → Exposition dans git
- **Symptôme :** Credentials exposés dans l'historique git (cas réel sur francetvpro.fr)
- **Cause :** `.env.dist` committé avec des valeurs réelles au lieu de placeholders
- **Correct :** Toujours des placeholders dans `.env.dist` — jamais de valeurs réelles sensibles
- **Prévention :** Ajouter `git-secrets` ou un hook pre-commit qui détecte les patterns de credentials

### Xdebug ne se connecte pas à l'IDE
- **Symptôme :** Breakpoints ignorés, aucune connexion Xdebug détectée
- **Cause 1 :** `xdebug.client_host = localhost` — localhost dans le container ≠ la machine hôte
- **Cause 2 :** `extra_hosts: host.docker.internal:host-gateway` absent du docker-compose.yml
- **Cause 3 :** L'IDE n'écoute pas sur le port 9003 avant la requête
- **Correct :** `xdebug.client_host = host.docker.internal` + `extra_hosts` dans docker-compose.yml + démarrer l'écoute IDE en premier
- **Prévention :** Vérifier : `docker compose exec php php -r "echo system('nc -zv host.docker.internal 9003');"`

### `Permission denied` sur `web/sites/default/files/` sur Linux
- **Symptôm :** Upload de fichiers impossible, Drupal ne peut pas écrire dans `files/`
- **Cause :** Le container PHP tourne avec UID 1000 mais les fichiers appartiennent à `root` ou à un UID différent
- **Correct :** `sudo chown -R 1000:www-data web/sites/default/files && sudo chmod -R g+w web/sites/default/files`
- **Prévention :** Ajouter `make install-linux` dans le README. Documenter que `DEV_UID/DEV_GID` doit correspondre à `id -u` et `id -g` sur la machine hôte

### Conflits de ports entre projets (plusieurs projets actifs en même temps)
- **Symptôme :** `Error: Bind for 0.0.0.0:80 failed: port is already allocated`
- **Cause :** Deux projets avec le même port exposé (`HTTPD_PORT=80`) démarrés en même temps
- **Correct :** Attribuer des ports uniques par projet dans `.env.dist` (ex: 80, 81, 82, 83...)
- **Prévention :** Convention d'équipe pour les ports. Ex: cci=80, vyv=81, canut=82, france=83

### `composer install` très lent dans le container
- **Symptôme :** `composer install` prend 5-10 minutes à chaque `make install`
- **Cause :** Le cache Composer n'est pas monté depuis l'hôte — tout re-télécharge
- **Correct :** Monter `${PERSONAL_GLOBAL_COMPOSER_FOLDER:-~/.composer}:/home/devuser/.composer` dans docker-compose.yml
- **Prévention :** Toujours inclure le volume Composer hôte. Vérifier que `PERSONAL_GLOBAL_COMPOSER_FOLDER=~/.composer` est dans `.env.dist`

### UUID du site ne correspond pas après `site:install` → `drush cim` échoue
- **Symptôme :** "Site UUID in source storage does not match the site UUID in target storage"
- **Cause :** `drush site:install` génère un nouveau UUID de site qui ne correspond pas au `config/sync/system.site.yml`
- **Correct :** `drush cset system.site uuid "$(grep uuid config/sync/system.site.yml | awk '{print $2}' | tr -d "'")" -y`
- **Prévention :** Inclure la synchronisation UUID dans `make install` (comme dans le Makefile template)

### Container `webpack_theming` ne voit pas les changements
- **Symptôme :** CSS/JS non mis à jour malgré `npm run watch` actif
- **Cause :** Le container Node.js monte la racine mais le `working_dir` est incorrect
- **Correct :** Vérifier que `working_dir` dans docker-compose.yml correspond exactement au chemin du thème
- **Prévention :** `docker compose logs webpack_theming -f` pour voir les erreurs de watch

### MariaDB pas encore prête → `drush site:install` échoue aléatoirement
- **Symptôme :** `make install` échoue avec "Connection refused" ou "Access denied" parfois, pas d'autres fois
- **Cause :** MariaDB prend 5-30s à démarrer. `docker compose up -d` rend la main immédiatement mais la DB n'est pas opérationnelle
- **Correct :** Ajouter `healthcheck` + `start_period: 30s` sur le service database + `depends_on: condition: service_healthy` sur PHP. Et/ou ajouter `wait-db` dans le Makefile
- **Prévention :** Toujours ajouter `healthcheck` sur MariaDB et `depends_on: service_healthy` sur PHP dans docker-compose.yml

### `drush site:install` crée des fichiers appartenant à `root`
- **Symptôme :** Après install, les fichiers dans `web/sites/default/files/` appartiennent à `root` → `Permission denied` à l'usage
- **Cause :** `docker compose exec php drush site:install` tourne en tant que `root` dans le container par défaut
- **Correct :** `docker compose exec --user www-data php drush site:install` — ou `make install-linux` en post-install
- **Prévention :** Documenter que toutes les commandes Drush qui créent des fichiers doivent utiliser `--user www-data`

### Build Docker lent (500MB+ de contexte)
- **Symptôme :** `docker compose build` prend 2-5 minutes même pour un changement mineur dans le Dockerfile
- **Cause :** Absence de `.dockerignore` — tout le projet (vendor, node_modules, .git) est envoyé comme contexte de build
- **Correct :** Créer `.dockerignore` à la racine du projet
- **Prévention :** `.dockerignore` TOUJOURS créé en même temps que le Dockerfile. Vérifier la taille du contexte avec `docker build . 2>&1 | head -3`

### Page Drupal blanche sans erreur visible
- **Symptôme :** HTTP 500, page blanche, aucun message d'erreur Drupal
- **Cause :** `error_level` à `hide` en local, ou erreur PHP fatale avant l'initialisation de Drupal
- **Correct :** 1) `docker compose logs php -f` pour voir les erreurs PHP/Apache 2) Vérifier `web/sites/default/settings.local.php` a `error_level = verbose`
- **Prévention :** settings.local.php avec `$config['system.logging']['error_level'] = 'verbose'` dans le README d'installation
