# Guide to "blue-green" and "canary" deployments using GitHub Actions

## What is Blue/Green deployment strategy in Kubernetes?
| ![blue-green-deploy-process](https://github.com/gauravthakur02/action-deployments/blob/2c3e03a0bc9503d87dbcf17606b6cdd377c77925/img/blue-green-deployment-process.gif) |
| :--: |
| *Blue/Green Deployment* |
>Blue/Green deployments are a form of progressive delivery where a new version of the application is deployed while the old version still exists. The two versions coexist for a brief period of time while user traffic is routed to the new version, before the old version is discarded (if all goes well).

---
## What is Canary deployment strategy in Kubernetes?
| ![canary-deploy-process](https://github.com/gauravthakur02/action-deployments/blob/2c3e03a0bc9503d87dbcf17606b6cdd377c77925/img/canary-deploy.gif) |
| :--: |
| *Canary Deployment* |
>Canary deployment strategy involves deploying new versions of an application next to stable production versions to see how the canary version compares against the baseline before promoting or rejecting the deployment. 

---
### **This is a simple tutorial on how to do [Blue/Green Deployment] and [Canary Deployment] on Kubernetes.**

### Prerequisites
* Any Kubernetes cluster 1.3+ should work. Create an [AKS Cluster](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough) using [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/):
```
az aks create --resource-group myResourceGroup --name myAKSCluster --node-count 1 --enable-addons monitoring --generate-ssh-keys
```
Connect, Start and Stop AKS Cluster using [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/aks/cluster?view=azure-cli-latest):
```
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
az aks start --name myAKSCluster --resource-group myResourceGroup
az aks stop --name myAKSCluster --resource-group myResourceGroup
```
* Application source code
* Docker image of the application
* Kubernetes manifest files _(blue-deploy.yaml, green-deploy.yaml, service.yaml)_

---
1. **Create Docker image from Dockerfile in `/nginx-html`:**
>Edit the `index.html` file to get different webpage message on each new docker image created from the below `Dockerfile`. (For this demo, I have created two docker images "_demo.azurecr.io/blue-nginx:1_" and "_demo.azurecr.io/green-nginx:1_")
```Dockerfile
FROM ubuntu

RUN apt-get update

RUN apt-get install nginx -y

COPY index.html /var/www/html/

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Build docker image using below command, e.g.:
```
docker build -t demo.azurecr.io/blue-nginx:1 .
```
```
docker build -t demo.azurecr.io/green-nginx:1 .
```
Upload the docker image to ACR/DockerHub (we are using ACR):
```
docker push demo.azurecr.io/blue-nginx:1
```
```
docker push demo.azurecr.io/green-nginx:1
```
---
2. **Create Kubernetes manifest files in `/kubernetes`:**
* Create `blue-deploy.yaml` file:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
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
          image: demogaurav.azurecr.io/blue-nginx:1
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: 200m
              memory: 200Mi
            limits:
              cpu: 500m
              memory: 500Mi
```
* Create `green-deploy.yaml` file:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-2
spec:
  replicas: 3
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
          image: demogaurav.azurecr.io/green-nginx:1
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: 200m
              memory: 200Mi
            limits:
              cpu: 500m
              memory: 500Mi
```
* Create `service.yaml` file:
```yml
apiVersion: v1
kind: Service
metadata: 
  name: nginx
  labels: 
    app: nginx
spec:
  ports:
    - name: http
      port: 80
      targetPort: 80
  selector: 
    app: nginx
  type: LoadBalancer
```
---
3. **Create GitHub Actions workflow in `/.github/workflows`:**
* For Blue/Green deployment, create `blue-green.yaml` file:
```yml
# This is a basic workflow to help you get started with Actions
name: Blue-Green-strategy

# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the main branch
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

env:
  ACR_NAME: demogaurav.azurecr.io
  ACR_REPO_NAME: demogaurav.azurecr.io/docker-java-app
  ARTIFACT_NAME: docker-java-app
  RESOURCE_GROUP: Gaurav-RG
  AKS_CLUSTER_NAME: Gaurav-AKS-cluster
  
# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  deployapp:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v2

      # This action can be used to set cluster context before other actions like azure/k8s-deploy, azure/k8s-create-secret or any kubectl commands (in script) can be run subsequently in the workflow.
      - name: Set cluster context
        uses: azure/aks-set-context@v1
        with:
          creds: '${{ secrets.AZURE_CREDENTIALS }}'
          cluster-name: ${{ env.AKS_CLUSTER_NAME }}
          resource-group: ${{ env.RESOURCE_GROUP }}
      
      # Runs a set of commands using the runners shell
      - name: Deploy app
        uses: azure/k8s-deploy@v1.3
        with:
          namespace: default
          manifests: |
            ./kubernetes/service.yaml
            ./kubernetes/blue-deploy.yaml
          images: |
            demogaurav.azurecr.io/green-nginx:1
          strategy: blue-green
          traffic-split-method: pod
          action: deploy  #deploy is the default; we will later use this to promote/reject

  approveapp:
    runs-on: ubuntu-latest
    needs: deployapp
    environment: akspromotion
    steps:
      - run: echo asked for approval

  promotereject:
    runs-on: ubuntu-latest
    needs: approveapp
    steps:
      - uses: actions/checkout@v2

      - name: Set cluster context
        uses: azure/aks-set-context@v1
        with:
          creds: '${{ secrets.AZURE_CREDENTIALS }}'
          cluster-name: ${{ env.AKS_CLUSTER_NAME }}
          resource-group: ${{ env.RESOURCE_GROUP }}

      - name: Promote App
        uses: azure/k8s-deploy@v1.3
        if: ${{ success() }}
        with:
          namespace: default
          manifests: |
            ./kubernetes/service.yaml
            ./kubernetes/green-deploy.yaml
          images: |
            demogaurav.azurecr.io/green-nginx:1
          strategy: blue-green
          traffic-split-method: pod
          action: promote  #deploy is the default; we will later use this to promote/reject

      - name: Reject App
        uses: azure/k8s-deploy@v1.3
        if: ${{ failure() }}
        with:
          namespace: default
          manifests: |
            ./kubernetes/service.yaml
            ./kubernetes/blue-deploy.yaml
          images: |
            demogaurav.azurecr.io/green-nginx:1
          strategy: blue-green
          traffic-split-method: pod
          action: reject  #deploy is the default; we will later use this to promote/reject
```
| ![Blue-Green Deployment Workflow](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/blue-green/blue-green.png) |
| :--: |
| *Blue/Green Deployment Workflow* |
* For Canary deployment, create `canary.yaml` file:
```yml
# This is a basic workflow to help you get started with Actions

name: Canary-strategy

# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the main branch
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

env:
  ACR_NAME: demogaurav.azurecr.io
  ACR_REPO_NAME: demogaurav.azurecr.io/docker-java-app
  ARTIFACT_NAME: docker-java-app
  RESOURCE_GROUP: Gaurav-RG
  AKS_CLUSTER_NAME: Gaurav-AKS-cluster
  
# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  deployapp:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v2

      - name: Set cluster context
        uses: azure/aks-set-context@v1
        with:
          creds: '${{ secrets.AZURE_CREDENTIALS }}'
          cluster-name: ${{ env.AKS_CLUSTER_NAME }}
          resource-group: ${{ env.RESOURCE_GROUP }}
      
      - name: Deploy app
        uses: azure/k8s-deploy@v1.3
        with:
          namespace: default
          manifests: |
            ./kubernetes/service.yaml
            ./kubernetes/blue-deploy.yaml
          images: |
            demogaurav.azurecr.io/green-nginx:1
          strategy: canary
          traffic-split-method: pod
          action: deploy  #deploy is the default; we will later use this to promote/reject
          percentage: 20
          baseline-and-canary-replicas: 2

  approveapp:
    runs-on: ubuntu-latest
    needs: deployapp
    environment: akspromotion
    steps:
      - run: echo asked for approval

  promotereject:
    runs-on: ubuntu-latest
    needs: approveapp
    steps:
      - uses: actions/checkout@v2

      - name: Set cluster context
        uses: azure/aks-set-context@v1
        with:
          creds: '${{ secrets.AZURE_CREDENTIALS }}'
          cluster-name: ${{ env.AKS_CLUSTER_NAME }}
          resource-group: ${{ env.RESOURCE_GROUP }}

      - name: Promote App
        uses: azure/k8s-deploy@v1.3
        if: ${{ success() }}
        with:
          namespace: default
          manifests: |
            ./kubernetes/service.yaml
            ./kubernetes/green-deploy.yaml
          images: |
            demogaurav.azurecr.io/green-nginx:1
          strategy: canary
          traffic-split-method: pod
          action: promote  #deploy is the default; we will later use this to promote/reject
          percentage: 20
          baseline-and-canary-replicas: 2

      - name: Reject App
        uses: azure/k8s-deploy@v1.3
        if: ${{ failure() }}
        with:
          namespace: default
          manifests: |
            ./kubernetes/service.yaml
            ./kubernetes/blue-deploy.yaml
          images: |
            demogaurav.azurecr.io/green-nginx:1
          strategy: canary
          traffic-split-method: pod
          action: reject  #deploy is the default; we will later use this to promote/reject
          percentage: 20
          baseline-and-canary-replicas: 2
```
| ![Canary Deployment Workflow](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/canary/canary.png) |
| :--: |
| *Canary Deployment Workflow* |

---
4. **Run the `blue-green-strategy` workflow:**
* | *Deployapp* |
  | :--: |
  | ![deployapp](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/blue-green/deployapp.png) |
    * | *Pods* |
      | :--: |
      | ![pod1](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/blue-green/pod1.png) |
    * | *Services* |
      | :--: |
      | ![service1](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/blue-green/service1.png) |
    * | *Application Version 1* |
      | :--: |
      | ![app1](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/blue-green/app1.png) |
* | *Approveapp* |
  | :--: |
  | ![approveapp](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/blue-green/approveapp.png) |
* | *Promotereject* |
  | :--: |
  | ![promotereject](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/blue-green/promotereject.png) |
    * | *Pods* |
      | :--: |
      | ![pod2](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/blue-green/pod2.png) |
    * | *Services* |
      | :--: |
      | ![service2](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/blue-green/service2.png) |
    * | *Application Version 2* |
      | :--: |
      | ![app2](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/blue-green/app2.png) |

---
5. **Run the `canary-strategy` workflow:**
* | *Deployapp* |
  | :--: |
  | ![deployapp](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/canary/deployapp.png) |
    * | *Pods* |
      | :--: |
      | ![pod1](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/canary/pod1.png) |
    * | *Services* |
      | :--: |
      | ![service1](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/canary/service1.png) |
    * | *Application Version 1* |
      | :--: |
      | ![app1](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/canary/app1.png) |
* | *Approveapp* |
  | :--: |
  | ![approveapp](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/canary/approveapp.png) |
* | *Promotereject* |
  | :--: |
  | ![promotereject](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/canary/promotereject.png) |
    * | *Pods* |
      | :--: |
      | ![pod2](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/canary/pod2.png) |
    * | *Services* |
      | :--: |
      | ![service2](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/canary/service2.png) |
    * | *Application Version 2* |
      | :--: |
      | ![app2](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/canary/app2.png) |

---
6. **Additional configurations for workflow setup:**
* Action Secrets:
>Secrets are environment variables that are encrypted. Anyone with collaborator access to this repository can use these secrets for Actions.
Secrets are not passed to workflows that are triggered by a pull request from a fork. Learn more about secrets [here](https://help.github.com/en/actions/automating-your-workflow-with-github-actions/creating-and-storing-encrypted-secrets).

| ![secrets](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/secrets.png) |
| :--: |
| *Action Secrets* |
* Environments:
>Environments are used to describe a general deployment target like `production`, `staging`, or `development`. When a GitHub Actions workflow deploys to an environment, the environment is displayed on the main page of the repository.
Learn more about [environments](https://help.github.com/en/github/working-with-github-actions/managing-environments-in-github-actions).

| ![environments](https://github.com/gauravthakur02/action-deployments/blob/3ee4eb928fa43851e3d27c4f6c39f279f85c2968/img/environments.png) |
| :--: |
| *Environments* |

---
