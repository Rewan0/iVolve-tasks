# Lab 5: Docker Image Vulnerability Scanning with Trivy

## Objective

Build a Docker image for a Node.js application, scan it for vulnerabilities using Trivy, generate HTML reports, and remediate all HIGH and CRITICAL vulnerabilities that have available fixes by updating the base image and application dependencies.

## Tools and Technologies

- Docker
- Trivy
- Node.js
- Express
- Alpine Linux

## Project Structure

```text
Lab5/
├── app.js
├── package.json
├── Dockerfile
├── html.tpl
├── report-v1.html
├── report-v2.html
└── README.md
```

## Step 1: Clone the Repository

```bash
git clone https://github.com/Ibrahim-Adel15/TrivyImage.git
cd TrivyImage
```

## Step 2: Build the Original Image

Build the Docker image from the provided Dockerfile and tag it as `trivy-lab:v1`:

```bash
docker build -t trivy-lab:v1 .
```

The original image is based on `node:18-bullseye-slim` (Debian 11) and runs the application using `npm start`.

## Step 3: Install Trivy

Install Trivy on Ubuntu/Debian:

```bash
sudo apt-get install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install -y trivy
```

Verify the installation:

```bash
trivy --version
```

## Step 4: Scan the Image

Run a full vulnerability scan:

```bash
trivy image trivy-lab:v1
```

Trivy scans both the operating system packages and the Node.js application dependencies.

## Step 5: Scan for HIGH and CRITICAL Vulnerabilities Only

```bash
trivy image --severity HIGH,CRITICAL trivy-lab:v1
```

Observed results:

| Target | Type | HIGH | CRITICAL | Total |
|---|---|---:|---:|---:|
| trivy-lab:v1 (debian 11.11) | OS packages | 86 | 7 | 93 |
| Node.js packages | node-pkg | 26 | 1 | 27 |
| **Total** | | **112** | **8** | **120** |

## Step 6: Generate an HTML Report

Download the Trivy HTML template:

```bash
wget -O html.tpl https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl
```

Generate the report:

```bash
trivy image --severity HIGH,CRITICAL \
  --format template --template "@html.tpl" \
  -o report-v1.html trivy-lab:v1
```

## Bonus: Vulnerability Remediation

### Step 7: Investigate the Vulnerabilities

The scan results show three sources of vulnerabilities:

- **Base image:** Debian 11 is end-of-life, so most OS vulnerabilities have no fix without changing the base image.
- **Application dependencies:** `express@4.18.2` pulls in vulnerable versions of `body-parser` (1.20.1) and `path-to-regexp` (0.1.7).
- **Bundled npm:** the npm included in the Node image contains vulnerable packages such as `tar`, `glob`, `minimatch`, and `brace-expansion`.

### Step 8: Fix the Issues

The following changes were applied:

- Changed the base image from `node:18-bullseye-slim` to `node:22-alpine`.
- Updated `express` from `4.18.2` to `^4.21.2`.
- Pinned `path-to-regexp` to `0.1.13` using npm `overrides`.
- Converted the Dockerfile to a multi-stage build.
- Removed `npm` and `npx` from the final image because they are not needed at runtime.
- Replaced `npm start` with `node app.js`.

Updated Dockerfile:

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json ./
RUN npm install --omit=dev

FROM node:22-alpine
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY package.json app.js ./
RUN rm -rf /usr/local/lib/node_modules/npm /usr/local/bin/npm /usr/local/bin/npx
EXPOSE 3000
CMD ["node", "app.js"]
```

Updated package.json:

```json
{
  "name": "trivy-lab",
  "version": "1.0.0",
  "description": "A simple Node.js application for Trivy scanning",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.21.2"
  },
  "overrides": {
    "path-to-regexp": "0.1.13"
  }
}
```

### Step 9: Rebuild the Image

```bash
docker build --no-cache -t ivolve-app:v2 .
```

### Step 10: Rescan and Generate a New Report

```bash
trivy image --severity HIGH,CRITICAL \
  --format template --template "@html.tpl" \
  -o report-v2.html ivolve-app:v2
