# Lab 23: Implementing a GitOps Workflow with Jenkins and Argo CD

## Overview

This lab extends Lab 22 by moving the deployment stage from an imperative `kubectl apply` to a GitOps workflow. Jenkins remains responsible for Continuous Integration: it builds the application, scans and publishes the image, and records the new image tag in the Kubernetes manifest stored in Git. Argo CD is responsible for Continuous Delivery: it watches the repository and automatically synchronizes the cluster with the state declared in Git.

Git becomes the single source of truth for the cluster, and Jenkins no longer needs direct access to the Kubernetes API.

## Objectives

- Install Argo CD in the Kubernetes cluster
- Access the Argo CD UI and log in
- Connect Argo CD to the GitHub repository
- Create an Argo CD Application named `ivolve`
- Point the Application at the manifest directory in the repository
- Enable automatic synchronization
- Modify the `deployOnK8s` shared library step to push the updated manifest to GitHub
- Trigger the Jenkins pipeline to build and publish a new image
- Confirm that Argo CD detects the change and synchronizes the application

## Tools and Technologies

| Tool | Role |
|---|---|
| Kubernetes | Container orchestration platform |
| Argo CD | GitOps continuous delivery controller |
| Jenkins | Continuous integration server |
| Jenkins Shared Libraries | Reusable pipeline steps (`buildApp`, `buildImage`, `deployOnK8s`) |
| Docker | Image build and registry login |
| Docker Hub | Container image registry |
| Trivy | Container image vulnerability scanner |
| GitHub | Git repository and source of truth for manifests |
| kubectl | Cluster administration and verification |

## Architecture

```
Developer / Jenkins
        |
        |  1. build, scan, push image
        v
  Docker Hub  (rewan2233/ivolve-app:lab23-N)
        |
        |  2. update image tag in deployment.yaml
        |     and git push
        v
  GitHub  (Rewan0/iVolve-tasks, path ArgoCD/Lab23)
        |
        |  3. Argo CD polls the repository
        v
     Argo CD  (namespace: argocd)
        |
        |  4. automatic sync
        v
  Kubernetes  (namespace: ivolve, Deployment ivolveapp)
```

## Environment

| Component | Details |
|---|---|
| Kubernetes cluster | master-1 and worker-1 (Calico CNI) |
| Argo CD | Installed in the `argocd` namespace, exposed through NodePort |
| Jenkins | Docker container on Docker Desktop, agent `linux-agent` (label `agent-1`) |
| Shared library | `jenkins-shared-lib` (registered in Jenkins as `shared-lib`) |
| Application repository | `Rewan0/iVolve-tasks` |
| Container image | `rewan2233/ivolve-app` |
| Target namespace | `ivolve` |

## Repository Structure

```
iVolve-tasks/
|-- Jenkins/
|   `-- Lab22/                 application source and Dockerfile
`-- ArgoCD/
    `-- Lab23/
        |-- Jenkinsfile        pipeline definition
        |-- deployment.yaml    manifest watched by Argo CD
        `-- README.md
```

The `ArgoCD/Lab23` directory contains the manifest that Argo CD monitors. Argo CD only processes Kubernetes manifest files in this path, so the Jenkinsfile and README are ignored.

## Implementation

### Step 1: Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
```

The `--server-side` flag is used because the Argo CD custom resource definitions are too large for a client-side apply. All pods in the `argocd` namespace must reach the `Running` state before continuing.

<img width="1294" height="300" alt="image" src="https://github.com/user-attachments/assets/d015f92f-2e67-4107-9d03-333663b3f60d" />

### Step 2: Access the Argo CD UI

The `argocd-server` service is switched to NodePort so it is reachable from outside the cluster.

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"NodePort"}}'
kubectl get svc argocd-server -n argocd
```

The initial administrator password is read from the generated secret:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

The UI is then opened at `https://<master-1-IP>:<NodePort>` (HTTPS port mapped to 443) and the login is performed with the user `admin`.

<img width="1278" height="686" alt="Screenshot 2026-11-08 145511" src="https://github.com/user-attachments/assets/2d4bf4fe-bb76-4aca-a71f-67f5f86225f1" />

### Step 3: Connect Argo CD to the GitHub Repository

In the UI, under **Settings > Repositories > Connect Repo**:

| Field | Value |
|---|---|
| Connection method | HTTPS |
| Type | git |
| Name | ivolve |
| Project | default |
| Repository URL | `https://github.com/Rewan0/iVolve-tasks.git` |

The Username and Password fields are left empty for a public repository. For a private repository, the GitHub username and a personal access token are used. The connection status must show **Successful**.

<img width="1273" height="688" alt="Screenshot 2026-11-08 152540" src="https://github.com/user-attachments/assets/29f3e636-6526-47ad-96c9-cfd5249f5f88" />

### Step 4: Create the Argo CD Application

The Application ivolve is created from the Argo CD UI under Applications > New App with the following settings:

| Setting | Value | Purpose |
|---|---|---|
| `path` | `ArgoCD/Lab23` | Directory containing the manifest to monitor |
| `targetRevision` | `main` | Branch tracked by Argo CD |
| `destination.namespace` | `ivolve` | Namespace where resources are applied |
| `automated` | enabled | Applies Git changes without manual intervention |
| `prune` | true | Removes cluster resources that were deleted from Git |
| `selfHeal` | true | Reverts manual changes made directly in the cluster |

The existing `ivolveapp` Deployment in the `ivolve` namespace is adopted by Argo CD. Once synchronized, the application reports `Synced` and `Healthy`.

