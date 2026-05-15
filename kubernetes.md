# Kubernetes — Drupal en Production

## Quand passer de Docker Compose à Kubernetes

| Critère | Docker Compose | Kubernetes |
|---------|---------------|-----------|
| Scaling horizontal PHP | ❌ Manuel | ✅ HPA automatique |
| Self-healing (restart pods) | 🟡 `restart: always` basique | ✅ Contrôleur de réplication |
| Rolling deploys sans downtime | ❌ | ✅ |
| Secrets chiffrés | ❌ `.env` en clair | ✅ `Secret` (+ Vault) |
| Multi-environnements (dev/staging/prod) | 🟡 Fichiers override | ✅ Namespaces |
| Usage typique Drupal | projets mono-serveur, dev local | projets haute dispo, multi-équipe |

---

## Structure `.docker_kube/` — Pattern Réel

```
.docker_kube/
├── namespace.yaml
├── configmap.yaml          # Variables non-sensibles
├── secret.yaml             # DB passwords, clés API (à chiffrer avec Sealed Secrets)
├── deployment.yaml         # Pod PHP/Apache
├── service.yaml            # Exposition interne du pod
├── ingress.yaml            # Exposition externe (domaine + TLS)
├── pvc.yaml                # Volume persistant pour les fichiers Drupal
└── hpa.yaml                # Auto-scaling (optionnel)
```

---

## ConfigMap — Variables Non-Sensibles

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: drupal-config
  namespace: mon-projet
data:
  MARIADB_HOSTNAME: "mariadb-service"
  MARIADB_PORT: "3306"
  MARIADB_DATABASE: "drupal"
  PHP_VERSION: "8.3"
  DRUPAL_ENV: "production"
```

---

## Secret — Variables Sensibles

```yaml
# secret.yaml (les valeurs sont en base64 : echo -n "valeur" | base64)
apiVersion: v1
kind: Secret
metadata:
  name: drupal-secrets
  namespace: mon-projet
type: Opaque
data:
  MARIADB_USER: ZHJ1cGFs           # drupal
  MARIADB_PASSWORD: c2VjcmV0MTIz   # secret123
  MARIADB_ROOT_PASSWORD: cm9vdA==  # root
  DRUPAL_HASH_SALT: <base64>
```

```bash
# Générer une valeur base64
echo -n "mon_mot_de_passe" | base64

# Appliquer
kubectl apply -f secret.yaml

# Vérifier (sans voir les valeurs)
kubectl get secret drupal-secrets -n mon-projet
```

---

## Deployment — Pod PHP/Apache

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drupal-php
  namespace: mon-projet
spec:
  replicas: 2   # Toujours ≥2 pour la haute dispo
  selector:
    matchLabels:
      app: drupal-php
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0   # Zéro downtime
  template:
    metadata:
      labels:
        app: drupal-php
    spec:
      containers:
        - name: php
          image: registry.gitlab.example.com/mon-projet/drupal-php:${CI_COMMIT_SHA}
          ports:
            - containerPort: 80
          envFrom:
            - configMapRef:
                name: drupal-config
            - secretRef:
                name: drupal-secrets
          volumeMounts:
            - name: drupal-files
              mountPath: /var/www/html/web/sites/default/files
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /core/install.php
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 5
      volumes:
        - name: drupal-files
          persistentVolumeClaim:
            claimName: drupal-files-pvc
```

---

## PersistentVolumeClaim — Fichiers Drupal

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: drupal-files-pvc
  namespace: mon-projet
spec:
  accessModes:
    - ReadWriteMany   # ← OBLIGATOIRE pour plusieurs replicas (NFS ou CephFS)
  storageClassName: nfs-client   # Adapter selon le cluster
  resources:
    requests:
      storage: 10Gi
```

**Attention :** `ReadWriteOnce` (RWO) ne fonctionne pas avec plusieurs replicas. Il faut `ReadWriteMany` (RWX) — vérifier que le cluster dispose d'un StorageClass RWX.

---

## Service — Exposition Interne

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: drupal-service
  namespace: mon-projet
spec:
  selector:
    app: drupal-php
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP   # Interne uniquement — exposé via Ingress
```

