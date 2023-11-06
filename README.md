# Overview
Effortless Secrets Management in Kubernetes: Streamlining App Deployment with GitOps and ArgoCD using the HashiCorp Vault Injector

## Install the School App
```bash
# Setup MongoDB DB in K8s
kubectl create ns schoolapp
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install schoolapp-mongodb --namespace schoolapp \
 --set auth.enabled=true \
 --set auth.rootUser=schoolapp \
 --set auth.rootPassword=mongoRootPass \
  bitnami/mongodb

# Add project repo to helm
helm repo add schoolapp https://gitlab.com/api/v4/projects/34240616/packages/helm/stable

# Install the Frontend
helm install frontend -n schoolapp schoolapp/schoolapp-frontend
# Install the API
helm install api -n schoolapp schoolapp/schoolapp-api
# Port forward the frontend
kubectl -n schoolapp port-forward service/frontend 8001:8080
# Port forward the api
kubectl -n schoolapp port-forward service/api 5000:5000
```

## Test the School App

From a browser, go to http://127.0.0.1:8001

## ArgoCD Setup

Now let's get ArgoCD ready for our school app.

## Install ArgoCD

### Install the ArgoCD Server

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait until all the pods are running

```bash
watch kubectl get po
```

## Access ArgoCD

### Expose the ArgoCD API Server

```bash
kubectl port-forward svc/argocd-server -n argocd 8002:443
```

### Get the admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

For this demo, I won't change the password, but you should go ahead and change the password and remove the K8s secret.

### Login using the UI

You can log in to the UI by opening a browser window and going to https://127.0.0.1:8002

```bash
username: admin
password: <THE_PASSWORD_YOU_GOT_ABOVE>
```

## School App Example with ArgoCD and Hardcoded Secrets

Now let's see how to use a GitOps approach with ArgoCD to run our school app.

## Delete the Current School App

```bash
kubectl delete ns schoolapp
```

## Create the School App Application in ArgoCD

```bash
kubectl apply -f argocdSchoolApp.yaml
```

## Test the School App with ArgoCD

```bash
# Port forward the frontend
kubectl port-forward service/frontend 8001:8080 -n schoolapp
# Port forward the api
kubectl port-forward service/api 5000:5000 -n schoolapp
```

Try creating and deleting a course and check the logs of the api pod.

```bash
kubectl logs -n schoolapp -f $(kubectl get pods --template '{{range .items}}{{.metadata.name}}{{end}}' --selector=app=api) -c api
```

## Making a Change in Git

Now let's see how Argo CD can detect drift between the live state and the target state. Let's make a simple change in our Git repo for the called schoolapp-subchart/values.yaml. Change the mongodb.auth.enabled value from true to false.

```yaml
mongodb:
  auth:
    enabled: false
    rootUser: 'schoolapp'
    rootPassword: 'mongoRootPass'
```
Then commit and push your changes.