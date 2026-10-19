## Building a CI/CD and GitOps Platform with Jenkins, Docker, Kubernetes and ArgoCD

**From Jenkins credential errors to a working pipeline, Kubernetes deployment, and automated GitOps reconciliation**

**Author:** Jamiu Olatunji Bakre  
**Role:** Cloud, DevOps and Platform Engineer

**Tags:** DevOps, Platform Engineering, Jenkins, CI/CD, Docker, Kubernetes, GitOps, ArgoCD, Maven, Kustomize, Minikube

---

## Project Overview

This project was an important step in my journey towards becoming a stronger Cloud, DevOps and Platform Engineer.

My goal was to understand how application code moves from a GitHub repository through an automated build process, becomes a Docker image, and is ultimately deployed and managed in Kubernetes.

I wanted to go beyond simply installing tools and following commands from a tutorial. I wanted to understand how the components communicate, where configuration belongs, how credentials are handled, how deployment problems are investigated, and how to verify that an application is actually working.

The journey included configuring Jenkins, working with GitHub repositories, building a Spring Boot application with Maven, packaging the application with Docker, and using Kubernetes and ArgoCD to manage the deployment.

One of the most memorable challenges was a Jenkins credential error that took considerable troubleshooting. The pipeline reported that it could not find a credential with the ID `github-credentials`. After working through the integration issues, I achieved a successful pipeline build on my 20th CI/CD pipeline build.

That experience taught me something important: building a pipeline is not just about writing a Jenkinsfile. It also requires understanding credentials, permissions, repository configuration, networking, container registries, deployment manifests, and the dependencies between each stage.


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/qrf2oh7v5wzc16r4ikks.png)

**ArgoCD Output**


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/63kyuiikky715s4d97zq.png)

**Scaled Up ArgoCD Output**


This documentation explains the architecture, the steps involved, the purpose of each component, the commands used to inspect and validate the system, and the improvements I would make when extending the project.

It also includes reference configurations for frontend and backend applications, persistent storage, monitoring, and production-oriented practices. These examples provide a foundation for future work; they should not be interpreted as proof that every optional component was deployed in the original lab.

## 1. Project Objectives

The project was designed around the following objectives:

1. Automate application build and verification using Jenkins.
2. Store application source code in GitHub.
3. Package the application into a Docker image.
4. Publish container images to Docker Hub.
5. Define Kubernetes resources declaratively.
6. Separate application source code from deployment configuration.
7. Use ArgoCD to monitor the GitOps repository.
8. Test automated reconciliation when the live cluster configuration differs from Git.
9. Verify the deployed application's availability through Kubernetes and an HTTP request.
10. Develop a repeatable troubleshooting and documentation process.

The wider objective was to understand how these capabilities fit together as a small platform engineering workflow.

## 2. Project Architecture

The architecture separates the application build process from the deployment reconciliation process.

Jenkins runs on an AWS EC2 instance. The Kubernetes application and ArgoCD run in a local Minikube environment.

This separation is important because a Jenkins server on EC2 cannot automatically reach a Kubernetes cluster running on a developer's local computer. The two environments have different network boundaries.

### Architecture diagram

```text
                    Developer
                        |
                        v
               Application Changes
                        |
                        v
                 GitHub Source Repo
                        |
                        v
                  Jenkins on EC2
                        |
             +----------+-----------+
             |          |           |
             v          v           v
           Maven     Code Checks   Docker Build
             |                      |
             v                      v
        Build/Test               Docker Hub
                                    |
                                    v
                          GitOps Repository
                                    |
                                    v
                                ArgoCD
                                    |
                                    v
                           Local Minikube
                                    |
                                    v
                            Kubernetes
                                    |
                                    v
                         Spring Boot App
```

The diagram describes the intended delivery flow. The exact automation depends on which stages are implemented in the Jenkinsfile. For example, updating the GitOps repository must be explicitly configured if it is to happen automatically after an image is published.

### The role of each component

| Component | Responsibility |
|---|---|
| GitHub | Stores application source code and deployment manifests |
| Jenkins | Orchestrates build, test, and image publication stages |
| Java and Spring Boot | Provide the application runtime and framework |
| Maven | Compiles, tests, and packages the application |
| Docker | Packages the application and its runtime dependencies |
| Docker Hub | Stores and distributes container images |
| Kubernetes | Schedules and manages application Pods |
| Minikube | Provides a local Kubernetes environment |
| ArgoCD | Reconciles Kubernetes resources with Git-defined desired state |
| Kustomize | Organises and renders Kubernetes configuration |
| SonarQube | Optional code-quality analysis when configured in the pipeline |

## 3. Repository Structure and Separation of Responsibilities

I used two GitHub repositories to separate application development from deployment configuration.

### 3.1 Application source repository

Repository: `platform-cicd-spring-boot-demo-source`

URL - https://github.com/Jahmeeu-Cloud/platform-cicd-spring-boot-demo-source

This repository holds the Spring Boot application and its Maven build configuration.

A typical structure looks like this:

```text
platform-cicd-spring-boot-demo-source/
├── spring-boot-app/
│   ├── pom.xml
│   ├── mvnw
│   ├── .mvn/
│   └── src/
│       ├── main/
│       │   ├── java/
│       │   └── resources/
│       └── test/
├── Dockerfile
├── Jenkinsfile
├── .gitignore
└── README.md
```

This is a recommended structure, not a claim that every file shown was present in the original repository.

The source repository should contain the code required to build the application. Generated output, credentials, and environment-specific secrets should not be committed.

For example, Maven's `target/` directory contains generated build artifacts. It is normally excluded from Git.

A suitable `.gitignore` could include:

```gitignore
target/
*.log
.env
.env.*
!.env.example
.idea/
.vscode/
.DS_Store
```

The exact exclusions should match the project's development tools and build process.

### 3.2 GitOps configuration repository

Repository: `platform-cicd-gitops-config`

URL: https://github.com/Jahmeeu-Cloud/platform-cicd-gitops-config

This repository contains the desired Kubernetes configuration.

The confirmed base directory is:

```text
platform-cicd-gitops-config/
└── apps/
    └── spring-boot-app/
        └── base/
            ├── deployment.yaml
            ├── service.yaml
            └── kustomization.yaml
```

