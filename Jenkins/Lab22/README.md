# Lab 22: CI/CD Pipeline with Jenkins Agents and Shared Libraries

A CI/CD pipeline for a Node.js application in which the pipeline logic lives in reusable Jenkins Shared Libraries and the build runs on a dedicated Jenkins agent connected over SSH.

---

## Objectives

| # | Task |
|---|------|
| 1 | Use the application repository (source code, Dockerfile, Kubernetes deployment file) |
| 2 | Create three shared libraries: `buildApp`, `buildImage`, `deployOnK8s` |
| 3 | Update the pipeline to use the shared libraries |
| 4 | Configure a Jenkins agent (slave) to run the pipeline |

---

## Architecture

```
  GitHub: iVolve-tasks                 GitHub: jenkins-shared-lib
  (app + Dockerfile + Jenkinsfile)     (vars/*.groovy)
            |                                    |
            |  Pipeline from SCM                 |  @Library('shared-lib')
            v                                    v
  +--------------------------------------------------------+
  |        Jenkins Controller (Docker Desktop container)   |
  +----------------------------+---------------------------+
                               |
                               |  SSH (port 22)
                               v
  +--------------------------------------------------------+
  |   Jenkins Agent: linux-agent   (label: agent-1)        |
  |   master-1 VM: Java 21, Node.js 20, Docker, Trivy,     |
  |   kubectl                                              |
  +------------+----------------------------+--------------+
               |                            |
               | docker push                | kubectl apply
               v                            v
         +-----------+              +-------------------+
         | Docker Hub|              | Kubernetes cluster|
         +-----------+              | namespace: ivolve |
                                    +-------------------+
```

### Pipeline Flow

```
Checkout --> Build App --> Build Image --> Scan --> Push --> Delete Local Image --> Update Deployment --> Deploy
             (buildApp)    |---------------- buildImage ----------------|          |------ deployOnK8s ------|
```

---

## Repositories Structure

### Application repository: `iVolve-tasks`

```
Jenkins/Lab22/
├── Jenkinsfile
├── Dockerfile
├── deployment.yaml
└── (application source code)
```

### Shared library repository: `jenkins-shared-lib`

```
vars/
├── buildApp.groovy
├── buildImage.groovy
└── deployOnK8s.groovy
```

The shared library must be in its own repository with `vars/` at the root. Jenkins looks for `vars/` at the top level of the library repository.

---

## Tools and Technologies

| Tool | Purpose |
|------|---------|
| Jenkins | CI/CD server (controller runs in Docker Desktop) |
| Jenkins Shared Library | Reusable pipeline steps |
| Jenkins SSH Agent | Runs the pipeline on a separate machine |
| Node.js / npm | Application build and tests |
| Docker | Build, push, and remove the image |
| Trivy | Image vulnerability scanning |
| Docker Hub | Image registry |
| Kubernetes / kubectl | Deployment target |
| Git / GitHub | Source control for the application and the library |

---

## Part 1: Jenkins Agent Setup

### 1.1 Prerequisites on the Agent VM

| Requirement | Check |
|-------------|-------|
| Java 21 (required by the current Jenkins remoting) | `java -version` |
| Node.js and npm | `node -v && npm -v` |
| Docker (user in the `docker` group) | `docker ps` |
| Trivy | `trivy --version` |
| kubectl | `kubectl get nodes` |
| SSH server | `systemctl status ssh` |

```bash
# Add the user to the docker group
sudo usermod -aG docker $USER

# Create the agent root directory
sudo mkdir /home/jenkins_home
sudo chmod 777 /home/jenkins_home
```

### 1.2 Generate an SSH Key (inside the Jenkins container)

```bash
docker exec -it jenkins bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/jenkins_agent_key
ssh-copy-id -i ~/.ssh/jenkins_agent_key.pub <USER>@<AGENT_IP>
ssh -i ~/.ssh/jenkins_agent_key <USER>@<AGENT_IP>     # test: no password prompt
cat ~/.ssh/jenkins_agent_key                          # copy the private key
```

### 1.3 Add the SSH Credential in Jenkins

