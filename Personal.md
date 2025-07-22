- Installed `dev` cluster via:

```
cat <<EOF | kind create cluster --name dev --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
- role: worker
EOF
```

# Chapter 2 - Setup all the application containers and storage systems

- `cd chapter-2;./kind-load.sh` - prefetch all the images we will use in the examples : Redis, Postgres, Kafka
- Installed NGinx Ingress controller
- Installed Conference app via Helm Charts

# Chapter 3 - Dagger

Dagger pipeline lives at: `conference-application/service-pipeline.go`
You can run build, test and publish for any of the services in `conference-application`
`go run service-pipeline.go build agenda-service`
`go run service-pipeline.go test agenda-service`
`go run service-pipeline.go publish agenda-service`

# Chapter 4 - ArgoCD

- Installed ArgoCD:

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

- Port forward to ArgoCD:
  `kubectl port-forward svc/argocd-server -n argocd 8080:443`

- To get admin password:
  `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo`

## Create Staging in ArgoCD

- `kubectl create ns staging`

- Port Forward to the application `kubectl port-forward svc/frontend -n staging 8081:80`

## Argo argo-rollouts

```
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
kubectl apply -k https://github.com/argoproj/argo-rollouts/manifests/crds\?ref\=stable
```

Also installed argo rollouts kubectl plugin and added it to system script

### Instal ArgoRollouts Kubectl plugin

```bash
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-darwin-amd64
chmod +x ./kubectl-argo-rollouts-darwin-amd64
sudo mv ./kubectl-argo-rollouts-darwin-amd64 /usr/local/bin/kubectl-argo-rollouts
```

### Start new version of rollouts running

```bash
kubectl argo rollouts dashboard
```

## Traefik capabilities

ArgoRollouts integrates Traefik in 2 ways:

1. Using [TraefikService's native capabilities](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/#kind-traefikservice) to do [weighted round robin](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/#weighted-round-robin) and [trafficmirroring](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/#mirroring) : [Argo Rollouts Traffic Management - Traefik](https://argoproj.github.io/argo-rollouts/features/traffic-management/traefik/)

- Note: this approach needs to not use `weight` if using ArgoCD since changes to the config made by Rollouts can make ArgoCD label the application as out of sync

2. Using the [Rollouts-GatewayAPI-plugin](https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-gatewayapi/tree/main/examples/traefik)

- Traefik needs to be configured with

# Prometheus, Grafana, and Keptn

All these tools have been bundled together in `chapter-9/keptn` with a MakeFile. We need to let KLT know what namespace to monitor which can be done by annotating it
To get at the lower level observability tools checkout: `chaper-9/keptn/support/observability/Makefile` : this file installs Jaeger, Prometheus, OtelCollector, and CertManager

```bash
make install
kubectl annotate ns default keptn.sh/lifecycle-toolkit="enabled"
```

## Portforwarding to installed observability tools

```bash
make port-forward-jaeger
make port-forward-grafana
make port-forward-prometheus

```

Jaeger : [http://localhost:16686/](http://localhost:16686/)
Grafana: [http://localhost:3000/](http://localhost:3000/) : admin/admin
Prometheus: [http://localhost:9090/](http://localhost:9090/)