The Deployment declares how the application should run. The Service provides a stable network endpoint. The Kustomize file identifies the manifests that belong to the application configuration.

The main principle is that the GitOps repository should describe the desired deployment state rather than merely store commands that someone runs manually.

### Why keep the repositories separate?

This separation makes it easier to review application changes independently from deployment changes.

It also creates a clear path for future environment promotion:

- Developers change application code.
- Jenkins validates and packages the code.
- A versioned image is published.
- The deployment configuration references that image.
- ArgoCD reconciles the cluster with the approved configuration.

In a larger platform, the same approach can support separate development, staging, and production configurations.


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/1cba4rv54tlynxng2lr0.png)

[**[SCREENSHOT: GITHUB SOURCE REPOSITORY HERE]**
](https://github.com/Jahmeeu-Cloud/platform-cicd-spring-boot-demo-source/tree/main/spring-boot-app)

*The source repository contains the application and build-related files.*


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/m0de9890aqb9t2j2p0cp.png)

[**[SCREENSHOT - GITHUB GITOPS REPOSITORY HERE]**](https://github.com/Jahmeeu-Cloud/platform-cicd-gitops-config/tree/main/apps/spring-boot-app/base)

*The GitOps repository stores the Kubernetes manifests used to define the desired application state.*

## 4. Understanding the Environment Before Starting

Before troubleshooting a CI/CD pipeline, it is useful to understand where each component is running.


In this project:

- Jenkins ran on AWS EC2.
- The Kubernetes cluster ran locally through Minikube.
- ArgoCD ran inside the Minikube cluster.
- The application image was stored in Docker Hub.
- GitHub stored both the source code and the deployment configuration.

This topology has an important implication: Jenkins does not automatically have network access to the local Kubernetes API.

A common mistake is to install `kubectl` on a remote Jenkins server and assume that this gives it access to a local cluster. It does not. The CLI needs a valid cluster endpoint, credentials, and network connectivity.

In a GitOps architecture, Jenkins can publish the image and update the deployment configuration in Git, while ArgoCD performs the cluster-side reconciliation.

This avoids requiring the Jenkins server to connect directly to the local cluster.

### 4.1 Check Jenkins

On the EC2 instance:

```bash
sudo systemctl status jenkins
```

If the service is not running:

```bash
sudo journalctl -u jenkins -n 100 --no-pager
```

Check the Java runtime if the Jenkins service reports startup or compatibility problems:

```bash
java -version
```

The Java version must be compatible with the installed Jenkins release. Check the official Jenkins documentation for the requirements of the specific version being used.

### 4.2 Check Docker

```bash
docker --version
docker info
```

These commands confirm that the Docker client is installed and can communicate with the daemon under the current user's permissions.

When Jenkins builds Docker images, test access using the actual Jenkins service account where appropriate:

```bash
sudo -u jenkins docker info
```

This command assumes a Linux installation with a service account named `jenkins`.

Do not automatically grant broad privileges to fix an access problem. Membership in the Docker group can effectively provide root-level access to the host. Investigate the specific error and choose a build architecture appropriate to the security requirements.

### 4.3 Check Minikube and Kubernetes

Run these commands on the machine where Minikube is installed:

```bash
minikube status
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -A
```

Expected results include a running Minikube cluster, a reachable Kubernetes API, and a node reporting `Ready`.


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/9jwvdt80nkk5tiyqadif.png)

**[SCREENSHOT - MINIKUBE AND KUBERNETES STATUS HERE]**

*The exact output varies by Kubernetes and Minikube version.*


## 5. How the CI/CD and GitOps Process Works

The complete delivery process has several logical stages.

### Stage 1: Source checkout

Jenkins retrieves the application source code from GitHub.

A checkout stage might look like this:

```groovy
stage('Checkout') {
    steps {
        git branch: 'main',
            url: 'https://github.com/OWNER/REPOSITORY.git'
    }
}
```

Replace `OWNER/REPOSITORY` with the actual repository path.

For a private repository, Jenkins needs a suitable credential. For a public repository, a read-only checkout may not require authentication.

The purpose of checkout is to ensure that the build uses a known source revision rather than files copied manually to the Jenkins server.

### Stage 2: Build and test

For a Maven-based Spring Boot application:

```bash
mvn clean verify
```

The `clean` goal removes previous build output. The `verify` lifecycle phase runs the configured checks leading up to verification.

A successful result means the build and configured verification steps passed. It does not prove that the application has no defects.

If the project uses the Maven Wrapper, prefer:

```bash
./mvnw clean verify
```

The wrapper helps the project use the intended Maven version, subject to the availability of a compatible Java runtime.

### Stage 3: Optional code analysis

SonarQube can be included to assess code quality and identify potential issues.

An illustrative command is:

```bash
mvn clean verify sonar:sonar \
  -Dsonar.host.url="$SONAR_HOST_URL" \
  -Dsonar.token="$SONAR_TOKEN"
```

This requires a correctly configured SonarQube server, compatible scanner configuration, and a valid token.

Do not add this stage to a pipeline until the scanner, server URL, and credentials are configured. A code-quality stage is optional in the reference implementation and should only be described as completed if it was actually executed.

### Stage 4: Build the Docker image

A Docker image packages the application into a standard format that can be stored and run in compatible container environments.

A simple example:

```bash
docker build \
  -t horlacloud/spring-boot-demo:BUILD_TAG \
  .
```

Replace `BUILD_TAG` with a valid version identifier.

For repeatable releases, use a unique build number, release version, or commit-based tag rather than depending only on `latest`.

For example:

```bash
docker build \
  -t horlacloud/spring-boot-demo:1.0.0 \
  .
```

This example assumes the Dockerfile is in the current build context. Adjust the path if it is stored elsewhere.

### Stage 5: Publish the image

After building the image, Jenkins authenticates to Docker Hub and pushes it.

A typical manual command sequence is:

```bash
docker login
docker push horlacloud/spring-boot-demo:1.0.0
```

For an automated pipeline, credentials should be supplied securely through Jenkins Credentials rather than typed into a pipeline or stored in the source repository.

A successful push means the registry accepted the image. It does not prove that Kubernetes can pull it or that the application will start correctly.

### Stage 6: Update deployment configuration

The GitOps configuration must reference the intended image.