<img width="1278" height="686" alt="Screenshot 2026-11-08 153505" src="https://github.com/user-attachments/assets/9e4e5847-d078-483d-aad6-6c2d897494f7" />

### Step 5: Modify the Shared Library Step

The `deployOnK8s` step in `jenkins-shared-lib` previously ran `kubectl apply` against the cluster. It now updates the image tag in the manifest and pushes the change to GitHub.

`vars/deployOnK8s.groovy`:

```groovy
def call(String manifestPath, String image, String tag, String credsId, String branch = 'main') {
    withCredentials([usernamePassword(credentialsId: credsId,
                     usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
        sh """
            sed -i 's|image: ${image}:.*|image: ${image}:${tag}|' ${manifestPath}
            git config user.email "jenkins@ivolve.local"
            git config user.name  "jenkins"
            git add ${manifestPath}
            git commit -m "Update image to ${image}:${tag} [skip ci]" || echo "No changes to commit"
            git push https://\${GIT_USER}:\${GIT_PASS}@github.com/Rewan0/iVolve-tasks.git HEAD:${branch}
        """
    }
}
```

| Library | Responsibility |
|---|---|
| `buildApp` | Install dependencies and run tests |
| `buildImage` | Build the image, scan it with Trivy, push to Docker Hub, remove the local image |
| `deployOnK8s` | Update the image tag in `deployment.yaml` and push it to GitHub so Argo CD synchronizes the cluster |

Key points:

- The `sed` expression targets only the application image, so the `mysql:8.0` image used by the init container is left unchanged.
- The commit message contains `[skip ci]` to prevent the push from triggering another build.
- The GitHub token is read from a Jenkins credential and escaped in the shell script, so it is never interpolated into the Groovy string.

### Step 6: Configure Jenkins

A credential of type **Username with password** is created in Jenkins:

| Field | Value |
|---|---|
| Username | GitHub username |
| Password | GitHub personal access token (`repo` scope) |
| ID | `github-creds` |

A pipeline job named `Lab23` is created with the following settings:

| Setting | Value |
|---|---|
| Definition | Pipeline script from SCM |
| SCM | Git |
| Repository URL | `https://github.com/Rewan0/iVolve-tasks.git` |
| Branch | `*/main` |
| Script Path | `ArgoCD/Lab23/Jenkinsfile` |

### Step 7: Jenkinsfile

```groovy
@Library('shared-lib') _

pipeline {
    agent { label 'agent-1' }

    environment {
        IMAGE    = "rewan2233/ivolve-app"
        TAG      = "lab23-${BUILD_NUMBER}"
        APP_DIR  = "Jenkins/Lab22"
        MANIFEST = "ArgoCD/Lab23/deployment.yaml"
    }

    stages {
        stage('Build App') {
            steps { buildApp(APP_DIR) }
        }
        stage('Build Image') {
            steps { buildImage(APP_DIR, IMAGE, TAG, 'dockerhub') }
        }
        stage('Update Manifest & Push to GitHub') {
            steps { deployOnK8s(MANIFEST, IMAGE, TAG, 'github-creds') }
        }
    }

    post {
        always  { echo 'Pipeline finished' }
        success { echo 'Manifest pushed, Argo CD will sync' }
        failure { echo 'Pipeline failed' }
    }
}
```

The image tag is prefixed with `lab23-` so it cannot collide with tags produced by earlier labs.

### Step 8: Trigger the Pipeline

The `Lab23` job is started with **Build Now**. The pipeline performs the following stages:

1. Install dependencies and run tests
2. Build the Docker image, scan it with Trivy, push it to Docker Hub, and delete the local copy
3. Update the image tag in `ArgoCD/Lab23/deployment.yaml` and push the commit to GitHub

<img width="2551" height="1368" alt="image" src="https://github.com/user-attachments/assets/f9393b3d-9d6e-4ba2-aac6-e8090db28bdd" />
<img width="2550" height="1368" alt="image" src="https://github.com/user-attachments/assets/12a7aa68-6eaa-4124-b81c-793644ed9faf" />
<img width="2550" height="1369" alt="image" src="https://github.com/user-attachments/assets/208a9798-5d84-4f67-8054-0bef363911c2" />
<img width="2550" height="1362" alt="image" src="https://github.com/user-attachments/assets/ee3038ae-79ff-4119-9697-4e863ebf6833" />


### Step 9: Verify the Synchronization

Argo CD detects the new commit during its next repository poll (about every three minutes) and applies it automatically. A refresh can be forced from the UI.

```bash
kubectl get app ivolve -n argocd
kubectl get deploy ivolveapp -n ivolve \
  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
kubectl get pods -n ivolve
```

Result:

```
NAME     SYNC STATUS   HEALTH STATUS
ivolve   Synced        Healthy

rewan2233/ivolve-app:lab23-4
```

The Deployment image changed from `rewan2233/ivolve-app:v1` to `rewan2233/ivolve-app:lab23-4` without any `kubectl` command being run by Jenkins, which confirms that Argo CD detected the Git change and synchronized the cluster.

<img width="1696" height="207" alt="image" src="https://github.com/user-attachments/assets/5a897288-e43f-4da9-8b34-97f44813a337" />

## Conclusion

This lab replaced the push-based deployment of Lab 22 with a GitOps workflow. Jenkins now stops at publishing the image and committing the new tag to the manifest, while Argo CD pulls the desired state from Git and reconciles the cluster automatically. This gives a full audit trail of every deployment through Git history, removes the need for cluster credentials in Jenkins, and ensures the cluster state is continuously corrected back to what is declared in the repository through the prune and self-heal options.
