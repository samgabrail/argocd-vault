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
kubens argocd
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
Now click on Sync inside the ArgoCD UI to sync the changes to the live cluster.
## Test the School App with ArgoCD
```bash
# Port forward the frontend
kubectl port-forward service/frontend 8001:8080 -n schoolapp
# Port forward the api
kubectl port-forward service/api 5000:5000 -n schoolapp
```
## Test the School App with Hardcoded Secrets
Try creating and deleting a course and check the logs of the api pod.
```bash
kubectl logs -n schoolapp -f $(kubectl get pods --template '{{range .items}}{{.metadata.name}}{{end}}' --selector=app=api) -c api
```
Notice this output:
Expected Output
```bash
MongoDB Credentials Using Hardcoded Values that appear in GitLab: 
Username = schoolapp
Password = mongoRootPass
```
## Install Vault via Helm
Let's install Vault with all the values defaults for the Helm chart.
```bash
kubectl create ns vault
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault --namespace vault \
    --set server.image.tag=1.15.1 \
    --set injector.enabled=true \
    --set csi.enabled=false \
    hashicorp/vault
```
## Initialize and Unseal Vault
Next we'll initialize and unseal Vault. Make sure to save your root token and unseal key
First exec into the vault container:
```bash
kubectl exec -it pod/vault-0 -- sh
```
Then run these commands
```bash
vault operator init -key-shares=1 -key-threshold=1
vault operator unseal <UNSEAL_KEY>
# Example:
vault operator unseal ox7fCUNO52KptEPJkycF1miOI28pZEUWQQms5kLkgCY=
```
## Prepare Vault for the School App
Let's prepare Vault to serve the MongoDB credentials to the API.
## Login to Vault
Let's now log in to Vault (while still inside the vault container)
```bash
# export the vault address and login
export VAULT_ADDR=http://127.0.0.1:8200
vault login <ROOT_TOKEN>
# Example:
vault login s.SBGKz0bx4USd6k4ExuhLUjYC
```
## Add mongoDB Credentials
We will add a static KV secret for the school app. This secret is the mongoDB credentials so that the API could connect to mongoDB.
```bash
vault secrets enable -version=2 -path=internal kv
vault kv put internal/schoolapp/mongodb schoolapp_DB_USERNAME=schoolapp schoolapp_DB_PASSWORD=mongoRootPass
```
You can check the credentials in the UI or the CLI by running:
```bash
vault kv get internal/schoolapp/mongodb
```
## Create a School App Policy
```bash
vault policy write schoolapp - <<EOF
path "internal/data/schoolapp/mongodb" {
  capabilities = ["list", "read"]
}
EOF
```
## Setup K8s Auth Method in Vault
```bash
# Enable K8s auth method
vault auth enable kubernetes
# Configure the K8s auth method
vault write auth/kubernetes/config \
    issuer="https://kubernetes.default.svc.cluster.local" \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
vault write auth/kubernetes/role/schoolapp \
    bound_service_account_names=schoolapp \
    bound_service_account_namespaces=schoolapp \
    policies=schoolapp
    ttl=5s \
    max_ttl=20s
```
Now exit from the container.
## ArgoCD and the Vault Injector
Now let's see how to use a GitOps approach with ArgoCD to run our school app while using the Vault injector for secrets.
## Test the School App with the Vault Injector
Update the `values.yaml` file under `schoolapp-subchart` with these values:
```yaml
schoolapp-api:
  vault:
    status: 'kv_static_injector_template_file'
    kind: 'injector'
```
Here we're using templating for a Vault unaware app
Commit and push the changes to GitLab. Then hit refresh in the argoCD UI.
Notice the app is now out of sync.
Click the APP DIFF button to see the changes and make sure the changes make sense then Click the sync button and Synchronize
Notice how the api pod now has 2/2 containers because the vault agent container is now running beside the api container.
**Expected Output:**
```
NAME                                     READY   STATUS    RESTARTS      AGE
pod/api-657f89d85b-ssnld                 2/2     Running   0             28s
pod/frontend-6c445b8775-ttgdx            1/1     Running   2 (14m ago)   52m
pod/schoolapp-mongodb-6cdf54d797-s2tnr   1/1     Running   4 (14m ago)   52m
```
Now let's check the school app
Make sure the api and frontend ports are being exposed to the host machine:
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
Notice this output:
**Expected Output**
```
MongoDB Credentials Using Vault KV with injector and templates for Vault unaware apps: 
Username = schoolapp
Password = mongoRootPass
```

	
