# Proyecto Keycloak (GitOps con ArgoCD + Kustomize)

Este repositorio despliega Keycloak en Kubernetes (minikube en entorno local) usando GitOps con ArgoCD.

## Objetivo

- Desplegar Keycloak con chart de Bitnami (sin valores embebidos en el `Application`).
- Bootstrap inicial de configuración del realm `demo` mediante un `Job` PostSync.
- Gestionar secretos sin texto plano usando Sealed Secrets.
- Mantener una estructura simple y mantenible.

## Estructura del repositorio

```text
charts/
manifests/
	apps/
		__overlays/
		keycloak/
			base/
			lab/
		configcli/
			base/
			lab/
resources/
	apps/
		keycloak/
			values.yaml
			secrets/
revissions/
	apps-revission.yaml
```

- `manifests/`: manifiestos Kubernetes y apps ArgoCD.
- `resources/`: valores Helm y recursos complementarios reutilizados por ArgoCD.
- `revissions/`: manifiesto raíz (`apps-keycloak`) que se aplica para arrancar todo el stack.

## Arquitectura GitOps

### 1) App raíz (app-of-apps)

Archivo: `revissions/apps-revission.yaml`

- Crea el `Application` `apps-keycloak` en namespace `argocd`.
- Fuente: `manifests/apps/__overlays`.
- `syncPolicy.automated` activa `prune` y `selfHeal`.

### 2) Overlay de apps

Archivo: `manifests/apps/__overlays/kustomization.yaml`

- Incluye:
	- `../keycloak/lab`
	- `../configcli/lab`

### 3) App hija de Keycloak

Archivo: `manifests/apps/keycloak/base/helmrelease.yaml`

- `Application` ArgoCD con `sources` múltiples:
	- Chart remoto `bitnami/keycloak`.
	- Manifiestos del repo (`manifests/apps/keycloak`).
	- Sealed Secrets de Helm (`resources/apps/keycloak/secrets`).
	- Referencia `ref: values` para consumir `resources/apps/keycloak/values.yaml`.

## Configuración de Keycloak (Helm)

Archivo: `resources/apps/keycloak/values.yaml`

Puntos relevantes:

- Imagen Keycloak: `docker.io/bitnamilegacy/keycloak:26.3.3-debian-12-r0`.
- PostgreSQL embebido del chart con imagen `bitnamilegacy/postgresql`.
- Credenciales por `existingSecret: keycloak-helm-secrets`.
- Service `ClusterIP` en puerto `80`.

Nota:

- Se migró a `bitnamilegacy/*` por errores de `manifest unknown` en imágenes/tags originales.

## App configcli (bootstrap del realm)

### Recursos base

Carpeta: `manifests/apps/configcli/base/`

- `configmap.yaml`: contiene `demo-realm.json` (realm, roles, cliente `demo-app`).
- `job.yaml`: Job PostSync que crea el realm si no existe.
- `serviceaccount.yaml`: SA dedicada `keycloak-configcli`.
- `secret.yaml`: `SealedSecret` de credenciales admin para el job.

### Comportamiento actual del Job

Archivo: `manifests/apps/configcli/base/job.yaml`

- Espera a Keycloak con un `initContainer` (`/realms/master`) y timeout explícito.
- Se autentica contra `master`.
- Si `demo` existe: termina en éxito con mensaje `Nothing to do`.
- Si `demo` no existe: crea el realm con `demo-realm.json`.

Importante:

- Este enfoque es intencionalmente simple.
- No hace reconciliación completa de cambios internos cuando el realm ya existe.

## Seguridad de secretos

Objetivo: no guardar passwords en texto plano en git.

Se usan dos secretos sellados:

- `manifests/apps/configcli/base/secret.yaml` (`keycloak-admin-creds`)
- `resources/apps/keycloak/secrets/keycloak-helm-sealedsecret.yaml` (`keycloak-helm-secrets`)

## Sealed Secrets (Minikube + ArgoCD)

### 1) Instalar controller

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.2/controller.yaml
kubectl -n kube-system rollout status deploy/sealed-secrets-controller --timeout=180s
```

### 2) Instalar `kubeseal`

```bash
set -e
ARCH=$(uname -m)
case "$ARCH" in
	x86_64) ARCH=amd64 ;;
	aarch64) ARCH=arm64 ;;
	*) echo "Arquitectura no soportada: $ARCH"; exit 1 ;;
esac

