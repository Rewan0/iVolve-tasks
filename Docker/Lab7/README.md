# Lab 7: Multi-Network Microservices, Volumes, and Bind Mounts

## Objective

Containerize a three-service microservices application (frontend, backend, and notification), connect the services using two custom bridge networks with separate subnets, persist logs using Docker volumes, and persist backend application data using a bind mount.

## Tools and Technologies

- Docker
- Python 3.12
- Flask
- Docker bridge networks
- Docker volumes and bind mounts

## Project Structure

```text
Lab7/
├── backend/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
├── frontend/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
├── notification/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
└── README.md
```

## Architecture

```text
                 frontend-backend            backend-notification
                 (172.28.0.0/16)             (172.29.0.0/16)
┌──────────┐        ┌───────────┐        ┌────────────────┐
│ Frontend │ ─────► │  Backend  │ ─────► │  Notification  │
│  :3000   │        │   :5000   │        │     :5001      │
└──────────┘        └───────────┘        └────────────────┘
 host :8080          │         │                  │
                     ▼         ▼                  ▼
              bind mount   backend_logs   notification_logs
           (~/backend-data)   (volume)        (volume)
```

| Service | Port | Configuration |
|---|---|---|
| frontend | 3000 (published as 8080) | `BACKEND_URL` |
| backend | 5000 | `NOTIFICATION_URL` |
| notification | 5001 | None |

The backend is the only container attached to both networks, so the frontend cannot reach the notification service directly.

## Step 1: Clone the Repository

```bash
git clone https://github.com/Ibrahim-Adel15/MicroserviceApp.git
cd MicroserviceApp
```

## Step 2: Build the Images

Each service has its own Dockerfile based on `python:3.12-slim`. It installs the dependencies from `requirements.txt`, copies `app.py`, creates the required directories, and starts the application.

```bash
docker build -t backend ./backend
docker build -t notification ./notification
docker build -t frontend ./frontend
```

Verify the images:

```bash
docker images | grep -E "backend|frontend|notification"
```

## Step 3: Create the Networks

Create two bridge networks with separate subnets:

```bash
docker network create --subnet 172.28.0.0/16 frontend-backend
docker network create --subnet 172.29.0.0/16 backend-notification
```

Verify the subnets:

```bash
docker network inspect frontend-backend backend-notification \
  --format '{{.Name}}: {{range .IPAM.Config}}{{.Subnet}}{{end}}'
```

## Step 4: Create the Volumes and Host Directory

Create two volumes for the backend and notification logs:

```bash
docker volume create backend_logs
docker volume create notification_logs
```

Create the host directory used by the bind mount:

```bash
mkdir -p ~/backend-data
```

Storage summary:

| Type | Source | Container Path | Purpose |
|---|---|---|---|
| Bind mount | `~/backend-data` | `/app/data` | Backend messages (`messages.json`) |
| Volume | `backend_logs` | `/app/logs` | Backend logs |
| Volume | `notification_logs` | `/app/logs` | Notification logs |

## Step 5: Run the Notification Container

```bash
docker run -d --name notification \
  --network backend-notification \
  --mount type=volume,source=notification_logs,target=/app/logs \
  notification
```

## Step 6: Run the Backend Container

Run the backend on the `frontend-backend` network with the bind mount, the logs volume, and the notification service URL:

```bash
docker run -d --name backend \
  --network frontend-backend \
  --mount type=bind,source=$HOME/backend-data,target=/app/data \
  --mount type=volume,source=backend_logs,target=/app/logs \
  -e NOTIFICATION_URL=http://notification:5001/notify \
  backend
```


```bash
docker network connect backend-notification backend
```

Verify that the backend is attached to both networks:

```bash
docker inspect -f '{{range $k,$v := .NetworkSettings.Networks}}{{$k}} {{end}}' backend
```

## Step 7: Run the Frontend Container

Run the frontend on the `frontend-backend` network, publish port 8080, and configure the backend URL:

```bash
docker run -d --name frontend \
  --network frontend-backend \
  -p 8080:3000 \
  -e BACKEND_URL=http://backend:5000 \
  frontend
```

Check the running containers:

```bash
docker ps --filter name=frontend --filter name=backend --filter name=notification
```

## Step 8: Access and Verify the Application

Open the web page in the browser:

```text
http://localhost:8080
```

Create a few messages and confirm that they appear on the page.

Verify the persisted data and logs:

```bash
cat ~/backend-data/messages.json
docker exec backend ls /app/logs
docker exec notification ls /app/logs
docker logs notification
```

Expected results:
- `messages.json` exists in the host directory and contains the created messages.
- `backend.log` exists in the backend logs volume.
- `notification.log` exists in the notification logs volume.
- The notification logs show `Notification received for message: ...`, which confirms that the backend reached the notification service.

## Screenshots

### Build the Images
<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/40814227-b22f-4d82-9f45-736097a772ef" />

### Networks and Subnets
<img width="2559" height="516" alt="image" src="https://github.com/user-attachments/assets/17f4bb0c-3f61-43dc-9a40-5008ef0f1d8e" />

### Volumes
<img width="1183" height="111" alt="image" src="https://github.com/user-attachments/assets/a2d7ed6c-ddef-4e7d-917a-8850cf1a306f" />

### Running Containers
<img width="2559" height="883" alt="image" src="https://github.com/user-attachments/assets/9abf2c50-4ee3-41b4-94b4-4309d91181b4" />

### Web Page with Messages
<img width="2560" height="1528" alt="image" src="https://github.com/user-attachments/assets/13a2d9c5-00ac-4bc7-98c4-8dfb58804d19" />

### Persisted Data and Logs
<img width="1423" height="328" alt="image" src="https://github.com/user-attachments/assets/b4e1a044-2ab1-4a9f-b1f7-ea9711b3778d" />

## Conclusion

This lab demonstrates how to isolate microservices using separate bridge networks, where the backend acts as the only bridge between the frontend and the notification service. It also demonstrates the difference between Docker volumes, used here for logs, and bind mounts, used here to persist backend data directly on the host.
