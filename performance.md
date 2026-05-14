# Performance — Bind Mounts et Optimisation

## Le Problème Fondamental sur Mac et Windows

Docker sur Mac/Windows utilise une couche de virtualisation pour les bind mounts (fichiers montés de l'hôte vers le container). Chaque accès au filesystem traverse cette couche → **Drupal charge en 5-15 secondes au lieu de <1s**.

```
Mac/Windows HôTE → (couche virtuelle lente) → Linux VM → Container PHP → site
```

Sur Linux natif, les bind mounts sont directs → pas de problème de performance.

---

## Solution 1 — Mutagen (avec mutagen-compose)

Mutagen synchronise les fichiers de manière asynchrone entre l'hôte et le container.

⚠️ **`x-mutagen` nécessite `mutagen-compose`**, pas le standard `docker compose`. Installer l'outil séparé.

```bash
# Installation de mutagen-compose (macOS)
brew install mutagen-io/mutagen/mutagen mutagen-io/mutagen/mutagen-compose

# Utiliser mutagen-compose à la place de docker compose
mutagen-compose up -d
mutagen-compose exec php drush cr
```

```yaml
# docker-compose.yml — config x-mutagen (pour mutagen-compose uniquement)
services:
  php:
    volumes:
      - drupal-sync:/var/www/html   # ← Volume synchronisé par Mutagen

volumes:
  drupal-sync:
    x-mutagen:
      sync:
        defaults:
          symlink:
            mode: ignore
          watching:
            pollingInterval: 10
          ignore:
            vcs: true
            paths:
              - vendor
              - web/core
              - node_modules
              - .git
```

---

## Solution 2 — DDEV (Gère Mutagen automatiquement)

DDEV configure automatiquement Mutagen sur Mac/Windows :

```yaml
# .ddev/config.yaml
mutagen_enabled: true   # Active Mutagen pour les performances
```

```bash
ddev start   # Démarre avec Mutagen automatiquement
ddev drush status
```

---

## Solution 3 — Docker Desktop VirtioFS (Apple Silicon)

Sur les Mac M1/M2/M3, Docker Desktop utilise VirtioFS par défaut qui est beaucoup plus rapide que gRPC FUSE.

Vérifier dans Docker Desktop → Settings → General → "Use VirtioFS for file sharing"

---

## OPcache — Configuration Critique

L'OPcache doit être activé ET bien configuré pour éviter les recompilations PHP à chaque requête :

```ini
# php.ini production (dans le container)
opcache.enable = 1
opcache.enable_cli = 1
opcache.memory_consumption = 256       # MB alloués
opcache.max_accelerated_files = 20000  # Nombre max de fichiers en cache
opcache.revalidate_freq = 0            # 0 = jamais revalider (prod)
opcache.validate_timestamps = 0        # 0 = ne pas vérifier les timestamps (prod)
opcache.save_comments = 1             # Requis par Drupal (annotations)
opcache.interned_strings_buffer = 16
```

```ini
# php.ini développement — revalider à chaque requête
opcache.enable = 1
opcache.revalidate_freq = 0     # 0 mais avec validate_timestamps=1
opcache.validate_timestamps = 1 # Vérifier si le fichier a changé
```

**Sans OPcache ou avec mauvaise config → Drupal recompile 200+ fichiers PHP à chaque requête.**

---

## Docker Build Cache — Ne pas Réinstaller Composer à Chaque Build

**Problème :** si `composer.json` change, Docker repart de la layer RUN composer install et tout se recompile.

**Solution :** copier d'abord `composer.json/composer.lock`, installer, PUIS copier le code source.

```dockerfile
# ❌ LENT — Si le code change, tout composer install re-tourne
COPY . /var/www/html
RUN composer install

# ✅ OPTIMAL — Docker cache la layer composer si les .json/.lock ne changent pas
COPY composer.json composer.lock /var/www/html/
RUN composer install --no-scripts --no-plugins --optimize-autoloader
COPY . /var/www/html
RUN composer run-script post-install-cmd   # Scripts Composer séparément
```

---

## APCu — Cache Objet PHP

```ini
# php.ini
apc.enabled = 1
apc.shm_size = 64M         # Mémoire partagée APCu
apc.ttl = 7200             # TTL par défaut 2h
apc.enable_cli = 0         # Ne pas utiliser en CLI (Drush)
```

```php
// settings.php — utiliser APCu pour le cache Drupal (si disponible)
if (extension_loaded('apcu') && PHP_SAPI !== 'cli') {
  $settings['cache']['bins']['bootstrap'] = 'cache.backend.apcu';
  $settings['cache']['bins']['config']    = 'cache.backend.apcu';
  $settings['cache']['bins']['discovery'] = 'cache.backend.apcu';
}
```

---

## Redis — Cache Distribué (Projets Multi-Containers)

```yaml
# docker-compose.yml
redis:
  image: redis:7-alpine
  ports:
    - "6379:6379"
  command: ["redis-server", "--maxmemory", "256mb", "--maxmemory-policy", "allkeys-lru"]
```

```php
// settings.php — activer Redis comme backend de cache
$settings['redis.connection']['interface'] = 'PhpRedis';
$settings['redis.connection']['host']      = 'redis';   // nom du service Docker
$settings['cache']['default']              = 'cache.backend.redis';
$settings['cache']['bins']['config']       = 'cache.backend.database';  // Config pas dans Redis
```

---

## Profilage — Identifier les Goulots

```bash
# Voir la consommation ressources en temps réel
docker compose stats

# Dans le container — profiler une requête Drupal
docker compose exec php drush php-eval "
  \$start = microtime(true);
  \Drupal::service('entity_type.manager')->getStorage('node')->loadMultiple();
  echo round((microtime(true) - \$start) * 1000) . 'ms';
"

# Activer le module Devel Kint (local uniquement)
docker compose exec php drush en devel kint -y
```

---

## Comparison des Solutions Mac/Windows

| Solution | Setup | Performance | Fiabilité |
|----------|-------|-------------|-----------|
| Bind mount direct | ✅ Aucun | ❌ 5-15s/page | ✅ Parfaite sync |
| VirtioFS (Apple M1+) | ✅ Automatique | 🟡 2-5s/page | ✅ Bonne |
| Mutagen + Docker | 🟡 Moyen | ✅ <1s/page | 🟡 Sync async |
| DDEV | ✅ Simple | ✅ <1s/page | ✅ Très bonne |
| Linux natif | ✅ Aucun | ✅ <0.5s/page | ✅ Parfaite |
