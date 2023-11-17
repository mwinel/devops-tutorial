### Download Helm

Download the Helm binary suitable for your operating system from the [official Helm GitHub releases page](https://github.com/helm/helm/releases). Select the version that suits your needs and download the appropriate binary.

<!-- ### Initialize Helm

Run the following command to initialize Helm and install Tiller, which is the server-side component of Helm that runs inside your Kubernetes cluster:

```bash
helm init
``` -->

### Verify Installation

Ensure Helm has been installed successfully by checking its version:

```bash
helm version
```

### Download and Install Terraform

Downloading the Terraform binary suitable for your operating system from the [official Terraform website](https://www.terraform.io/downloads.html). Once downloaded, follow the installation instructions provided for your platform.

### Verify Terraform Installation

In your command prompt, run the following command to verify that Terraform has been successfully installed:

```bash
terraform --version
```

### Create a Namespace

Use the following command to create a new namespace named "argocd":

```bash
kubectl create ns argocd
```

### Verify Namespace Creation

You can verify that the namespace has been created by running:

```bash
kubectl get ns
```

Look for "argocd" in the list of namespaces to ensure it was created successfully.

### Add argo helm repository:

Helm uses repositories to store and fetch charts. The default repository is stable, but you can add more repositories based on your needs. Run the following command to add an argocd update:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
```

### Update helm index:

Run the following command to add to include the new charts:

```bash
helm repo update
```

### Search for Charts:

You can search for available argocd charts in the added repositories using the following command:

```bash
helm search repo argocd
```

You should be able to see such a response:

```bash
NAME                      	CHART VERSION	APP VERSION	DESCRIPTION
argo/argocd-applicationset	1.12.1       	v0.4.1     	A Helm chart for installing ArgoCD ApplicationSet
argo/argocd-apps          	1.4.1        	           	A Helm chart for managing additional Argo CD Ap...
argo/argocd-image-updater 	0.9.1        	v0.12.2    	A Helm chart for Argo CD Image Updater, a tool ...
argo/argocd-notifications 	1.8.1        	v1.2.1     	A Helm chart for ArgoCD notifications, an add-o...
argo/argo-cd              	3.35.4       	v2.2.5     	A Helm chart for ArgoCD, a declarative, GitOps ...
```

### Configure Terraform to install ArgoCD and ArgoCD Image Updater

Create a terraform folder. Inside your terraform folder add a values folder. The values folder holds essential configuration details tailored to specific components of your infrastructure.

In this folder add two yaml files `argocd.yaml` and `image-updater.yaml`.

The `argocd.yaml` contains parameters related to the configuration of **ArgoCD**, a GitOps continuous delivery tool for Kubernetes. This could include settings such as repository information, synchronization intervals, or other ArgoCD-specific configurations.

```bash
# terraform/values/argocd.yaml
global:
  image:
    tag: "v2.9.0"

server:
  extraArgs:
    - --insecure
```

The `image-updater.yaml` file is tailored for the Image Updater component within your Terraform-managed infrastructure.

```bash
# terraform/values/image-updater.yaml
image:
  tag: "v0.12.2"

metrics:
  enabled: true
```

In your terraform folder add a `provider.tf` file.

This configures the Helm provider, enabling Terraform to interact with Helm charts for deploying and managing applications on Kubernetes.

```bash
# terraform/provider.tf
provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}
```

Add the `argocd.tf` file. The `argocd.tf` file orchestrates the deployment of the ArgoCD to your Kubernetes cluster using the Helm provider.

```bash
# terraform/argocd.tf

# helm repo add argo https://argoproj.github.io/argo-helm
# helm repo update
# helm install argocd -n argocd --create-namespace argo/argo-cd --version 3.35.4 -f terraform/values/argocd.yaml

resource "helm_release" "argocd" {
  name = "argocd"

  repository       = "https://argoproj.github.io/argo-helm"
  chart            = "argo-cd"
  namespace        = "argocd"
  create_namespace = true
  version          = "3.35.4"

  values = [file("values/argocd.yaml")]
}
```

Add the `image-updater.tf` file. The `image-updater.tf` file orchestrates the deployment of the ArgoCD Image Updater to your Kubernetes cluster using the Helm provider.

```bash
# terraform/image-updater.tf

# helm repo add argo https://argoproj.github.io/argo-helm
# helm repo update
# helm install updater -n argocd argo/argocd-image-updater --version 0.8.4 -f terraform/values/image-updater.yaml

resource "helm_release" "updater" {
  name = "updater"

  repository       = "https://argoproj.github.io/argo-helm"
  chart            = "argocd-image-updater"
  namespace        = "argocd"
  create_namespace = true
  version          = "0.8.4"

  values = [file("values/image-updater.yaml")]
}
```

In the terraform dierectory, run the following command:

```bash
terraform init
```

Run the following command to install argocd and argocd image updater helm charts to your kubernetes cluster:

```bash
terraform apply
```

Run the following command to verify your installations:

```bash
kubectl get pods -n argocd
```

You should be able to see such a response with your pods successfully running.

```bash
NAME                                             READY   STATUS    RESTARTS   AGE
argocd-application-controller-5cbc87966c-whhkn   1/1     Running   0          82s
argocd-dex-server-667cbbf744-fgrbg               1/1     Running   0          82s
argocd-redis-79dd4dc75-d9phr                     1/1     Running   0          82s
argocd-repo-server-5fb9fcb8b6-q9r42              1/1     Running   0          82s
argocd-server-67c894749d-2srjs                   1/1     Running   0          82s
updater-argocd-image-updater-5b5c5575bd-r5v4j    1/1     Running   0          87s
```

### Download Argo CD CLI

Download the latest Argo CD version from [https://github.com/argoproj/argo-cd/releases/latest](https://argo-cd.readthedocs.io/en/stable/cli_installation/).

### Expose the ArgoCD server

In your root folder add the `argocd-server-nodeport.yaml` file that defines a Kubernetes Service resource to expose the ArgoCD server using NodePort, allowing external access to the server.

```bash
# argocd-server-nodeport.yaml

apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: argocd
    meta.helm.sh/release-namespace: argocd
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/instance: argocd
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: argocd-server
    app.kubernetes.io/part-of: argocd
    helm.sh/chart: argo-cd-3.35.4
  name: argocd-server-nodeport
  namespace: argocd
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 8080
    nodePort: 30007
  - name: https
    port: 443
    protocol: TCP
    targetPort: 8080
    nodePort: 30008
  selector:
    app.kubernetes.io/instance: argocd
    app.kubernetes.io/name: argocd-server
  sessionAffinity: None
  type: NodePort
```

In the above resource we specify two ports;

- **HTTP Port**: Exposes port `80` on the service, forwarding traffic to port `8080` on the ArgoCD server. NodePort is set to `30007`.
- **HTTPS Port**: Exposes port `443` on the service, forwarding traffic to port `8080` on the ArgoCD server. NodePort is set to `30008`.

Run the following command to create nodeport:

```bash
kubectl apply -f argocd-server-nodeport.yaml
```

Run the following command to verify nodeport service was created successfully:

```bash
kubectl get svc -n argocd
```

You should see such a response.

```bash
argocd-application-controller          ClusterIP   10.99.20.135     <none>        8082/TCP                     17m
argocd-dex-server                      ClusterIP   10.108.112.244   <none>        5556/TCP,5557/TCP            17m
argocd-redis                           ClusterIP   10.98.91.25      <none>        6379/TCP                     17m
argocd-repo-server                     ClusterIP   10.105.245.96    <none>        8081/TCP                     17m
argocd-server                          ClusterIP   10.109.86.19     <none>        80/TCP,443/TCP               17m
argocd-server-nodeport                 NodePort    10.110.252.205   <none>        80:30007/TCP,443:30008/TCP   73s
updater-argocd-image-updater-metrics   ClusterIP   10.106.62.201    <none>        8081/TCP                     17m
```

Now you can access your argocd dashboard on [http://localhost:30007](http://localhost:30007) or [https://localhost:30008](https://localhost:30008). Please note `locahost` is your current cluster IP address.

### Login into ArgoCD Dashboard

Login password for admin can be retrieved using the command below and later you can update your passord on the dashboard.

```bash
argocd admin initial-password -n argocd
```

### How to log Image Updater status

```bash
kubectl logs -f -l app.kubernetes.io/instance=updater -n argocd
```

### Create a GitOps repo on GitHub

In the context of GitOps, a GitOps repository plays a central and crucial role in managing the entire lifecycle of your applications and infrastructure as a single source of truth. Typically this should be a private repository.

### Add GitHub repository to your ArgoCD server

Prerequisites:
- Ensure that your ArgoCD server is up and running.
- Access you can log into the ArgoCD dashboard.

### Base folder

Clone the GitOps repo create earlier on. In this repo create a base folder named `base`. In the context of GitOps or Kubernetes configuration management, a `base` folder typically serves as a central location for storing foundational and common configurations that are shared across multiple environments or applications.

In the base folder add a `namespace.yaml` file. This file contains the YAML configuration for creating a Kubernetes namespace.

```bash
# base/namespace.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: default
```

Now add a `deployment.yaml` file. This file contains the YAML configuration for deploying a Kubernetes Deployment, which is used to create and manage a set of replicated application pods.

```bash
# base/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          imagePullPolicy: Always
          image: mwinel/nginx-devops-lesson
          ports:
            - containerPort: 80
```

Now add a `kustomization.yaml`. Kustomize is a tool for customizing Kubernetes manifests, allowing you to manage and customize your Kubernetes configurations in a more modular and flexible way.

```bash
# base/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: arbitrary
resources:
  - deployment.yaml
  - namespace.yaml
```

### Enviroments folder

Still in your GitOps repo, create an `enviroments` folder. Inside this folder create a `staging` folder and inside the staging folder create another folder called `my-app`. The enviroments folder will be used to hold our Kustomize overlays used to overide specific parameters in the base resources.

Add a `kustomization.yaml` file in the `environments/staging/my-app` folder. Its purpose is to customize and tailor the base configuration for the specific needs of the staging environment.

```bash
# environments/staging/my-app/kustomization.yaml

namespace: staging
replicas:
  - name: nginx-deployment
    count: 2
images:
  - name: mwinel/nginx-devops-lesson
    newTag: v0.1.0
resources:
  - ../../../base
```

### Initial Commit

Commit all the files and push them to the GitOps repository.

### Add the ArgoCD application deployment resource

Create a folder named `app`. Inside it add a file called `application.yaml`. This file is a configuration for an Argo CD Application, defining how an application should be deployed and synchronized on a Kubernetes cluster.

```bash
# app/application.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: mwinel/nginx-devops-lesson:~v0.1
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/mwinel/devops-tutorial
    targetRevision: main # branch to track
    path: environments/staging/my-app
  destination:
    server: https://kubernetes.default.svc # cluster where argocd is running
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - Validate=true
      - CreateNamespace=false
      - PrunePropagationPolicy=foreground
      - PruneLast=true
```

### Deploy the app

Run the following command to create an initial deployment of your applicaction.

```bash
kubectl apply -f app/application.yaml
```

Check your argocd dashboard to verify your app was deployed successfully.

Update your docker image and push it to docker hub.

Examine your application resource using the following command:

```bash
kubectl get application -n argocd my-app -o yaml | less
```

You should be able to see the later image deployed.