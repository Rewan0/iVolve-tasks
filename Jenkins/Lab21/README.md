# Lab 21: Jenkins Pipeline for Application Deployment

## Objective

Build a complete CI/CD pipeline in Jenkins that takes the Node.js application from Lab 8 (the same application deployed to Kubernetes in Lab 15) from source code to a running Deployment on the Kubernetes cluster.

The pipeline automates:

1. Build the application
2. Build the Docker image
3. Scan the Docker image
4. Push the image to Docker Hub
5. Delete the image locally
6. Update the image in the deployment file
7. Deploy to the Kubernetes cluster

It also defines post actions for `always`, `success`, and `failure`.

This links Docker, Kubernetes, and Jenkins together, and prepares the ground for GitOps with Argo CD.

## Tools and Technologies

| Tool | Purpose |
|---|---|
| Jenkins (Docker container) | CI/CD server |
| Docker | Build, tag, and push the application image |
| Trivy | Vulnerability scanning of the Docker image |
| Docker Hub | Image registry (`rewan2233/ivolve-app`) |
| Kubernetes (kubeadm, 1 master + 1 worker) | Deployment target |
| kubectl | Applying the deployment and checking the rollout |
| Node.js / npm | Application runtime and dependency installation |
| GitHub | Source repository (`Rewan0/iVolve-tasks`) |

## Architecture

```
  GitHub repo                 Jenkins (Docker container)                Docker Hub
 +-----------+   checkout   +---------------------------------+  push  +-----------+
 | Jenkins/  | -----------> | 1. Build App (npm install)      | -----> | rewan2233/|
 |  Lab21    |              | 2. Build Docker Image           |        | ivolve-app|
 | Jenkinsfile              | 3. Scan Image (Trivy)           |        |   :<build>|
 | Dockerfile|              | 4. Push to DockerHub            |        +-----------+
 | deployment.yaml          | 5. Delete Local Image           |              |
 +-----------+              | 6. Update Image in deployment   |              | pull
                            | 7. Deploy to Kubernetes         |              v
                            +----------------+----------------+     +------------------+
                                             | kubectl apply        | Kubernetes       |
                                             +--------------------> | ns: ivolve       |
                                                                    | Deployment       |
                                                                    | ivolveapp        |
                                                                    +------------------+
```

## Repository Structure

```text
Jenkins/Lab21/
├── Dockerfile
├── Jenkinsfile
├── deployment.yaml
├── server.js
├── db.js
├── package.json
└── frontend/
```

## Prerequisites

### 1. Tools inside the Jenkins container

Jenkins runs in a Docker container with the host Docker socket mounted at `/var/run/docker.sock`, so builds use the host Docker daemon. The default Jenkins image does not include the required tools, so they were installed inside the container:

```bash
docker exec -it -u root jenkins bash

apt-get update && apt-get install -y docker.io nodejs npm curl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
install kubectl /usr/local/bin/kubectl

apt-get install -y wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor -o /usr/share/keyrings/trivy.gpg
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" > /etc/apt/sources.list.d/trivy.list
apt-get update && apt-get install -y trivy

chmod 666 /var/run/docker.sock
```

Verification:

```bash
docker version --format '{{.Server.Version}}'
node -v
kubectl version --client
trivy --version
su jenkins -s /bin/bash -c "docker ps"
```

<img width="2515" height="717" alt="image" src="https://github.com/user-attachments/assets/a2954a6d-6287-43d2-8aba-83737ececadf" />

### 2. Jenkins credentials

`Manage Jenkins` -> `Credentials` -> `Global credentials` -> `Add Credentials`:

| ID | Kind | Content |
|---|---|---|
| `dockerhub` | Username with password | Docker Hub username and access token |
| `k8s-config` | Secret file | kubeconfig of the Kubernetes cluster |

The credentials are used only inside `withCredentials` blocks, so no secret is stored in the repository, and Jenkins masks their values in the console output.

<img width="2523" height="1351" alt="image" src="https://github.com/user-attachments/assets/29d662c8-ccbd-4107-9fc2-d9354e27bee7" />

### 3. Connectivity from Jenkins to the cluster

The kubeconfig was copied into the container and tested before using it in the pipeline:

```bash
docker cp /mnt/e/NTI/iVolve/k8s-config.txt jenkins:/tmp/k8s-config
docker exec jenkins kubectl --kubeconfig=/tmp/k8s-config get nodes
```

Both `master-1` and `worker-1` were listed as `Ready`.

## Implementation

### Step 1: Prepare the repository

The application code, `Dockerfile`, and `deployment.yaml` from the previous labs were placed in `Jenkins/Lab21` in the GitHub repository.

The image line of the application container in `deployment.yaml`:

```yaml
image: rewan2233/ivolve-app:v1
```

