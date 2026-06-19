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