Manage Jenkins > Credentials > Global > Add Credentials

| Field | Value |
|-------|-------|
| Kind | SSH Username with private key |
| ID | `agent-ssh` |
| Username | The agent VM user |
| Private Key | Enter directly, paste the private key |

<img width="1276" height="682" alt="Screenshot 2026-11-05 172832" src="https://github.com/user-attachments/assets/f9bbbfb5-5ecb-4273-85ab-cc07d0bf461d" />

### 1.4 Create the Agent Node

Manage Jenkins > Nodes > New Node

| Setting | Value |
|---------|-------|
| Name | `linux-agent` |
| Type | Permanent Agent |
| Number of executors | `1` |
| Remote root directory | `/home/jenkins_home` |
| Labels | `agent-1` |
| Usage | Use this node as much as possible |
| Launch method | Launch agents via SSH |
| Host | `<AGENT_IP>` |
| Credentials | `agent-ssh` |
| Host Key Verification | Non verifying Verification Strategy |

After saving, the node log shows `Agent successfully connected and online`.

**Screenshot: agent node online**

<img width="1265" height="686" alt="Screenshot 2026-11-05 173148" src="https://github.com/user-attachments/assets/925d56de-9697-4c59-9d99-7d09b1210008" />
<img width="1274" height="683" alt="Screenshot 2026-11-05 174311" src="https://github.com/user-attachments/assets/4da3714b-751d-4d93-9222-b0f49f5b0dfb" />


---

## Part 2: Shared Libraries

Each file in `vars/` defines one function named `call`. The file name is the function name used in the Jenkinsfile.

### 2.1 `vars/buildApp.groovy` (unit test and build)

```groovy
def call(String appDir) {
    dir(appDir) {
        sh 'npm install'
        sh 'npm test --if-present'
    }
}
```

### 2.2 `vars/buildImage.groovy` (build, scan, push, delete locally)

```groovy
def call(String appDir, String image, String tag, String dockerCredId) {
    dir(appDir) {
        sh "docker build -t ${image}:${tag} ."
    }
    sh "trivy image --severity HIGH,CRITICAL ${image}:${tag}"
    withCredentials([usernamePassword(credentialsId: dockerCredId,
        usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
        sh 'echo $DH_PASS | docker login -u $DH_USER --password-stdin'
        sh "docker push ${image}:${tag}"
    }
    sh "docker rmi ${image}:${tag}"
}
```

### 2.3 `vars/deployOnK8s.groovy` (edit deployment file, deploy to cluster)

```groovy
def call(String appDir, String image, String tag, String kubeCredId) {
    dir(appDir) {
        sh "sed -i 's|image: ${image}:.*|image: ${image}:${tag}|' deployment.yaml"
        withCredentials([file(credentialsId: kubeCredId, variable: 'KUBECONFIG')]) {
            sh 'kubectl apply -f deployment.yaml'
            sh 'kubectl rollout status deployment/ivolveapp -n ivolve --timeout=180s'
        }
    }
}
```

| Library | Responsibility |
|---------|----------------|
| `buildApp` | Install dependencies and run tests |
| `buildImage` | Build the image, scan it with Trivy, push to Docker Hub, remove the local image |
| `deployOnK8s` | Update the image tag in `deployment.yaml`, apply it, and wait for the rollout |

**Screenshot: shared library repository**

<img width="2532" height="1285" alt="image" src="https://github.com/user-attachments/assets/f24bbd32-85b1-47a1-9877-8c466fe88ffe" />

---

## Part 3: Jenkins Configuration

### 3.1 Register the Shared Library

Manage Jenkins > System > Global Trusted Pipeline Libraries > Add

| Field | Value |
|-------|-------|
| Name | `shared-lib` |
| Default version | `main` |
| Retrieval method | Modern SCM |
| Source Code Management | Git |
| Project Repository | URL of the `jenkins-shared-lib` repository |
| Credentials | None (public repository) |

**Screenshot: library configuration**

<img width="1279" height="686" alt="Screenshot (7)" src="https://github.com/user-attachments/assets/8608f46d-f814-4e66-86a9-3c8dbffd480b" />

