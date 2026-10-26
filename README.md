# Kanban sur Kubernetes (minikube)

Déploiement d'une application Kanban en 4 composants sur un cluster Kubernetes local (minikube), avec une architecture type production : namespace dédié, stockage persistant, Secret, probes, Ingress.

## Architecture

```
                         Ingress (nginx)
Navigateur ──► adminer.k8s.com      ──► Service adminer:8080    ──► Adminer ──┐
          ├──► kanban.k8s.com       ──► Service kanban-ui:80    ──► Frontend  │
          └──► kanban.k8s.com/api   ──► Service kanban-app:8080 ──► Backend ──┤
                                                                              ▼
                                        Service postgres:5432 ──► PostgreSQL ──► PVC ──► PV
```

| Composant | Image | Rôle |
|---|---|---|
| postgres | `postgres:9.6-alpine` | base de données |
| kanban-app | `wkrzywiec/kanban-app:k8s` | API REST (Spring Boot), tables créées par Liquibase |
| kanban-ui | `wkrzywiec/kanban-ui:k8s` | frontend servi par nginx |
| adminer | `adminer:4.7.6-standalone` | administration de la base |

## Objets Kubernetes

| Fichier | Objet |
|---|---|
| `namespace.yaml` | Namespace `kanban` |
| `postgres-config.yaml` | ConfigMap (nom de base, utilisateur) |
| `postgres-secret.yaml` | Secret (mot de passe), **non versionné** |
| `postgres-pv.yaml` / `postgres-pvc.yaml` | stockage persistant (hostPath, `Retain`) |
| `*-deployment.yaml` | Deployments avec readiness / liveness probes |
| `*-svc.yaml` | Services ClusterIP (rien n'est exposé directement) |
| `ingress-controller.yaml` | Ingress : routage par domaine et par chemin |

Choix notables :
- **Recreate** pour PostgreSQL : jamais deux bases sur le même disque.
- **2 réplicas** pour le frontend (sans état).
- `/api` routé directement vers le backend : page et API sous le même domaine (évite les refus CORS).

## Prérequis

Docker, minikube, kubectl.

## Déploiement

```bash
minikube start --driver=docker --memory=4096 --cpus=2
minikube addons enable ingress

kubectl apply -f namespace.yaml
kubectl config set-context --current --namespace=kanban

# Secret (non versionné)
kubectl create secret generic postgres-secret --from-literal=POSTGRES_PASSWORD=<mot_de_passe> -n kanban

kubectl apply -f postgres-config.yaml -f postgres-pv.yaml -f postgres-pvc.yaml
kubectl apply -f postgres-deployment.yaml -f postgres-svc.yaml
kubectl apply -f adminer-deployment.yaml -f adminer-svc.yaml
kubectl apply -f kanban-app-svc.yaml -f kanban-app-deployment.yaml
kubectl apply -f kanban-ui-deployment.yaml -f kanban-ui-svc.yaml
kubectl apply -f ingress-controller.yaml
```

## Accès

1. Fichier `hosts` (Windows : `C:\Windows\System32\drivers\etc\hosts`, Linux : `/etc/hosts`) :
   ```
   127.0.0.1 kanban.k8s.com adminer.k8s.com
   ```
2. Tunnel vers l'Ingress :
   ```bash
   kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8088:80
   ```
3. Navigateur :
   - application : http://kanban.k8s.com:8088
   - administration : http://adminer.k8s.com:8088 (PostgreSQL, serveur `postgres`)

## Vérification

```bash
kubectl get pods,svc,pvc,ingress
kubectl logs deploy/kanban-app | grep -E "HikariPool|Started"
kubectl describe ingress kanban-ingress
```

## Nettoyage

```bash
kubectl delete namespace kanban
kubectl delete pv postgres-pv
```
