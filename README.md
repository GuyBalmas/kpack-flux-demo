# GitOps CICD Demo
### Using Kpack and Flux V2

The goal of this demo is to demonstrate how to implement CICD workflow with image update automation for a CRD (HelmRelease).

We will be using:
- **Kpack** – for continuous integration
- **Flux V2** – for continuous delivery

### What is Kpack?
`Kpack` a Kubernetes-native build service that builds container images on Kubernetes using Cloud Native Buildpacks. 
It takes source code repositories (like `GitHub`), builds the code into a container image, and uploads it to the container registry of your choice.

### Kpack main benefits for this demo
- **No Dockerfile** - Kpack builds OCI-compliant containers directly from source code, eliminating the need for developers to build and maintain dockerfiles.
- **Manage image build process** – build and push to image repository.
- **Manage image updates** - detects changes to source code, dependency, or OS components and automatically updates containers.

### What is Flux?
- `Flux` is a tool for keeping Kubernetes clusters in sync with sources of configuration (like Git repositories), and automating updates to configuration when there is new code to deploy.
- `Flux` is Based on a set of Kubernetes API extensions (CRDs) which control how git repositories and other sources of configuration (like helm repositories) are synced into the cluster.

### Flux main benefits for this demo
- **Just push to Git and Flux does the rest** - enables continuous application deployment (CD) through automatic reconciliation.
- **Provides GitOps for both apps and infrastructure** - can also manage any Kubernetes resource.
- **Automatic image update** – Flux V2 can even push back to `Git` with automated container image updates, deploy those changes, and keep your deployments and CRDs up-to-date.

### Prerequisites 
What I used is in (`parenthesis`)
1. Running Kubernetes cluster (`minikube version: v1.25.2`)
2. Running Docker (`Docker version 20.10.16`)

### Steps
1. Write simplistic Java code and push to git (`Spring-Boot`)
2. After the code is pushed to git, build the code to an image with `Kpack` and push it to an image repository (`DockerHub`)
3. Create a helm chart to deploy the application, and put in helm repository (`ChartMuseum`)
4. Configure Flux to update the helm repository and deploy the application automatically once there is a new image in the repository.

## Step 1 - Write a Java app
I implemented a simple SpringBoot web app with a '`Hello World!`' page, starting up on `http://localhost:8080/`
```
git clone https://github.com/GuyBalmas/example-app.git
```
delete the `.git` folder and push the repo to your `GitHub`

## Step 2 - build the code to an image with `Kpack`
Follow the `Kpack` README.md at:
```aidl
kpack-flux-demo/kpack/README.md

#or

https://github.com/GuyBalmas/kpack-flux-demo/blob/main/kpack/README.md
```

## Step 3 - Create a helm chart and push to helm repository
I implemented a helm chart for the `example-app` SpringBoot web app, and it exists in its repo.
```aidl
https://github.com/GuyBalmas/example-app/tree/main/chart
```

Follow the `Install ChartMuseum` section of the `Kpack` README.md:
```aidl
kpack-flux-demo/kpack/README.md

#or

https://github.com/GuyBalmas/kpack-flux-demo/blob/main/kpack/README.md#install-chartmuseum
```

## Step 4 - Configure Flux
Follow the `Flux` README.md at:
```aidl
kpack-flux-demo/flux/README.md

#or

https://github.com/GuyBalmas/kpack-flux-demo/blob/main/flux/README.md
```

