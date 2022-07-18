# Kpack
kpack is a Kubernetes native platform maintained by VMware under the VMware Tanzu project that utilizes unprivileged Kubernetes primitives to provide builds of OCI images as a platform implementation of Cloud Native Buildpacks (CNB).

### Prerequisites
1. kpack is installed and available on a kubernetes cluster.

2. kpack cli - Get the kp cli from the github release

### Install Kpack on a k8s cluster
```aidl
# Will be applied to kpack namespace by default
kubectl apply -f https://github.com/pivotal/kpack/releases/download/v0.5.4/release-0.5.4.yaml

# Make sure all pods are running and the enviroment is ready
kubectl get pods --namespace kpack --watch

```

### Install Kpack CLI
```aidl
# Go to:
https://github.com/vmware-tanzu/kpack-cli/releases/tag/v0.6.0

# Download the binary file of kp, for example:
kp-windows-0.6.0.exe 

# Configurate kp binary in your PATH enviroment variable
```

### Configure Kpack to push images to a docker registry 
1. Create a secret to store docker registery config
```aidl
#export creds as env vars
export DOCKERHUB_USER=<username>
export DOCKERHUB_PASSWORD=<password>

kubectl create secret docker-registry docker-registry-credentials \
    --docker-username=$DOCKERHUB_USER \
    --docker-password=$DOCKERHUB_PASSWORD \
    --docker-server=https://index.docker.io/v1/ \
    --namespace default
```
2. Create a service account that references the registry secret created above
```aidl
kubectl apply -f cluster/kpack-sa.yaml
kubectl delete -f cluster/kpack-sa.yaml

kubectl get sa
```

3. Create a cluster store configuration
```aidl
kubectl apply -f cluster/cluster-store.yaml
kubectl delete -f cluster/cluster-store.yaml

kp clusterstore status default-cluster-store
```

4. Create a cluster stack configuration

- A stack resource is the specification for a cloud native buildpacks stack used during build and in the resulting app image.
```aidl
kubectl apply -f cluster/cluster-stack.yaml
kubectl delete -f cluster/cluster-stack.yaml

kp clusterstack status base
```

5. Create a Builder configuration

- A Builder is the kpack configuration for a builder image that includes the stack and buildpacks needed to build an OCI image from your app source code.
- The Builder configuration will write to the registry with the secret configured in step one and will reference the stack and store created in step three and four. The builder order will determine the order in which buildpacks are used in the builder.
```aidl
kubectl apply -f cluster/builder.yaml
kubectl delete -f cluster/builder.yaml

# View your newly created cluster-builder
kp clusterbuilder status my-cluster-builder

kubectl get clusterbuilders

```

6. Create an image resource
- An image resource is the specification for an OCI image that kpack should build and manage.
```aidl
kubectl apply -f cluster/image.yaml
kubectl delete -f cluster/image.yaml

# View image resource status
kubectl get images
kubectl get image my-image
kp image status my-image

# View build logs of my-image
kp build logs my-image

# View builds
kubectl get builds
#
```

# Install ChartMuseum 

Download ChartMuseum CLI
```aidl
curl https://raw.githubusercontent.com/helm/chartmuseum/main/scripts/get-chartmuseum | bash

# or download chartmuseum CLI binary from
# https://github.com/helm/chartmuseum/releases
```

Run ChartMuseum 
```aidl
chartmuseum --debug --port=9090 \
  --storage="local" \
  --storage-local-rootdir="./chartstorage"
```
Add ChartMuseum to repos
```aidl
helm repo add chartmuseum http://localhost:9090/
```

Install helm plugin `cm-push`
```aidl
helm plugin install https://github.com/chartmuseum/helm-push
```
Push app chart to chartmuseum
```aidl
helm cm-push chart/ chartmuseum
```
Install your chart from chartmuseum onto the cluster
```aidl
helm repo update
helm search repo chartmuseum/example-app-chart

helm install chartmuseum/example-app-chart --generate-name
helm list
helm uninstall <release name>

# View deployment
kubectl get deployments
kubectl get deployment example-app -owide

# View pod 
kubectl get pods

kubectl describe pod example-app-c4db594b-4zvxz
kubectl logs example-app-c4db594b-4zvxz


#View app 
kubectl port-forward example-app-c4db594b-4zvxz 8080:8080
#app should be running on http://localhost:8080/
```

# FluxCD

Install flux CLI onto windows from binary:
```aidl
https://github.com/fluxcd/flux2/releases
```
Create a Github personal access token to allow flux configure the monitored repos 
```aidl
https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token
```
Export your GitHub personal access token and username as env vars:
```aidl
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```
Check you have everything needed to run Flux by running the following command:
```aidl
flux check --pre
```

## Install Flux onto your cluster
The bootstrap command above does following:

1. Creates a git repository called `infrastructure` under your GitHub account.
2. Adds Flux component manifests to the `infrastructure` repository
3. Deploys Flux Components to your Kubernetes Cluster
4. Configures Flux components to track the path specified in the repository (`/my-cluster`).
```aidl
flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=$GITHUB_USER \
  --repository=infrastructure \
  --branch=main \
  --path=./my-cluster \
  --read-write-key \
  --personal

//flux bootstrap github \
//  --owner=$GITHUB_USER \
//  --repository=infrastructure \
//  --branch=main \
//  --path=./my-cluster \
//  --personal
//
//flux bootstrap github \
//  --components-extra=image-reflector-controller,image-automation-controller \
//  --owner=$GITHUB_USER \
//  --repository=example-app \
//  --branch=main \
//  --path=./manifests \
//  --read-write-key \
//  --personal
```

## Infrastructure repo
Clone the newly created repo
```aidl
git clone https://github.com/GuyBalmas/infrastructure.git

cd infrastructure
```
kubectl -n default get deployments,services -owide

kubectl -n flux-system get deployments,services

