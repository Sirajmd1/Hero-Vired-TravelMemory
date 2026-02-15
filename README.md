# TravelMemory — Production Deployment (AWS EC2 + Nginx + ALB + Cloudflare + MongoDB Atlas)

This repository documents how to deploy the TravelMemory MERN application in a production-ready way on AWS EC2, including:

- Backend (Node/Express) running on port 3000 and managed by PM2
- Nginx reverse proxy in front of the backend
- React frontend build served by Nginx on a separate EC2 instance
- Horizontal scaling using multiple EC2 instances and an Application Load Balancer (ALB)
- Custom domain configured with Cloudflare DNS
- MongoDB Atlas as the database

> ✅ Designed to meet common deployment assignment requirements: multi-instance scaling, load balancing, reverse proxy, and custom domain.

---

## Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Backend Deployment (EC2)](#backend-deployment-ec2)
- [Frontend Deployment (Separate EC2)](#frontend-deployment-separate-ec2)
- [Frontend ↔ Backend Connection](#frontend--backend-connection)
- [Scaling with Load Balancer](#scaling-with-load-balancer)
- [Cloudflare Domain Setup](#cloudflare-domain-setup)
- [Validation](#validation)
- [Troubleshooting](#troubleshooting)
- [Security Best Practices](#security-best-practices)

---

## Architecture

Recommended production layout:

```
Users
  |
  v
Cloudflare (DNS)
  |
  |  A record: sirajtm.in  -> Frontend EC2 Public IP
  v
Frontend EC2 (Nginx serves React build)
  |
  |  API calls -> https://api.sirajtm.in
  v
Backend ALB (HTTP/HTTPS)
  |
  v
Backend EC2 instances (Nginx -> Node.js on :3000, PM2)
  |
  v
MongoDB Atlas
```

---

## Prerequisites

- AWS account (EC2 + ALB permissions)
- Cloudflare-managed domain
- MongoDB Atlas cluster (connection string)
- Ubuntu 22.04 (recommended) EC2 instances
- SSH keypair (.pem)

---

## Repository Structure

```
TravelMemory/
  backend/     # Node/Express API
  frontend/    # React UI
```

---

## Backend Deployment (EC2)

### 1) Provision Backend EC2 Instances
Create 2+ EC2 instances for the backend (for scaling), Ubuntu 22.04 recommended.

Security Group (backend instances):
- Inbound 80 from ALB Security Group
- Inbound 22 from your IP (SSH)
- ❌ Do NOT expose port 3000 publicly

### 2) Install Packages
Run on each backend instance:

```bash
sudo apt update -y && sudo apt upgrade -y
sudo apt install -y git nginx
# Install Node.js (ensure node/npm available)
sudo npm install -g pm2
```

> If Node.js is not installed, install it using NodeSource or your preferred method.

### 3) Clone & Configure Backend

```bash
git clone https://github.com/UnpredictablePrashant/TravelMemory.git
cd TravelMemory/backend
npm install
nano .env
```

backend/.env

```env
MONGO_URI=mongodb+srv://admin:pwd@cluster0.c8fpquj.mongodb.net/TravelMemory
PORT=3000
```

### 4) Run Backend with PM2

```bash
pm2 start index.js --name travelmemory-api
pm2 save
pm2 startup
pm2 list
```

### 5) Configure Nginx Reverse Proxy (Backend)

Edit:

```bash
sudo nano /etc/nginx/sites-available/default
```

Use:

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://10.214.16.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_cache_bypass $http_upgrade;
    }
}
```

Restart:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

## Frontend Deployment (Separate EC2)

### 1) Provision Frontend EC2 Instances
Create 1+ EC2 instances for the frontend.

Security Group (frontend instances):
- Inbound 80 from internet (or Cloudflare IPs if locking down)
- Inbound 22 from your IP (SSH)

### 2) Install Packages

```bash
sudo apt update -y && sudo apt upgrade -y
sudo apt install -y git nginx
```

### 3) Clone, Configure, and Build React

```bash
git clone https://github.com/UnpredictablePrashant/TravelMemory.git
cd TravelMemory/frontend
npm install
```

#### Update `urls.js`
Update `frontend/src/urls.js` so the UI calls the backend correctly.

Option A (recommended for assignment): Use API subdomain

```js
export const baseUrl = process.env.REACT_APP_BACKEND_URL || "https://api.sirajtm.in";

```

Build:

```bash
npm run build
```

### 4) Copy Build to Nginx Web Root

```bash
sudo mkdir -p /var/www/travelmemory
sudo cp -r build/* /var/www/travelmemory/
```

### 5) Configure Nginx to Serve React Build

Edit:

```bash
sudo nano /etc/nginx/sites-available/default
```

Use:

```nginx
server {
  listen 80;
  server_name sirajtm.in;

  root /var/www/travelmemory;
  index index.html;

  # React SPA routing
  location / {
    try_files $uri /index.html;
  }
}
```

Restart:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

## Frontend ↔ Backend Connection

### Why this matters
React runs in the browser and must call the backend API using a reachable URL.

✅ Recommended approach:
- Frontend domain: `sirajtm.in`
- Backend API domain: `api.sirajtm.in`

Then set `BASE_URL` in `urls.js` to `https://api.sirajtm.in`.

---

## Scaling with Load Balancer

### Backend ALB (Required)
1. Create an Application Load Balancer (internet-facing)
2. Create a Target Group
   - Target type: Instances
   - Protocol: HTTP
   - Port: 80 (Nginx on backend instances)
3. Register 2+ backend EC2 instances
4. Configure health check path:
   - Use `/` if it returns 200
   - Or implement `/health` endpoint that returns 200

> Ensure backend instance security group allows inbound 80 from the ALB security group.

### Frontend Scaling (Optional / If required)
If the assignment requires frontend load balancing too:
- Add 2+ frontend EC2 instances
- Create a separate ALB and point `sirajtm.in` to it

---

## Cloudflare Domain Setup

### DNS Records
In Cloudflare DNS:

1) A Record (Frontend)
- Name: `sirajtm.in`
- Type: `A`
- Value: `<Frontend EC2 Public IP>`

2) CNAME Record (Backend ALB)
- Name: `api`
- Type: `CNAME`
- Value: `<your-alb-dns-name>.elb.amazonaws.com>`

> Tip: If using Cloudflare proxy (orange cloud), ensure ALB allows Cloudflare IP ranges.

---

## Validation

- Open `http://sirajtm.in` → React UI loads
- Confirm API works:
  - Open browser DevTools → Network
  - Verify API calls return 200
- In AWS Target Group, confirm all backend instances show Healthy

---

## Troubleshooting

### 502 Bad Gateway (Nginx)
- Backend Node not running: `pm2 list`, `pm2 logs travelmemory-api`
- Nginx config typo: `sudo nginx -t`
- Wrong proxy port: ensure backend listens on `3000`

### Target group unhealthy
- Health check path returns non-200
- Security groups not allowing ALB → EC2 port 80

### React routes 404 on refresh
- Ensure SPA routing:
  ```nginx
  location / { try_files $uri /index.html; }
  ```

---

## Security Best Practices

- Restrict SSH to your IP
- Do not expose Node port 3000 publicly
- Use HTTPS (ACM on ALB + Cloudflare Full/Strict)
- Store secrets only in `.env` and never commit them
- Keep OS packages updated

---

## License / Credits

- Application source: https://github.com/UnpredictablePrashant/TravelMemory