For example:

```yaml
containers:
  - name: spring-boot-app
    image: horlacloud/spring-boot-demo:1.0.0
```

If Jenkins is expected to update this value automatically, the Jenkinsfile needs an explicit step to modify the GitOps repository and push the change using an appropriately restricted GitHub credential.

Publishing an image does not automatically update Kubernetes unless the delivery design includes an image-update mechanism or a corresponding configuration change.


### Stage 7: ArgoCD reconciliation

ArgoCD monitors the configured Git repository and path.

When a new desired configuration is committed, ArgoCD can detect the difference and synchronise the managed Kubernetes resources according to its application configuration.

This is the GitOps part of the workflow.


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/ip2ciu948lxw5ziodcil.png)

---


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/7ls820wici2uzbw3204a.png)


**[SCREENSHOT - JENKINS PIPELINE STAGES HERE]**


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/uhpno1ziwry4se5hl93n.png)



**[SCREENSHOT - SUCCESSFUL DOCKER HUB IMAGE PUBLICATION HERE]**

*The build process packages the application into a container image that can be referenced by Kubernetes deployment configuration.*


## 6. The Jenkins Credential Error That Took Hours to Resolve

One of the most valuable lessons from this project came from a Jenkins credential lookup failure.

The error was:

```text
ERROR: Could not find credentials entry with ID 'github-credentials'
```

The important detail is the wording: Jenkins could not find the credential entry with the requested ID.

This is different from a rejected token or an access-denied response.



### 6.1 What is a credential ID?

Jenkins stores credentials separately from the pipeline source code. Each saved credential has an identifier.

For example:

```groovy
credentialsId: 'github-credentials'
```

Here, `github-credentials` is the credential ID. It is not the token itself.

If the saved credential is named `github-token`, referencing `github-credentials` will not work unless a credential with that exact ID exists in the relevant credential store and scope.

### 6.2 Troubleshooting the problem

I approached the error by focusing on the stage that failed instead of repeatedly rebuilding the application.

The checks to perform are:

1. Identify the exact failed stage in the Jenkins console.
2. Inspect the credential ID referenced by the Jenkinsfile.
3. Open Jenkins Credentials and locate the corresponding entry.
4. Confirm the credential's type.
5. Check the credential's scope and availability to the job.
6. Verify that the credential has the permissions required for the GitHub operation.
7. Confirm that the repository URL and branch are correct.

The exact credential ID must match the identifier used in the Jenkinsfile.

The credential type must also match the pipeline binding. A Secret text credential is not the same as a username/password credential or an SSH private key.


### 6.3 Example: username and token

If the credential is stored as a username/password pair, an illustrative Jenkins binding is:

```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'github-credentials',
        usernameVariable: 'GIT_USERNAME',
        passwordVariable: 'GIT_TOKEN'
    )
]) {
    sh '''
        set +x
        # Perform the authenticated operation here.
    '''
}
```

The ID and credential type must match the saved Jenkins configuration.

For a Secret text credential, the binding would instead look like this:

```groovy
withCredentials([
    string(
        credentialsId: 'github-token',
        variable: 'GIT_TOKEN'
    )
]) {
    sh '''
        set +x
        # Use the token without printing it.
    '''
}
```

These are reference patterns, not the exact final code from my environment.

### 6.4 Why secrets should not be stored in the Jenkinsfile

A Jenkinsfile is normally stored in version control. Hardcoding a token into it creates a risk of exposing that token through Git history, logs, pull requests, or copied source code.

The safer approach is to store secrets in Jenkins Credentials and reference them by ID.

Use narrowly scoped credentials, protect pipeline changes, and rotate exposed secrets promptly.

### 6.5 Three similar errors that require different fixes

| Error | Likely meaning | First check |
|---|---|---|
| Credential not found | Jenkins cannot resolve the referenced ID | Credential ID and scope |
| Authentication failed | The authentication method or secret was rejected | Token validity, username, credential type |
| Permission denied | The identity lacks access to the requested resource | Repository permissions and token scope |

This distinction saves time because each error points to a different part of the configuration.


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/udps0p2q9z6d0szahq5k.png)



**[SCREENSHOT - THE SUCCESSFUL PIPELINE BUILD HERE]**

*By the 20th CI/CD pipeline build, I had worked through the integration problems and achieved a successful pipeline build.*


## 7. Building a Spring Boot Application Container

A container image needs a Dockerfile that explains how the application is built or how its packaged artifact is executed.

There are two common approaches.

### Option A: Build the JAR before creating the image

If Maven has already generated the JAR, a simple Dockerfile can copy that artifact into a Java runtime image.

```dockerfile
FROM eclipse-temurin:21-jre

WORKDIR /app

COPY target/*.jar app.jar

EXPOSE 8080

USER 10001

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

This is an illustrative example. The Java runtime version must match the application's requirements, and the build process must ensure that exactly the intended JAR is copied.

The `EXPOSE` instruction documents the expected container port; it does not publish the port by itself.

The non-root user improves the default security posture, provided the application can run with the required filesystem permissions.

### Option B: Use a multi-stage Docker build

A multi-stage Dockerfile can compile the application in a build image and copy only the packaged artifact into the runtime image.

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build

WORKDIR /workspace

COPY pom.xml .
COPY src ./src

RUN mvn -B clean package -DskipTests

FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=build /workspace/target/*.jar app.jar

EXPOSE 8080

USER 10001

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

This example assumes the Maven project is at the Docker build context root and produces one intended executable JAR.

Notice that this sample skips tests during packaging. In a complete CI/CD workflow, tests should run in a dedicated stage before image publication, and the pipeline should stop if they fail. Avoid using `-DskipTests` as a replacement for automated testing.

The runtime image does not need the entire Maven build environment, which can reduce the amount of tooling included in the final container.

### Build and test locally

```bash
mvn clean verify
docker build -t spring-boot-app:local .
docker run --rm -p 8080:8080 spring-boot-app:local
```

Then, in another terminal:

```bash
curl -i http://localhost:8080
```

Use an actual application route if the root path is not defined.


## 8. Deploying the Application with Kubernetes

Kubernetes manages the desired state of containerised workloads. It schedules Pods, monitors workload state, and attempts to maintain the replica count specified in the Deployment.

The project used a Deployment and a Service for the Spring Boot application.


### 8.1 Deployment manifest

The confirmed GitOps configuration declares three replicas and the image `horlacloud/spring-boot-demo:20`.

An illustrative Deployment is:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-app
  labels:
    app: spring-boot-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-boot-app
  template:
    metadata:
      labels:
        app: spring-boot-app
    spec:
      containers:
        - name: spring-boot-app
          image: horlacloud/spring-boot-demo:20
          ports:
            - name: http
              containerPort: 8080
```

