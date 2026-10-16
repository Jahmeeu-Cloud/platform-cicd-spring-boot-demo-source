# Platform CI/CD and GitOps with Jenkins, Docker, Kubernetes & Argo CD

## Project Overview

This project demonstrates an automated CI/CD and GitOps workflow for deploying a Spring Boot application to Kubernetes.

The goal is to streamline application delivery, improve deployment consistency, and manage Kubernetes deployments through version-controlled configuration. Jenkins handles the CI/CD workflow, Docker packages the application, Kubernetes runs the workloads, and Argo CD synchronizes the cluster with the desired state stored in Git.

This project is part of my hands-on journey toward Platform Engineering and Cloud-Native DevOps.

## Architecture

```text
Developer
   |
   v
Application Source Code (GitHub)
   |
   v
Jenkins CI/CD Pipeline
   |
   v
Build and Package Spring Boot Application
   |
   v
Docker Image
   |
   v
Container Registry
   |
   v
GitOps Configuration Repository
   |
   v
Argo CD
   |
   v
Kubernetes Cluster
   |
   v
Spring Boot Application
```

**Note:** The diagram represents the intended workflow. The exact pipeline stages and registry integration depend on the current Jenkinsfile configuration.

## Technology Stack

| Technology | Purpose |
|---|---|
| Jenkins | CI/CD pipeline automation |
| Git and GitHub | Source control and versioned configuration |
| Docker | Containerization |
| Kubernetes | Container orchestration |
| Argo CD | GitOps-based deployment synchronization |
| Spring Boot | Application framework |
| YAML / Kustomize | Kubernetes configuration management |
| Minikube | Local Kubernetes development and testing |

## Repository Structure

The project uses separate repositories to keep application source code and deployment configuration organised.

### 1. Application Source Repository

`platform-cicd-spring-boot-demo-source`

Contains the Spring Boot application source code and build configuration.

### 2. GitOps Configuration Repository

`platform-cicd-gitops-config`

Contains Kubernetes manifests and the configuration Argo CD uses to reconcile the desired application state with the cluster.

Example structure:

```text
platform-cicd-gitops-config/
└── apps/
    └── spring-boot-app/
        └── base/
            ├── deployment.yaml
            └── service.yaml
```

## Key Features

- **Automated CI/CD:** Demonstrates the use of Jenkins to automate application build and delivery tasks.
- **Containerization:** Packages the Spring Boot application for container-based deployment.
- **Kubernetes Deployment:** Runs the application as a Kubernetes Deployment and exposes it through a Service.
- **GitOps Workflow:** Stores deployment configuration in Git for version control and repeatable deployments.
- **Argo CD Synchronization:** Reconciles the Kubernetes cluster with the configuration in the Git repository.
- **Self-Healing:** Configures Argo CD to restore the desired state when supported resources drift from the Git-defined configuration.
- **Reproducible Configuration:** Keeps application code and deployment manifests in separate repositories.

## GitOps and Self-Healing Demonstration

One of the project's practical exercises is testing Kubernetes drift correction.

The desired replica count is set to three in the GitOps configuration. When the live Deployment was manually scaled down to one replica, Argo CD detected the difference and restored the Deployment to three replicas.

This demonstrates how GitOps helps maintain consistency between the declared configuration and the running cluster.

## Running the Project Locally

### Prerequisites

Install and configure:

- Git
- Docker
- kubectl
- Minikube or another Kubernetes cluster
- Jenkins
- Argo CD
- Java and Maven, if building the application locally

### 1. Clone the GitOps repository

```bash
git clone https://github.com/Jahmeeu-Cloud/platform-cicd-gitops-config.git

cd platform-cicd-gitops-config
```

### 2. Validate the Kubernetes manifests

```bash
kubectl kustomize apps/spring-boot-app/base
```

### 3. Inspect the Kubernetes cluster

```bash
kubectl get nodes
kubectl get pods
kubectl get deployments
kubectl get services
```

### 4. Check the Argo CD application

```bash
kubectl get applications -n argocd
kubectl get application spring-boot-app -n argocd
```

### 5. Verify the deployment

```bash
kubectl get deployment spring-boot-app
kubectl rollout status deployment/spring-boot-app
```

These commands assume that the Kubernetes cluster is running and the Argo CD application has already been configured.

## Project Outcomes

Through this project, I have practised:

- Structuring application source and GitOps configuration repositories.
- Working with Jenkins and CI/CD pipeline concepts.
- Packaging applications for container-based environments.
- Deploying applications using Kubernetes resources.
- Managing deployments through Argo CD.
- Understanding declarative infrastructure and Git as the source of truth.
- Testing automated reconciliation and recovery from configuration drift.

## Future Improvements

- Add automated unit and integration tests.
- Integrate image scanning and dependency vulnerability checks.
- Add secure handling of pipeline credentials and secrets.
- Implement automated Docker image tagging and registry publishing.
- Add Prometheus and Grafana for monitoring and visualisation.
- Introduce deployment notifications and alerting.
- Add automated validation of Kubernetes manifests in CI.
- Expand the project with Helm and environment-specific deployment configurations.

## Author

**Jamiu Olatunji Bakre**

Cloud & DevOps Engineer | Kubernetes | Docker | CI/CD | Terraform | Platform Engineering

- GitHub: [Jahmeeu-Cloud](https://github.com/Jahmeeu-Cloud)
- Portfolio: [CloudWithHorla Portfolio](https://jahmeeu-cloud.github.io/portfolio-project/)
- Dev.to: [@jamiu_cloud](https://dev.to/jamiu_cloud/)

---

This is a hands-on cloud-native engineering project focused on building practical skills in CI/CD automation, Kubernetes deployments, and GitOps-driven application delivery.
