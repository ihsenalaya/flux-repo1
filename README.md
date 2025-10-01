
# GitOps AKS – cert-manager, ExternalDNS, External Secrets (Workload Identity + UAMI)

Ce guide décrit **toutes les étapes et modifications** pour faire fonctionner une solution GitOps (FluxCD + Kustomize + Helm) sur AKS avec :
- **cert-manager** (ACME Let’s Encrypt, HTTP-01 via Traefik)
- **external-dns** (Azure DNS)
- **external-secrets** (Azure Key Vault)
- **Workload Identity** **avec** **User-Assigned Managed Identity (UAMI)**

La structure **base/** (commune) + **clusters/<env>/** (overlays) est respectée.

---

## 0) Prérequis

- AKS ≥ 1.25 recommandé, avec **OIDC issuer** et **Workload Identity** activés.
- Traefik (ou autre Ingress Controller) déjà déployé et fonctionnel (classe `traefik`).
- FluxCD installé et relié à ce dépôt (via `flux bootstrap` ou équivalent).
- Azure CLI connecté avec des droits suffisants (Owner/Contributor au minimum sur les scopes nécessaires).
- Vous disposez de :
  - `SUB_ID` (ID Subscription)
  - `TENANT_ID` (ID Tenant)
  - Ressource Group du cluster AKS: `RG_AKS`, nom du cluster `AKS_NAME`
  - Ressource Group des identities: `RG_IDENTITY`
  - Ressource Group qui contient la/les zone(s) DNS: `DNS_ZONE_RG`
  - Nom du Key Vault: `KV_NAME`
  - Environnement cible d’exemple: `staging`

> Si OIDC/WI ne sont pas encore actifs :
```bash
az aks update -g $RG_AKS -n $AKS_NAME \
  --enable-oidc-issuer --enable-workload-identity
```

Récupérer l’issuer OIDC du cluster :
```bash
OIDC=$(az aks show -g $RG_AKS -n $AKS_NAME --query "oidcIssuerProfile.issuerUrl" -o tsv)
echo "$OIDC"
```

---

## 1) Arborescence GitOps cible

```
.
├─ base/
│  ├─ flux/helmrepositories/
│  │  ├─ cert-manager.yaml
│  │  ├─ external-dns.yaml
│  │  └─ external-secrets.yaml
│  ├─ namespaces/
│  │  ├─ cert-manager.yaml
│  │  ├─ external-dns.yaml
│  │  └─ external-secrets.yaml
│  └─ platform/
│     ├─ cert-manager/
│     │  ├─ helmrelease.yaml
│     │  └─ kustomization.yaml
│     ├─ external-dns/
│     │  ├─ helmrelease.yaml      # SA créé en overlay → create:false
│     │  └─ kustomization.yaml
│     └─ external-secrets/
│        ├─ helmrelease.yaml      # SA créé en overlay → create:false
│        └─ kustomization.yaml
└─ clusters/
   └─ staging/
      ├─ kustomization.yaml        # inclut toutes les Kustomization CRs
      ├─ kustomizations/
      │  ├─ cert-manager.yaml
      │  ├─ cert-issuers.yaml      # ← Kustomization séparée pour les Issuers
      │  ├─ external-dns.yaml
      │  └─ external-secrets.yaml
      └─ platform/
         ├─ cert-manager/
         │  ├─ kustomization.yaml
         │  └─ clusterissuers.yaml
         ├─ external-dns/
         │  ├─ kustomization.yaml
         │  ├─ values-staging.yaml
         │  └─ sa.yaml             # SA avec annotation/label WI (UAMI)
         └─ external-secrets/
            ├─ kustomization.yaml
            ├─ values-staging.yaml
            ├─ sa.yaml             # SA avec annotation/label WI (UAMI)
            └─ clustersecretstore-azure.yaml
```

---

## 2) Modifications **nécessaires** côté manifests (par rapport à une base “classique”)

### 2.1 HelmRelease – API version
Utiliser **`apiVersion: helm.toolkit.fluxcd.io/v2`** (stable) pour **tous** les HelmRelease.

### 2.2 Workload Identity – ServiceAccounts **séparés** en overlay
- Dans les `HelmRelease` de **external-dns** et **external-secrets**, mettre :
  ```yaml
  values:
    serviceAccount:
      create: false
      name: <nom-SA>
  ```
- Créer le **ServiceAccount** dans l’overlay **clusters/<env>/** avec :
  - **label** `azure.workload.identity/use: "true"`
  - **annotation** `azure.workload.identity/client-id: "<CLIENT_ID_DE_LA_UAMI>"`

> Les charts ne permettent pas toujours d’ajouter le **label** requis, d’où la création du SA **en dehors** d’Helm.

### 2.3 External Secrets – lier le ClusterSecretStore au SA
Dans `ClusterSecretStore`, ajouter :
```yaml
spec:
  provider:
    azurekv:
      authType: WorkloadIdentity
      serviceAccountRef:
        name: external-secrets
        namespace: external-secrets
```

### 2.4 Cert-Manager – séparer l’installation des Issuers
Les CRDs de cert-manager doivent être prêts avant d’appliquer les `ClusterIssuer`.  
Créer une **Kustomization dédiée** (`cert-issuers`) qui **dependsOn** `cert-manager`.

---

## 3) Contenu des manifests (extraits prêts à coller)

> **Base** (commun) – *inchangé sauf `apiVersion` v2 et SA create:false*

**`base/platform/external-dns/helmrelease.yaml`**
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: external-dns
  namespace: external-dns
spec:
  interval: 10m
  chart:
    spec:
      chart: external-dns
      version: ">=1.15.0"
      sourceRef:
        kind: HelmRepository
        name: external-dns
        namespace: flux-system
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
  values:
    sources: ["ingress"]
    provider: azure
    registry: txt
    policy: sync
    serviceAccount:
      create: false          # ← SA fourni par l’overlay
      name: external-dns
    extraArgs:
      - --interval=1m
      - --log-format=json
```

**`base/platform/external-secrets/helmrelease.yaml`**
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: external-secrets
  namespace: external-secrets
spec:
  interval: 10m
  chart:
    spec:
      chart: external-secrets
      version: ">=0.9.0"
      sourceRef:
        kind: HelmRepository
        name: external-secrets
        namespace: flux-system
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
  values:
    installCRDs: true
    serviceAccount:
      create: false          # ← SA fourni par l’overlay
      name: external-secrets
```

> **Overlays – staging**

**External-DNS – SA + valeurs**

`clusters/staging/platform/external-dns/sa.yaml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  namespace: external-dns
  annotations:
    azure.workload.identity/client-id: "<CLIENT_ID_UAMI_EXTERNAL_DNS>"
  labels:
    azure.workload.identity/use: "true"
```

`clusters/staging/platform/external-dns/values-staging.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: external-dns
  namespace: external-dns
spec:
  values:
    txtOwnerId: "aks-staging-01"
    env:
      - name: AZURE_TENANT_ID
        value: "<TENANT_ID>"
      - name: AZURE_SUBSCRIPTION_ID
        value: "<SUB_ID>"
      - name: AZURE_CLIENT_ID
        value: "<CLIENT_ID_UAMI_EXTERNAL_DNS>"
    extraArgs:
      - --azure-resource-group=<DNS_ZONE_RG>
      # - --domain-filter=<example.com>       # optionnel
      # - --azure-use-private-dns              # si zones privées
```

`clusters/staging/platform/external-dns/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../base/platform/external-dns
  - sa.yaml
patches:
  - path: values-staging.yaml
    target:
      kind: HelmRelease
      name: external-dns
      namespace: external-dns
```

**External-Secrets – SA + Store + valeurs**

`clusters/staging/platform/external-secrets/sa.yaml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets
  namespace: external-secrets
  annotations:
    azure.workload.identity/client-id: "<CLIENT_ID_UAMI_EXTERNAL_SECRETS>"
  labels:
    azure.workload.identity/use: "true"
```

`clusters/staging/platform/external-secrets/clustersecretstore-azure.yaml`
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: azure-kv
spec:
  provider:
    azurekv:
      vaultUrl: "https://<KV_NAME>.vault.azure.net/"
      tenantId: "<TENANT_ID>"
      authType: WorkloadIdentity
      serviceAccountRef:
        name: external-secrets
        namespace: external-secrets
```

`clusters/staging/platform/external-secrets/values-staging.yaml`
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: external-secrets
  namespace: external-secrets
spec:
  values:
    # Ajouts spécifiques si besoin (généralement rien d'autre)
```

`clusters/staging/platform/external-secrets/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../base/platform/external-secrets
  - sa.yaml
  - clustersecretstore-azure.yaml
patches:
  - path: values-staging.yaml
    target:
      kind: HelmRelease
      name: external-secrets
      namespace: external-secrets
```

**Cert-Manager – Issuers séparés**

`clusters/staging/platform/cert-manager/clusterissuers.yaml`
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: "<your-email@example.com>"
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
      - http01:
          ingress:
            class: traefik
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: "<your-email@example.com>"
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: traefik
```

`clusters/staging/platform/cert-manager/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../../base/platform/cert-manager
```

`clusters/staging/platform/cert-manager/issuers/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../clusterissuers.yaml
```

**Kustomization CRs (Flux)**

`clusters/staging/kustomizations/cert-manager.yaml`
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cert-manager-staging
  namespace: flux-system
spec:
  interval: 5m
  prune: true
  wait: true
  timeout: 5m
  dependsOn:
    - name: traefik-staging
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  path: ./clusters/staging/platform/cert-manager
```

`clusters/staging/kustomizations/cert-issuers.yaml`
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cert-issuers-staging
  namespace: flux-system
spec:
  interval: 5m
  prune: true
  wait: true
  timeout: 5m
  dependsOn:
    - name: cert-manager-staging
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  path: ./clusters/staging/platform/cert-manager/issuers
```

`clusters/staging/kustomizations/external-dns.yaml`
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: external-dns-staging
  namespace: flux-system
spec:
  interval: 5m
  prune: true
  wait: true
  timeout: 5m
  dependsOn:
    - name: traefik-staging
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  path: ./clusters/staging/platform/external-dns
```

`clusters/staging/kustomizations/external-secrets.yaml`
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: external-secrets-staging
  namespace: flux-system
spec:
  interval: 5m
  prune: true
  wait: true
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  path: ./clusters/staging/platform/external-secrets
```

> Ajoutez ces entrées à `clusters/staging/kustomization.yaml` :
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - kustomizations/traefik.yaml
  - kustomizations/monitoring.yaml
  - kustomizations/cert-manager.yaml
  - kustomizations/cert-issuers.yaml
  - kustomizations/external-dns.yaml
  - kustomizations/external-secrets.yaml
```

---

## 4) Ce qu’il faut faire **côté Azure** (UAMI + Workload Identity)

> Variables d’exemple :
```bash
SUB_ID="<YOUR_SUB_ID>"
TENANT_ID="<YOUR_TENANT_ID>"
RG_AKS="<RG_AKS>"
AKS_NAME="<AKS_NAME>"
RG_IDENTITY="<RG_FOR_UAMI>"
DNS_ZONE_RG="<DNS_ZONE_RG>"
KV_NAME="<KV_NAME>"
```

### 4.1 Créer les UAMI
```bash
# External-DNS
az identity create -g $RG_IDENTITY -n uami-external-dns \
  --query "{clientId:clientId, principalId:principalId}" -o tsv

# External-Secrets
az identity create -g $RG_IDENTITY -n uami-external-secrets \
  --query "{clientId:clientId, principalId:principalId}" -o tsv
```

> Notez bien **clientId** et **principalId** de chaque UAMI.

### 4.2 Créer les *federated credentials* (lier SA ↔ UAMI)

Récupérer l’issuer OIDC si pas fait :
```bash
OIDC=$(az aks show -g $RG_AKS -n $AKS_NAME --query "oidcIssuerProfile.issuerUrl" -o tsv)
```

**External-DNS → SA `system:serviceaccount:external-dns:external-dns`**
```bash
az identity federated-credential create \
  --name ksa-external-dns \
  --identity-name uami-external-dns \
  --resource-group $RG_IDENTITY \
  --issuer "$OIDC" \
  --subject "system:serviceaccount:external-dns:external-dns" \
  --audience "api://AzureADTokenExchange"
```

**External-Secrets → SA `system:serviceaccount:external-secrets:external-secrets`**
```bash
az identity federated-credential create \
  --name ksa-external-secrets \
  --identity-name uami-external-secrets \
  --resource-group $RG_IDENTITY \
  --issuer "$OIDC" \
  --subject "system:serviceaccount:external-secrets:external-secrets" \
  --audience "api://AzureADTokenExchange"
```

### 4.3 Donner les rôles nécessaires

**External-DNS** – accès aux zones DNS (au niveau **Resource Group** ou **Zone**)
```bash
UAMI_DNS_PRINCIPAL_ID=$(az identity show -g $RG_IDENTITY -n uami-external-dns --query principalId -o tsv)

az role assignment create \
  --assignee $UAMI_DNS_PRINCIPAL_ID \
  --role "DNS Zone Contributor" \
  --scope "/subscriptions/$SUB_ID/resourceGroups/$DNS_ZONE_RG"
```

**External-Secrets** – accès au Key Vault (choisissez RBAC data-plane **ou** access policy)

- **RBAC data-plane (recommandé)** :
```bash
UAMI_ESO_PRINCIPAL_ID=$(az identity show -g $RG_IDENTITY -n uami-external-secrets --query principalId -o tsv)
KV_ID=$(az keyvault show -n $KV_NAME --query id -o tsv)

az role assignment create \
  --assignee $UAMI_ESO_PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope "$KV_ID"
```

- **Access policy (alternative)** :
```bash
UAMI_ESO_CLIENT_ID=$(az identity show -g $RG_IDENTITY -n uami-external-secrets --query clientId -o tsv)

az keyvault set-policy \
  -n $KV_NAME \
  --secret-permissions get list \
  --spn $UAMI_ESO_CLIENT_ID
```

> **Important** : Pour External-DNS **ne pas** utiliser `--azure-use-managed-identity` avec Workload Identity. Renseignez `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`, `AZURE_CLIENT_ID` via `values-staging.yaml`.

---

## 5) Déploiement

1. Commit & push de la structure/fichiers ci-dessus.
2. Lancer une reconciliation (au choix) :
   ```bash
   flux reconcile source git flux-system -n flux-system
   flux reconcile kustomization <name> -n flux-system
   ```
3. Vérifier les namespaces, SAs, CRDs et pods :
   ```bash
   kubectl get ns cert-manager external-dns external-secrets
   kubectl -n external-dns get sa external-dns -o yaml | grep -E "azure.workload.identity"
   kubectl -n external-secrets get sa external-secrets -o yaml | grep -E "azure.workload.identity"
   kubectl -n cert-manager get pods
   kubectl -n external-dns get pods
   kubectl -n external-secrets get pods
   ```
4. Vérifier les logs :
   ```bash
   kubectl -n external-dns logs deploy/external-dns --tail=200
   kubectl -n external-secrets logs deploy/external-secrets --tail=200
   ```

---

## 6) Tests rapides

### 6.1 External-DNS
- Créer un Ingress avec un host sur votre domaine (géré par la zone dans `DNS_ZONE_RG`).
- Observer la création du record `A` et du `TXT` (owner=aks-staging-01).

### 6.2 cert-manager
- Créer un Ingress avec annotation `cert-manager.io/cluster-issuer: letsencrypt-staging` et TLS.
- Vérifier que l’HTTP-01 passe via Traefik, puis qu’un secret TLS est créé.

### 6.3 External-Secrets
Créer un `ExternalSecret` d’exemple :
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: demo-secret
  namespace: default
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: azure-kv
    kind: ClusterSecretStore
  target:
    name: demo-secret
  data:
    - secretKey: my-value
      remoteRef:
        key: my-secret-in-kv     # Nom du secret dans Key Vault
```
Vérifier :
```bash
kubectl -n default get secret demo-secret -o yaml
```

---

## 7) Dépannage (FAQ)

- **Pod en CrashLoopBackOff (external-dns/eso)** : vérifier le SA (`annotation` + `label`), la federated credential (`subject` exact), et les rôles Azure (scope correct).
- **ClusterIssuer appliqué trop tôt** : mettre en place la Kustomization `cert-issuers` avec `dependsOn: cert-manager`.
- **Pas de création DNS** : vérifier `--azure-resource-group`, les variables `AZURE_*`, et `txtOwnerId` (éviter conflits multi-clusters).
- **Key Vault permissions** : en RBAC, utiliser le rôle **Key Vault Secrets User**; en policies, `get`/`list` sur `secrets` pour le **clientId** UAMI.
- **Zones privées** : ajouter `--azure-use-private-dns`.

---

## 8) Récap des champs à **personnaliser**

- `<TENANT_ID>`, `<SUB_ID>`
- `<RG_AKS>`, `<AKS_NAME>`, `<RG_IDENTITY>`
- `<DNS_ZONE_RG>`
- `<KV_NAME>`
- `<CLIENT_ID_UAMI_EXTERNAL_DNS>`, `<CLIENT_ID_UAMI_EXTERNAL_SECRETS>`
- emails ACME pour cert-manager (`ClusterIssuer`)
- `txtOwnerId` (unique par cluster)
- (option) `--domain-filter=<votre-domaine>`

---

## 9) Bonnes pratiques

- 1 UAMI **par composant** (isolation & moindre blast radius).
- Nommer les SA/UAMI par composant + environnement (`external-dns`, `external-secrets`).
- Toujours séparer **base** vs **overlays**; limiter les valeurs spécifiques à l’overlay.
- Garder les **CRDs** gérés par Helm pour cert-manager & external-secrets (installCRDs: true).
- Utiliser `dependsOn` entre Kustomization Flux pour l’ordre d’installation.

---

_Fin – prêt à copier/coller dans `README.md` de votre dépôt._