This shows the core configuration. A production-oriented Deployment should normally also define appropriate probes, resource requests and limits, and a security context.

The `replicas` field is the desired number of Pods. Kubernetes attempts to maintain that number, but scheduling constraints, image pull failures, application crashes, and insufficient capacity can prevent all replicas from becoming ready.

The selector must match the labels on the Pod template. If the selector and template labels do not match, the Deployment will be invalid or fail to manage the intended Pods.

### 8.2 Service manifest

A Service provides a stable endpoint for matching Pods.

The project's Service is named `spring-boot-app-service`. A representative configuration is:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-boot-app-service
spec:
  type: NodePort
  selector:
    app: spring-boot-app
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
```

Here:

- `port: 80` is the Service port.
- `targetPort: 8080` is the port on the application container.
- `type: NodePort` exposes the Service through a port on the Kubernetes node.

The `selector` identifies the Pods that should receive traffic.

A Service can exist even when it has no ready endpoints. That is why it is important to inspect both the Service and the Pods behind it.

### 8.3 Validate the resources

```bash
kubectl get deployments
kubectl get pods -o wide
kubectl get services
kubectl get endpoints spring-boot-app-service
```

To inspect the Deployment in more detail:

```bash
kubectl describe deployment spring-boot-app
```

To inspect an individual Pod:

```bash
kubectl describe pod POD_NAME
kubectl logs POD_NAME
```

Replace `POD_NAME` with the name returned by `kubectl get pods`.



![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/iva22vhob98fxp5ap4ol.png)


**[SCREENSHOT - KUBERNETES DEPLOYMENT AND POD STATUS HERE]**



![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/zokfty2dfw6pbz8239r7.png)


**[SCREENSHOT - KUBERNETES SERVICE AND ENDPOINTS HERE]**

*Kubernetes was managing the application replicas and exposing them through a stable Service.*


## 9. ConfigMaps, Secrets, and Application Configuration

Applications often need configuration that should not be embedded directly into the container image.

Examples include:

- Application mode or environment name.
- Log level.
- API endpoint URLs.
- Database connection settings.
- Authentication credentials.

Kubernetes provides ConfigMaps for non-sensitive configuration and Secrets for sensitive values.

### 9.1 ConfigMap example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: spring-boot-config
data:
  APP_ENV: "development"
  LOG_LEVEL: "INFO"
```

A Deployment can expose these values to the container:

```yaml
envFrom:
  - configMapRef:
      name: spring-boot-config
```

This belongs under the container specification.

### 9.2 Secret example

A Secret can hold sensitive configuration such as a database password.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
stringData:
  username: app_user
  password: REPLACE_WITH_A_SECRET
```

This is a template only. Replace the placeholder using an appropriate secret-management process.

Do not commit real passwords, access tokens, or private keys to Git. Kubernetes Secret values are not automatically safe simply because they are stored in a Secret object; access controls and encryption-at-rest configuration also matter.

For production environments, consider a suitable external secret-management integration.

### 9.3 Why configuration should be separate

Separating configuration from application code makes it easier to promote the same image across environments without rebuilding it for every setting change.

It also reduces the risk of accidentally baking environment-specific values into a reusable image.

## 10. Persistent Volumes and Persistent Volume Claims

Persistent storage deserves special attention because container storage and application data have different lifecycles.

A container's writable filesystem is generally not intended to be the durable home for important data. If a container is replaced, its writable layer may be lost.

Kubernetes persistent storage provides a way to manage data independently of an individual container's lifecycle.

### 10.1 Understanding the storage components

**PersistentVolume (PV):** Represents a piece of storage available to a Kubernetes cluster.

**PersistentVolumeClaim (PVC):** A request for storage made by an application or workload.

**StorageClass:** Defines how storage can be provisioned. Depending on the cluster, it may support dynamic provisioning so that a PVC can trigger creation of a volume.

**Pod volume mount:** Connects the claimed storage to a path inside a container.

The relationship is:

```text
Application Pod
      |
      v
  Volume Mount
      |
      v
 PersistentVolumeClaim
      |
      v
 PersistentVolume
      |
      v
Underlying Storage
```

When dynamic provisioning is enabled, a PVC may trigger the creation of a PV through its StorageClass.

### 10.2 When should persistent storage be used?

Persistent storage may be appropriate for:

- Database data directories.
- Application files that must survive Pod replacement.
- Durable uploads or other persistent application data.
- Workloads that explicitly require filesystem persistence.

It is generally not necessary for every application container.

For example, a stateless Spring Boot web application may not need a PVC if it stores no durable files locally and uses an external database or object storage.

The right storage design depends on the application's data requirements.

### 10.3 Inspect available StorageClasses

Before creating a PVC, check the cluster:

```bash
kubectl get storageclass
kubectl describe storageclass
```

The available StorageClasses depend on the Kubernetes environment and installed storage provisioners.

A Minikube installation may provide a default StorageClass, but this should be checked rather than assumed.

### 10.4 PVC example

This example requests 1 GiB of persistent storage:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

This example leaves `storageClassName` unspecified so that the cluster's default StorageClass can be used when one is configured.

The claim may remain `Pending` if no suitable StorageClass or matching volume is available.

The access mode `ReadWriteOnce` generally allows a volume to be mounted read-write by a single node at a time; it does not necessarily mean only one Pod can use it. Actual access behaviour depends on the storage implementation.

### 10.5 Mount the PVC into a Pod

The claim can be mounted into a container through a Pod or Deployment specification.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: storage-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: storage-demo
  template:
    metadata:
      labels:
        app: storage-demo
    spec:
      containers:
        - name: storage-demo
          image: busybox:1.36
          command:
            - sh
            - -c
            - "while true; do date >> /data/timestamps.txt; sleep 30; done"
          volumeMounts:
            - name: app-data
              mountPath: /data
      volumes:
        - name: app-data
          persistentVolumeClaim:
            claimName: app-data
```

