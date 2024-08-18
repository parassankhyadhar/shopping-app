# Spring Boot Application with Azure DevOps, Docker, and Kubernetes

## Overview

This project is a simple Spring Boot application running on port 8080. The application is containerized using Docker and deployed to an Azure Kubernetes Service (AKS) cluster. The source code, including the Dockerfile and Kubernetes manifest file, is managed in Azure Repos. An Azure DevOps pipeline is configured to build the Docker image, push it to Azure Container Registry (ACR), and deploy it to AKS.

## Project Structure

- **Source Code:** The Spring Boot application source code is located in the root directory.
- **Dockerfile:** The Dockerfile is used to build a Docker image for the Spring Boot application.
- **Kubernetes Manifests:** The manifest file defines the deployment and service for the application in the AKS cluster.
- **Azure Pipelines YAML:** The pipeline YAML file automates the build, push, and deployment process.

## Dockerfile

The Dockerfile is divided into two stages:
1. **Build Stage:** Uses Maven to build the Spring Boot application and package it as a WAR file.
2. **Run Stage:** Deploys the WAR file on a Tomcat server.

```dockerfile
FROM maven:3.6.3-openjdk-17 AS builder
WORKDIR /app
COPY . .
RUN mvn clean package

FROM tomcat:10.1.20-jdk17-temurin
COPY --from=builder /app/target/demoapp-0.0.1-SNAPSHOT.war .
RUN mv demoapp-0.0.1-SNAPSHOT.war demoapp.war && cp /usr/local/tomcat/demoapp.war /usr/local/tomcat/webapps/
EXPOSE 8080
CMD [ "catalina.sh", "run" ]
```
## Kubernetes Deployment

The Kubernetes manifest file includes a Deployment and Service definition. The Deployment uses a rolling update strategy to ensure smooth updates, and the Service exposes the application on port 8080.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  shopping-app-deployment
  labels:
    name:  shopping-app-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      name: shopping-app-pod
  strategy:
    rollingUpdate:
      maxSurge: 4
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: shopping-app-pod
    spec:
      containers:
      - image: k8sacr.azurecr.io/shopping-app:latest
        name:  app-con
      restartPolicy: Always

---

apiVersion: v1
kind: Service
metadata:
  name: shopping-app-service
spec:
  type: ClusterIP
  ports:
    - port: 8080
  selector:
    name: shopping-app-pod
```
## Azure DevOps Pipeline

The Azure DevOps pipeline automates the following steps:
1. **Build and Push Docker Image:** Builds the Docker image from the Dockerfile and pushes it to Azure Container Registry (ACR).
2. **Create Kubernetes Secret:** Creates a Docker registry secret in the AKS cluster to authenticate with ACR.
3. **Deploy to AKS:** Deploys the application to the AKS cluster using the Kubernetes manifest file.

### Pipeline YAML

```yaml
trigger:
- main

variables:
  acrName: "k8sacr.azurecr.io"
  imageName: "shopping-app"
  tag: "$(Build.BuildId)"

stages:
  - stage: Build_and_Push
    displayName: Build and Push to ACR
    jobs:
      - job: Build
        displayName: Build Image
        pool:
          vmImage: 'ubuntu-latest'
        steps:
        - task: Docker@2
          inputs:
            containerRegistry: 'acrsc'
            repository: $(imageName)
            command: 'buildAndPush'
            Dockerfile: '**/Dockerfile'
        - task: KubernetesManifest@1
          inputs:
            action: 'createSecret'
            connectionType: 'kubernetesServiceConnection'
            kubernetesServiceConnection: 'akssc'
            namespace: 'default'
            secretType: 'dockerRegistry'
            secretName: 'acr-secret'
            dockerRegistryEndpoint: 'acrsc'
        - task: KubernetesManifest@1
          inputs:
            action: 'deploy'
            connectionType: 'kubernetesServiceConnection'
            kubernetesServiceConnection: 'akssc'
            namespace: 'default'
            manifests: '$(System.DefaultWorkingDirectory)/manifests/deployment.yaml'
            containers: 'k8sacr.azurecr.io/shopping-app:$(tag)'
            imagePullSecrets: 'acr-secret'
```
## Conclusion

This project demonstrates a simple yet powerful way to manage, build, and deploy a Spring Boot application using Docker, Kubernetes, and Azure DevOps. The setup ensures continuous integration and delivery, providing a scalable and maintainable solution for running applications in a cloud environment.