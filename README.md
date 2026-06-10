# B-DOP-200 — Mémo de lancement du projet

## Prérequis (une seule fois, déjà normalement fait)

- Docker installé dans WSL
- `kubectl` installé
- `k3d` installé
- Cluster `dop-cluster` créé (sinon voir [Créer le cluster](#créer-le-cluster-de-zéro-si-supprimé--1ère-install))

---

## 1. Démarrage quotidien (après reboot du PC)

```bash
# Lancer Docker
sudo service docker start

# Lancer le cluster K3d (les pods reprennent où ils s'étaient arrêtés)
k3d cluster start dop-cluster

# Vérifier que tout est UP (attendre ~30s avant de check)
kubectl get pods -A
```
## 2. Créer le cluster de zéro (si supprimé / 1ère install)
```
k3d cluster create dop-cluster \
  --servers 1 \
  --agents 2 \
  --port "30021:30021@server:0" \
  --port "30042:30042@server:0" \
  --k3s-arg "--disable=traefik@server:*"
```

## 3. Déployer les manifests (dans l'ordre)
```
cd ~/B-DOP-200_popeye_applications

# cAdvisor (monitoring)
kubectl apply -f cadvisor.daemonset.yaml

# PostgreSQL
kubectl apply -f postgres.secret.yaml \
              -f postgres.configmap.yaml \
              -f postgres.volume.yaml \
              -f postgres.deployment.yaml \
              -f postgres.service.yaml

# Redis
kubectl apply -f redis.configmap.yaml \
              -f redis.deployment.yaml \
              -f redis.service.yaml

# Apps (poll, worker, result) + Ingress
kubectl apply -f poll.deployment.yaml \
              -f worker.deployment.yaml \
              -f result.deployment.yaml \
              -f poll.service.yaml \
              -f result.service.yaml \
              -f poll.ingress.yaml \
              -f result.ingress.yaml

# Traefik (RBAC en premier !)
kubectl apply -f traefik.rbac.yaml \
              -f traefik.deployment.yaml \
              -f traefik.service.yaml
```
## 4. Créer la table votes dans PostgreSQL (à faire 1 fois)
```
POD=$(kubectl get pod -l app=postgres -o jsonpath='{.items[0].metadata.name}')
echo "CREATE TABLE votes (id text PRIMARY KEY, vote text NOT NULL);" | \
  kubectl exec -i $POD -c postgres -- psql -U postgres
```
## 5. Configurer /etc/hosts (à faire 1 fois)
### Côté WSL
```
echo "127.0.0.1 poll.dop.io result.dop.io" | sudo tee -a /etc/hosts
```
### Côté Windows (PowerShell admin)
```
Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value "127.0.0.1 poll.dop.io result.dop.io"
```

## 6. URLs de test
Application	URL
Poll	http://poll.dop.io:30021

Result	http://result.dop.io:30021

Dashboard Traefik	http://localhost:30042

## 7. Commandes utiles de debug
```
# État de tous les pods
kubectl get pods -A

# Détail d'un pod
kubectl describe pod <nom-du-pod>

# Logs en live
kubectl logs -f -l app=<poll|result|worker|postgres|redis|traefik>

# Redémarrer un deployment
kubectl rollout restart deployment/<nom>

# Voir les ingress
kubectl get ingress
```

## 8. Nettoyage / Arrêt 
```
# Stopper le cluster (sans le supprimer)
k3d cluster stop dop-cluster

# Supprimer complètement le cluster
k3d cluster delete dop-cluster

# Supprimer tous les manifests sans toucher au cluster
kubectl delete -f .
```
## 9. Notes importantes

Si l'IP WSL change après reboot (vérifier avec hostname -I), refaire les netsh portproxy côté Windows pour l'accès depuis le téléphone.

cAdvisor reste en RunContainerError sur K3d (limitation du runtime). Le manifest est correct, c'est juste l'environnement qui pose souci.

En cas de problème "Page 404 introuvable" sur result :
Vérifier les logs de result (problème probable de connexion PostgreSQL).
Le ConfigMap postgres-config doit contenir POSTGRES_HOST, POSTGRES_PORT, POSTGRES_DB.

En cas de problème "404 page not found" sur poll/result via Traefik :
Vérifier que l'IngressClass traefik existe :
```
kubectl get ingressclass
```
Si l’IngressClass traefik est absente (vérifiable avec kubectl get ingressclass),
il suffit de réappliquer le fichier traefik.rbac.yaml pour la recréer.
```
kubectl apply -f traefik.rbac.yaml

```

  
