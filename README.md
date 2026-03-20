# Proyecto Keycloak (GitOps con ArgoCD + Helm)

Este repositorio despliega Keycloak en Kubernetes (minikube en entorno local) usando GitOps con ArgoCD y el Helm Chart v25.2.0 de Bitnami.

## Objetivo

- Desplegar Keycloak con Helm Chart de Bitnami (25.2.0).
- Bootstrap automático de realm `demo` mediante `keycloakConfigCli` (hook PostSync del chart).
- Gestionar secretos sin texto plano en git usando Sealed Secrets.
- Mantener configuración versionada en git (valores Helm y manifiestos).
- Infraestructura reproducible: desplegar en cualquier cluster con SealedSecrets Controller.

## Estructura del repositorio

```text
manifests/
  apps/
    __overlays/
      kustomization.yaml          # Orquestador: incluye keycloak/lab
    keycloak/
      base/
        helmrelease.yaml          # Application ArgoCD (3 sources)
      lab/
        kustomization.yaml
resources/
  apps/
    keycloak/
      values.yaml                 # Configuración Helm: realm, env, images
      secrets/
        keycloak-helm-sealedsecret.yaml  # Credenciales encriptadas
revissions/
  apps-revission.yaml             # App raíz (app-of-apps)
```

- **`manifests/`**: Applications ArgoCD y Kustomize overlays.
- **`resources/`**: Valores Helm, secretos encriptados, configuración compartida.
- **`revissions/`**: Manifiesto raíz (`apps-keycloak`), punto de entrada del despliegue.

## Arquitectura GitOps

### 1) App raíz (app-of-apps)

**Archivo**: `revissions/apps-revission.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps-keycloak
  namespace: argocd
spec:
  source:
    repoURL: <tu-repo>
    path: manifests/apps/__overlays
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

- Punto de entrada único del stack.
- Referencia overlay raíz que orquesta todas las apps hijas.
- `syncPolicy.automated` con `prune` y `selfHeal` habilitados.

### 2) Overlay raíz

**Archivo**: `manifests/apps/__overlays/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../keycloak/lab
```

- Agrupa todas las aplicaciones Keycloak.
- Referencia solo `keycloak/lab` (el configcli está ahora dentro del chart).

### 3) App Keycloak (multi-source)

**Archivo**: `manifests/apps/keycloak/base/helmrelease.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: keycloak
  namespace: argocd
spec:
  sources:
    # Source 1: Bitnami Helm Chart
    - repoURL: https://charts.bitnami.com/bitnami
      chart: keycloak
      targetRevision: "25.2.0"
      helm:
        releaseName: keycloak
        valueFiles:
          - $values/resources/apps/keycloak/values.yaml
    
    # Source 2: Sealed Secrets (credenciales encriptadas)
    - repoURL: <tu-repo>
      path: resources/apps/keycloak/secrets
      targetRevision: main

    # Source 3: Valores Helm
    - repoURL: <tu-repo>
      targetRevision: main
      ref: values

  destination:
    server: https://kubernetes.default.svc
    namespace: keycloak
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

**Flujo de sincronización**:
1. ArgoCD descarga Chart v25.2.0 desde Bitnami.
2. Merge de valores: `values.yaml` del chart + `resources/apps/keycloak/values.yaml`.
3. Aplica `resources/apps/keycloak/secrets/keycloak-helm-sealedsecret.yaml`.
4. SealedSecrets Controller desencripta → crea Secret nativo `keycloak-helm-secrets`.
5. Helm instala release con todas las variables interpoladas.
6. PostSync hook: `keycloakConfigCli` crea realm `demo` si no existe (idempotente).

## Configuración de Keycloak (Helm Chart v25.2.0)

**Archivo**: `resources/apps/keycloak/values.yaml`

### Imágenes

```yaml
image:
  registry: bitnamilegacy
  repository: keycloak
  tag: 26.3.3-debian-12-r0

postgresql:
  image:
    registry: bitnamilegacy
    repository: postgresql
    tag: 17.6.0-debian-12-r0

keycloakConfigCli:
  image:
    registry: bitnamilegacy
    repository: keycloak-config-cli
    tag: 6.4.0-debian-12-r11
```