This is a storage demonstration, not part of the confirmed Spring Boot deployment. It writes timestamps to a file so that persistence can be tested.

Apply the example only in a suitable test namespace:

```bash
kubectl apply -f pvc.yaml
kubectl apply -f storage-demo.yaml
```

Check the claim and Pod:

```bash
kubectl get pvc
kubectl get pv
kubectl get pods
```

Once the Pod is running, inspect the file:

```bash
kubectl exec deploy/storage-demo -- cat /data/timestamps.txt
```

Delete the Pod without deleting the PVC, wait for the replacement Pod, and check the file again. If the same claim is successfully remounted, the file should still contain the previously written data.

The exact behaviour depends on the provisioner and storage configuration.

### 10.6 What happens when a PVC is deleted?

Deleting a PVC can affect the underlying storage depending on the PV reclaim policy and provisioner.

Common reclaim policies include:

- `Retain`: The volume is retained for manual recovery or reuse.
- `Delete`: The underlying storage may be deleted when the claim is released.

Never assume that deleting a PVC is harmless.

Before testing cleanup, inspect the PV and its reclaim policy:

```bash
kubectl get pv
kubectl describe pv PV_NAME
```

Replace `PV_NAME` with the actual volume name.

Persistent storage is not a substitute for backups. A durable volume may survive Pod replacement while still being vulnerable to accidental deletion, corruption, or infrastructure failure.


## 11. Extending the Project to a Frontend and Backend

The original confirmed deployment is a Spring Boot application. If I extend the capstone to a frontend and backend, I would treat them as separate workloads with clearly defined network paths.

For example:

```text
Browser
   |
   v
Frontend Service
   |
   v
Frontend Pods
   |
   v
Backend Service
   |
   v
Backend Pods
   |
   v
Database Service
   |
   v
Database Pods or External Database
```

This is a reference architecture for a future extension.

The frontend could be a React application served through Nginx, while the backend could be a Spring Boot API or a Node.js service. The backend technology depends on the actual application.

### 11.1 Frontend Dockerfile example

For a React application built with Vite:

```dockerfile
FROM node:22-alpine AS build

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM nginx:alpine

COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80
```

This example assumes the project uses `npm`, has a valid lockfile, and generates its production build in `dist/`.

For a single-page application, Nginx may also need a custom configuration to route client-side paths back to `index.html`.

### 11.2 Frontend Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: example/frontend:1.0.0
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

Replace the example image with an image that actually exists in your registry.

The `ClusterIP` Service is reachable inside the cluster. External access can be provided through an Ingress, Gateway, LoadBalancer, or a local development mechanism appropriate to the environment.

### 11.3 Backend Deployment and Service

A generic backend might look like this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: example/backend:1.0.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
```

This is a generic example. The backend must actually expose `/health` and return an appropriate successful response. Spring Boot applications may use a different health endpoint depending on their configuration.

The readiness probe determines whether a Pod should receive Service traffic. The liveness probe helps Kubernetes detect certain application states that require a container restart. They serve different purposes, so they should not automatically use the same endpoint in every application.

### 11.4 How should the frontend reach the backend?

Inside Kubernetes, the frontend can call a backend Service using its DNS name, such as:

```text
http://backend:8080
```

However, a browser-based React application executes on the user's device, not inside the frontend container. A URL such as `http://backend:8080` will not normally resolve in the user's browser.

For browser-based applications, the frontend usually needs a publicly reachable API URL or an Ingress/reverse-proxy arrangement that routes requests to the backend.

The correct approach depends on whether the request originates from browser JavaScript or from a server-side process.

### 11.5 How would I connect this to GitOps?

The frontend and backend manifests could be organised under one application directory:

```text
apps/
└── web-platform/
    ├── base/
    │   ├── frontend.yaml
    │   ├── backend.yaml
    │   └── kustomization.yaml
    └── overlays/
        ├── staging/
        └── production/
```

The base would contain shared configuration. Overlays could define environment-specific image tags, replica counts, resources, and other differences.

This layout should be introduced only when the additional workloads have actually been created and tested.

## 12. Databases and Stateful Workloads

Databases require additional planning because application availability and data durability are different concerns.

A Kubernetes Deployment is commonly used for stateless services. A StatefulSet may be appropriate for a workload that needs stable Pod identities and persistent storage.

A StatefulSet does not automatically make a database highly available, provide safe backups, or configure replication. Those behaviours depend on the database and its operational design.

For a learning environment, a database could use a PVC for its data directory. In a production system, I would evaluate whether a managed database service is more appropriate.

### Important database considerations

- Persistent storage must be provisioned and mounted correctly.
- Database credentials should not be committed to Git.
- The database should have appropriate readiness and health checks.
- Application connections should use the correct Service name and port.
- Backups should be scheduled and restore procedures tested.
- Database schema migrations should be managed deliberately.
- Storage capacity and performance should be monitored.
- A storage snapshot alone should not be assumed to be a consistent database backup.

The application should also handle database startup and temporary connectivity failures without entering an uncontrolled restart loop.

## 13. GitOps with ArgoCD

GitOps is the practice of using a version-controlled repository to declare the desired state of an environment and a controller to reconcile the actual environment with that desired state.

In this project, ArgoCD monitored:

```text
Repository:
https://github.com/Jahmeeu-Cloud/platform-cicd-gitops-config.git

Path:
apps/spring-boot-app/base

Branch:
main
```

The application was configured to deploy into the Kubernetes `default` namespace.

### 13.1 Inspect the ArgoCD application

```bash
kubectl get applications -n argocd
kubectl get application spring-boot-app -n argocd
```

The expected successful state was `Synced` and `Healthy`.

`Synced` indicates that the managed resources match the desired configuration according to ArgoCD's comparison.

`Healthy` indicates that ArgoCD considers the resources healthy under its health assessment rules.


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/815u0ej04oc9yrumzb0b.png)



Neither status guarantees that every business function works correctly. An HTTP request and application-specific tests provide additional evidence.

### 13.2 Example ArgoCD Application manifest

