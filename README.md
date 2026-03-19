- charts: Los charts que no puedo pillar directamente desde git/ helm
- manifests: helmreleases.yaml de cada aplicacion los manifiestos de cada aplicacion
- resources: Van los values.yaml que vamos a referenciar desde los helmreleases
- revissions: Van las revisiones de argo de los stacks enteros de componentes y lo unico que tengo que aplicar

## Sealed Secrets (Minikube + ArgoCD)

Objetivo: no guardar passwords en texto plano en git.

Flujo:
1. Crear un `Secret` normal en memoria (`--dry-run=client`).
2. Cifrarlo con `kubeseal` para generar un `SealedSecret`.
3. Subir el `SealedSecret` al repo para que ArgoCD lo aplique.

### 1) Instalar Sealed Secrets Controller en Minikube

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.2/controller.yaml
kubectl -n kube-system rollout status deploy/sealed-secrets-controller --timeout=180s
```

### 2) Instalar kubeseal CLI

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

### 3) Crear un SealedSecret (ejemplo)

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

Notas:
- `keycloak-admin-creds` y `namespace keycloak` deben coincidir con lo que consume el manifiesto (`secretKeyRef`).
- El archivo generado debe tener `kind: SealedSecret`.

### 4) Validar y subir

```bash
kustomize build manifests/apps/configcli/base >/tmp/configcli-check.yaml
git add manifests/apps/configcli/base/secret.yaml
git commit -m "chore(security): update sealed secret"
git push origin main
```

### Ejemplo usado en este repo

- Sealed secret de Helm Keycloak: `resources/apps/keycloak/secrets/keycloak-helm-sealedsecret.yaml`
- Values sin passwords en plano: `resources/apps/keycloak/values.yaml`