### Step 2: Write the Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        IMAGE = "rewan2233/ivolve-app"
        TAG   = "${BUILD_NUMBER}"
    }

    stages {
        stage('Build App') {
            steps { dir('Jenkins/Lab21') { sh 'npm install' } }
        }
        stage('Build Docker Image') {
            steps { dir('Jenkins/Lab21') { sh 'docker build -t $IMAGE:$TAG .' } }
        }
        stage('Scan Docker Image') {
            steps { sh 'trivy image --severity HIGH,CRITICAL $IMAGE:$TAG' }
        }
        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub',
                    usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh 'echo $DH_PASS | docker login -u $DH_USER --password-stdin'
                    sh 'docker push $IMAGE:$TAG'
                }
            }
        }
        stage('Delete Local Image') {
            steps { sh 'docker rmi $IMAGE:$TAG' }
        }
        stage('Update Image in Deployment File') {
            steps { dir('Jenkins/Lab21') { sh 'sed -i "s|image: $IMAGE:.*|image: $IMAGE:$TAG|" deployment.yaml' } }
        }
        stage('Deploy to Kubernetes') {
            steps {
                dir('Jenkins/Lab21') {
                    withCredentials([file(credentialsId: 'k8s-config', variable: 'KUBECONFIG')]) {
                        sh 'kubectl apply -f deployment.yaml'
                        sh 'kubectl rollout status deployment/ivolveapp -n ivolve --timeout=180s'
                    }
                }
            }
        }
    }

    post {
        always  { echo 'Pipeline finished' }
        success { echo 'Deployed successfully' }
        failure { echo 'Pipeline failed' }
    }
}
```

Explanation of each stage:

| Stage | What it does | Why |
|---|---|---|
| Build App | `npm install` inside `Jenkins/Lab21` | Validates that the dependencies resolve; the pipeline stops early if the application cannot be built |
| Build Docker Image | `docker build` with the build number as the tag | Every build produces a unique, traceable image version instead of `latest` |
| Scan Docker Image | `trivy image` for HIGH and CRITICAL findings | The DevSecOps step: vulnerabilities are reported before the image is published |
| Push to DockerHub | `docker login` with the credential, then `docker push` | The cluster nodes pull the image from the registry |
| Delete Local Image | `docker rmi` | Keeps the build host clean; the image already exists on Docker Hub |
| Update Image in Deployment File | `sed` replaces the application image tag | Only the line matching `rewan2233/ivolve-app:` is changed, so the MySQL init container image is not touched |
| Deploy to Kubernetes | `kubectl apply` then `kubectl rollout status` | Applies the new image and waits until the rollout completes, failing the build if it does not |

Post actions:

| Block | Runs when | Purpose |
|---|---|---|
| `always` | Every run | Final message, suitable for cleanup |
| `success` | All stages passed | Success notification |
| `failure` | Any stage failed | Failure notification |

### Step 3: Create the Jenkins pipeline job

1. `Dashboard` -> `New Item` -> name `ivolve-app-pipeline` -> **Pipeline**.
2. Under **Pipeline**:
   - Definition: `Pipeline script from SCM`
   - SCM: `Git`
   - Repository URL: `https://github.com/Rewan0/iVolve-tasks.git`
   - Branch: `*/main`
   - Script Path: `Jenkins/Lab21/Jenkinsfile`
3. Save, then **Build Now**.

### Step 4: Run the pipeline and verify

The final successful run (build #10) completed all stages:

- The image `rewan2233/ivolve-app:10` was built, scanned, and pushed to Docker Hub.
- The local image was deleted.
- The deployment file was updated with the new tag.
- `kubectl apply` configured the Deployment and `kubectl rollout status` reported `successfully rolled out`.
- The post action printed `Deployed successfully`.

<img width="2550" height="1360" alt="image" src="https://github.com/user-attachments/assets/23af603f-e26f-4f0d-949e-797f8f52acc1" />

<img width="2554" height="1357" alt="image" src="https://github.com/user-attachments/assets/fe0ff0e0-fe77-4bb2-a7fb-711d8c37ddb7" />

<img width="2559" height="1369" alt="image" src="https://github.com/user-attachments/assets/2694fe23-5c77-44dc-8e7a-ff6b1b51bf54" />

<img width="1276" height="678" alt="Screenshot 2026-11-04 151350" src="https://github.com/user-attachments/assets/6021829f-c871-411d-83ce-39483cb48251" />

<img width="2559" height="1357" alt="image" src="https://github.com/user-attachments/assets/bf87036b-df56-43a1-9ede-9e21a64d0820" />

<img width="2548" height="1372" alt="image" src="https://github.com/user-attachments/assets/56ecdefa-0919-46e3-b3f2-1a614870b335" />

<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/fd908edd-26f0-4233-979f-a80a5f22e316" />

## Conclusion

A Jenkins pipeline was built that moves the Node.js application from source code to a running Deployment on Kubernetes: it builds the application and the Docker image, scans it with Trivy, pushes it to Docker Hub, removes the local image, updates the deployment manifest, and deploys it, finishing with a rollout check and post actions. Credentials are managed securely inside Jenkins, and each build produces a uniquely tagged image. Troubleshooting the registry credentials, namespace quota, node resources, and CNI issues showed how the Docker, Kubernetes, and Jenkins layers depend on each other.
