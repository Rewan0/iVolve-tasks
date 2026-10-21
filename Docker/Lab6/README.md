# Lab 6: Docker Environment Variables and Resource Management

## Objective

Containerize a Python Flask application using Docker, pass environment variables in three different ways (`-e`, `--env-file`, and `ENV` in the Dockerfile), limit the CPU and memory of a container, and monitor its resource usage.

## Tools and Technologies

- Docker
- Python 3.12
- Flask
- `python:3.12-slim` base image

## Project Structure

```text
Lab6/
├── app.py
├── requirements.txt
├── Dockerfile
├── env.txt
└── README.md
```

## Step 1: Clone the Repository

```bash
git clone https://github.com/Ibrahim-Adel15/PythonApp.git
cd PythonApp
```

## Step 2: Create the Dockerfile

Create a `Dockerfile` using the `python:3.12-slim` base image:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

The Dockerfile:
- Uses a lightweight Python 3.12 base image.
- Copies and installs the dependencies before copying the rest of the code, so Docker can cache the dependency layer.
- Exposes port `5000`, which the application listens on.

## Step 3: Build the Image

Build the `ivolve1` image:

```bash
docker build -t ivolve1 .
```

List the image:

```bash
docker images ivolve1
```

## Step 4: Run container1 with Environment Variables with Limiting the Container Resources

Run `container1`, map port `5000`, and set the environment variables using `-e`:
Limit the container to 1 CPU and 256 MB of memory. The limits are applied in the same `docker run` command:

```bash
docker run -d --name container1 -p 5000:5000 \
  -e APP_NAME=iVolveApp1 \
  -e APP_ENV=stg \
  --cpus="1" \
  --memory="256m" \
  ivolve1
```

Verify the configured limits:

```bash
docker inspect container1 --format 'CPU={{.HostConfig.NanoCpus}} MEM={{.HostConfig.Memory}}'
```

The expected output is `CPU=1000000000 MEM=268435456`, which equals 1 CPU and 256 MB.

## Step 5: Verify the Application

Test the application:

```bash
curl http://localhost:5000
```

The application should return a JSON response containing `iVolveApp1` and `stg`.

## Step 6: Verify the Environment Variables

Check the environment variables inside the container:

```bash
docker exec container1 env | grep APP_
```

Expected output:

```text
APP_NAME=iVolveApp1
APP_ENV=stg
```

## Step 7: Monitor CPU and Memory Usage

Monitor the container resource usage:

```bash
docker stats --no-stream container1
```

The `MEM USAGE / LIMIT` column should show a limit of `256MiB`.

## Step 8: Create the env.txt File

Create `env.txt` with the following content:

```text
APP_NAME=iVolveApp2
APP_ENV=prod
```

Do not add quotes or spaces around `=` inside an env file.

## Step 9: Run container2 Using env.txt

Stop `container1` first because it is using host port `5000`:

```bash
docker stop container1
```

Run `container2` and load the variables from `env.txt`:

```bash
docker run -d --name container2 -p 5000:5000 --env-file env.txt ivolve1
```

Verify the environment variables:

```bash
docker exec container2 env | grep APP_
```

Expected output:

```text
APP_NAME=iVolveApp2
APP_ENV=prod
```

## Step 10: Build an Image with Hardcoded Environment Variables

Create `Dockerfile.env`, which is the same as the original Dockerfile with these lines added before `CMD`:

```dockerfile
ENV APP_NAME=iVolveApp3
ENV APP_ENV=dev
```

Build the new image:

```bash
docker build -f Dockerfile.env -t ivolve2 .
```

## Step 11: Run container2 and Validate the Dockerfile Variables

Remove the old `container2` so the name can be reused:

```bash
docker rm -f container2
```

Run `container2` from the new image:

```bash
docker run -d --name container2 -p 5000:5000 ivolve2
```

Validate the environment variables and the application response:

```bash
docker exec container2 env | grep APP_
curl http://localhost:5000
```

Expected output:

```text
APP_NAME=iVolveApp3
APP_ENV=dev
```

## Summary of Results

| Container | Image | Variables Source | APP_NAME | APP_ENV |
|---|---|---|---|---|
| container1 | ivolve1 | `-e` flags | iVolveApp1 | stg |
| container2 | ivolve1 | `--env-file env.txt` | iVolveApp2 | prod |
| container2 | ivolve2 | `ENV` in Dockerfile | iVolveApp3 | dev |

## Screenshots

### Run container1 with Environment Variables and Resource Limits, Verify the Application and Environment Variables
<img width="2559" height="199" alt="image" src="https://github.com/user-attachments/assets/6610c642-0fc1-43c8-b865-04172aa0b58b" />

### Monitor CPU and Memory Usage
<img width="2559" height="91" alt="image" src="https://github.com/user-attachments/assets/012bc050-1850-44e8-ba17-b6f1f04a10d9" />

### Run container2 Using env.txt
<img width="2559" height="571" alt="image" src="https://github.com/user-attachments/assets/55f3ce87-4210-49b8-8f7f-13dc17e73816" />

### Build the ivolve2 Image with Hardcoded Variables
<img width="2559" height="709" alt="image" src="https://github.com/user-attachments/assets/1f450cd1-a8ad-4b76-b6d8-08ff2a6cd84e" />

### Run container2 and Validate the Dockerfile Variables
<img width="2476" height="256" alt="image" src="https://github.com/user-attachments/assets/46987936-3589-4812-aa1f-c9af5e99ec9d" />

## Conclusion

This lab demonstrates three ways to configure a containerized application with environment variables: runtime flags with `-e`, an external file with `--env-file`, and default values baked into the image with `ENV`. It also demonstrates limiting a container to 1 CPU and 256 MB of memory and monitoring its usage with `docker stats`.
