# Containerized Web Application with PostgreSQL using Docker Compose and IPvlan

University of Petroleum and Energy Studies

Course: Containerization and DevOps
Assignment: Containerized Web Application with PostgreSQL using Docker Compose and IPvlan

## Project Overview

This project demonstrates how to design, containerize, and deploy a web application using Docker.

The application consists of:

- **Backend API:** Python + Flask
- **Database:** PostgreSQL
- **Orchestration:** Docker Compose
- **Networking:** Docker IPvlan
- **Storage:** Docker Named Volume

Containers communicate using an IPvlan network, allowing each container to have its own static IP address within the LAN.

## Architecture

### Network Design Diagram

```
Client (Browser)
        │
        ▼
localhost:5000 (Port Mapping)
        │
        ▼
Docker Host (WSL2 + Docker Desktop)
        │
        ▼
Docker IPvlan Network
(192.168.160.0/20)
        │
        ├───────────────────────┐
        ▼                       ▼
Backend Container          PostgreSQL Container
(Python + Flask)           (PostgreSQL DB)
IP: 192.168.170.10         IP: 192.168.170.11
Port: 5000                 Port: 5432
                                │
                                ▼
                     Docker Volume (pgdata)
                       Persistent Storage
```

## Repository Structure

```
Assignment 1/
│
├── Backend/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py
│   └── templates/
│       └── index.html
│
├── database/
│   └── init.sql
│
├── docker-compose.yml
└── README.md
```

## File Contents

### Backend/main.py

```python
from flask import Flask, render_template

app = Flask(__name__)

@app.route('/')
def home():
    return render_template("index.html")

@app.route('/api')
def api():
    return {
        "message": "DevOps Project 🚀",
        "status": "Backend running successfully"
    }

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### Backend/requirements.txt

```
flask
```

### Backend/Dockerfile

```dockerfile
FROM python:3.9

WORKDIR /app

COPY . .

RUN pip install -r requirements.txt

EXPOSE 5000

CMD ["python", "app.py"]
```

### database/init.sql

```sql
CREATE TABLE IF NOT EXISTS users(
  id SERIAL PRIMARY KEY,
  name TEXT
);
```

### docker-compose.yml

```yaml
version: "3.9"

services:

  backend:
    build: ./Backend
    container_name: pa1_backend
    ports:
      - "5000:5000"
    environment:
      DB_HOST: db
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: mydb
    depends_on:
      - db
    networks:
      ipvlan_net:
        ipv4_address: 192.168.170.10

  db:
    image: postgres:15-alpine
    container_name: pa1_db
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      ipvlan_net:
        ipv4_address: 192.168.170.11

networks:
  ipvlan_net:
    external: true

volumes:
  pgdata:
```

## Create IPvlan Network

Create the IPvlan network required for container communication.

```bash
docker network create \
  -d ipvlan \
  --subnet=192.168.160.0/20 \
  --gateway=192.168.160.1 \
  -o ipvlan_mode=l2 \
  -o parent=eth0 \
  ipvlan_net
```

## Build and Run

### Create Project Directory

```bash
mkdir devops-project
cd devops-project
```

### Build and Start Containers

```bash
docker compose up --build -d
```

### Check Running Containers

```bash
docker ps
```

## Test Application

### Open in Browser

```
http://localhost:5000
```

### Test API Endpoint

```bash
curl http://localhost:5000/api
```

Output:

```json
{
  "message": "DevOps Project 🚀",
  "status": "Backend running successfully"
}
```

## Verify Containers

```bash
docker ps
```

## Verify Network

```bash
docker network inspect ipvlan_net
```

## Verify Images

```bash
docker images
```

## Verify Volumes

```bash
docker volume ls
```

## Verify Volume Persistence

This project uses a Docker named volume (pgdata) to ensure that PostgreSQL data persists even if containers are stopped or restarted.

### Step 1 — Start Containers

```bash
docker compose up -d
```

### Step 2 — Stop Containers

Stop and remove the running containers.

```bash
docker compose down
```

### Step 3 — Verify Volume Still Exists

```bash
docker volume ls | grep pgdata
```

### Step 4 — Restart Containers

```bash
docker compose up -d
```

### Step 5 — Verify Data Persistence

The volume and its data are still present after restart.

```bash
docker volume ls
```

## Container IPs

| Container    | IP Address       | Port |
|-------------|-----------------|------|
| pa1_backend | 192.168.170.10  | 5000 |
| pa1_db      | 192.168.170.11  | 5432 |

## Notes on WSL2 + IPvlan

In WSL2, containers on an IPvlan network cannot be reached directly from the Windows host due to Hyper-V virtual switch NAT isolation. All testing is done from inside the container using `docker exec`. This is a known limitation.