### 3.2 Credentials Used by the Pipeline

| Credential ID | Type | Used for |
|---------------|------|----------|
| `dockerhub` | Username with password | Docker Hub login and push |
| `k8s-config` | Secret file | kubeconfig for `kubectl` |
| `agent-ssh` | SSH key | Controller to agent connection |

### 3.3 Create the Pipeline Job

New Item > `Lab22` > Pipeline

| Setting | Value |
|---------|-------|
| Definition | Pipeline script from SCM |
| SCM | Git |
| Repository URL | The `iVolve-tasks` repository |
| Branch | `*/main` |
| Script Path | `Jenkins/Lab22/Jenkinsfile` |

---

## Part 4: Jenkinsfile

```groovy
@Library('shared-lib') _

pipeline {
    agent { label 'agent-1' }

    environment {
        IMAGE   = "rewan2233/ivolve-app"
        TAG     = "${BUILD_NUMBER}"
        APP_DIR = "Jenkins/Lab22"
    }

    stages {
        stage('Build App') {
            steps { buildApp(APP_DIR) }
        }
        stage('Build Image') {
            steps { buildImage(APP_DIR, IMAGE, TAG, 'dockerhub') }
        }
        stage('Deploy to Kubernetes') {
            steps { deployOnK8s(APP_DIR, IMAGE, TAG, 'k8s-config') }
        }
    }

    post {
        always  { echo 'Pipeline finished' }
        success { echo 'Deployed successfully' }
        failure { echo 'Pipeline failed' }
    }
}
```

| Line | Meaning |
|------|---------|
| `@Library('shared-lib') _` | Loads the shared library registered in Jenkins |
| `agent { label 'agent-1' }` | Runs the whole pipeline on the SSH agent |
| `buildApp(...)`, `buildImage(...)`, `deployOnK8s(...)` | Call the functions defined in `vars/` |

---

## Run and Verify

1. Open the `Lab22` job and click Build Now.
2. In the Console Output, confirm:
   - `Running on linux-agent`
   - The `shared-lib` library is loaded
   - All stages pass and the build ends with `Finished: SUCCESS`
3. Verify on the cluster:

```bash
kubectl get pods -n ivolve
kubectl get deployment ivolveapp -n ivolve -o jsonpath='{.spec.template.spec.containers[0].image}'
```

The image tag matches the Jenkins build number (build `#14` deployed `rewan2233/ivolve-app:14`).

**Screenshots**

| Screenshot | File |
|------------|------|
| Stage View (all stages passed) | `screenshots/04-stage-view.png` |
| Console output: `Running on linux-agent` | `screenshots/05-console-agent.png` |
| Rollout completed successfully | `screenshots/06-rollout.png` |
| `kubectl get pods -n ivolve` | `screenshots/07-pods.png` |

<img width="2548" height="1360" alt="image" src="https://github.com/user-attachments/assets/cb97993b-e3ad-48fa-9a75-4edf256e266a" />
<img width="2413" height="718" alt="image" src="https://github.com/user-attachments/assets/24487b01-da34-4eef-8602-4c53d1b2fd00" />
<img width="2545" height="1368" alt="image" src="https://github.com/user-attachments/assets/18d3b4dd-0b88-4de0-9194-6287a3f4910d" />
<img width="1722" height="252" alt="image" src="https://github.com/user-attachments/assets/94b6ec5d-d899-4829-8d71-e71db5e0b4bd" />

---

## Notes

- The Trivy scan reports HIGH and CRITICAL findings but does not fail the build. To enforce a quality gate, add `--exit-code 1` to the `trivy` command.
- Docker credentials are masked in the console output by `withCredentials`.
- Updating a function in `jenkins-shared-lib` affects every pipeline that uses it, with no change to the Jenkinsfile.

---

## Conclusion

The pipeline logic is split into three reusable shared libraries (`buildApp`, `buildImage`, `deployOnK8s`) and runs on a dedicated SSH agent instead of the Jenkins controller. This keeps the Jenkinsfile short, makes the steps reusable across projects, and moves build load off the controller, while the full flow from build to Kubernetes deployment runs end to end.
