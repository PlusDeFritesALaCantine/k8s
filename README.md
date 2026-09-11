# Déploiement Kubernetes FutureKawa

Ce dépôt contient les manifests Kubernetes et les valeurs Helm nécessaires au déploiement de l'environnement FutureKawa :

- l'application FutureKawa et ses services associés ;
- Odoo 16 ;
- Prometheus et Grafana ;
- GitHub Exporter ;
- les runners GitHub Actions autoscalés.

## Prérequis

- Un cluster Kubernetes fonctionnel ;
- `kubectl` configuré avec le bon contexte ;
- Helm 3 ;
- Traefik installé comme contrôleur Ingress ;
- les services applicatifs FutureKawa, PostgreSQL, EMQX et Mailpit disponibles dans le namespace `futurekawa-app` ;
- les CRD et le contrôleur Actions Runner Controller installés pour les runners GitHub.

Vérifier le contexte actif :

```bash
kubectl config current-context
kubectl get nodes
kubectl get ingressclass
```

## Sécurité

Les fichiers `secret.yaml` et `github-exporter-values.yaml` contiennent actuellement des tokens et mots de passe en clair. Ces secrets doivent être considérés comme compromis :

1. Révoquer et régénérer les tokens GitHub concernés ;
2. Remplacer les valeurs en clair par des Secrets Kubernetes ou un gestionnaire de secrets ;
3. Ne pas versionner de nouveaux secrets dans ce dépôt.

Le mot de passe administrateur Grafana est également défini dans `grafana-values.yaml` et doit être changé avant toute exposition du service.

## Namespaces

Créer les namespaces utilisés par les manifests, s'ils n'existent pas déjà :

```bash
kubectl create namespace futurekawa-app --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace arc-runners --dry-run=client -o yaml | kubectl apply -f -
```

## Déploiement de l'application

Les manifests applicatifs supposent que les services `futurekawa-backend`, `futurekawa-front-service`, `futurekawa-api-service`, `postgres-service`, `mailpit-service` et `emqx-service` sont déjà déployés dans `futurekawa-app`.

Déployer Odoo et son Ingress :

```bash
kubectl apply -f odoo-deployment.yaml
kubectl apply -f ingress.yaml
```

Vérifier les ressources :

```bash
kubectl get pods,svc,ingress -n futurekawa-app
kubectl rollout status deployment/odoo -n futurekawa-app
```

## Monitoring

Ajouter les dépôts Helm si nécessaire :

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Installer ou mettre à jour la stack Prometheus/Grafana :

```bash
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
	--namespace monitoring \
	--create-namespace

helm upgrade --install grafana grafana/grafana \
	--namespace monitoring \
	--values grafana-values.yaml
```

Appliquer les Ingress de monitoring :

```bash
kubectl apply -f prometheus-ingress.yaml
kubectl apply -f grafana-ingress.yaml
```

Vérifier l'état du monitoring :

```bash
kubectl get pods,svc,ingress -n monitoring
kubectl get servicemonitor -n monitoring
```

## GitHub Exporter

Le fichier `github-exporter-values.yaml` configure la collecte des workflows de l'organisation `PlusDeFritesALaCantine` et crée un `ServiceMonitor` dans `monitoring`.

Installer ou mettre à jour le chart GitHub Exporter avec le nom du dépôt Helm utilisé par votre cluster :

```bash
helm upgrade --install github-exporter <repository>/github-exporter \
	--namespace monitoring \
	--values github-exporter-values.yaml
```

Remplacer `<repository>/github-exporter` par le nom réel du chart disponible dans votre dépôt Helm.

## Runners GitHub Actions

Le fichier `secret.yaml` crée le secret utilisé par Actions Runner Controller. Appliquer le secret puis les valeurs du runner avec le chart ARC installé dans le cluster :

```bash
kubectl apply -f secret.yaml

helm upgrade --install shared-runners <repository>/gha-runner-scale-set \
	--namespace arc-runners \
	--create-namespace \
	--values runner-values.yaml
```

Le scale set est nommé `shared-runners`, avec zéro runner minimum et deux runners maximum. Remplacer `<repository>/gha-runner-scale-set` par le chart ARC utilisé par votre installation.

## Accès local

Les Ingress utilisent les noms d'hôtes suivants :

| Service | Adresse |
| --- | --- |
| Application | `http://app.futurekawa.local` |
| API | `http://api.futurekawa.local` |
| Backend | `http://backend.futurekawa.local` |
| Mailpit | `http://mail.futurekawa.local` |
| MQTT/EMQX | `http://mqtt.futurekawa.local` |
| Odoo | `http://odoo.futurekawa.local` |
| Grafana | `http://grafana.local` |
| Prometheus | `http://prometheus.local` |

Pour un accès local, faire pointer ces noms vers l'adresse IP du contrôleur Traefik dans le fichier `hosts` de la machine cliente, par exemple :

```text
<IP_DE_TRAEFIK> app.futurekawa.local api.futurekawa.local backend.futurekawa.local
<IP_DE_TRAEFIK> mail.futurekawa.local mqtt.futurekawa.local odoo.futurekawa.local
<IP_DE_TRAEFIK> grafana.local prometheus.local
```

Retrouver l'adresse du contrôleur Traefik :

```bash
kubectl get svc -A | grep -i traefik
```

## Dépannage

```bash
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe pod <nom-du-pod> -n <namespace>
kubectl logs deployment/odoo -n futurekawa-app
kubectl logs deployment/<nom-du-deploiement> -n monitoring
```

Tester la configuration sans modifier le cluster :

```bash
kubectl apply --dry-run=client -f odoo-deployment.yaml
kubectl apply --dry-run=client -f ingress.yaml
kubectl apply --dry-run=client -f prometheus-ingress.yaml
kubectl apply --dry-run=client -f grafana-ingress.yaml
kubectl apply --dry-run=client -f secret.yaml
```

## Fichiers principaux

| Fichier | Rôle |
| --- | --- |
| `odoo-deployment.yaml` | Deployment et Service Odoo |
| `ingress.yaml` | Routage Ingress de l'application et des services |
| `grafana-values.yaml` | Configuration Helm de Grafana |
| `grafana-ingress.yaml` | Accès Ingress à Grafana |
| `prometheus-ingress.yaml` | Accès Ingress à Prometheus |
| `github-exporter-values.yaml` | Configuration du GitHub Exporter |
| `runner-values.yaml` | Configuration du scale set GitHub Actions |
| `secret.yaml` | Secret GitHub pour les runners |
