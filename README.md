# streaming-applications-gitops


In this exercise, we're going to see how to deploy and run Kakfa applications with a GitOps approach.

You will:
1. Create a local Kubernetes cluster
2. Install the FluxCD GitOps tool
3. Write and package a simple kafka producing application
4. Deploy this application by just committing code to GitHub, you will not interact with the Kubernetes Cluster directly



First, [install kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation). If you are on a Mac with Homebrew installed, you can just run:
```shell
brew install kind
```

### Install FluxCD

In order to install the FluxCD agent into your Kubernetes cluster,  you will need [install the Flux CI](https://fluxcd.io/flux/installation/#install-the-flux-cli):
```
brew install fluxcd/tap/flux
```

You will need to export your GitHub credentials:

```sh
export GITHUB_USER=<your github username>
export GITHUB_TOKEN=<a github personal access token which can read/write repos AND packages>
```

## Create a local Kubernetes cluster
Next up, create a cluster 
```sh
kind create cluster --name streaming-apps-staging
Creating cluster "streaming-apps-staging" ...
 ✓ Ensuring node image (kindest/node:v1.27.1) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-streaming-apps-staging"
You can now use your cluster with:

kubectl cluster-info --context kind-streaming-apps-staging
```

Activate the cluster with the following command:
```sh
kubectl cluster-info --context kind-streaming-apps-staging
Kubernetes control plane is running at https://127.0.0.1:52561
CoreDNS is running at https://127.0.0.1:52561/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

You will also need `kubetctl` so let's install it too:
```sh
brew install kubectl
```

Let's check if we can see the Kubernetes node created by Kind:
```sh
kubectl get nodes
NAME                                   STATUS   ROLES           AGE   VERSION
streaming-apps-staging-control-plane   Ready    control-plane   16m   v1.27.1
```

Let's verify that we have all we need before going further:
```sh
flux check --pre
► checking prerequisites
✔ Kubernetes 1.27.1 >=1.25.0-0
✔ prerequisites checks passed
```

Let's bootstrap flux to "connect" a new dedicated GitHub repo to your freshly installed Kubernetes cluster. You will have to type or paste your personal access token.

```sh
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=streaming-applications-gitops \
  --branch=main \
  --context=staging \
  --path=./clusters/staging \
  --personal
```

If you list the namespaces, you should see that the bootstrap command has created a `flux-system` namespace:
```
kubectl get ns
NAME                 STATUS   AGE
default              Active   47m
flux-system          Active   85s
kube-node-lease      Active   47m
kube-public          Active   47m
kube-system          Active   47m
local-path-storage   Active   47m
```

FluxCD Clone the GitHub repository on your machine:

```sh
git clone https://github.com/$GITHUB_USER/streaming-applications-gitops
cd streaming-applications-gitops
```

Let's create two folders to store the Helm charts and Kustomize files:
``` sh
mkdir -p apps/base/simple-streaming-app apps/staging
```

```sh
docker pull confluentinc/kafka-streams-examples
```

```sh
git add -A && git commit -m "add dev cluster" && git push
```

## Creating the Helm Chart
```shell
mkdir deploy && cd deploy
helm create simple-streaming-app
```
TODO: make adjustments

## Install the WeaveOps UI

```shell
brew tap weaveworks/tap  
brew install weaveworks/tap/gitops
```

Install the OSS Weave GitOps dashboard with:
```shell
PASSWORD="admin"  
gitops create dashboard ww-gitops \  
  --password=$PASSWORD \  
  --export > ./clusters/staging/weave-gitops-dashboard.yaml
```


Log into the Github Container Registry with Docker:
```shell
echo $CR_PAT | docker login ghcr.io -u $GITHUB_USER --password-stdin
```

## Build and publish the Docker image

```shell
docker build  -t ghcr.io/gphilipp/simple-streaming-app:0.1.0 .
```

```shell
docker push ghcr.io/gphilipp/simple-streaming-app:0.1.0
```

## Create and publish the Helm Chart

TODO
```shell
helm create 
```

Log into the Helm Registry
```sh
echo $GITHUB_TOKEN | helm registry login ghcr.io/$GITHUB_USER --username $GITHUB_USER --password-stdin
```

```shell
 cd deploy 
 helm package simple-streaming-app`
```
 
Next,  push it to GitHub Container Registry
```sh
 export CHART_VERSION=$(grep 'version:' ./simple-streaming-app/Chart.yaml | tail -n1 | awk '{ print $2 }')
helm push ./simple-streaming-app-${CHART_VERSION}.tgz oci://ghcr.io/$GITHUB_USER/charts/
```

Browse your Helm Chart repository and verify that it's there:
```sh
open https://github.com/users/$GITHUB_USER/packages/container
```

Create a secret for your token:
```shell
flux create secret oci ghcr-auth \
  --url=ghcr.io \
  --username=flux \
  --password=${GITHUB_TOKEN}
# oci secret 'ghcr-auth' created in 'flux-system' namespace
```

Save this file under `apps/base/my-helm-repository.yaml` 
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: my-helm-repository
  namespace: flux-system
spec:
  interval: 10m
  url: oci://ghcr.io/gphilipp/charts
  type: oci
  secretRef:  
    name: ghcr-auth
```

```sh
flux get kustomizations --watch
```

```sh
kubectl create namespace simple-streaming-app
```
Make this namespace the active one:
```
kubens simple-streaming-app
```
## Install the Sealed Secrets controller

Install the `kubeseal` CLI:
```bash
brew install kubeseal
```

Create a Flux `Helm repository` resource that points to the sealed-secrets Helm chart:
```shell
flux create source helm sealed-secrets \
    --url https://bitnami-labs.github.io/sealed-secrets \
    --interval 1h \
    --export \
    > infrastructure/controllers/sealed-secrets-source.yaml
```

Create a Flux `Helm release` resource:

```bash
flux create helmrelease sealed-secrets \
    --interval=1h \
    --release-name=sealed-secrets \
    --target-namespace=flux-system \
    --source=HelmRepository/sealed-secrets \
    --chart=sealed-secrets \
    --chart-version=">=1.15.0-0" \
    --crds=CreateReplace \
    --export \
    > infrastructure/controllers/sealed-secrets-release.yaml
```

Deploy the Sealed Secrets controller via GitOps:
```
git add infrastructure/
git commit -m "Deploy Bitnami sealed secrets"
git push
```
The reconciliation process will automatically deploy the controller.

Now, retrieve the public key from the `sealed-secrets` controller with `kubeseal`:
```sh
kubeseal --fetch-cert \
    --controller-name=sealed-secrets \
    --controller-namespace=flux-system \
    > pub-sealed-secrets.pem
```


We're going to create a Kubernetes secret locally for each confidential parameter. Let's start with the bootstrap server.
```shell
kubectl create secret generic bootstrap-server \  
    --from-literal=secret=YOUR_BOOTSTRAP_SERVER_HERE \  
    --dry-run=client \  
    -o yaml > bootstrap-server-secret.yaml
    
```

## Sealing secrets
Now seal this secret
```bash
kubeseal --format=yaml --cert=pub-sealed-secrets.pem \  
    < bootstrap-server-secret.yaml \  
    > bootstrap-server-sealed-secret.yaml
```

Now, perform those two steps above for each of the remaining secrets we need: 
- the Cluster API key and secret. 
- the Schema Registry URL.
- the Schema Registry API key and secret.

You will end up with 6 files: 
- `bootstrap-server-sealed-secret.yaml`
- `cluster-api-sealed-secret.yaml`
- `cluster-secret-sealed-secret.yaml`
- `schema-registry-url-sealed-secret.yaml`
- `schema-registry-api-key-sealed-secret.yaml`
- `schema-registry-api-secret-sealed-secret.yaml`

Move those under `apps/staging` and commit:
```shell
mv *sealed-secret.yaml apps/staging
git add apps/staging
git commit -m "Add sealed secrets"
git push
```


## Docker secret sealed

Generate the docker registry secret:
```shell
kubectl create secret docker-registry docker-regcred \
--dry-run=client \
--docker-server=ghcr.io \
--docker-username=gphilipp \
--docker-password=$GITHUB_PAT \
--namespace=demo-apps \
-o yaml > docker-secret.yaml
```

Seal the docker registry secret:
```shell
kubeseal --format=yaml --cert=pub-sealed-secrets.pem < docker-secret.yaml > docker-secret-sealed-secret.yaml
```

```shell
kubectl apply -f docker-secret-sealed-secret.yaml
```
