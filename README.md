# streaming-applications-gitops


In this exercise, we're going to see how to deploy and run Kakfa applications with a GitOps approach.

You will:
1. Create a local Kubernetes cluster
2. Install the FluxCD GitOps tool
3. Write and package a simple kafka producing application
4. Deploy this application by just committing code to GitHub, you will not interact with the Kubernetes Cluster directly


### Install Kind

First, [install kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation). 
If you are on a Mac with Homebrew installed, you can just run:

```shell
brew install kind
```

### Install FluxCD

In order to install the FluxCD agent into your Kubernetes cluster,  you will need [install the Flux CI](https://fluxcd.io/flux/installation/#install-the-flux-cli):
```
brew install fluxcd/tap/flux
```

You will need to export your GitHub username and a classic GitHub Personal Access Token, just make sure that this token has the permissions to read/write repositories AND packages too.

```sh
export GITHUB_USER=<your github username>
export GITHUB_TOKEN=<your github personal access token>
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

Let's bootstrap flux to create a new GitHub repository and link it to your freshly installed Kubernetes cluster. You will have to type or paste your personal access token.

```sh
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=streaming-applications-gitops \
  --branch=main \
  --context=staging \
  --path=./clusters/staging \
  --personal
```

Once the bootstrap is done, when you list the namespaces, you should see that the bootstrap command has created a `flux-system` namespace:
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

In order to make changes, you must clone the `streaming-applications-gitops` GitHub repository on your machine:

```sh
git clone https://github.com/$GITHUB_USER/streaming-applications-gitops
cd streaming-applications-gitops
```

## Install the Weave GitOps Dashboard
Before we move on to creating the files necessary to deploy our apps, we're going to install a UI dashboard to help us understand what's going on.

Install the open source Weave GitOps dashboard with:
```shell
mkdir -p infrastructure/controllers

brew tap weaveworks/tap
brew install weaveworks/tap/gitops

PASSWORD="admin"
gitops create dashboard ww-gitops \  
  --password=$PASSWORD \  
  --export > infrastructure/controllers/weave-gitops-dashboard.yaml
```

When the controller is up and running, forward the port to your host machine:
```shell
kubectl port-forward svc/ww-gitops-weave-gitops -n flux-system 9001:9001
```

Point your browser to `https://localhost:9001`. This dashboard will give you a clue to visualize what's going on and troubleshoot issues.

```sh
git add -A && git commit -m "Add Weave Gitops dashboard" && git push
```

## Install the Sealed Secret controller
In order to encrypt our secrets (as Kubernetes just computes a hash for classic secrets) and store them safely in the `streaming-applications-gitops` repository, we're going to use Bitnami's Sealed Secrets.

The first step in doing that is to deploy the Sealed Secrets controller, to do that we need the `kubeseal` CLI:

```bash
brew install kubeseal
```

Next, create a Flux `Helm repository` resource that points to the sealed-secrets Helm chart:
```shell
flux create source helm sealed-secrets \
    --url https://bitnami-labs.github.io/sealed-secrets \
    --interval 1h \
    --export \
    > infrastructure/controllers/sealed-secrets-source.yaml
```

Also create a Flux `Helm release` resource:

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

Deploy the Sealed Secrets controller by pushing the files to GitHub:

```
git add infrastructure/
git commit -m "Deploy Bitnami Sealed Secrets"
git push
```

The reconciliation process will automatically deploy the controller.

You can now retrieve and store on your disk the public key from the `sealed-secrets` controller with `kubeseal`:
```sh
kubeseal --fetch-cert \
    --controller-name=sealed-secrets \
    --controller-namespace=flux-system \
    > pub-sealed-secrets.pem
```

## Create a secret for your Helm Chart registry
Flux will need permission to access your Helm chart registry in order to fetch the helm charts from your own private Github Container Registry. 

Create a secret for your token:
```shell
flux create secret oci ghcr-auth \
  --url=ghcr.io \
  --username=flux \
  --password=${GITHUB_TOKEN}
```

## Create a secret to access your Docker Images registry
You need to generate a docker registry secret, so that Flux can pull images from your own private Github Container Registry.

```shell
kubectl create secret docker-registry docker-regcred \
--dry-run=client \
--docker-server=ghcr.io \
--docker-username=$GITHUB_USER \
--docker-password=$GITHUB_TOKEN \
--namespace=demo-apps \
-o yaml > docker-secret.yaml
```

Just like before, we're going to seal this secret:

kubeseal --format=yaml --cert=pub-sealed-secrets.pem < docker-secret.yaml > apps/base/simple-streaming-app/docker-secret-sealed-secret.yaml

## Create the files under the ./apps folder

Let's create a couple of folders to store the Helm charts and Kustomize files for our future streaming application:
``` sh
mkdir -p apps/base/simple-streaming-app apps/staging
```

Create a file `apps/base/simple-streaming-app/namespace.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-apps
  labels:
    toolkit.fluxcd.io/tenant: demo-dev-team
```

Create a file `apps/base/simple-streaming-app/release.yaml`:
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: simple-streaming-app
  namespace: demo-apps
spec:
  chart:
    spec:
      chart: simple-streaming-app
      reconcileStrategy: ChartVersion
      sourceRef:
        kind: HelmRepository
        name: simple-streaming-app-helm-repo
  install:
    createNamespace: true
  interval: 2m
  releaseName: simple-streaming-app-release-name
```

Create a file `apps/base/simple-streaming-app/repository.yaml`, dont' forget to replace `YOUR_GIHTUB_USER` in the file with your own GitHub username:
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: simple-streaming-app-helm-repo
  namespace: demo-apps
spec:
  interval: 1m
  type: oci
  url: oci://ghcr.io/YOUR_GIHTUB_USER/charts
  secretRef:
    name: docker-regcred
```

Create a file `apps/base/simple-streaming-app/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: demo-apps
resources:
  - namespace.yaml
  - repository.yaml
  - release.yaml
  - docker-secret-sealed-secret.yaml
```

## Build and deploy the example application

Fork the https://github.com/gphilipp/simple-streaming-app repository under your own username.

In the `deploy/simple-streaming-app/values.yaml` file, replace `YOUR_GITHUB_USERNAME` with your own GitHub username.

In this hands-on exercise, for the sake of brevity, we're going to build and package the app manually instead of building a CI/CD pipeline. 

Log into the Github Container Registry with Docker:

```shell
echo $GITHUB_TOKEN | docker login ghcr.io -u $GITHUB_USER --password-stdin
```

First, let's build the Docker image:

```shell
docker build  -t ghcr.io/$GITHUB_USER/simple-streaming-app:1.0.0 . 
docker push ghcr.io/$GITHUB_USER/simple-streaming-app:1.0.0
```
Next up, log into the Helm Registry.
```sh
echo $GITHUB_TOKEN | helm registry login ghcr.io/$GITHUB_USER --username $GITHUB_USER --password-stdin
```

You should now package up the application helm chart.  
```shell
 cd deploy 
 helm package simple-streaming-app`
```
 
Finally, publish it as a package to GitHub Container Registry:
```sh
 export CHART_VERSION=$(grep 'version:' ./simple-streaming-app/Chart.yaml | tail -n1 | awk '{ print $2 }')
helm push ./simple-streaming-app-${CHART_VERSION}.tgz oci://ghcr.io/$GITHUB_USER/charts/
```

Point your browser to your own Helm Chart repository and verify that it's there:
```sh
open https://github.com/users/$GITHUB_USER/packages/container
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

## Create secrets to connect to Confluent Cloud

Create a dry-run normal secret in Kubernetes format into a file: 
```shell
kubectl create secret generic client-credentials \
    --from-literal=bootstrap-server=YOUR_BOOTSTRAP_SERVER \
    --from-literal=cluster-api-key=YOUR_CLUSTER_API_KEY \
    --from-literal=cluster-api-secret=YOUR_CLUSTER_API_SECRET \
    --from-literal=schema-registry-url=YOUR_SCHEMA_REGISTRY_URL \
    --from-literal=schema-registry-api-key=YOUR_SCHEMA-REGISTRY-API-KEY \
    --from-literal=schema-registry-api-secret=YOUR_SCHEMA-REGISTRY-API-SECRET \
    --dry-run=client \
    -o yaml > client-credentials.yaml
```

In my case, the `client-credentials-secret.yaml` file looks like this:
```shell
apiVersion: v1
data:
  bootstrap-server: WU9VUl********
  cluster-api-key: WU9VUl********
  cluster-api-secret: WU9VUl********
  schema-registry-api-key: WU9VUl********
  schema-registry-api-secret: WU9VUl********
  schema-registry-url: WU9VUl********
kind: Secret
metadata:
  creationTimestamp: null
  name: client-credentials
```


```bash
kubeseal --format=yaml --cert=pub-sealed-secrets.pem \
    < client-credentials-secret.yaml \
    > client-credentials-sealed-secret.yaml
```

Let's have a look at the file:
```shell
cat client-credentials-sealed-secret.yaml
```

It should look like this, note that it represents a SealedSecret object.

```shell
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: client-credentials
  namespace: demo-apps
spec:
  encryptedData:
    bootstrap-server: ***********************************
    cluster-api-key: ***********************************==
    cluster-api-secret: ***********************************==
    schema-registry-api-key: ***********************************
    schema-registry-api-secret: ***********************************
    schema-registry-url: ***********************************=
  template:
    metadata:
      creationTimestamp: null
      name: client-credentials
      namespace: demo-apps

```


Move the file under `apps/staging` and commit:
```shell
mv client-credentials-sealed-secret.yaml apps/staging
git add apps/staging
git commit -m "Add sealed secret"
git push
```

## Additional resources
https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry