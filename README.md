# 🚀 Three-Tier MERN Application Deployment using Docker Compose on AWS EC2

A real-world DevOps project demonstrating how to **containerize, orchestrate, and deploy** a full-stack **MERN** application (React frontend + Node.js backend + MongoDB database) using **Docker Compose**, hosted on an **AWS EC2** instance.

---

## 📌 Project Overview

This project simulates an industry-level scenario where a startup needs to quickly prototype and deploy a three-tier web application with consistent environments across development and production. Docker Compose is used to define, build, and run all three services together with a single command.

**Architecture:**

```
                 ┌────────────────────────────┐
                 │        Web Application      │
                 └────────────────────────────┘
                          │        │        │
                  ┌───────┘        │        └───────┐
                  ▼                ▼                ▼
            ┌──────────┐     ┌──────────┐     ┌──────────┐
            │ Frontend │     │ Backend  │     │ MongoDB  │
            │ (React)  │ ──► │ (Node.js)│ ──► │(Database)│
            │ Port 3000│     │ Port 5000│     │Port 27017│
            └──────────┘     └──────────┘     └──────────┘
```

Each service runs in its own Docker container, orchestrated via `docker-compose.yml`.

---

## 🛠️ Tech Stack

| Layer        | Technology              |
|--------------|--------------------------|
| Frontend     | React.js                 |
| Backend      | Node.js + Express        |
| Database     | MongoDB                  |
| Orchestration| Docker Compose           |
| Hosting      | AWS EC2 (Ubuntu)         |

---

## 📁 Project Structure

```
three-tier-Docker-compose/
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
├── backend/
│   ├── Dockerfile
│   ├── server.js
│   ├── package.json
│   └── .env
├── docker-compose.yml
└── README.md
```

---

## ⚙️ Prerequisites

- AWS EC2 instance (Ubuntu, t2.medium or higher recommended — **at least 20GB storage**)
- Docker & Docker Compose installed
- Security Group with required ports opened (3000, 5000, 27017)

---

## 📦 1. Install Docker and Docker Compose

```bash
sudo apt update
sudo apt install -y docker.io
sudo apt install -y docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
```

> ✅ Verify installation:
> ```bash
> docker --version
> docker compose version
> ```

---

## 📥 2. Clone the Repository

```bash
git clone <your-repo-url>
cd three-tier-Docker-compose
```

---

## 🐳 3. Dockerfiles

### `frontend/Dockerfile`
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npx", "serve", "-s", "build"]
```

### `backend/Dockerfile`
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["node", "server.js"]
```

> ⚠️ **Important:** Inside `server.js`, make sure Express listens on `0.0.0.0`, not `127.0.0.1` or `localhost` — otherwise the container's exposed port won't be reachable from outside:
> ```js
> app.listen(5000, () => console.log("Backend running on port 5000"));
> ```

---

## 🔐 4. Environment Variables

Create a `.env` file inside `backend/`:

```env
MONGO_URI=mongodb://mongo:27017/three-tier-db
```

> 🔑 Note: Since MongoDB runs as a service **inside the same Docker network**, use the **service name `mongo`** as the hostname — not `localhost` and not a MongoDB Atlas URI (unless you're using a cloud-hosted database instead of the local container).

---

## 📝 5. Docker Compose File

```yaml
services:
  mongo:
    image: mongo
    container_name: mongo
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

  backend:
    build: ./backend
    container_name: backend
    ports:
      - "5000:5000"
    depends_on:
      - mongo
    env_file:
      - ./backend/.env

  frontend:
    build: ./frontend
    container_name: frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend

volumes:
  mongo-data:
```

> Note: the `version:` key has been removed — it's obsolete in modern Docker Compose and only triggers a warning.

---

## ▶️ 6. Run the Application

```bash
docker compose up -d
```

Check running containers:
```bash
docker compose ps
```

View logs for any service:
```bash
docker compose logs -f backend
```

Stop all containers:
```bash
docker compose down
```

Rebuild after code changes:
```bash
docker compose up -d --build
```

---

## ☁️ 7. AWS EC2 Security Group Configuration

Go to **EC2 → Security Groups → Inbound Rules → Edit inbound rules**, and add:

| Type        | Protocol | Port Range | Source      |
|-------------|----------|------------|-------------|
| Custom TCP  | TCP      | 3000       | 0.0.0.0/0   |
| Custom TCP  | TCP      | 5000       | 0.0.0.0/0   |
| Custom TCP  | TCP      | 27017      | 0.0.0.0/0 *(optional, only for external DB access)* |

> 🔒 For production, restrict the `Source` to specific IPs instead of `0.0.0.0/0`.

---

## ✅ 8. Validate the Deployment

| Check | Command / URL |
|---|---|
| Frontend loads | `http://<EC2_PUBLIC_IP>:3000` |
| Backend API responds | `curl http://<EC2_PUBLIC_IP>:5000/api/items` |
| MongoDB connected | Check backend logs for `MongoDB connected` |

---

## 🩺 Troubleshooting Guide

Real issues encountered during this deployment, and how they were resolved:

### 1. `docker-compose: command not found`
Modern Docker installs the Compose **plugin**, which uses a space instead of a hyphen:
```bash
docker compose up -d
```
Or install the standalone binary / plugin explicitly:
```bash
sudo apt install docker-compose-plugin -y
```

### 2. `permission denied while trying to connect to the Docker daemon socket`
The current user isn't in the `docker` group:
```bash
sudo usermod -aG docker $USER
newgrp docker
```

### 3. `no space left on device` during build
EC2 root volume (commonly 8GB by default) runs out of space during `npm install` / image builds.

**Fix — extend the EBS volume without downtime:**
1. AWS Console → EC2 → Volumes → select volume → **Modify Volume** → increase size
2. On the server:
   ```bash
   sudo growpart /dev/xvda 1
   sudo resize2fs /dev/xvda1
   df -h   # confirm new size
   ```
3. Clean up old Docker data:
   ```bash
   docker system prune -a --volumes
   ```

### 4. Frontend loads but data doesn't save / API calls fail
- Check the browser console (F12 → Console/Network tab) for the exact error.
- `ERR_CONNECTION_TIMED_OUT` on the backend port almost always means the **AWS Security Group** isn't allowing inbound traffic on that port — add the rule as shown above.
- Confirm the backend is actually listening on `0.0.0.0`, not `127.0.0.1`, inside the container.
- Confirm `docker ps` shows port mapping as `0.0.0.0:5000->5000/tcp` (not `127.0.0.1:...`).

### 5. Backend can't connect to MongoDB
Double-check `MONGO_URI` in `backend/.env` uses the **Docker Compose service name** (`mongo`), not `localhost`:
```env
MONGO_URI=mongodb://mongo:27017/three-tier-db
```

---

## 📖 Key Concepts Covered

- ✅ What is YAML and why it's used in DevOps configs
- ✅ What is Docker Compose and how it orchestrates multi-container apps
- ✅ Writing Dockerfiles for React and Node.js services
- ✅ Connecting services via Docker's internal network (service-name-based DNS)
- ✅ Persisting MongoDB data with Docker volumes
- ✅ Exposing services securely via AWS Security Groups
- ✅ Debugging real-world container networking and storage issues on EC2

---

## 🎯 Conclusion

This project demonstrates a complete, real-world DevOps workflow: containerizing a three-tier MERN application, orchestrating it with Docker Compose, and deploying it on AWS EC2 with proper networking, storage, and security configuration — along with the practical troubleshooting steps every engineer runs into along the way.

---

## 📌 Future Improvements

- [ ] Add Nginx as a reverse proxy in front of frontend/backend
- [ ] Add HTTPS via Let's Encrypt / Certbot
- [ ] Set up CI/CD with GitHub Actions for automated build & deploy
- [ ] Add health checks in `docker-compose.yml`
- [ ] Use Docker secrets / AWS Secrets Manager instead of plain `.env` for production

---

## 👤 Author

Maintained as part of a hands-on DevOps learning project (MERN + Docker + AWS EC2).

'''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''''
# 🚀 Secure Deployment of a Full-Stack MERN App using Docker, Docker Compose & AWS EC2

A production-style deployment of a **MERN stack** (MongoDB, Express/Node.js, React) application using **Docker multi-stage builds**, **Alpine base images**, **non-root containers**, and **healthchecks** — orchestrated with **Docker Compose** and deployed on an **AWS EC2** instance.

---

## 📌 Industry Scenario

A travel-tech startup wants to containerize and securely deploy their web application (React frontend, Node.js backend, MongoDB database) on an AWS EC2 instance. The goal is to apply Docker best practices — multi-stage builds for small image sizes, security hardening with non-root users, and monitoring with healthchecks — using Docker Compose.

## 🎯 Objectives

- Clone and containerize the app (React + Node.js + MongoDB)
- Use Alpine images and Docker multi-stage builds
- Apply Docker security best practices (non-root user, minimal images, healthchecks)
- Deploy and expose the app securely on AWS EC2 via Docker Compose

## 🛠️ Tech Stack

| Layer      | Technology            |
|------------|------------------------|
| Frontend   | React (served via `serve`) |
| Backend    | Node.js / Express      |
| Database   | MongoDB 6.0             |
| Container  | Docker (multi-stage, Alpine) |
| Orchestration | Docker Compose       |
| Hosting    | AWS EC2 (Ubuntu)        |

---

## 📂 Project Structure

```
three-tier-app/
├── backend/
│   └── Dockerfile
├── frontend/
│   └── Dockerfile
└── docker-compose.yml
```

---

## ✅ Step-by-Step Implementation

### 1. Clone the Repository

```bash
git clone https://github.com/Umair1012/three-tier-app.git
cd three-tier-app
```

### 2. Backend Dockerfile (Node.js — Multi-Stage & Alpine)

`backend/Dockerfile`

```dockerfile
# --- Stage 1: Build ---
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .

# --- Stage 2: Production ---
FROM node:18-alpine
RUN adduser -S appuser
WORKDIR /app
COPY --from=builder /app .
USER appuser
EXPOSE 5000
CMD ["node", "server.js"]
```

### 3. Frontend Dockerfile (React — Multi-Stage & Alpine)

`frontend/Dockerfile`

```dockerfile
# --- Stage 1: Build React app ---
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# --- Stage 2: Serve with 'serve' ---
FROM node:18-alpine
RUN npm install -g serve
COPY --from=builder /app/build /build
RUN adduser -S appuser
USER appuser
EXPOSE 3000
CMD ["serve", "-s", "/build"]
```

### 4. MongoDB

No custom Dockerfile needed — handled directly via the official `mongo:6.0` image in Docker Compose.

### 5. Docker Compose File

`docker-compose.yml`

```yaml
version: '3.8'

services:
  backend:
    build: ./backend
    ports:
      - "5000:5000"
    depends_on:
      - mongo
    env_file:
      - ./backend/.env
    user: "1000:1000"
    read_only: true
    restart: always
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend
    env_file:
      - ./frontend/.env
    user: "1000:1000"
    read_only: true
    restart: always
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3

  mongo:
    image: mongo:6.0
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
    restart: always
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  mongo-data:

networks:
  app-network:
    driver: bridge
```

> ⚠️ **Note:** The original source doc had a mismatch — frontend was mapped to port `3000` in Compose but referenced as `8000` in the verification/security-group steps, and the Mongo healthcheck had no `test` command. Both are fixed above for consistency — adjust the frontend port back to `8000:3000` if that's what your security group/CDN expects.

### 6. Launch EC2 & Install Docker

**Requirements:**
- Ubuntu EC2 instance (`t2.medium` or higher)
- Security group allows inbound ports: `22`, `3000`, `5000`, `27017`

```bash
# Connect to EC2
ssh -i your-key.pem ubuntu@your-ec2-ip

# Install Docker & Docker Compose
sudo apt update && sudo apt install -y docker.io docker-compose
sudo usermod -aG docker ubuntu
newgrp docker



### 7. Build & Run with Docker Compose

```bash
docker-compose up --build -d
```

**Reset everything (if needed):**

```bash
sudo docker-compose down --volumes --remove-orphans
sudo docker-compose build
sudo docker-compose up -d
```

**Debug failing services:**

```bash
docker-compose logs frontend
docker-compose logs backend
```

### 8. Verify Deployment

| Service        | URL                                |
|-----------------|-------------------------------------|
| Frontend        | `http://<EC2-IP>:3000`             |
| Backend API     | `http://<EC2-IP>:5000`             |
| MongoDB         | Internal only — `mongo:27017`      |

### 9. AWS EC2 Security Group Rules

From the AWS Console → **Inbound Rules → Add custom TCP:**

- `3000` (React frontend)
- `5000` (Node.js API)
- `27017` (MongoDB — debugging/dev only)

> 🛑 **Production tip:** Restrict MongoDB access to the backend container only — never expose `27017` publicly in production.

---

## 🔐 Docker Security Best Practices Applied

- ✅ Multi-stage builds (smaller images, no leftover build tools)
- ✅ Alpine-based minimal images (reduced attack surface)
- ✅ Non-root users in every container
- ✅ Healthcheck endpoints for backend & MongoDB
- ✅ Read-only root filesystem on app containers
- ✅ Isolated bridge network for service-to-service communication
- ✅ Port-specific security group rules

---

## 🎯 Conclusion

This project demonstrates end-to-end Dockerization and deployment of a production-ready, full-stack MERN application using Docker best practices on AWS. Multi-stage builds and Alpine images minimize the attack surface, while healthchecks and non-root users improve security and observability — reflecting how modern DevOps teams ship microservices securely on cloud infrastructure.

---

## 📜 License

This project is for educational/demo purposes as part of a DevOps learning series.