**Nota**: Todas las imágenes usan registry `bitnamilegacy/` (las imágenes en `bitnami/` tienen issues de manifest unknown en algunos tags).

### Credenciales (Sealed Secrets)

```yaml
auth:
  adminPassword: ""  # Inyectado desde keycloak-helm-secrets
  existingSecret: keycloak-helm-secrets

postgresql:
  auth:
    password: ""  # Inyectado desde keycloak-helm-secrets
    existingSecret: keycloak-helm-secrets
```

**Flujo**:
1. `resources/apps/keycloak/secrets/keycloak-helm-sealedsecret.yaml` contiene credenciales encriptadas.
2. SealedSecrets Controller desencripta → crea Secret `keycloak-helm-secrets` en namespace `keycloak`.
3. Helm chart referencia ese Secret existente (no crea uno nuevo).
4. Las credenciales se inyectan en las variables de entorno del Pod.

### Variables de entorno (Keycloak ↔ PostgreSQL)

```yaml
keycloak:
  extraEnvVars:
    - name: KC_DB
      value: postgres
    - name: KC_DB_URL_HOST
      value: keycloak-postgresql
    - name: KC_DB_URL_PORT
      value: "5432"
    - name: KC_DB_URL_DATABASE
      value: keycloak
    - name: KC_DB_USERNAME
      value: keycloak
    - name: KC_DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: keycloak-helm-secrets
          key: password
    - name: KC_PROXY
      value: edge
    - name: KC_HOSTNAME
      value: keycloak.local
```

**Explicación**:
- `KC_DB`: Driver de base de datos (postgres).
- `KC_DB_URL_*`: Conexión a PostgreSQL (hostname, puerto, BD).
- `KC_DB_USERNAME/PASSWORD`: Credenciales de BD.
- `KC_PROXY`: Confía en headers de reverse proxy (X-Forwarded-*).
- `KC_HOSTNAME`: Hostname usado en redirect URIs y token audience.

### Configuración inicial del realm (keycloakConfigCli)

```yaml
keycloakConfigCli:
  enabled: true
  containerSecurityContext:
    runAsUser: 1001
  
  configuration:
    demo-realm.json: |
      {
        "realm": "demo",
        "enabled": true,
        "clients": [
          {
            "clientId": "demo-app",
            "name": "Demo App",
            "rootUrl": "http://localhost:8080",
            "redirectUris": ["http://localhost:8080/*"],
            "webOrigins": ["*"]
          }
        ],
        "roles": {
          "realm": [
            {"name": "admin"},
            {"name": "user"},
            {"name": "manager"}
          ]
        },
        "clientScopes": [
          {
            "name": "demo-scope",
            "description": "Demo scope"
          }
        ],
        "defaultClientScopes": ["demo-scope"]
      }
```

**Comportamiento**:
- Hook PostSync del chart ejecuta `keycloak-config-cli`.
- Lee configuración de `demo-realm.json`.
- Crea realm `demo` si no existe (idempotente, no falla si existe).
- Scope es: crear/actualizar realm, clientes, roles, scopes.

**Ventajas vs Job manual**:
- ✅ Incluido en el chart (menos manifiestos).
- ✅ Automático con el Helm release.
- ✅ Idempotente (seguro para re-syncs).
- ✅ Versionado con el chart.

Importante:

- Este enfoque es intencionalmente simple.
- No hace reconciliación completa de cambios internos cuando el realm ya existe.

## Seguridad de secretos

**Objetivo**: no guardar passwords en texto plano en git.

### Enfoque: Sealed Secrets + SealedSecret Controller

1. **Git**: `keycloak-helm-sealedsecret.yaml` contiene credenciales **encriptadas**.
2. **Cluster**: SealedSecrets Controller desencripta automáticamente.
3. **Runtime**: Secret de Kubernetes se inyecta en los Pods.

**Ventaja**: Puedes hacer `git clone` y deployar sin miedo. Las credenciales solo son útiles con la **clave privada del cluster**.

