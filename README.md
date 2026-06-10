# 🐳 B-DOP-200 — Popeye · Mémo de lancement

> Stack Kubernetes locale via **K3d** · PostgreSQL · Redis · Poll / Worker / Result · Traefik

---

## 📋 Table des matières

1. [Prérequis](#-prérequis)
2. [Démarrage quotidien](#-démarrage-quotidien)
3. [Créer le cluster de zéro](#-créer-le-cluster-de-zéro)
4. [Déployer les manifestes](#-déployer-les-manifestes)
5. [Initialiser la base PostgreSQL](#-initialiser-la-base-postgresql)
6. [Configurer `/etc/hosts`](#-configurer-etchosts)
7. [URLs de test](#-urls-de-test)
8. [Commandes de débogage](#-commandes-de-débogage)
9. [Nettoyage & Arrêt](#-nettoyage--arrêt)
10. [Notes & Problèmes connus](#-notes--problèmes-connus)

---

## ✅ Prérequis

> Ces étapes sont à faire **une seule fois**. Si le cluster existe déjà, passe directement au [Démarrage quotidien](#-démarrage-quotidien).

| Outil | Statut attendu |
|-------|---------------|
| Docker (dans WSL) | `sudo service docker status` → running |
| `kubectl` | `kubectl version --client` |
| `k3d` | `k3d version` |
| Cluster `dop-cluster` | `k3d cluster list` → dop-cluster |

Si le cluster n'existe pas encore → [Créer le cluster de zéro](#-créer-le-cluster-de-zéro)

---

## 🚀 Démarrage quotidien

> À faire à **chaque redémarrage du PC**. Les pods reprennent là où ils s'étaient arrêtés.

```bash
# 1. Lancer Docker
sudo service docker start

# 2. Relancer le cluster K3d
k3d cluster start dop-cluster

# 3. Vérifier que tout est UP (attendre ~30s)
kubectl get pods -A
```

---

## 🏗️ Créer le cluster de zéro

> À faire uniquement si le cluster a été **supprimé** ou lors de la **première installation**.

```bash
k3d cluster create dop-cluster \
  --servers 1 \
  --agents 2 \
  --port "30021:30021@server:0" \
  --port "30042:30042@server:0" \
  --k3s-arg "--disable=traefik@server:*"
```

---

## 📦 Déployer les manifestes

> Depuis le répertoire `~/B-DOP-200_popeye_applications`, **dans l'ordre ci-dessous**.

```bash
cd ~/B-DOP-200_popeye_applications
```

### 1 — cAdvisor (monitoring)

```bash
kubectl apply -f cadvisor.daemonset.yaml
```

### 2 — PostgreSQL

```bash
kubectl apply -f postgres.secret.yaml \
              -f postgres.configmap.yaml \
              -f postgres.volume.yaml \
              -f postgres.deployment.yaml \
              -f postgres.service.yaml
```

### 3 — Redis

```bash
kubectl apply -f redis.configmap.yaml \
              -f redis.deployment.yaml \
              -f redis.service.yaml
```

### 4 — Applications (poll · worker · result) + Ingress

```bash
kubectl apply -f poll.deployment.yaml \
              -f worker.deployment.yaml \
              -f result.deployment.yaml \
              -f poll.service.yaml \
              -f result.service.yaml \
              -f poll.ingress.yaml \
              -f result.ingress.yaml
```

### 5 — Traefik (RBAC **en premier** !)

```bash
kubectl apply -f traefik.rbac.yaml \
              -f traefik.deployment.yaml \
              -f traefik.service.yaml
```

---

## 🗄️ Initialiser la base PostgreSQL

> À faire **une seule fois** après le premier déploiement.

```bash
POD=$(kubectl get pod -l app=postgres -o jsonpath='{.items[0].metadata.name}')
echo "CREATE TABLE votes (id text PRIMARY KEY, vote text NOT NULL);" | \
  kubectl exec -i $POD -c postgres -- psql -U postgres
```

---

## 🌐 Configurer `/etc/hosts`

> À faire **une seule fois** sur WSL et sur Windows.

**WSL**
```bash
echo "127.0.0.1 poll.dop.io result.dop.io" | sudo tee -a /etc/hosts
```

**Windows — PowerShell (administrateur)**
```powershell
Add-Content -Path C:\Windows\System32\drivers\etc\hosts `
            -Value "127.0.0.1 poll.dop.io result.dop.io"
```

---

## 🔗 URLs de test

| Service | URL |
|---------|-----|
| 🗳️ Application Poll | http://poll.dop.io:30021 |
| 📊 Résultats | http://result.dop.io:30021 |
| 🛡️ Dashboard Traefik | http://localhost:30042 |

---

## 🛠️ Commandes de débogage

```bash
# État de tous les pods
kubectl get pods -A

# Détail d'un pod spécifique
kubectl describe pod <nom-du-pod>

# Logs en live (remplacer <app> par poll, result, worker, postgres, redis ou traefik)
kubectl logs -f -l app=<app>

# Redémarrer un deployment
kubectl rollout restart deployment/<nom>

# Voir les ingress configurés
kubectl get ingress

# Vérifier l'IngressClass Traefik
kubectl get ingressclass
```

---

## 🧹 Nettoyage & Arrêt

```bash
# Stopper le cluster (sans le supprimer — les données sont conservées)
k3d cluster stop dop-cluster

# Supprimer complètement le cluster
k3d cluster delete dop-cluster

# Supprimer tous les manifests sans toucher au cluster
kubectl delete -f .
```

---

## ⚠️ Notes & Problèmes connus

### IP WSL changeante après reboot
Si l'IP WSL change (vérifier avec `hostname -I`), refaire les règles `netsh portproxy` côté Windows pour l'accès depuis d'autres appareils.

### cAdvisor en `RunContainerError`
Comportement attendu sur K3d — limitation du runtime conteneur. Le manifeste est correct, c'est l'environnement qui pose souci. Pas bloquant.

### "Page 404" sur `result`
Problème probable de connexion PostgreSQL. Vérifier que le ConfigMap `postgres-config` contient bien :
- `POSTGRES_HOST`
- `POSTGRES_PORT`
- `POSTGRES_DB`

```bash
kubectl logs -f -l app=result
```

### "404 page not found" via Traefik sur poll/result
L'IngressClass `traefik` est peut-être absente. Vérifier :

```bash
kubectl get ingressclass
```

Si absente, la recréer :

```bash
kubectl apply -f traefik.rbac.yaml
```