VER=0.27.2
curl -fL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${VER}/kubeseal-${VER}-linux-${ARCH}.tar.gz" -o /tmp/kubeseal.tgz
tar -xzf /tmp/kubeseal.tgz -C /tmp
sudo install -m 0755 /tmp/kubeseal /usr/local/bin/kubeseal
kubeseal --version
```

### 3) Crear/update de secreto sellado (ejemplo)

```bash
kubectl create secret generic keycloak-admin-creds \
	--namespace keycloak \
	--from-literal=password='CAMBIA_ESTE_VALOR' \
	--dry-run=client -o yaml \
| kubeseal \
	--controller-name sealed-secrets-controller \
	--controller-namespace kube-system \
	--format yaml \
> manifests/apps/configcli/base/secret.yaml
```

## Flujo de despliegue

### Aplicar stack completo

```bash
kubectl apply -f revissions/apps-revission.yaml
```

### Ver estado ArgoCD

```bash
kubectl -n argocd get applications
kubectl -n argocd get application apps-keycloak -o jsonpath='{.status.sync.status}{"\n"}{.status.health.status}{"\n"}'
```

### Ver estado de Keycloak

```bash
kubectl -n keycloak get pods
kubectl -n keycloak get svc
```

### Ver estado del job de bootstrap

```bash
kubectl -n keycloak get job keycloak-configcli
kubectl -n keycloak logs job/keycloak-configcli --all-containers=true
```

## Validación funcional

Comprobar realm/roles/client por API desde el pod de Keycloak:

```bash
kubectl -n keycloak exec keycloak-0 -- sh -lc '
TOKEN=$(curl -s -X POST http://localhost:8080/realms/master/protocol/openid-connect/token \
	-d client_id=admin-cli -d username=admin \
	-d password=$(cat /opt/bitnami/keycloak/secrets/admin-password) \
	-d grant_type=password | sed -n "s/.*\"access_token\":\"\([^\"]*\)\".*/\1/p")

echo "realm demo:" $(curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $TOKEN" http://localhost:8080/admin/realms/demo)
for r in admin user manager; do
	echo "role $r:" $(curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $TOKEN" http://localhost:8080/admin/realms/demo/roles/$r)
done
echo "client demo-app bytes:" $(curl -s -H "Authorization: Bearer $TOKEN" "http://localhost:8080/admin/realms/demo/clients?clientId=demo-app" | wc -c)
'
```

## Troubleshooting aplicado en este proyecto

### `kustomize create --autodetect --recursive` no hacía cambios

- Causa: no modifica `kustomization.yaml` existentes y/o no detecta manifiestos en paths esperados.
- Solución: definir `kustomization.yaml` explícitos por base/lab/overlay.

### `ImagePullBackOff` en Keycloak/PostgreSQL

- Causa: tags no disponibles en repo original.
- Solución: usar imágenes `bitnamilegacy/*` con tags validados.

### Sync bloqueado en ArgoCD (`another operation is already in progress`)

- Causa: operación fantasma esperando hook que ya no existía.
- Solución operativa usada:
	- `kubectl -n argocd delete application apps-keycloak --cascade=orphan`
	- `kubectl -n argocd apply -f revissions/apps-revission.yaml`

### Job atascado esperando readiness

- Causa: endpoint `/health/ready` devolvía 404 en este despliegue.
- Solución: esperar sobre `/realms/master`.

### `jq: command not found` en el Job

- Solución: parseo de token sin dependencia de `jq`.

### Complejidad excesiva del script

- Decisión final: simplificar a bootstrap mínimo y estable.

## Estado actual y limitaciones

Estado actual:

- `apps-keycloak` sincroniza correctamente.
- Keycloak y PostgreSQL levantan en `keycloak` namespace.
- El job `keycloak-configcli` funciona para crear `demo` si no existe.

Limitación actual (consciente):

- El Job no reconcilia cambios finos de `demo-realm.json` cuando el realm ya existe.

## Archivos clave

- App raíz ArgoCD: `revissions/apps-revission.yaml`
- Overlay apps: `manifests/apps/__overlays/kustomization.yaml`
- App Keycloak (ArgoCD multi-source): `manifests/apps/keycloak/base/helmrelease.yaml`
- Values Helm Keycloak: `resources/apps/keycloak/values.yaml`
- Sealed secret Helm: `resources/apps/keycloak/secrets/keycloak-helm-sealedsecret.yaml`
- Config realm: `manifests/apps/configcli/base/configmap.yaml`
- Job bootstrap realm: `manifests/apps/configcli/base/job.yaml`
- Sealed secret job admin: `manifests/apps/configcli/base/secret.yaml`