The following is an illustrative manifest for the repository and path used in this project:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: spring-boot-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Jahmeeu-Cloud/platform-cicd-gitops-config.git
    targetRevision: main
    path: apps/spring-boot-app/base
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

This is a reference configuration. Use the existing application manifest rather than replacing it blindly.

Automatic pruning can delete resources that are removed from the desired Git configuration. Review its implications before enabling it for important workloads.

### 13.3 Validate the manifests before committing

From the GitOps repository:

```bash
kubectl kustomize apps/spring-boot-app/base
```

This renders the Kustomize configuration. It is a useful early check that the directory can be processed.

After a change, inspect the rendered output and review the diff:

```bash
git status
git diff
```

Then commit and push the intended files:

```bash
git add apps/spring-boot-app/base
git commit -m "Update application deployment configuration"
git push origin main
```

Run these commands from the GitOps repository and confirm that the correct remote and branch are configured before pushing.

### 13.4 Verify reconciliation

```bash
kubectl get application spring-boot-app -n argocd
kubectl get deployment spring-boot-app
kubectl get pods
```

The aim is to confirm that the Git-defined desired state has been applied and the expected Pods are running.


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/m6avibmb3lt5l8crm28t.png)


**[SCREENSHOT - ARGOCD APPLICATION STATUS HERE]**

*Suggested caption: ArgoCD was tracking the GitOps repository and reporting the synchronisation and health of the application.*

## 14. Testing GitOps Self-Healing

The most useful test in this project was deliberately changing the live Deployment and observing how ArgoCD responded.

The GitOps configuration declared three replicas.

I manually changed the live Deployment to one replica:

```bash
kubectl scale deployment spring-boot-app --replicas=1
```

Then I watched the Deployment:

```bash
kubectl get deployment spring-boot-app --watch
```

Because the ArgoCD application was configured for automated synchronisation and self-healing, ArgoCD detected the difference between the live configuration and the desired state in Git.

The replica count returned to three.

### What did this prove?

The test demonstrated that the GitOps controller could correct this particular kind of configuration drift.

The process was:

1. Git declared three replicas.
2. The live Deployment was manually scaled down to one.
3. The live state no longer matched the desired state.
4. ArgoCD detected the difference.
5. Automated reconciliation restored the declared replica count.
6. Kubernetes worked towards the desired three-replica state.

This is a practical example of declarative operations.

It also shows why manual changes to GitOps-managed resources may be temporary. If the desired state remains unchanged in Git, the controller may restore it.

### Does self-healing fix every problem?

No.

ArgoCD can reconcile managed resources with the desired configuration, but it cannot automatically solve every application bug, exhausted storage volume, invalid image, external database outage, or cluster networking problem.

Self-healing is a reconciliation capability, not a replacement for monitoring, testing, and operational troubleshooting.


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/iiv980uc60j4rfc8udrf.png)



**[SCREENSHOT - THE DEPLOYMENT AT ONE REPLICA HERE AND THE DEPLOYMENT RESTORED TO THREE REPLICAS HERE]**

*Suggested caption: I deliberately changed the live replica count and observed ArgoCD restore the desired state defined in Git.*

## 15. Verifying the Application End to End

A successful Jenkins build is not the same as a successful application deployment.

I validated the Kubernetes resources and tested the application endpoint independently.

### 15.1 Check the Deployment

```bash
kubectl get deployment spring-boot-app
```

Inspect the replica counts and confirm that the expected Pods become ready.

### 15.2 Check the Pods

```bash
kubectl get pods -o wide
```

If a Pod is not ready, investigate it:

```bash
kubectl describe pod POD_NAME
kubectl logs POD_NAME
```

Replace `POD_NAME` with the actual Pod name.

### 15.3 Check the Service

```bash
kubectl get service spring-boot-app-service
kubectl get endpoints spring-boot-app-service
```

The endpoints help show whether the Service has selected and is routing to suitable Pods.

### 15.4 Find the current Minikube access address

For a NodePort Service:

```bash
minikube ip
kubectl get service spring-boot-app-service
```

Use the returned IP and NodePort to construct the URL, provided the Minikube networking setup supports direct access that way.

Alternatively:

```bash
minikube service spring-boot-app-service --url
```

The command may open a tunnel or return a URL depending on the driver and operating environment. Follow any instructions printed by Minikube.

### 15.5 Send an HTTP request

For example:

```bash
curl -i http://MINIKUBE_IP:NODE_PORT
```

Replace the placeholders with the actual address and port.

In the test performed for this project, the application endpoint returned HTTP 200.

That result confirmed that the tested route responded successfully. It did not, by itself, prove that every application function was correct.

### What did the end-to-end test validate?

- The Deployment existed.
- Kubernetes created application Pods.
- The Service selected the intended workload.
- The endpoint was reachable through the tested access path.
- The application returned a successful HTTP response.

Together, these checks provided stronger evidence than a successful build result alone.


## 16. Probes, Resource Requests, and Limits

For a more robust deployment, I would extend the Deployment with resource settings and appropriate health probes.

An illustrative container section is:

```yaml
containers:
  - name: spring-boot-app
    image: horlacloud/spring-boot-demo:20
    ports:
      - name: http
        containerPort: 8080
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 20
      periodSeconds: 20
```

These values are examples, not measurements from the capstone.

The application must actually expose the configured health endpoint. A Spring Boot application may require an appropriate health endpoint configuration, and the route may differ from `/health`.

Resource requests influence scheduling. Resource limits constrain how much CPU or memory the container may consume under the configured runtime rules.

A readiness probe helps determine when a container should receive Service traffic. A liveness probe helps Kubernetes decide whether to restart a container that appears unable to recover.

Probe configuration must reflect the application's real startup and recovery behaviour. Aggressive probes can make a slow-starting application less reliable rather than more reliable.


## 17. Availability, Scaling, and Reliability

These concepts are related, but they are not interchangeable.


### Availability

Three replicas provide multiple application instances, but three replicas on a single-node local cluster do not provide node-level high availability.

If the only node fails, all Pods on that node may become unavailable.

A production design would consider multiple nodes, appropriate failure-domain placement, load balancing, cluster capacity, persistent storage, and recovery procedures.


### Scalability

A Deployment's replica count can be changed manually:

```bash
kubectl scale deployment spring-boot-app --replicas=3
```

