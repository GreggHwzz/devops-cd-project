# Guide de déploiement

Ce document contient toutes les commandes nécessaires pour exécuter l'application localement, construire l'image Docker et déployer l'application sur Kubernetes avec Minikube, ainsi que mettre en place le monitoring avec Prometheus et Grafana et la gestion des logs avec Loki.

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
kubectl apply -f namespaces.yaml

# Charger l'image dans Minikube (nécessaire car imagePullPolicy: Never)
minikube image load go-app:v1

# Déployer dans l'environnement de développement
kubectl apply -f deployment.yaml -n development
kubectl apply -f service.yaml -n development

# Déployer dans l'environnement de production
kubectl apply -f deployment.yaml -n production
kubectl apply -f service.yaml -n production
```

## Configuration du monitoring avec Prometheus et Grafana

```bash
# Ajouter les repositories Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Installer Prometheus Stack avec les valeurs personnalisées
helm install prometheus prometheus-community/kube-prometheus-stack -f values-prometheus.yaml

# Installer Grafana (si non inclus dans le stack Prometheus)
helm install grafana grafana/grafana -f values-grafana.yaml

# Configurer les règles d'alerte
kubectl apply -f high-memory-usage-rule.yaml
kubectl apply -f pod-not-running-rule.yaml

# Configurer AlertManager pour les alertes par email
kubectl apply -f alertmanager-config.yaml
```

## Configuration de la gestion des logs avec Loki

```bash
# Installer Loki Stack via Helm
helm install loki grafana/loki-stack -f values-loki.yaml --set grafana.enabled=false

# Configurer Loki comme source de données dans Grafana
kubectl apply -f update-datasources.yaml
kubectl rollout restart deployment grafana

# Déployer le dashboard pour les logs d'erreur
kubectl apply -f dashboard-configmap.yaml

# Déployer un pod générant des logs d'erreur pour tester
kubectl apply -f error-generator.yaml -n production
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

## Test des alertes et des logs

```bash
# Tester l'alerte "PodNotRunning"
kubectl run test-pod --image=non-existent-image

# Tester l'alerte "HighMemoryUsage"
kubectl run memory-test --image=progrium/stress -- -m 1 --vm-bytes 512M --timeout 300s

# Générer un log d'erreur spécifique
kubectl exec -n production error-generator -- sh -c 'echo "CUSTOM ERROR: test message $(date)"'
```

## Visualisation des logs d'erreur dans Grafana

1. Accéder à Grafana via l'URL ou le port-forward
2. Se connecter avec les identifiants (admin/admin par défaut)
3. Pour visualiser les logs via le dashboard:
   - Dans le menu de gauche, sélectionner "Dashboards"
   - Trouver et ouvrir le dashboard "Production Error Logs"
   
4. Pour visualiser les logs via Explore:
   - Dans le menu de gauche, cliquer sur "Explore"
   - Sélectionner "Loki" comme source de données
   - Entrer la requête: `{namespace="production"} |= "error"`
   - Cliquer sur "Run Query"

## Vérification de la configuration

```bash
# Vérifier que Loki est correctement déployé
kubectl get pods | grep loki

# Vérifier que le pod générateur d'erreurs fonctionne
kubectl get pods -n production
kubectl logs -n production error-generator
```

## Requêtes Loki utiles

- Tous les logs d'erreur dans le namespace production:
  `{namespace="production"} |= "error"`

- Logs d'erreur personnalisés:
  `{namespace="production"} |= "CUSTOM ERROR"`

- Logs d'erreur sans les messages normaux:
  `{namespace="production"} |= "error" !~ "normal"`