---

## Ingress — Domaine + TLS

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: drupal-ingress
  namespace: mon-projet
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "64m"   # Upload max Drupal
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - mon-projet.example.com
      secretName: drupal-tls
  rules:
    - host: mon-projet.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: drupal-service
                port:
                  number: 80
```

---

## settings.php — Lire les Secrets Kubernetes

```php
// web/sites/default/settings.php
// En Kubernetes, les variables d'env viennent de ConfigMap + Secret
$databases['default']['default'] = [
  'driver'   => getenv('DB_DRIVER') ?: 'mysql',
  'host'     => getenv('MARIADB_HOSTNAME') ?: 'mariadb-service',
  'port'     => getenv('MARIADB_PORT') ?: '3306',
  'database' => getenv('MARIADB_DATABASE') ?: 'drupal',
  'username' => getenv('MARIADB_USER'),
  'password' => getenv('MARIADB_PASSWORD'),
  'prefix'   => '',
  'charset'  => 'utf8mb4',
];

$settings['hash_salt'] = getenv('DRUPAL_HASH_SALT');

// trusted_host_patterns — obligatoire en prod
$settings['trusted_host_patterns'] = [
  '^mon-projet\.example\.com$',
];

// Fichiers — le PVC est monté ici
$settings['file_public_path'] = 'sites/default/files';
$settings['file_private_path'] = '/var/www/private';
```

---

## GitLab CI — Déploiement vers Kubernetes

```yaml
# .gitlab-ci.yml — stage deploy
deploy:production:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    # Injecter le SHA de l'image dans le deployment
    - kubectl set image deployment/drupal-php php=${CI_REGISTRY_IMAGE}/drupal-php:${CI_COMMIT_SHA} -n mon-projet
    # Attendre que le rollout soit complet
    - kubectl rollout status deployment/drupal-php -n mon-projet --timeout=5m
    # Post-deploy : drush updb + cim
    - kubectl exec -n mon-projet deployment/drupal-php -- drush deploy -y
  environment:
    name: production
    url: https://mon-projet.example.com
  only:
    - main
```

---

## Commandes kubectl Courantes

```bash
# Voir les pods
kubectl get pods -n mon-projet

# Logs d'un pod
kubectl logs -f deployment/drupal-php -n mon-projet

# Exécuter drush dans un pod
kubectl exec -it deployment/drupal-php -n mon-projet -- drush cr
kubectl exec -it deployment/drupal-php -n mon-projet -- drush cim -y

# Redémarrer les pods (sans downtime — rolling restart)
kubectl rollout restart deployment/drupal-php -n mon-projet

# Voir l'état du rollout
kubectl rollout status deployment/drupal-php -n mon-projet

# Rollback vers la version précédente
kubectl rollout undo deployment/drupal-php -n mon-projet

# Voir les events (debug)
kubectl describe pod <nom-du-pod> -n mon-projet
kubectl get events -n mon-projet --sort-by='.lastTimestamp'
```

---

## Anti-Patterns Kubernetes + Drupal

| ❌ À ne jamais faire | ✅ Bonne pratique | Raison |
|---------------------|------------------|--------|
| PVC `ReadWriteOnce` avec replicas > 1 | `ReadWriteMany` (NFS/Ceph) | Pods ne peuvent pas monter le même PVC RWO |
| Secrets en clair dans git | Sealed Secrets ou Vault | Rotation impossible, audit trail cassé |
| Image `:latest` en production | Tag précis (`${CI_COMMIT_SHA}`) | Impossible de savoir quelle version tourne |
| `drush cim` dans l'entrypoint Docker | Dans le pipeline CI après deploy | Race condition si plusieurs replicas démarrent en même temps |
| Volume `emptyDir` pour les fichiers Drupal | PVC persistant | Données perdues au redémarrage du pod |
| `settings.php` hardcodé | Variables d'env via ConfigMap/Secret | Non portable entre environnements |