```
git (encriptado)
    ↓ [ArgoCD sync]
cluster
    ↓ [SealedSecrets Controller desencripta]
Secret nativo (base64 en etcd)
    ↓ [Inyectado en Pod]
variable de entorno / archivo
```

### Instalación (una sola vez por cluster)

```bash
# 1. Instalar Sealed Secrets Controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.2/controller.yaml
kubectl -n kube-system rollout status deploy/sealed-secrets-controller --timeout=180s

# 2. Instalar el cliente kubeseal
VER=0.27.2
ARCH=$(uname -m | sed 's/x86_64/amd64/' | sed 's/aarch64/arm64/')
curl -fL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${VER}/kubeseal-${VER}-linux-${ARCH}.tar.gz" | tar -xz -C /tmp
sudo install -m 0755 /tmp/kubeseal /usr/local/bin/kubeseal
```

### Crear/actualizar un Sealed Secret

```bash
# Crear un Secret plaintext
kubectl create secret generic keycloak-helm-secrets \
  --namespace keycloak \
  --from-literal=admin-password='changeme' \
  --from-literal=password='keycloak123' \
  --from-literal=postgres-password='keycloak123' \
  --dry-run=client -o yaml \
| kubeseal \
  --controller-name sealed-secrets-controller \
  --controller-namespace kube-system \
  --format yaml \
> resources/apps/keycloak/secrets/keycloak-helm-sealedsecret.yaml

# Commitear a git (está encriptado)
git add resources/apps/keycloak/secrets/keycloak-helm-sealedsecret.yaml
git commit -m "Update sealed secrets"
```

## Despliegue

### 1) Requisitos previos

- Kubernetes cluster (minikube, EKS, GKE, etc.)
- ArgoCD instalado
- Sealed Secrets Controller instalado
- `kubectl` y `kubeseal` configurados

### 2) Desplegar stack completo

```bash
# Clonar repo
git clone https://github.com/GivenCloud/Proyecto-Keycloak.git
cd Proyecto-Keycloak

# Aplicar app raíz (inicia sincronización)
kubectl apply -f revissions/apps-revission.yaml

# Ver progreso
watch kubectl -n argocd get application
```

### 3) Ver estado

```bash
# Apps ArgoCD
kubectl -n argocd get applications -o wide

# Pods Keycloak
kubectl -n keycloak get pods -w

# Servicios
kubectl -n keycloak get svc

# Secrets
kubectl -n keycloak get secrets
```

### 4) Verificar desencriptación de secretos

```bash
# Ver Secret desencriptado (base64)
kubectl -n keycloak get secret keycloak-helm-secrets -o yaml

# Decodificar una contraseña
kubectl -n keycloak get secret keycloak-helm-secrets \
  -o jsonpath='{.data.password}' | base64 -d && echo ""
```

## Validación funcional

### 1) Verificar que Keycloak está respondiendo

```bash
# Port-forward
kubectl -n keycloak port-forward svc/keycloak 8080:80 &

# Probar conexión
curl -s http://localhost:8080/realms/master/.well-known/openid-configuration | jq .issuer
```

### 2) Verificar realm demo creado

```bash
# Obtener token de admin
TOKEN=$(kubectl -n keycloak exec keycloak-0 -- sh -c '
  curl -s -X POST http://localhost:8080/realms/master/protocol/openid-connect/token \
    -d client_id=admin-cli \
    -d username=admin \
    -d "password=$(cat /opt/bitnami/keycloak/secrets/admin-password)" \
    -d grant_type=password \
  | grep -o "access_token\":\"[^\"]*" | cut -d'"' -f3
')

# Verificar realm existe
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/admin/realms/demo | jq .realm

# Verificar client existe
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8080/admin/realms/demo/clients?clientId=demo-app" | jq .[0].clientId

# Verificar roles
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/admin/realms/demo/roles | jq '.[].name'
```

### 3) Verificar PostgreSQL

