# Flux v2: 
### Image update automation for helm releases 
## Prerequisites:
### 1. Install flux CLI
Install flux CLI onto windows from binary:
```aidl
https://github.com/fluxcd/flux2/releases
```
### 2. Configure GitHub personal access token
Create a Github personal access token to allow flux configure the monitored repos.

Be sure to give it read-write access to your projects. 
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
The bootstrap command does following:

- Creates a git repository called `infrastructure` under your GitHub account.
- Adds Flux component manifests to the `infrastructure` repository.
- Deploys Flux Components to your Kubernetes Cluster.
- Configures Flux components to track the path specified in the repository (`/my-cluster`).

1. Install Flux with the image automation controllers, by boostrap command:
https://fluxcd.io/docs/guides/image-update/
```aidl
flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=$GITHUB_USER \
  --repository=infrastructure \
  --branch=main \
  --path=./my-cluster \
  --read-write-key \
  --personal
```
2. Create an `ImageRepository` to tell Flux which container registry to scan for new tags:
```aidl
flux create image repository dockerhub \
--image=guybalmas/example-app \
--interval=1m \
--export > ./my-cluster/dockerhub-registry.yaml

#if you are accessing a private repo, create a secret, then add it to `spec.secretRef.name` to the ImageRepository
kubectl create secret docker-registry docker-registry-credentials \
    --docker-username=$DOCKERHUB_USER \
    --docker-password=$DOCKERHUB_PASSWORD \
    --docker-server=https://index.docker.io/v1/ \
    --namespace flux-system

#list secrets flux-system namespace
kubectl get secrets -n flux-system
```

3. Create an `ImagePolicy` to tell Flux which tags to use when filtering tags:
```aidl
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImagePolicy
metadata:
  name: example-app
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: dockerhub
  filterTags:
    pattern: '^b(\d)+\.(\d)+\.(\d)+'
  policy:
    alphabetical:
      order: asc
```
Or create your own image policy using flux CLI:
```aidl
flux create image policy example-app \
--image-ref=dockerhub \
--select-semver=x.x.x \
--export > ./my-cluster/example-app-new-policy.yaml
```

4. Create an `ImageUpdateAutomation` to tell Flux which Git repository to write image updates to:
```aidl
---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: example-app
  namespace: flux-system
spec:
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        name: imageUpdateBot
        email: fluxbot@guy.com
      messageTemplate: Updated image version
    push:
      branch: main
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  update:
    path: ./my-cluster/apps
    strategy: Setters
```
Or create your own image automation update using flux CLI:
```aidl
flux create image update example-app \
--git-repo-ref=flux-system \
--git-repo-path="./my-cluster/apps" \
--checkout-branch=main \
--push-branch=main \
--author-name=imageUpdateBot \
--author-email=fluxbot@guy.com \
--commit-template="Updated image version" \
--export > ./my-cluster/example-app-image-new-update-automation.yaml
```

check to see if all image resources are synced:
```aidl
#First, commit and push changes to GitHub!

#get image related resources
flux get images all -A

#get each resource individually
flux get images repository -A
flux get images policy -A
flux get images update -A
```

5. Create a `HelmRepository` to tell Flux which helm registry to scan for new tags:
```aidl
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: chartmuseum
  namespace: flux-system
spec:
  interval: 5m0s
  url: http://192.168.229.1:9090/
```

6. Create a `HelmRelease` to tell Flux which charts to deploy and update upon new images.
This file should be placed inside `my-cluster/apps/` folder
```aidl
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: example-app
  namespace: flux-system
spec:
  interval: 5m
  chart:
    spec:
      chart: example-app-chart
      version: ">=3.0.0"
      sourceRef:
        kind: HelmRepository
        name: chartmuseum
        namespace: flux-system
      interval: 1m
  values:
    image: guybalmas/example-app:b1.20220717.034631 # {"$imagepolicy": "flux-system:example-app"}
```
After you added the `HelmRepository` and `HelmRelease` components, commit and push your changes to GitHub.

## Continuous Delivery
Reconcile everything:
```aidl
#reconcile the whole flux-system
flux reconcile ks flux-system --with-source

#or reconcile individual components
flux reconcile source helm chartmuseum
flux reconcile source git example-app
flux reconcile helmrelease example-app
flux reconcile image update example-app

#watch helm related resources
flux get helmreleases all -A
flux get sources all -A

#debug helmRelease
kubectl describe helmrelease -n flux-system example-app

#debug helmChart
kubectl describe helmchart flux-system-example-app -n flux-system

#check chart version 
helm list -n flux-system

#check for image version
kubectl get deployments,services -owide

#or 
kubectl get pods 
kubectl get pod example-app-f5fcdd5c4-pb5g4 -o yaml | grep "image:"
kubectl describe pod example-app-f5fcdd5c4-pb5g4

#port-forword for visability
kubectl get pods 
kubectl port-forward example-app-d8c586df9-s6r7g 8080:8080 
```

## Image Update Automation
1. Commit & push to GitHub a new source code change from your application repository.
(I will be adding a new endpoint to the application controller under `/welcome`)


2. Watch the new image being built:
```aidl
# View image resource status
kubectl get images
kp image status my-image

# View build logs of my-image
kp build logs my-image

# View builds
kubectl get builds
```
3. Once the image is uploaded to the artifactory, the new tag should be picked up by the `ImagePolicy` we created:
```aidl
#reconcile the whole flux-system
flux reconcile ks flux-system --with-source

#get image related resources
flux get images all -A
```
4. After the tag was picked up, the helm controller should patch the application pod with the new image:
```aidl
#check chart version 
helm list -n flux-system

#check for image version
kubectl get deployments,services -owide
```
5. View commit made by the `ImageUpdateAutomation` config we created, patching the image version tag:
```aidl
https://github.com/GuyBalmas/infrastructure/commits
```

## Chart Automation Updates
1. Commit & push to GitHub a chart configuration change, in the application repo:
   Change the `message.greetings` value located in the `values.yaml` of the app chart.
```aidl
message:
  greetings: "Welcome to Kpack + Flux demo!"
```
2. Bump the chart version under the `Chart.yaml` file of the app chart.
```aidl
name: example-app-chart
version: 3.0.1
```
3. Push the new chart to ChartMuseum:
```aidl
helm cm-push chart/ chartmuseum
```
4. Reconcile 
```aidl
#reconcile the whole flux-system
flux reconcile ks flux-system --with-source

#get image related resources
flux get images all -A
```
5. Watch the new chart being deployed with the new image:
```aidl
#check chart version 
helm list -n flux-system

#check for image version
kubectl get deployments,services -owide
```