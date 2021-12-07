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
