# Guide de déploiement

Ce document contient toutes les commandes nécessaires pour exécuter l'application localement, construire l'image Docker et déployer l'application sur Kubernetes avec Minikube, ainsi que mettre en place le monitoring avec Prometheus et Grafana.

## Exécution locale de l'application Go

```bash
# Exécuter l'application en local
go run main.go

# Test dans un autre terminal
curl http://localhost:8080/whoami
```

## Construction et exécution de l'image Docker

```bash
# Construction de l'image
docker build -t go-app:v1 .

# Exécution du conteneur Docker
docker run -p 8080:8080 go-app:v1

# Test de l'application dans le conteneur
curl http://localhost:8080/whoami
```

## Déploiement sur Kubernetes avec Minikube

```bash
# Démarrer Minikube (si ce n'est pas déjà fait)
minikube start

# Créer les namespaces pour dev et prod
kubectl apply -f kubernetes-namespaces.yaml

# Charger l'image dans Minikube (nécessaire car imagePullPolicy: Never)
minikube image load go-app:v1

# Déployer dans l'environnement de développement
kubectl apply -f kubernetes/dev/deployment.yaml
kubectl apply -f kubernetes/dev/service.yaml

# Déployer dans l'environnement de production
kubectl apply -f kubernetes/prod/deployment.yaml
kubectl apply -f kubernetes/prod/service.yaml
```

## Configuration du monitoring avec Prometheus et Grafana

```bash
# Ajouter les repositories Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Installer Prometheus Stack
helm install prometheus prometheus-community/kube-prometheus-stack -f values-prometheus.yaml

# Installer Grafana (si non inclus dans le stack Prometheus)
helm install grafana grafana/grafana -f values-grafana.yaml

# Configurer les règles d'alerte
kubectl apply -f pod-not-running-rule.yaml
kubectl apply -f high-memory-usage-rule.yaml

# Configurer AlertManager pour les alertes par email
kubectl apply -f alertmanager-config.yaml
```

## Accès aux interfaces Web

```bash
# Accéder à Grafana (option 1)
minikube service grafana --url

# Accéder à Grafana (option 2)
kubectl port-forward svc/grafana 3000:80
# Puis ouvrir http://localhost:3000 dans le navigateur
# Identifiants: admin/admin

# Accéder à Prometheus
kubectl port-forward svc/prometheus-server 9090:80
# Puis ouvrir http://localhost:9090 dans le navigateur
```

## Test des alertes

```bash
# Tester l'alerte "PodNotRunning"
kubectl run test-pod --image=non-existent-image

# Tester l'alerte "HighMemoryUsage"
kubectl run memory-test --image=progrium/stress -- -m 1 --vm-bytes 512M --timeout 300s
```