# K8s
This was an experiment to see how we could deploy the app to k8s.
I got the apps working, I got the temporal cluster deployed, but I had to stop due to lack of time and costs.

## TODO
- [ ] Is everything working end-2-end?
- [ ] Add liveness and readiness probes 
    - [ ] portal-web 
    - [ ] publication-web
    - [ ] webhooks-server
    - [ ] worker
- [ ] Setup CI/CD pipeline (use kustomize or helm?)
- [ ] Figure out how to scale to zero to reduce costs while not working on the app

## Before you get started
Install gcloud, authenticate, and select the proper project. 

I've created this alias to help me switch between accounts and projects.

```bash
alias fun='gcloud config set account ricardo.canastro@gmail.com && gcloud config set project chronical-dev && gcloud auth application-default login'
```

## To use locally
Start minikube

```bash
minikube start
```

Run kubectl and apply all files, and install all helm charts:

```bash
kubectl apply --recursive  -f .
helmfile apply
```

Run minikube tunnel

```bash
minikube tunnel

or 

minikube service portal-web-service publication-web-service webhooks-server-service
```

If you need to re-deploy an image, you can use the following command

```bash
kubectl rollout restart deployment portal-web
```

If you want to see some info about the local cluster, you can use the following command

```bash
minikube dashboard
```

To check logs, you can use the following command

```bash
kubectl logs -l app=portal-web
```

To connect to the database locally, you can open a tunnel to the pod:

```bash
kubectl port-forward postgres-0 5432:5432
```

## GCP
* region: us-east1
* project: chronical-dev
* repository: chronical-artifacts-repository

### Build and push images to GCP
```bash
# To build for GCP add `--platform=linux/amd64`
docker build . -f ./apps/portal-web/Dockerfile -t us-east1-docker.pkg.dev/chronical-dev/chronical-artifacts-repository/chronical-portal-web:0.0.2 --platform=linux/amd64
docker build . -f ./apps/publication-web/Dockerfile -t us-east1-docker.pkg.dev/chronical-dev/chronical-artifacts-repository/chronical-publication-web:0.0.1 --platform=linux/amd64
docker build . -f ./apps/webhooks-server/Dockerfile -t us-east1-docker.pkg.dev/chronical-dev/chronical-artifacts-repository/chronical-webhooks-server:0.0.2 --platform=linux/amd64
docker build . -f ./apps/worker/Dockerfile -t us-east1-docker.pkg.dev/chronical-dev/chronical-artifacts-repository/chronical-worker:0.0.1 --platform=linux/amd64

# Push images to GCP
docker push us-east1-docker.pkg.dev/chronical-dev/chronical-artifacts-repository/chronical-portal-web:0.0.3
docker push us-east1-docker.pkg.dev/chronical-dev/chronical-artifacts-repository/chronical-publication-web:0.0.1
docker push us-east1-docker.pkg.dev/chronical-dev/chronical-artifacts-repository/chronical-webhooks-server:0.0.2
docker push us-east1-docker.pkg.dev/chronical-dev/chronical-artifacts-repository/chronical-worker:0.0.1
# Load images to minikube
minikube image load us-east1-docker.pkg.dev/chronical-dev/chronical-artifacts-repository/chronical-portal-web:0.0.1
minikube image load us-east1-docker.pkg.dev/chronical-dev/chronical-artifacts-repository/chronical-publication-web:0.0.1
minikube image load us-east1-docker.pkg.dev/chronical-dev/chronical-artifacts-repository/chronical-webhooks-server:0.0.1
minikube image load chronical-migrator
```


### Create service account & workload identity

```bash
# Create service account for workload identity
gcloud iam service-accounts create web-wli

# Add permission
gcloud projects add-iam-policy-binding chronical-dev \
    --member='serviceAccount:web-wli@chronical-dev.iam.gserviceaccount.com' \
    --role='roles/secretmanager.secretAccessor'

# Map K8s service account to workload identity
gcloud iam service-accounts \
  add-iam-policy-binding web-wli@chronical-dev.iam.gserviceaccount.com \
  --role='roles/iam.workloadIdentityUser' \
  --member='serviceAccount:chronical-dev.svc.id.goog[default/chronical-web-sa]'
```

### Certificates 

```bash
gcloud certificate-manager maps create root-dev-map

gcloud certificate-manager maps entries create root-dev-entry \
    --map="root-dev-map" \
    --certificates="chronical-dev-cert" \
    --hostname="chronical.dev"

gcloud certificate-manager maps entries create wildcard-dev-entry \
    --map="root-dev-map" \
    --certificates="chronical-dev-cert" \
    --hostname="*.chronical.dev"

gcloud certificate-manager maps entries describe root-dev-entry \
    --map="root-dev-map"

gcloud certificate-manager maps entries describe wildcard-dev-entry \
    --map="root-dev-map"
```

### Temporal

```bash
kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user ricardo.canastro@gmail.com

kubectl apply -f temporal-values/temporal.secrets.yaml

helm install \
    --repo https://go.temporal.io/helm-charts \
    -f temporal-values/temporal.values.yaml \
    --set server.replicaCount=1 \
    chronical-temporal temporal \
    --timeout 15m
```

### Scale down when not in use

```bash
kubectl scale --replicas=0 deployment worker
kubectl scale --replicas=0 deployment webhooks-server
kubectl scale --replicas=0 deployment publication-web
kubectl scale --replicas=0 deployment portal-web
kubectl scale --replicas=0 deployment chronical-temporal-admintools
kubectl scale --replicas=0 deployment chronical-temporal-frontend
kubectl scale --replicas=0 deployment chronical-temporal-history
kubectl scale --replicas=0 deployment chronical-temporal-matching
kubectl scale --replicas=0 deployment chronical-temporal-web
kubectl scale --replicas=0 deployment chronical-temporal-worker
kubectl scale --replicas=0 deployment redis
gcloud services disable monitoring.googleapis.com --force

```



### Reading list
* https://cloud.google.com/kubernetes-engine/docs/concepts/gateway-api
* https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-gateways#external-gateway