```bash
# Conectar a la BD
PG_PASS=$(kubectl -n keycloak get secret keycloak-helm-secrets \
  -o jsonpath='{.data.password}' | base64 -d)

kubectl -n keycloak exec keycloak-postgresql-0 -- \
  env PGPASSWORD="$PG_PASS" psql -U keycloak -d keycloak \
  -c "SELECT schema_name FROM information_schema.schemata WHERE schema_name = 'public';"

# Resultado: public (significa que Keycloak inicializó la BD correctamente)
```

## Troubleshooting

### Imagen `ImagePullBackOff`

**Síntoma**: Pod en `ImagePullBackOff`.

**Causa**: Las imágenes en `bitnami/` no existen con esos tags (manifest unknown error).

**Solución**: Usar `bitnamilegacy/` en lugar de `bitnami/`.

```yaml
# ✅ Correcto
image:
  registry: bitnamilegacy

# ❌ Incorrecto  
image:
  registry: bitnami
```

### ArgoCD "another operation is already in progress"

**Síntoma**: Application sincroniza pero queda bloqueada; new syncs fallan.

**Causa**: Operación fantasma (job/pod que falló pero el estado quedó en Running).

**Solución**:

```bash
# Opción 1: Limpiar operación (conserva recursos)
kubectl -n argocd patch application keycloak --type merge \
  -p '{"operation": null}'

# Opción 2: Recrear app (más agresivo)
kubectl -n argocd delete application keycloak --cascade=orphan
kubectl apply -f manifests/apps/keycloak/base/helmrelease.yaml
```

### Keycloak no se sincroniza (stuck en Helm hook)

**Síntoma**: keycloakConfigCli job en `ImagePullBackOff` o `CrashLoopBackOff`.

**Causa**: Hook PostSync fallando (probablemente la imagen del configcli).

**Solución**:

```bash
# Ver logs del job
kubectl -n keycloak logs job/keycloak-keycloak-config-cli --all-containers=true

# Limpiar job viejo
kubectl -n keycloak delete job keycloak-keycloak-config-cli

# Forzar nuevo sync en ArgoCD
argocd app sync apps-keycloak --force
```

### PostgreSQL no está inicializando

**Síntoma**: PVC Bound pero pod en `Pending` o `CrashLoopBackOff`.

**Causa**: Storage class no disponible o credenciales incorrectas en Secret.

**Solución**:

```bash
# Ver storage classes disponibles
kubectl get storageclass

# Ver eventos del pod
kubectl -n keycloak describe pod keycloak-postgresql-0

# Verificar Secret tiene contraseña
kubectl -n keycloak get secret keycloak-helm-secrets -o yaml
```

### SealedSecret no se desencripta

**Síntoma**: SealedSecret aplicado pero Secret no aparece.

**Causa**: Controller parado, clave privada corrupta, o namespace incorrecto.

**Solución**:

```bash
# Verificar controller está running
kubectl -n kube-system get deploy sealed-secrets-controller

# Ver logs del controller
kubectl -n kube-system logs deploy/sealed-secrets-controller

# Recrear sealed secret asegurando namespace correcto
kubectl -n keycloak create secret generic keycloak-helm-secrets \
  --from-literal=password='contraseña' \
  --dry-run=client -o yaml \
| kubeseal --format yaml > sealed.yaml
kubectl apply -f sealed.yaml
```

## Archivos clave

| Archivo | Propósito |
|---------|-----------|
| `revissions/apps-revission.yaml` | Application raíz (app-of-apps) |
| `manifests/apps/__overlays/kustomization.yaml` | Overlay raíz que agrupa apps |
| `manifests/apps/keycloak/base/helmrelease.yaml` | Application Keycloak (multi-source) |
| `manifests/apps/keycloak/lab/kustomization.yaml` | Overlay Keycloak lab (despliegue local) |
| `resources/apps/keycloak/values.yaml` | Configuración Helm (realm, images, env vars) |
| `resources/apps/keycloak/secrets/keycloak-helm-sealedsecret.yaml` | Credenciales encriptadas (Sealed Secrets) |

## Referencias

- [Bitnami Keycloak Chart](https://github.com/bitnami/charts/tree/main/bitnami/keycloak)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Keycloak Admin REST API](https://www.keycloak.org/docs/latest/server_admin/#_admin_rest_api)