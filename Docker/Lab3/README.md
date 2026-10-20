# Lab 3: Create a Docker Image for a Java Application

## Objective

The objective of this lab is to containerize a Java Calculator application built with Maven using Docker. Two Docker image-building approaches were explored and compared.

## Tools and Technologies

- Ubuntu Linux
- Java 17
- Apache Maven
- Docker
- Amazon Corretto 17
- Maven Docker image

## Project Structure

```text
Lab3/
├── Dockerfile
├── pom.xml
├── src/
└── README.md
```

## Part 1: Build inside Docker

### 1. Dockerfile

The first approach uses a Maven image with Eclipse Temurin Java 17. Maven builds the application inside the Docker image.

```dockerfile
FROM maven:3.10-eclipse-temurin-17-alpine

WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --chown=appuser:appgroup . .

RUN mvn package

CMD ["java", "-jar", "target/calculator.jar"]
```

### 2. Build the Docker Image

Command:

```bash
docker build -t app1 .
```

**Result:** The Docker image was built successfully and tagged as `app1:latest`.

### 3. Run the Container

Command:

```bash
docker run -d --name cont1 -p 8085:8080 app1
```

The container was created, but it exited with status code `1`.

###



## Part 2: Build Locally

### 1. Dockerfile

The second approach uses Amazon Corretto 17 as the runtime image. The application JAR is built separately with Maven and then copied into the image.

```dockerfile
FROM amazoncorretto:17-alpine3.21-full

WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

COPY --chown=appuser:appgroup target/calculator.jar app.jar

USER appuser

CMD ["java", "-jar", "calculator.jar"]
```

### 2. Build the Docker Image

Command:

```bash
docker build -t app2 .
```

**Result:** The Docker image was built successfully and tagged as `app2:latest`.

### 3. Run the Container

Command:

```bash
docker run -d --name container2 -p 8086:8080 app2
```

The container was created, but it also exited with status code `1` because the Calculator requires interactive console input.

### 4. Verify Docker Images

Command:

```bash
docker images app1
docker images app2
```

Both images were present locally after successful builds.

## Comparison Between Part 1 and Part 2

| Feature             | Part 1                                        | Part 2                             |
| ------------------- | --------------------------------------------- | ---------------------------------- |
| Base image          | Maven with Eclipse Temurin 17                 | Amazon Corretto 17                 |
| Maven build         | Inside the Docker image                       | Performed before the Docker build  |
| JAR handling        | Copied with the project source, then packaged | Existing JAR copied into the image |
| Image tag           | `app1:latest`                                 | `app2:latest`                      |
| Docker build        | Successful                                    | Successful                         |
| Container execution | Exited with code 1                            | Exited with code 1                 |

## Security Considerations

- A non-root user named `appuser` is created in both images.
- Part 2 explicitly switches to `USER appuser` before running the application.
- Part 2 uses a runtime-focused image and copies only the packaged JAR, rather than the complete source project.
- A `.dockerignore` file can exclude unnecessary files such as `.git/` and `target/` when they are not needed in the build context.



## Screenshots

## Part 1
<img width="1707" height="1019" alt="lab3Part1" src="https://github.com/user-attachments/assets/47e47ebb-211a-4c46-8387-2e68d8598960" />
<img width="1707" height="1019" alt="image" src="https://github.com/user-attachments/assets/40178570-604c-4aab-aff1-c2e77b301727" />

## Part 2
<img width="1707" height="1019" alt="lab3Part2" src="https://github.com/user-attachments/assets/fbccb246-cd0c-492f-bb74-9ffb5f89afbe" />
<img width="1707" height="1019" alt="image" src="https://github.com/user-attachments/assets/6d35fb7d-713d-400c-9c95-ae7b8487f05a" />

### Comparison
<img width="1706" height="322" alt="image" src="https://github.com/user-attachments/assets/7cf0bc3c-0614-44dd-93c4-811a9e30cf29" />


## Conclusion
Successfully built two Docker images for the Java Maven Calculator application. The Amazon Corretto image (app2, 453 MB) was smaller than the Maven-based image (app1, 602 MB) because it contains only the Java runtime and the pre-built JAR, without the additional Maven build tools.

