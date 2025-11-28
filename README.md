# MEAN CRUD App – Dockerized Deployment with Jenkins CI/CD

This project is a **CRUD MEAN application** (MongoDB, Express, Angular, Node.js) that has been:

- Containerized using **Docker**
- Orchestrated using **Docker Compose**
- Deployed on an **Ubuntu VM** on cloud
- Automated end-to-end using a **Jenkins CI/CD pipeline**
- Exposed behind **Nginx** as a reverse proxy on **port 80**

This implementation is done as part of the **DevOps Engineer Intern assignment** for Discover Dollar.

---

## 🔧 Tech Stack & Tools

### Application Stack

- **Frontend**: Angular
- **Backend**: Node.js + Express
- **Database**: MongoDB

### DevOps & Infrastructure

- **Cloud VM**: Ubuntu (AWS / Azure / Any)
- **Containers**: Docker
- **Orchestration**: Docker Compose
- **Reverse Proxy**: Nginx
- **CI/CD**: Jenkins Pipeline
- **Registry**: Docker Hub

---

## 📁 Project Structure

```text
crud-dd-task-mean-app/
├─ backend/
│  ├─ server.js
│  ├─ package.json
│  ├─ Dockerfile
│  └─ app/
│     ├─ config/db.config.js
│     ├─ models/
│     ├─ controllers/
│     └─ routes/
├─ frontend/
│  ├─ src/
│  ├─ package.json
│  ├─ Dockerfile
│  └─ nginx/
│     └─ default.conf
├─ docker-compose.yml
├─ Jenkinsfile
└─ README.md
```

---

## ⚙️ Local Development (Optional)

### 1. Prerequisites

- Node.js (LTS)
- npm
- MongoDB
- Angular CLI (`npm install -g @angular/cli`)

### 2. Backend

```bash
cd backend
npm install
npm start
```

Backend will run on: `http://localhost:8080`

### 3. Frontend

Update API base URL to relative path (already done in `tutorial.service.ts`):

```ts
const baseUrl = '/api/tutorials';
```

Run Angular dev server:

```bash
cd frontend
npm install
npm start     # ng serve
```

Frontend will run on: `http://localhost:4200`

---

## 🐳 Dockerization

### Backend Dockerfile (`backend/Dockerfile`)

```dockerfile
FROM node:18-alpine

WORKDIR /usr/src/app

COPY package*.json ./
RUN npm install --only=production

COPY . .

ENV PORT=8080
EXPOSE 8080

CMD ["node", "server.js"]
```

### Frontend Dockerfile (`frontend/Dockerfile`)

```dockerfile
FROM node:18-alpine AS build

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build --prod

FROM nginx:alpine

COPY nginx/default.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist/angular-15-crud /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Nginx Reverse Proxy (`frontend/nginx/default.conf`)

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://backend:8080/api/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

---

## 🧩 Docker Compose Setup

**File:** `docker-compose.yml`

```yaml
version: "3.9"

services:
  mongo:
    image: mongo:6
    container_name: dd-mongo
    restart: unless-stopped
    volumes:
      - mongo-data:/data/db

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    image: <your-dockerhub-username>/dd-mean-backend:latest
    container_name: dd-backend
    restart: unless-stopped
    environment:
      - MONGO_URL=mongodb://mongo:27017/dd_db
    depends_on:
      - mongo

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    image: <your-dockerhub-username>/dd-mean-frontend:latest
    container_name: dd-frontend
    restart: unless-stopped
    ports:
      - "80:80"
    depends_on:
      - backend

volumes:
  mongo-data:
```

### First-time deployment on VM

```bash
docker compose up -d --build
docker ps
```

Application will be available at:

> `http://<VM-PUBLIC-IP>/`

---

## ☁️ Cloud VM Setup

1. Create an **Ubuntu VM** on AWS / Azure / etc.
2. Open:
   - Port **22** (SSH)
   - Port **80** (HTTP)
3. Install:
   - Docker
   - Docker Compose plugin
4. Clone the GitHub repo:

```bash
cd /opt
sudo mkdir -p discoverdollar-mean-app
sudo chown $USER:$USER discoverdollar-mean-app
cd discoverdollar-mean-app

git clone https://github.com/<your-github-username>/<your-repo-name>.git .
```

5. Run:

```bash
docker compose up -d --build
```

---

## 🤖 Jenkins CI/CD Pipeline

### Jenkins Setup

- Jenkins is installed on the **same Ubuntu VM** as Docker.
- Jenkins user is added to `docker` group to allow Docker commands.
- GitHub repository is configured as the SCM source.

### Docker Hub Credentials

In Jenkins:

- Credentials ID: `dockerhub-creds`
- Type: Username + Password / Token
- Used to **login and push images** to Docker Hub.

### Jenkinsfile

```groovy
pipeline {
    agent any

    environment {
        DOCKERHUB_BACKEND = "<your-dockerhub-username>/dd-mean-backend"
        DOCKERHUB_FRONTEND = "<your-dockerhub-username>/dd-mean-frontend"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Backend Image') {
            steps {
                sh '''
                  echo "[INFO] Building backend image..."
                  docker build -t ${DOCKERHUB_BACKEND}:latest ./backend
                '''
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh '''
                  echo "[INFO] Building frontend image..."
                  docker build -t ${DOCKERHUB_FRONTEND}:latest ./frontend
                '''
            }
        }

        stage('Login to Docker Hub & Push Images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                      echo "[INFO] Logging in to Docker Hub..."
                      echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                      echo "[INFO] Pushing backend image..."
                      docker push ${DOCKERHUB_BACKEND}:latest

                      echo "[INFO] Pushing frontend image..."
                      docker push ${DOCKERHUB_FRONTEND}:latest

                      echo "[INFO] Docker logout"
                      docker logout || true
                    '''
                }
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                sh '''
                  echo "[INFO] Pulling latest images and redeploying containers..."
                  docker compose pull
                  docker compose up -d --remove-orphans
                  docker image prune -f || true
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Deployment successful!'
        }
        failure {
            echo '❌ Pipeline failed. Please check the logs.'
        }
    }
}
```

---

## 🔄 CI/CD Flow

1. Developer pushes code to **GitHub**.
2. **Jenkins** pipeline is triggered.
3. Jenkins:
   - Builds Docker images for backend + frontend
   - Pushes images to Docker Hub
   - Runs deploy using Docker Compose on VM
4. Application updates automatically.
5. App is available at:

> `http://<VM-PUBLIC-IP>/`

---

## 📸 Recommended Screenshots for Submission

- Jenkins pipeline run success
- Docker Hub images
- Docker Compose running on VM (`docker ps`)
- Application UI running in browser
- Nginx reverse proxy confirmation
- Architecture diagram (optional)

---

## ✅ Summary

- Fully containerized MEAN CRUD app  
- Automated CI/CD with Jenkins  
- Docker Hub image management  
- Deployment via Docker Compose  
- Nginx reverse proxy on port 80  
- Cloud-ready production deployment  

---