```

### Step 11: Verify No Fixable HIGH or CRITICAL Vulnerabilities Remain

```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ivolve-app:v2
echo $?
```

The exit code `0` confirms that no HIGH or CRITICAL vulnerabilities with an available fix remain.

### Step 12: Run the Application

```bash
docker run --rm -p 3000:3000 ivolve-app:v2
```

The application starts successfully on port 3000 without npm.

## Results Comparison

| Image | Base Image | OS Vulnerabilities | Node.js Vulnerabilities | Total HIGH/CRITICAL |
|---|---|---:|---:|---:|
| trivy-lab:v1 | node:18-bullseye-slim | 93 | 27 | 120 |
| ivolve-app:v2 | node:22-alpine | 0 | 0 | 0 |

Remediation progression:

| Change | Remaining HIGH/CRITICAL |
|---|---:|
| Original image | 120 |
| `node:22-alpine` + updated express | 11 |
| Updated npm | 4 |
| Multi-stage build, npm removed, `overrides` | 0 |

## Screenshots

### Clone the Repository
<!-- ضيفي screenshot هنا: screenshots/01-clone.png -->

### Build the Original Image
<!-- ضيفي screenshot هنا: screenshots/02-build-v1.png -->

### Install Trivy and Verify
<!-- ضيفي screenshot هنا: screenshots/03-trivy-version.png -->

### Full Vulnerability Scan
<!-- ضيفي screenshot هنا: screenshots/04-scan-full.png -->

### HIGH and CRITICAL Scan
<!-- ضيفي screenshot هنا: screenshots/05-scan-high-critical.png -->

### HTML Report (v1)
<!-- ضيفي screenshot هنا: screenshots/06-report-v1.png -->

### Dockerfile and package.json Changes
<!-- ضيفي screenshot هنا: screenshots/07-dockerfile-changes.png -->

### Build the Updated Image
<!-- ضيفي screenshot هنا: screenshots/08-build-v2.png -->

### Docker Images
<!-- ضيفي screenshot هنا: screenshots/09-docker-images.png -->

### HTML Report (v2)
<!-- ضيفي screenshot هنا: screenshots/10-report-v2.png -->

### Final Scan and Exit Code
<!-- ضيفي screenshot هنا: screenshots/11-final-scan.png -->

### Application Running
<!-- ضيفي screenshot هنا: screenshots/12-app-running.png -->

## Conclusion

This lab demonstrates how Trivy can be used to identify vulnerabilities in a Docker image and how to remediate them. Updating the base image from an end-of-life Debian release to a maintained Alpine image removed all OS-level vulnerabilities, while updating dependencies and using a multi-stage build removed the remaining Node.js findings. Removing build tools such as npm from the runtime image also reduces the attack surface.# Lab 5: Docker Image Vulnerability Scanning with Trivy



| Change | Remaining HIGH/CRITICAL |
|---|---:|
| Original image | 120 |
| `node:22-alpine` + updated express | 11 |
| Updated npm | 4 |
| Multi-stage build, npm removed, `overrides` | 0 |

## Screenshots

### Build the Original Image
<img width="1280" height="476" alt="Screenshot 2026-10-21 233415" src="https://github.com/user-attachments/assets/040ee7ac-1fce-451e-b0dc-2a9428a73aaf" />

### Install Trivy and Full Vulnerability Scan
<img width="1280" height="764" alt="Screenshot 2026-10-21 233627" src="https://github.com/user-attachments/assets/22dc7a14-5404-42cd-bb7f-0089f771435e" />

### HIGH and CRITICAL Scan, HTML Report (v1), Dockerfile and package.json Changes, Build the Updated Image
<img width="1280" height="764" alt="Screenshot 2026-10-21 233825" src="https://github.com/user-attachments/assets/91a76144-ae93-407f-9f14-859928f0c77a" />

### HTML Report (v2)
<img width="1280" height="764" alt="Screenshot 2026-10-21 234018" src="https://github.com/user-attachments/assets/08f1f6a4-976c-415f-b781-c5d1a050c86e" />

### Final Scan and Exit Code
<img width="1280" height="764" alt="Screenshot 2026-10-21 234042" src="https://github.com/user-attachments/assets/4e99645a-fe41-417c-9c44-e6f7f941f198" />

### Application Running
<img width="1280" height="764" alt="Screenshot 2026-10-21 234102" src="https://github.com/user-attachments/assets/6b3c5a45-0870-4eee-b6fd-1f67a467670a" />

## Conclusion

This lab demonstrates how Trivy can be used to identify vulnerabilities in a Docker image and how to remediate them. Updating the base image from an end-of-life Debian release to a maintained Alpine image removed all OS-level vulnerabilities, while updating dependencies and using a multi-stage build removed the remaining Node.js findings. Removing build tools such as npm from the runtime image also reduces the attack surface.
