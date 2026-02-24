# MEAN Stack CRUD App – Docker + Jenkins + AWS Deployment

Hi! This is my DevOps project where I took an existing MEAN stack app (MongoDB, Express, Angular, Node.js), containerized it with Docker, and deployed it on AWS EC2 with a fully automated Jenkins CI/CD pipeline and Nginx as a reverse proxy.

---

## Table of Contents

1. [What the App Does](#what-the-app-does)
2. [How Everything is Connected](#how-everything-is-connected)
3. [Project Structure](#project-structure)
4. [Changes I Made to the Original Code](#changes-i-made-to-the-original-code)
5. [Step 1 – GitHub Setup](#step-1--github-setup)
6. [Step 2 – Docker Images](#step-2--docker-images)
7. [Step 3 – AWS EC2 Setup](#step-3--aws-ec2-setup)
8. [Step 4 – Deploy with Docker Compose](#step-4--deploy-with-docker-compose)
9. [Step 5 – Jenkins CICD Pipeline](#step-5--jenkins-cicd-pipeline)
10. [Nginx Reverse Proxy](#nginx-reverse-proxy)
11. [API Reference](#api-reference)
12. [Troubleshooting](#troubleshooting)
13. [Screenshot Checklist](#screenshot-checklist)

---

## What the App Does

It is a simple Tutorial Management app where you can:
- Add a new tutorial with a title and description
- View all tutorials in a list
- Click a tutorial to see its details
- Update or delete a tutorial
- Search tutorials by title

---

## How Everything is Connected

```
Your Browser
     |
     v
  Port 80
     |
  [ Nginx ]  <-- reverse proxy running as a Docker container
     |
     |---- /api/*  --------> [ Backend ]   Node.js + Express (port 8080)
     |                              |
     |                              v
     |                         [ MongoDB ]  (port 27017)
     |
     `---- /*  ------------> [ Frontend ]  Angular served by Nginx inside container
```

All 4 services run as Docker containers on a single AWS EC2 instance, connected on a private Docker network. Only port 80 is open to the internet.

---

## Project Structure

```
crud-dd-task-mean-app/
|
|-- backend/
|   |-- app/
|   |   |-- config/
|   |   |   `-- db.config.js          [MODIFIED] reads MongoDB URL from env variable
|   |   |-- controllers/
|   |   |   `-- tutorial.controller.js
|   |   |-- models/
|   |   |   |-- index.js
|   |   |   `-- tutorial.model.js
|   |   `-- routes/
|   |       `-- turorial.routes.js
|   |-- server.js                     [MODIFIED] enabled CORS
|   |-- package.json
|   `-- Dockerfile                    [NEW] multi-stage Node.js Docker image
|
|-- frontend/
|   |-- src/
|   |   |-- app/
|   |   |   |-- components/
|   |   |   |   |-- add-tutorial/
|   |   |   |   |-- tutorial-details/
|   |   |   |   `-- tutorials-list/
|   |   |   |-- models/
|   |   |   |   `-- tutorial.model.ts
|   |   |   |-- services/
|   |   |   |   `-- tutorial.service.ts    [NEW] was missing from original
|   |   |   |-- app.component.html         [NEW] was missing from original
|   |   |   |-- app.component.ts
|   |   |   |-- app.module.ts
|   |   |   `-- app-routing.module.ts
|   |   |-- index.html                     [NEW] was missing from original
|   |   `-- styles.css
|   |-- angular.json
|   |-- package.json                       [NEW] was missing from original
|   |-- tsconfig.json
|   |-- tsconfig.app.json                  [NEW] was missing from original
|   |-- nginx.conf                         [NEW] nginx config inside the container
|   `-- Dockerfile                         [NEW] multi-stage Angular build image
|                                          [MODIFIED] pinned @types/node version fix
|
|-- nginx/
|   `-- nginx.conf                         [NEW] main reverse proxy config
|
|-- docker-compose.yml                     [NEW] orchestrates all 4 containers
|-- Jenkinsfile                            [NEW] CI/CD pipeline
|-- .gitignore
`-- README.md
```

---

## Changes I Made to the Original Code

Here is every single change I made and why.

---

### 1. backend/app/config/db.config.js

Before:
```js
module.exports = {
  url: "mongodb://localhost:27017/dd_db"
};
```

After:
```js
module.exports = {
  url: process.env.MONGODB_URL || "mongodb://localhost:27017/dd_db"
};
```

Why: When running inside Docker, the backend and MongoDB are in separate containers. Using localhost would not work. We now read the URL from an environment variable set in docker-compose.yml, pointing to the mongodb container by its service name.

---

### 2. backend/server.js

What changed: Uncommented the cors import and added app.use(cors()).

Why: The Angular frontend and Node.js backend run on different ports inside Docker. Without CORS enabled, the browser blocks API requests coming from the frontend.

---

### 3. frontend/Dockerfile

What changed: Added this line after npm install:
```dockerfile
RUN npm install @types/node@18.11.18 --save-dev
```

Why: We hit a TypeScript build error because the latest @types/node package was too new for TypeScript 4.8 (used by Angular 15). Pinning it to version 18.11.18 fixed the build error completely.

---

### 4. frontend/src/app/services/tutorial.service.ts (New file)

This file was referenced in the components but was completely missing from the original project. It handles all HTTP calls to the backend API. The base URL is set to /api/tutorials as a relative path so all requests go through Nginx, which then routes them to the backend container.

---

### 5. frontend/src/app/app.component.html (New file)

Was missing from the original. This is the main layout file with the navbar showing the Tutorials and Add links, plus the router-outlet where Angular renders page content.

---

### 6. frontend/src/index.html (New file)

Was missing from the original. This is the HTML entry point that Angular uses to bootstrap the entire app.

---

### 7. frontend/package.json (New file)

Was missing from the original. Contains all Angular 15 dependencies needed to install and build the app inside the Docker container.

---

### 8. frontend/tsconfig.app.json (New file)

Was missing from the original. Angular CLI requires this file to compile the app. Without it the build fails immediately.

---

### 9. Jenkinsfile (New file)

We went through two fixes to get this working:

Fix 1 - Removed cd /root/crud-dd-task-mean-app from the Deploy stage.
Jenkins runs as the jenkins user and cannot access /root/. The workspace already has the checked-out code so no cd is needed.

Fix 2 - Added container cleanup before deploying.
The containers we started manually earlier were still running. Jenkins was trying to create containers with the same names, causing a conflict error. We now stop and remove old containers before bringing up new ones.

---

## Step 1 - GitHub Setup

```bash
git clone https://github.com/carbo-x/crud-dd-task-mean-app.git
cd crud-dd-task-mean-app
```

<img width="1907" height="802" alt="Screenshot 2026-02-24 114348" src="https://github.com/user-attachments/assets/798cb191-4462-40f4-b4b6-d68bd7071926" />


---

## Step 2 - Docker Images

### Build the backend image

```bash
export DOCKER_USERNAME=your_dockerhub_username (my case denish136)
docker build -t $DOCKER_USERNAME/mean-backend:latest ./backend
```

### Build the frontend image (takes 3 to 5 minutes)

```bash
docker build -t $DOCKER_USERNAME/mean-frontend:latest ./frontend
```

### Login and push both images

```bash
docker login -u $DOCKER_USERNAME
docker push $DOCKER_USERNAME/mean-backend:latest
docker push $DOCKER_USERNAME/mean-frontend:latest
```

<img width="1919" height="874" alt="image" src="https://github.com/user-attachments/assets/7ed49bd7-c818-48ac-b68f-0649e80adf5d" />


### Why two Dockerfiles?

Backend Dockerfile uses a two-stage build:
- Stage 1 installs npm packages
- Stage 2 copies only what is needed into a clean Node 18 Alpine image

Frontend Dockerfile uses a two-stage build:
- Stage 1 runs npm run build to compile the Angular app
- Stage 2 copies the compiled files into a lightweight nginx:alpine image

---

## Step 3 - AWS EC2 Setup

### Instance details

- OS: Ubuntu Server 22.04 LTS
- Type: t3.small
- Security Group inbound rules:

| Port | Source | Purpose |
|---|---|---|
| 22 | Your IP | SSH access |
| 80 | 0.0.0.0/0 | Application |
| 8080 | Your IP | Jenkins dashboard |

<img width="1915" height="911" alt="Screenshot 2026-02-24 115108" src="https://github.com/user-attachments/assets/1ceb52b6-38d2-43ec-a890-d053fd885e93" />

<img width="1919" height="911" alt="Screenshot 2026-02-24 115035" src="https://github.com/user-attachments/assets/784fa055-dfdd-4897-a9a2-22aa1400a971" />


### Install Docker on EC2

```bash
sudo apt-get update -y
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker ubuntu
newgrp docker
docker --version
docker compose version
```

---

## Step 4 - Deploy with Docker Compose

### Pull images and start all 4 containers

```bash
export DOCKER_USERNAME=your_dockerhub_username
DOCKER_USERNAME=$DOCKER_USERNAME IMAGE_TAG=latest docker compose pull
DOCKER_USERNAME=$DOCKER_USERNAME IMAGE_TAG=latest docker compose up -d
```

### Verify everything is running

```bash
docker compose ps
```

Expected output:

```
NAME       IMAGE                            STATUS
backend    your-username/mean-backend:latest    Up
frontend   your-username/mean-frontend:latest   Up
mongodb    mongo:6.0                            Up (healthy)
nginx      nginx:alpine                         Up  0.0.0.0:80->80/tcp
```

<img width="1919" height="992" alt="Screenshot 2026-02-24 121816" src="https://github.com/user-attachments/assets/c4431318-3299-410e-a45b-b2487e73f0b4" />


### Test the API

```bash
curl http://localhost/api/tutorials
# Expected: []
```

<img width="1919" height="971" alt="Screenshot 2026-02-24 121945" src="https://github.com/user-attachments/assets/c5128437-c09b-4595-9f10-fdfb8feecf6a" />

<img width="1919" height="967" alt="Screenshot 2026-02-24 122736" src="https://github.com/user-attachments/assets/dcde8d84-d5cc-4518-b391-f55fb24076e9" />

<img width="1919" height="910" alt="Screenshot 2026-02-24 122822" src="https://github.com/user-attachments/assets/b3530b0d-a27d-4d22-b7b2-c9f75ed226b2" />

---

## Step 5 - Jenkins CICD Pipeline

### How the pipeline works

Every time you push code to GitHub, Jenkins automatically runs these 6 stages:

1. Checkout - pulls the latest code from GitHub
2. Docker Login - logs into Docker Hub using stored credentials
3. Build Backend Image - builds and tags mean-backend with build number and latest
4. Build Frontend Image - builds and tags mean-frontend with build number and latest
5. Push Images - pushes both images to Docker Hub
6. Deploy - stops old containers, pulls new images, starts fresh containers

<img width="1919" height="972" alt="Screenshot 2026-02-24 121720" src="https://github.com/user-attachments/assets/967bbe74-5544-48c5-b3b6-cf00cc1443cb" />


### Jenkins credentials to add

Go to: Dashboard > Manage Jenkins > Credentials > System > Global credentials > Add Credentials

| Credential ID | Kind | Value |
|---|---|---|
| docker-hub-username | Secret text | Your Docker Hub username |
| docker-hub-password | Secret text | Your Docker Hub password or access token |
| github-credentials | Username with password | GitHub username + Personal Access Token |

Note: GitHub no longer accepts your account password for Git operations. Create a Personal Access Token at https://github.com/settings/tokens and use that as the password. Tick the repo scope when creating it.

<img width="1919" height="962" alt="Screenshot 2026-02-24 120533" src="https://github.com/user-attachments/assets/5d12f601-8e47-4ac2-9926-5ba9de0823b2" />


### Create the Pipeline job

1. Click New Item, give it a name like mean-app-pipeline, select Pipeline, click OK
2. Scroll to the Pipeline section and fill in:
   - Definition: Pipeline script from SCM
   - SCM: Git
   - Repository URL: https://github.com/your-username/crud-dd-task-mean-app.git
   - Credentials: github-credentials
   - Branch: */main
   - Script Path: Jenkinsfile
3. Click Save then Build Now

<img width="1919" height="970" alt="image" src="https://github.com/user-attachments/assets/8246a703-a03d-43b8-9791-63cc47899b93" />


<img width="1919" height="980" alt="Screenshot 2026-02-24 121514" src="https://github.com/user-attachments/assets/04bb6560-b44a-4f39-939d-751ac1c73b39" />

### Auto-trigger on every GitHub push (Webhook)

1. In your GitHub repo go to Settings > Webhooks > Add webhook
2. Payload URL: http://your-ec2-ip:8080/github-webhook/
3. Content type: application/json
4. Events: select Just the push event
5. In Jenkins job go to Configure and enable GitHub hook trigger for GITScm polling

After this every git push automatically triggers a full build and deployment.

---

## Nginx Reverse Proxy

Nginx is the single entry point for all traffic. It listens on port 80 and routes requests based on the URL path.

File: nginx/nginx.conf

```
/api/*  -->  backend container on port 8080    (Node.js REST API)
/*      -->  frontend container on port 80     (Angular SPA)
```

The frontend container also has its own nginx.conf at frontend/nginx.conf. This one handles Angular client-side routing using try_files so that refreshing the page or navigating directly to a route does not return a 404 error.

<img width="1854" height="223" alt="Screenshot 2026-02-24 122517" src="https://github.com/user-attachments/assets/04b93532-16c9-4773-af47-e535a9c1cd2b" />


---

## API Reference

Base path: /api/tutorials

| Method | URL | What it does |
|---|---|---|
| GET | /api/tutorials | Get all tutorials |
| GET | /api/tutorials/:id | Get one tutorial by ID |
| POST | /api/tutorials | Create a new tutorial |
| PUT | /api/tutorials/:id | Update a tutorial |
| DELETE | /api/tutorials/:id | Delete one tutorial |
| DELETE | /api/tutorials | Delete all tutorials |
| GET | /api/tutorials/published | Get all published tutorials |
| GET | /api/tutorials?title=xyz | Search by title |

---

## Troubleshooting

Check which containers are running:
```bash
docker compose ps
```

View logs for a specific service:
```bash
docker compose logs -f backend
docker compose logs -f nginx
docker compose logs -f mongodb
```

Restart a single container:
```bash
docker compose restart backend
```

Backend cannot connect to MongoDB:
Make sure MONGODB_URL is set to mongodb://mongodb:27017/dd_db in docker-compose.yml. The hostname mongodb must exactly match the service name, Docker uses this for internal DNS resolution between containers.

Frontend build fails with TypeScript errors:
This is a @types/node version mismatch with Angular 15. The fix is already included in the Dockerfile:
```dockerfile
RUN npm install @types/node@18.11.18 --save-dev
```

Jenkins Deploy stage fails with container name already in use:
Old containers are still running from a previous manual deployment. The Jenkinsfile now handles this automatically. If it still happens, manually run:
```bash
docker compose down
```

Port 80 not accessible from browser:
Check your EC2 Security Group and make sure port 80 is open for 0.0.0.0/0 in the inbound rules.

---

## Screenshot Checklist

Here is every place in this README where you need to add a screenshot:

- [ ] GitHub repo showing all files and folders
- [ ] Docker Hub showing both mean-backend and mean-frontend images with tags
- [ ] AWS EC2 console showing the running instance
- [ ] AWS Security Group inbound rules
- [ ] Terminal showing docker compose ps with all 4 containers Up
- [ ] Browser showing the app at http://your-ec2-ip
- [ ] Browser showing Tutorials List page with a tutorial added
- [ ] Jenkins credentials page with all 3 credentials
- [ ] Jenkins pipeline job configuration page
- [ ] Jenkins Stage View showing all 6 stages in green
- [ ] Jenkins console output showing Finished: SUCCESS
- [ ] Terminal showing curl returning JSON from the API
