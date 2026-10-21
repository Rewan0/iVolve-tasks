# Lab 4: Docker Image Optimization and Export/Load

## Objective

Build a Java Spring Boot application using Docker, optimize its image using a multi-stage Dockerfile, run the application as a non-root user, compare image sizes, and practice exporting and loading Docker images.

## Tools and Technologies

- Docker
- Java 17
- Maven
- Spring Boot
- Eclipse Temurin Alpine images

## Project Structure

```text
Lab4/
├── Dockerfile
├── Dockerfile2
├── pom.xml
├── src/
└── README.md
```

## Step 1: Clone the Repository

```bash
git clone https://github.com/Ibrahim-Adel15/ImageOptimization-.git
cd ImageOptimization-
```

## Step 2: Build the Original Image

Build the Docker image using the original Dockerfile:

```bash
docker build -t image1 .
```

List the image:

```bash
docker images image1
```

The original image uses Maven and Java 17 to build and run the application.

## Step 3: Build the Optimized Image

Build the optimized image using the multi-stage Dockerfile:

```bash
docker build -t image2 -f Dockerfile2 .
```

The optimized Dockerfile uses:
- A Maven builder stage to compile and package the application.
- A separate Java Runtime stage to run the packaged JAR.
- A dedicated non-root user named `appuser`.

## Step 4: Compare Image Sizes

Check the image sizes:

```bash
docker images image1
docker images image2
```

Observed results:

| Image | Disk Usage | Content Size |
|---|---:|---:|
| image1 | 671 MB | 239 MB |
| image2 | 302 MB | 88.2 MB |

Based on the reported disk usage, the optimized image reduced the image size by approximately 55%.

The actual storage usage can vary depending on Docker's image layers and shared storage.

## Step 5: Run and Verify the Optimized Container

Run the optimized image:

```bash
docker run -d --name container2 -p 8087:8080 image2
```

Check the container status and application logs:

```bash
docker ps
docker logs container2
```

Test the application:

```bash
curl http://localhost:8087
```

Verify that the application runs as a non-root user:

```bash
docker exec container2 whoami
docker exec container2 id
```

The expected username is `appuser`, and the UID must not be `0`.

## Step 6: Export the Optimized Image

Save `image2` to a TAR archive:

```bash
docker save -o lab-image.tar image2
```

Verify that the archive exists:

```bash
ls -lh lab-image.tar
```

## Step 7: Remove the Image

Remove the optimized image from the local Docker host:

```bash
docker rmi image2
```

If Docker reports that a container is using the image, stop and remove that container before removing the image.

## Step 8: Load the Image

Restore the image from the TAR archive:

```bash
docker load -i lab-image.tar
```

Verify that the image has been restored:

```bash
docker images image2
```
## Screenshots

## Build the Original Image
<img width="2560" height="1600" alt="image" src="https://github.com/user-attachments/assets/3d4769ce-6cf9-4b56-ac7e-9f20bdf280ba" />
## Build the Optimized Image
<img width="1280" height="764" alt="Screenshot 2026-10-21 211614" src="https://github.com/user-attachments/assets/321708e2-d845-483f-aea3-ff1d87ac03bd" />
## Compare Image Sizes
<img width="2559" height="307" alt="image" src="https://github.com/user-attachments/assets/40b10105-567b-4955-babd-2504983421b2" />
## Run the Optimized Container, Export, Remove & Load the Image
<img width="2559" height="741" alt="image" src="https://github.com/user-attachments/assets/565306fc-ee0b-4a5d-96fe-61ed84476dc5" />


## Conclusion

This lab demonstrates how multi-stage Docker builds can reduce the size of a Java application's runtime image by separating build dependencies from runtime dependencies. It also demonstrates running a container with a non-root user and exporting, deleting, and restoring a Docker image using TAR archives.