A Horizontal Pod Autoscaler can be introduced to adjust the number of replicas based on metrics.

For example:

```bash
kubectl autoscale deployment spring-boot-app \
  --cpu-percent=70 \
  --min=2 \
  --max=10
```

This is an optional extension. It requires a working metrics source and suitable resource requests. It should not be described as implemented unless the HPA was actually created and verified.

Scaling Pods also does not guarantee greater throughput if the application depends on a saturated database or another bottleneck.


### Reliability

Reliability depends on more than the number of replicas. It also depends on application health checks, dependency availability, resource planning, configuration management, observability, and tested recovery procedures.

GitOps adds a useful form of configuration consistency by making the desired state explicit and version-controlled.


## 18. Troubleshooting Guide

A repeatable troubleshooting approach starts by identifying which layer is failing.


### Jenkins pipeline fails

Inspect the failed stage and the console output.

```text
Question: Did checkout, build, test, image build, or push fail?
```

If checkout fails, check the repository URL, branch, network connectivity, and credential requirements.

If Maven fails, inspect the compilation or test output before changing Docker configuration.

If Docker fails, inspect the Dockerfile, build context, daemon access, and image tag.

If the GitOps update fails, inspect the GitHub credential ID, credential type, token permissions, branch rules, and remote URL.


### ImagePullBackOff

Check the Pod events:

```bash
kubectl describe pod POD_NAME
```

Common causes include:

- Incorrect image name or tag.
- Image not present in the registry.
- Private registry authentication not configured.
- Registry connectivity issues.
- Architecture incompatibility.

Do not assume that an image was successfully published merely because the local Docker build passed.


### CrashLoopBackOff

Inspect the container logs and events:

```bash
kubectl logs POD_NAME
kubectl describe pod POD_NAME
```

Possible causes include a bad startup command, application configuration problems, missing environment variables, dependency failures, or memory constraints.

### Service has no endpoints

Inspect the Service selector and Pod labels:

```bash
kubectl describe service spring-boot-app-service
kubectl get pods --show-labels
kubectl get endpoints spring-boot-app-service
```

The Service selector must match the intended Pods. Pods that are not ready may not be included as normal Service endpoints.

### PVC remains Pending

Check the claim and StorageClass:

```bash
kubectl describe pvc app-data
kubectl get storageclass
kubectl get pv
```

Possible causes include an unavailable provisioner, no matching volume, an unsupported access mode, or insufficient capacity.


### ArgoCD reports OutOfSync

Inspect the application status and compare the live state with Git.

Check whether:

- The correct repository is configured.
- The expected branch and path are selected.
- The commit was pushed successfully.
- The manifests render correctly.
- Automatic synchronisation is configured.
- The controller can access the repository.

If manual changes keep being reverted, check whether self-healing is enabled and whether the resource is managed by the ArgoCD application.


### HTTP request fails

Work from the inside out:

1. Check Pod readiness.
2. Inspect application logs.
3. Check the Service selector.
4. Check Service endpoints.
5. Confirm the application listens on the expected container port.
6. Confirm the correct NodePort, tunnel, or ingress route.
7. Test the correct application path.

This process avoids changing several layers at once without knowing which one is responsible.


## 19. Security Practices

The project interacts with several systems that require controlled access.


### Jenkins

- Restrict administrative access.
- Maintain Jenkins and its plugins.
- Store secrets in Jenkins Credentials.
- Review Jenkinsfile changes.
- Avoid exposing secrets in console output.
- Restrict the privileges available to build agents.


### GitHub

- Use narrowly scoped credentials.
- Protect important branches.
- Require reviews where appropriate.
- Never commit tokens or passwords.
- Review repository permissions before allowing automated writes.


### Docker

- Prefer maintained base images.
- Scan images for vulnerabilities where tooling is available.
- Use non-root containers when practical.
- Avoid embedding secrets in images.
- Use traceable image versions.


### Kubernetes

- Apply appropriate resource requests and limits.
- Use security contexts.
- Restrict network access where practical.
- Use readiness and liveness probes thoughtfully.
- Protect Secrets through access control and appropriate encryption configuration.
- Limit service-account permissions.


### GitOps

- Review changes before promoting them to important environments.
- Understand the effect of automated pruning.
- Protect the branch used by ArgoCD.
- Keep sensitive values out of ordinary deployment manifests.
- Make rollback procedures explicit and testable.

These controls are part of the delivery architecture, not optional additions that can be ignored until a security incident occurs.


## 20. Reusability and Future Platform Improvements

One of the main benefits of declarative configuration is the ability to reuse it.

Kustomize allows a project to maintain a common base and apply environment-specific overlays.

A possible future layout is:

```text
apps/
└── spring-boot-app/
    ├── base/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── kustomization.yaml
    └── overlays/
        ├── staging/
        │   └── kustomization.yaml
        └── production/
            └── kustomization.yaml
```

The base defines shared resources. Overlays can customise image tags, replica counts, and other settings for different environments.

This is a suggested extension rather than a description of already completed overlays.

Before committing configuration changes, render the manifests:

```bash
kubectl kustomize apps/spring-boot-app/base
```

Then review the result and the Git diff:

```bash
git diff
git status
```

A reusable deployment pattern should reduce repetitive work while keeping the resulting configuration understandable to the engineers responsible for operating it.


## 21. Version Control, Backups, and Cleanup

Version control records configuration changes, but it is not a complete backup strategy for every kind of project data.

The source and GitOps repositories provide a record of the files committed at specific revisions.

The GitOps repository's last confirmed commit during this work was:

```text
dc9e323 Fix deployment YAML and set replicas to three
```

That commit recorded a change to the deployment configuration.

For future maintenance, I would ensure that:

- The intended files are committed.
- Generated build output is excluded when appropriate.
- The remote URL points to the correct repository.
- The default branch contains the intended configuration.
- Deployment manifests render successfully.
- The deployed image tag corresponds to a published image.
- Credentials remain outside Git.
- Persistent data is backed up separately when required.

Before cleaning up a cloud environment, identify which resources belong to the project and whether any contain data that must be retained.

Before deleting a PVC, inspect its reclaim policy and confirm that important data is backed up.

Before deleting an AWS resource, verify that it is not shared with another workload or still required by a running service.

