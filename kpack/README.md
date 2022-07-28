# Kpack
kpack is a Kubernetes native platform maintained by VMware under the VMware Tanzu project that utilizes unprivileged Kubernetes primitives to provide builds of OCI images as a platform implementation of Cloud Native Buildpacks (CNB).

## Prerequisites
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

# Rename it as kp.exe and configurate kp binary in your PATH enviroment variable
```

## Configure Kpack to push images to a docker registry
1. Create a secret to store docker registery config for DockerHub
```aidl
#export creds as env vars
export DOCKERHUB_USER=<username>
export DOCKERHUB_PASSWORD=<password>

#create credentials secret
kubectl create secret docker-registry docker-registry-credentials \
    --docker-username=$DOCKERHUB_USER \
    --docker-password=$DOCKERHUB_PASSWORD \
    --docker-server=https://index.docker.io/v1/ \
    --namespace default
```
2. Create a service account that references the registry secret created above
```aidl
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kpack-service-account
  namespace: default
secrets:
  - name: docker-registry-credentials
imagePullSecrets:
  - name: docker-registry-credentials
```
apply onto the cluster:
```aidl
kubectl apply -f kpack/kpack-sa.yaml

#if needed
kubectl delete -f kpack/kpack-sa.yaml

#get all
kubectl get sa
```

3. Create a `ClusterStore` configuration
```aidl
apiVersion: kpack.io/v1alpha2
kind: ClusterStore
metadata:
  name: default-cluster-store
spec:
  sources:
    - image: gcr.io/paketo-buildpacks/java
```
apply onto the cluster:
```aidl
kubectl apply -f kpack/cluster-store.yaml

#if needed
kubectl delete -f kpack/cluster-store.yaml

#get status
kp clusterstore status default-cluster-store
```

4. Create a `ClusterStack` configuration

- A stack resource is the specification for a cloud native buildpacks stack used during build and in the resulting app image.
```aidl
apiVersion: kpack.io/v1alpha2
kind: ClusterStack
metadata:
  name: base
spec:
  id: "io.buildpacks.stacks.bionic" 
  buildImage:
    image: "paketobuildpacks/build:base-cnb"
  runImage:
    image: "paketobuildpacks/run:base-cnb"
```
apply onto the cluster:
```aidl
kubectl apply -f kpack/cluster-stack.yaml

#if needed
kubectl delete -f kpack/cluster-stack.yaml

#get status
kp clusterstack status base
```

5. Create a `ClusterBuilder` configuration

- A Builder is the kpack configuration for a builder image that includes the stack and buildpacks needed to build an OCI image from your app source code.
- The Builder configuration will write to the registry with the secret configured in step one and will reference the stack and store created in step three and four. 
- The builder order will determine the order in which buildpacks are used in the builder.
```aidl
kubectl apply -f kpack/builder.yaml

#if needed
kubectl delete -f kpack/builder.yaml

# View your newly created cluster-builder
kp clusterbuilder status my-cluster-builder

kubectl get clusterbuilders

```

6. Create an `Image` resource
- An image resource is the specification for an OCI image that kpack should build and manage.
```aidl
apiVersion: kpack.io/v1alpha2
kind: Image
metadata:
  name: my-image
  namespace: default
spec:
  tag: guybalmas/example-app
  serviceAccountName: kpack-service-account
  builder:
    kind: ClusterBuilder
    name: my-cluster-builder
  source:
    git:
      revision: main
      url: https://github.com/guybalmas/example-app
  build:
    env:
      - name: "BPL_JAVA_NMT_ENABLED"
        value: "false"
      - name: "BP_JVM_TYPE"
        value: "JRE"
      - name: "BP_JVM_VERSION"
        value: "11"
      - name: "BP_OCI_AUTHORS"
        value: "GuyBalmas"
      - name: "BP_OCI_TITLE"
        value: "example-app"
```
apply onto the cluster:
```aidl
kubectl apply -f kpack/image.yaml

#If needed
kubectl delete -f kpack/image.yaml

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
### We will run a ChartMuseum server outside of the cluster, to simulate a remote Helm Repository
* ChartMuseum will be running on localhost of Win10 OS (IP address)

Download ChartMuseum CLI
```aidl
curl https://raw.githubusercontent.com/helm/chartmuseum/main/scripts/get-chartmuseum | bash

# or download chartmuseum CLI binary from:
# https://github.com/helm/chartmuseum/releases
```

Open terminal and run ChartMuseum
```aidl
chartmuseum --debug --port=9090 \
  --storage="local" \
  --storage-local-rootdir="./chartstorage"
```
Add ChartMuseum to known helm repos
```aidl
helm repo add chartmuseum http://localhost:9090/
```

Install helm plugin `cm-push`
```aidl
helm plugin install https://github.com/chartmuseum/helm-push
```
Push app chart to chartmuseum **from the application repo**
```aidl
helm cm-push chart/ chartmuseum
```
Install your chart from chartmuseum onto the cluster
```aidl
helm repo update
helm search repo chartmuseum/example-app-chart

helm install chartmuseum/example-app-chart --generate-name

#check if helm release has been installed
helm list

#if needed
helm uninstall <release name>

# View deployment
kubectl get deployments
kubectl get deployment example-app -owide

# View pod 
kubectl get pods

# View pod details
kubectl describe pod example-app-c4db594b-4zvxz
kubectl logs example-app-c4db594b-4zvxz

#View app 
kubectl port-forward example-app-c4db594b-4zvxz 8080:8080
#app should be running on http://localhost:8080/
```