Cleanup should be deliberate, documented, and reversible where practical.


## 22. What I Would Improve Before Calling This Production-Ready

The project was a hands-on learning environment, not a claim of a production-grade platform.

The next improvements I would prioritise are:

1. **Immutable image references:** Use unique release tags or image digests.
2. **Automated testing:** Ensure the pipeline stops when tests fail.
3. **Pull-request validation:** Run build and manifest checks before changes reach the deployment branch.
4. **Controlled promotion:** Promote a tested image through staging and production with appropriate approvals.
5. **Supply-chain security:** Scan dependencies and container images.
6. **Observability:** Add application metrics, dashboards, logs, and actionable alerts.
7. **Production networking:** Use a suitable ingress or gateway and secure TLS configuration.
8. **High availability:** Design for node failures and relevant failure domains.
9. **Data protection:** Implement and test database backup and restore procedures where persistent data is involved.
10. **Recovery testing:** Practise rolling back to a known-good configuration and image.
11. **Least privilege:** Reduce the permissions available to Jenkins, ArgoCD, and application workloads.
12. **Environment isolation:** Separate staging and production access and configuration.

A platform becomes more mature when it can not only deploy software but also explain its state, recover from expected failures, and make unsafe changes more difficult.


## 23. Final Results and Validation Checklist

The following table separates the demonstrated project outcomes from optional extensions.

| Area | Result or validation |
|---|---|
| Jenkins | A successful pipeline build was achieved by the 20th CI/CD pipeline build |
| Credential troubleshooting | Identified the `github-credentials` lookup error as a credential reference issue |
| Source control | Application source stored in its GitHub repository |
| GitOps configuration | Deployment configuration stored in a separate repository |
| Container image | Docker image publication was part of the workflow |
| Kubernetes | Spring Boot application managed through a Deployment and Service |
| Replicas | GitOps configuration declared three replicas |
| ArgoCD | Application managed through the GitOps repository |
| Self-healing | A manual scale-down was corrected back to the Git-defined replica count |
| Application testing | The tested endpoint returned HTTP 200 |
| Persistent storage | Documented as a reusable extension; verify separately if it was implemented |
| Frontend and backend | Documented as a reference extension, not part of the confirmed Spring Boot-only deployment |
| Production monitoring and HA | Recommended future improvements, not assumed to be implemented |


### Final validation commands

Run these commands against the relevant environment:

```bash
# Kubernetes cluster
kubectl get nodes -o wide

# Deployment and Pods
kubectl get deployment spring-boot-app
kubectl get pods -o wide

# Service and endpoints
kubectl get service spring-boot-app-service
kubectl get endpoints spring-boot-app-service

# ArgoCD application
kubectl get application spring-boot-app -n argocd

# Render the GitOps manifests
kubectl kustomize apps/spring-boot-app/base

# Review Git changes from the GitOps repository
git status
git log -5 --oneline
```

The last two Git commands should be run from the GitOps repository. The Kubernetes commands should be run against the cluster and namespace where the application is deployed.


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/6f6mjapvyp68sd0j3361.png)

---

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/phd0c44475ofv2wn6ng6.png)

---

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/hkgdijpei9vbmbfltv5q.png)


---

![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/3v5rx2d1ku4dltx3exh1.png)




**[SCREENSHOT - FINAL KUBERNETES STATUS HERE]**


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/v3devdjtnmpe4mojzspu.png)



**[SCREENSHOT - FINAL ARGOCD SYNCHRONISATION STATUS HERE]**


![Image description](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/hgcrychsghok72yas7sf.png)


**[SCREENSHOT - THE FINAL SUCCESSFUL APPLICATION TEST HERE]**


## 24. What I Learned From This Capstone

This project strengthened my understanding of how application delivery works across several connected systems.

I learned that:

- A pipeline is a sequence of dependent operations, and each stage must be debugged at the correct layer.
- Credential IDs, credential types, authentication, and permissions are separate concerns.
- Docker image publication does not prove that an application has deployed successfully.
- Kubernetes Deployments manage desired replica counts, while Services provide stable access to selected Pods.
- Persistent storage requires deliberate planning around claims, volumes, lifecycle, and recovery.
- GitOps makes the desired state visible and version-controlled.
- ArgoCD self-healing can correct configuration drift when the desired state and synchronisation policy support it.
- Application health must be verified independently of build and deployment status.
- Three replicas on a local Minikube cluster do not constitute production high availability.
- Security, observability, backups, and rollback procedures are part of operating a platform.

The most memorable part was the troubleshooting. The Jenkins credential error was frustrating, but it taught me to stop guessing and investigate the exact failure.

When an earlier pipeline stage succeeds and a later stage fails, the answer is not necessarily to rebuild everything. The better approach is to identify the failing operation, inspect its dependencies, and verify the relevant configuration.

## Conclusion

This capstone gave me practical experience connecting Jenkins, GitHub, Maven, Docker, Kubernetes, and ArgoCD into a repeatable delivery workflow.

It also helped me understand why CI/CD and GitOps are complementary. CI/CD automates the process of validating and packaging application changes. GitOps provides a declarative way to manage the desired deployment state and reconcile the running environment with it.

The project is a foundation I can continue improving with stronger automated testing, a more complete frontend/backend architecture, persistent data services where needed, better monitoring, and more production-oriented security and availability controls.

My biggest takeaway is that the tools are only part of the work. Understanding the connections between them, investigating failures systematically, and being able to explain the result are just as important.

For anyone learning DevOps or platform engineering, I would encourage you to document not only what worked but also the errors you encountered, how you investigated them, and what the final tests proved.

That is what turns a practical exercise into knowledge you can reuse in your next project.

---

## Project References

**GitHub profile:**  
https://github.com/Jahmeeu-Cloud

**Application source repository:**  
https://github.com/Jahmeeu-Cloud/platform-cicd-spring-boot-demo-source

**GitOps configuration repository:**  
https://github.com/Jahmeeu-Cloud/platform-cicd-gitops-config

**Author:** Jamiu Olatunji Bakre 
*Cloud, DevOps and Platform Engineer*

*This article documents a hands-on learning project and includes additional reference configurations for future extensions. Optional components are identified separately from the functionality verified in the original environment.*
