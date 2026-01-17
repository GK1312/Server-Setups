# Complete Guide: Deploy Multiple Applications with Cloudflare Tunnels

A step-by-step guide to deploying portfolio website and XChangeHub on a 1GB DigitalOcean droplet using Docker and Cloudflare Tunnels.

**Working Setup:**

- 🌐 **Portfolio**: portfolio.gkverse.in (Next.js on port 4000)
- 🌐 **XChangeHub**: xchangehub.meelabs.site (React + Nginx on port 3000)
- 🚇 **Two Cloudflare Tunnels**: Separate tunnels for each domain
- 🐳 **Unified Docker Compose**: Single configuration managing all services

---

## Table of Contents

1. [Server Setup & Security](#1-server-setup--security)
2. [Install Required Dependencies](#2-install-required-dependencies)
3. [Configure Cloudflare Tunnels](#3-configure-cloudflare-tunnels)
4. [Setup Docker & Docker Compose](#4-setup-docker--docker-compose)
5. [Deploy Your Applications](#5-deploy-your-applications)
6. [Troubleshooting](#6-troubleshooting)

---

## 1. Server Setup & Security

### 1.1 Create SSH Key Pair

**Command:**

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

**Why:** SSH keys provide a more secure authentication method than passwords. They use public-key cryptography where you keep the private key and share only the public key with servers.

**What happens:** This generates two files:

- Private key (`~/.ssh/id_ed25519`) - stays on your local machine
- Public key (`~/.ssh/id_ed25519.pub`) - added to servers you want to access

### 1.2 Add SSH Key to DigitalOcean

**Command:**

```bash
cat ~/.ssh/id_ed25519.pub
```

**Why:** By adding your public key to DigitalOcean before creating the droplet, you can access the server immediately without password authentication.

**What happens:** Copy the output and add it in DigitalOcean's "Settings → Security → SSH Keys" section. When you create a new droplet, select this SSH key.

### 1.3 Connect to Your Server

**Command:**

```bash
ssh root@YOUR_SERVER_IP
```

**Why:** Initial connection is made as root to perform system-level configurations and create a non-root user.

**What happens:** You'll connect to your server for the first time. You may see a fingerprint confirmation - type "yes" to continue.

### 1.4 Create a Non-Root User

**Command:**

```bash
adduser deploy
```

**Why:** Running services as root is a security risk. If an attacker compromises your application, they have full system access. A non-root user limits potential damage.

**What happens:** You'll be prompted to set a password and optional user details. This creates a new user account named "deploy".

### 1.5 Grant Sudo Privileges

**Command:**

```bash
usermod -aG sudo deploy
```

**Why:** The new user needs sudo privileges to perform administrative tasks when necessary, but operates as a regular user by default.

**What happens:** Adds the "deploy" user to the "sudo" group, allowing them to run commands with elevated privileges using `sudo`.

### 1.6 Copy SSH Keys to New User

**Command:**

```bash
rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy
```

**Why:** The new user needs the authorized SSH keys to login. Without this, you won't be able to SSH as the new user.

**What happens:** Copies the `.ssh` directory from root to the new user's home directory with correct ownership and permissions.

### 1.7 Disable Root Login

**Command:**

```bash
sudo nano /etc/ssh/sshd_config
```

**Configuration changes:**

```
PermitRootLogin no
PasswordAuthentication no
```

**Why:** Disabling root login and password authentication prevents brute-force attacks and unauthorized access. Attackers commonly target the root account.

**What happens:** After saving, SSH will only accept key-based authentication for non-root users, significantly improving security.

### 1.8 Restart SSH Service

**Command:**

```bash
sudo systemctl restart ssh
```

**Why:** SSH configuration changes only take effect after the service restarts.

**What happens:** The SSH daemon reloads with new security settings. Your current session stays active, but new connections must use the new rules.

### 1.9 Test New User Login

**Command:**

```bash
exit
ssh deploy@YOUR_SERVER_IP
```

**Why:** Always verify the new user can login before closing your root session. This prevents lockouts.

**What happens:** You'll disconnect from the server and reconnect as the "deploy" user using SSH key authentication.

---

## 2. Install Required Dependencies

### 2.1 Update Package Lists

**Command:**

```bash
sudo apt update
```

**Why:** Ensures you're installing the latest versions of packages and security updates.

**What happens:** Refreshes the package index from Ubuntu repositories but doesn't install anything yet.

### 2.2 Install Essential Packages

**Command:**

```bash
sudo apt install -y nginx git curl
```

**Why:**

- **nginx**: Web server and reverse proxy for serving applications
- **git**: Version control to clone repositories
- **curl**: Tool for downloading files and making HTTP requests

**What happens:** Installs these three essential packages needed for web development and deployment.

### 2.3 Install Node.js

**Commands:**

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt-get install -y nodejs
```

**Why:** Node.js is required to run JavaScript applications on the server. The LTS (Long Term Support) version ensures stability.

**What happens:** Adds NodeSource repository, then installs Node.js and npm (Node Package Manager).

**Verify installation:**

```bash
node --version
npm --version
```

### 2.4 Setup Nginx

**Commands:**

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

**Why:**

- `start`: Immediately starts the nginx service
- `enable`: Ensures nginx starts automatically on system boot

**What happens:** Nginx starts serving on port 80. Visit `http://YOUR_SERVER_IP` to see the welcome page.

---

## 3. Configure Cloudflare Tunnels

### 3.1 Why Use Cloudflare Tunnel?

**Benefits:**

- **No port forwarding required**: Securely expose applications without opening firewall ports
- **DDoS protection**: Cloudflare's network protects your origin server
- **SSL/TLS encryption**: Automatic HTTPS for all domains
- **Multiple applications**: Route multiple domains/subdomains to different ports
- **Hide origin IP**: Your server's IP is never exposed publicly

**Our Setup:**

- **Tunnel 1 (gkverse)**: Routes portfolio.gkverse.in → port 4000
- **Tunnel 2 (meelabs)**: Routes xchangehub.meelabs.site → port 3000

### 3.2 Install Cloudflared

**Commands:**

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update
sudo apt install cloudflared
```

**Why:** This adds Cloudflare's official repository and installs the cloudflared daemon, which creates secure tunnels to Cloudflare's network.

**What happens:** Cloudflared is installed system-wide and ready to create tunnels.

**Verify installation:**

```bash
cloudflared --version
```

### 3.3 Authenticate with Cloudflare

**Command:**

```bash
cloudflared tunnel login
```

**Why:** Authenticates your server with your Cloudflare account to manage tunnels and DNS records.

**What happens:** Opens a browser link. Login to Cloudflare and select the domain you want to use. A certificate file is saved to `~/.cloudflared/cert.pem`.

### 3.4 Create Tunnels

**Commands:**

```bash
# Create first tunnel for portfolio
cloudflared tunnel create gkverse

# Create second tunnel for XChangeHub
cloudflared tunnel create meelabs
```

**Why:** We use separate tunnels for different domains to isolate traffic and improve organization.

**What happens:** Generates two credentials files at `~/.cloudflared/<TUNNEL_ID>.json`. **Save both tunnel IDs** - you'll need them later.

**Example output:**

```
Created tunnel gkverse with id f523b141-d0c3-4dd4-afc7-6a2b4eec7f57
Created tunnel meelabs with id 20810283-2c55-4d4c-b5fa-51f3af64f137
```

# Route portfolio domain to gkverse tunnel

cloudflared tunnel route dns gkverse portfolio.gkverse.in

# Route XChangeHub domain to meelabs tunnel

cloudflared tunnel route dns meelabs xchangehub.meelabs.site

```

**Why:** Creates CNAME records in Cloudflare DNS that point your domains to the respective tunnels.

**What happens:** Each command creates a DNS record. Visit Cloudflare dashboard → DNS to verify the CNAME records were created pointing to:
- `f523b141-d0c3-4dd4-afc7-6a2b4eec7f57.cfargotunnel.com`
- `20810283-2c55-4d4c-b5fa-51f3af64f137.cfargotunnel.com`
```

**Why:** Creates CNAME records in Cloudflare DNS that point your domains to the tunnel.

**What happens:** Each command creates a DNS record. Visit Cloudflare dashboard → DNS to verify the CNAME records were created.

### 3.6 Configure Tunnel Routing

# Create cloudflared config directory

sudo mkdir -p /etc/cloudflared
sudo chown deploy:deploy /etc/cloudflared

# Create config for gkverse tunnel

sudo nano /etc/cloudflared/gkverse-config.yml

````

**gkverse-config.yml:**

```yaml
tunnel: f523b141-d0c3-4dd4-afc7-6a2b4eec7f57
credentials-file: /home/nonroot/.cloudflared/f523b141-d0c3-4dd4-afc7-6a2b4eec7f57.json

# Connection settings for better performance
protocol: quic
no-tls-verify: false

ingress:
  # Portfolio Website
  - hostname: portfolio.gkverse.in
    service: http://gkverse-portfolio:4000
    originRequest:
      connectTimeout: 30s
      noTLSVerify: false
      disableChunkedEncoding: false

  # Catch-all rule (required)
  - service: http_status:404
````

**Create config for meelabs tunnel:**

```bash
sudo nano /etc/cloudflared/meelabs-config.yml
```

**meelabs-config.yml:**

````yaml
tunnel: 20810283-2c55-4d4c-b5fa-51f3af64f137
credentials-file: /home/nonroot/.cloudflared/20810283-2c55-4d4c-b5fa-51f3af64f137.json

# Connection settings for better performance
protocol: quic
no-tls-verify: false

ingress:
  # XChangeHub Frontend
  - hostname: xchangehub.meelabs.site
    service: http://xchangehub-frontend:3000
    originRequest:
      connectTimeout: 30s
      noTLSVerify: false
      disableChunkedEncoding: false
Add Swap Space (Critical for 1GB RAM)

**⚠️ IMPORTANT:** Building Next.js apps on a 1GB RAM server will fail without swap space.

**Commands:**

```bash
# Create 2GB swap file
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make swap permanent (survives reboots)
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Verify swap is active
free -h
````

**Why:** Next.js builds are memory-intensive. Without swap, the build process will hang/timeout on low-memory servers.

**What happens:** Your server now has 3GB total memory (1GB RAM + 2GB swap), enough for building modern JavaScript applications.

### 4.2 Why Use Docker?

**Benefits:**

- **Isolation**: Each application runs in its own container with dependencies
- **Consistency**: Same environment in development and production
- **Easy updates**: Pull new images and restart containers
- **Resource efficiency**: Lightweight compared to virtual machines
- **Multiple apps**: Run multiple applications on same server without conflicts
  4

### 4.3entials-file`: Path to tunnel credentials (inside Docker container)

- `protocol: quic`: Uses QUIC for faster, more reliable connections
- `service`: Points to Docker container name and port
- `originRequest`: Optimized timeout and encoding settings
- Last rule (`http_status:404`): Catch-all for unmatched requests

**What happens:** When someone visits the domain, Cloudflare routes through the tunnel to the specified container and port

- `ingress`: List of routing rules (order matters - first match wins)
- Last rule (`http_status:404`): Catch-all for unmatched requests

**What happens:** When someone visits `portfolio.gkverse.in`, Cloudflare routes the request through the tunnel to your server's port 4000.

### 3.7 Fix Credentials File Permissions

**Command:**

```bash
chmod 644 ~/.cloudflared/*.json
```

**Why:** The cloudflared container runs as a non-root user and needs read permission for the credentials file.

**What happens:** Makes the credentials file readable by all users while keeping it write-protected.

---

## 4. Setup Docker & Docker Compose

### 4.1 Why Use Docker?

**Benefits:**

- **Isolation**: Each application runs in its own container with dependencies
- **Consistency**: Same environment in development and production
- **Easy updates**: Pull new images and restart containers
- **Resource efficiency**: Lightweight compared to virtual machines
- **Multiple apps**: Run multiple applications on same server without conflicts

### 4.2 Install Docker

**Commands:**

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker deploy
```

**Why:**

- First command: Official Docker installation script
- Second command: Adds your user to docker group to run docker without sudo

**What happens:** Docker engine is installed and your user can now manage containers.

**Important:** Log out and back in for group changes to take effect:

```bash
exit
ssh deploy@YOUR_SERVER_IPs
```

### 4.3 Install Docker Compose Plugin

**Command:**

# Create deployment directory

cd ~
mkdir -p app/gkverse/portfolio
mkdir -p app/meelabs/XChangeHub/frontend
mkdir -p etc/cloudflared

# Clone your repositories

cd ~/app/gkverse/portfolio
git clone YOUR_PORTFOLIO_REPO_URL .

cd ~/app/meelabs/XChangeHub/frontend
git clone YOUR_XCHANGEHUB_REPO_URL .

cd ~

```

**Why:** Organized directory structure matches the docker-compose configuration and makes it easier to manage multiple applications.

**Expected structure:**
```

~
├── app/
│ ├── gkverse/
│ │ └── portfolio/ (Next.js app)
│ └── meelabs/
│ └── XChangeHub/
│ └── frontend/ (React app)
└── etc/
└── cloudflared/
├── gkverse-config.yml
└── meelabs-config.yml

````

**What happens:** Createss

#### Portfolio Dockerfile (Next.js)

**Command:**

```bash
nano ~/app/gkverse/portfolio/Dockerfile
````

**Dockerfile content:**

```dockerfile
FROM node:24-alpine AS build
WORKDIR /app

COPY package*.json yarn.lock ./
RUN corepack enable
RUN yarn install --frozen-lockfile

COPY . .
RUN yarn run build

FROM node:24-alpine
WORKDIR /app

COPY --from=build /app .
ENV NODE_ENV=production

EXPOSE 4000
CMD ["npm", "start"]
```

**Why:** Multi-stage build reduces final image size. Next.js runs on Node.js so we keep the runtime.

#### XChangeHub Dockerfile (React + Nginx)

**Command:**

````bashUnified Docker Compose Configuration

**Command:**

```bash
cd ~
nano docker-compose.yml
````

**Docker Compose content:**

````yaml
networks:
  internal:
    driver: bridge

services:
  # Portfolio Service (Next.js)
  gkverse-portfolio:
    build: ./app/gkverse/portfolio
    container_name: gkverse-portfolio
    restart: always
    networks:
      - internal
    expose:
      - "4000"
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:4000"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # XChangeHub Frontend Service (React + Nginx)
  xchangehub-frontend:
    build: ./app/meelabs/XChangeHub/frontend
    container_name: xchangehub-frontend
    restart: always
    networks:
      - internal
    expose:
      - "3000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # Cloudflared Tunnel for gkverse
  cloudflared-gkverse:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared-gkverse
    command: tunnel --config /etc/cloudflared/gkverse-config.yml run
    networks:
      - internal
    volumes:
      - /home/deploy/.cloudflared/f523b141-d0c3-4dd4-afc7-6a2b4eec7f57.json:/home/nonroot/.cloudflared/f523b141-d0c3-4dd4-afc7-6a2b4eec7f57.json:ro
      - /etc/cloudflared/gkverse-config.yml:/etc/cloudflared/gkverse-config.yml:ro
    restart: unless-stopped
    depends_on:
      gkverse-portfolio:
        condition: service_healthy
All Containers

**Command:**

```bash
cd ~
docker compose up -d --build
````

**Flags explained:**

- `up`: Creates and starts containers
- `-d`: Detached mode (runs in background)
- `--build`: Forces rebuild of images

**What happens:**

1. Builds portfolio image (~2-3 minutes)
2. Builds XChangeHub image (~5-10 minutes with swap)
3. Starts all 4 containers
4. Waits for health checks to pass
5. Starts tunnel connections

**Expected build time:** 10-15 minutes total on a 1GB server with swap.

**⚠️ If build gets stuck:** The build may appear frozen during Next.js compilation. This is normal with limited RAM. Wait patiently or check logs:

```bash
# Watch build progress in another terminal
docker compose logs -f gkverse-portfolio
docker compose logs -f xchangehub-fronten exposes ports to internal network (more secure)
- **volumes**: Mounts configs from `/etc/cloudflared/` (absolute paths prevent mounting directories)
- **depends_on with condition**: Tunnels wait for services to be healthy
- **restart policies**: Automatically restarts containers on failure

**What happens:** Single command deploys everything - both apps and both tunnels
```

**nginx.conf content:**

```nginx
server {
    listen 3000;
    server_name _;

    root /usr/share/nginx/html;
# Check all containers
docker compose ps

# Should show 4 containers:
# gkverse-portfolio      Up (healthy)
# xchangehub-frontend    Up (healthy)
# cloudflared-gkverse    Up
# cloudflared-meelabs    Up

# Check individual logs
docker compose logs gkverse-portfolio
docker compose logs xchangehub-frontend
docker compose logs cloudflared-gkverse
docker compose logs cloudflared-meelabs

# Follow all logs in real-time
docker compose logs -f
```

**Why:** Monitor container health and troubleshoot issues.

**What happens:**s

Visit your domains in a browser:

- 🌐 **Portfolio**: `https://portfolio.gkverse.in`
- 🌐 **XChangeHub**: `https://xchangehub.meelabs.site`

**What happens:**

1. Browser connects to Cloudflare's edge network
2. Cloudflare identifies which tunnel to use based on hostname
3. Routes through the appropriate tunnel to your server
4. Cloudflared forwards to the correct container (port 4000 or 3000)
5. Application responds with content
6. Response travels back through tunnel with SSL encryption

**DNS Propagation:** May take a few minutes for DNS changes to propagate globally.
INF Registered tunnel connection connIndex=3

```

**Successful health check:**
- Portfolio: `wget` can connect to port 4000
- XChangeHub: `curl` can connect to port 3000 # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
}
```

**Why:** Configures nginx to listen on port 3000 and handle React Router properly.

**What happens:** Both Dockerfiles define how to build and run each application
WORKDIR /app

COPY package\*.json yarn.lock ./
RUN corepack enable
RUN yarn install --frozen-lockfile

COPY . .
RUN yarn run build

FROM node:24-alpine
WORKDIR /app

COPY --from=build /app .
ENV NODE_ENV=pError 1033 - Cloudflare Tunnel Error

**Symptoms:** Website shows "Error 1033 - Cloudflare Tunnel error"

**Possible causes:**

- Services not running
- Services unhealthy (health checks failing)
- Wrong service name in tunnel config

**Solution:**

```bash
# Check container status
docker compose ps

# Check which services are unhealthy
docker compose logs gkverse-portfolio
docker compose logs xchangehub-frontend

# Common fixes:
# 1. Restart unhealthy service
docker compose restart xchangehub-frontend

# 2. Verify tunnels can reach services
docker exec cloudflared-gkverse wget -O- http://gkverse-portfolio:4000
docker exec cloudflared-meelabs curl http://xchangehub-frontend:3000
```

#### Issue 2: Build Gets Stuck / Times Out

**Symptoms:** Build freezes during `RUN yarn run build` or takes forever

**Cause:** Insufficient memory on 1GB RAM server

**Solution:**

```bash
# Add swap space if not already done
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Verify swap is active
free -h

# Retry build
docker compose down
docker compose up -d --build
```

#### Issue 3: Container Shows "Unhealthy"

**Symptoms:** `docker compose ps` shows container as "unhealthy"

**For XChangeHub (nginx):**

```bash
# Test if nginx is responding
docker exec xchangehub-frontend curl -I http://localhost:3000

# Check if curl is available (health check uses curl)
docker exec xchangehub-frontend which curl

# If curl not found, health check will fail
# Solution: Use curl in health check (not wget)
```

**For Portfolio (Next.js):**

```bash
# Test if app is responding
docker exec gkverse-portfolio wget -O- http://localhost:4000

# Check app logs
docker compose logs gkverse-portfolio
```

#### Issue 4: Cloudflared "read is a directory" Error

**Error:** `error parsing YAML in config file: read /etc/cloudflared/gkverse-config.yml: is a directory`

**Cause:** Using relative path (`./etc/cloudflared/...`) which Docker mounts as a directory

**Solution:** Use absolute paths in docker-compose.yml:

```yaml
volumes:
  - /etc/cloudflared/gkverse-config.yml:/etc/cloudflared/gkverse-config.yml:ro
  # NOT: ./etc/cloudflared/gkverse-config.yml:...
```

#### Issue 5: "Connection Refused" in Health Check

**Symptoms:** Container logs show `wget: can't connect to remote host: Connection refused`

**Possible causes:**

- App not listening on expected port
- Wrong tool in health check (wget vs curl)
- App crashed or still starting

**Solution:**

```bash
# Check what's actually listening
docker exec xchangehub-frontend netstat -tulpn

# Verify nginx config
docker exec xchangehub-frontend cat /etc/nginx/conf.d/default.conf

# Check if port matches health check
# nginx.conf: listen 3000
# health check: http://localhost:3000
```

#### Issue 6: Permission Denied (Credentials File)

**Error:** `couldn't read tunnel credentials: permission denied`

**Solution:**

```bash
chmod 644 ~/.cloudflared/*.json
docker compose restart cloudflared-gkverse
docker compose restart cloudflared-meelabs
```

#### Issue 7: DNS Not Resolving

**Symptoms:** Domain doesn't resolve or shows Cloudflare parking page

**Solution:**

````bash
# Verify DNS records in Cloudflare dashboard
# Should see CNAME records:
# portfolio.gkverse.in → f523b141-d0c3-4dd4-afc7-6a2b4eec7f57.cfargotunnel.com
# xchangehub.meelabs.site → 20810283-2c55-4d4c-b5fa-51f3af64f137.cfargotunnel.com

# Re-create routes if needed
cloudflared tunnel route dns gkverse portfolio.gkverse.in
cloudflared tunnel route dns meelabs xchangehub.meelabs.site
**What happens:** Docker Compose orchestrates both your app and cloudflared tunnel in a single network.

### 5.4 Handle Project Structure Issues

**If your repository has nested folders:**

```bash
mv Portfolio/* .
mv Portfolio/.* . 2>/dev/null
rmdir Portfoliostatus
docker compose ps

# View all containers (including stopped)
docker ps -a

# Stop all containers
docker compose down

# Restart specific service
docker compose restart xchangehub-frontend

# Rebuild and restart
docker compose up -d --build

# Rebuild only one service
docker compose up -d --build gkverse-portfolio

# View logs (all services)
docker compose logs -f

# View logs (specific service)
docker compose logs -f cloudflared-gkverse

# Execute command in container
docker exec -it gkverse-portfolio sh
docker exec -it xchangehub-frontend sh

# Check container health
docker inspect gkverse-portfolio --format='{{json .State.Health}}' | jq

# Remove all stopped containers
docker container prune

# Remove unused images
docker image prune -a

# Remove everything and start fresh
docker compose down
docker system prune -a
docker compose up -d --build
````

### Update Your Applications

**Update Portfolio:**

```bash
cd ~/app/gkverse/portfolio
git pull
cd ~
docker compose up -d --build gkverse-portfolio
```

**Update XChangeHub:**

```bash
cd ~/app/meelabs/XChangeHub/frontend
git pull
cd ~
docker compose up -d --build xchangehub-frontend
```

**Update Both:**

```bash
cd ~/app/gkverse/portfolio && git pull
cd ~/app/meelabs/XChangeHub/frontend && git pull
cd ~epack enable
yarn install
git add yarn.lock
git commit -m "fix: update yarn.lock"
git push
docker compose down
docker compose up -d --build
```

### 5.6 Test Your Application

**Command:**

```bash
curl http://localhost:4000
```

**Why:** Verifies your application is running and responding on the correct port.

\*\*Wh2GB swap space for building on 1GB RAM server

- ✅ Docker environment for containerized applications
- ✅ **Two Cloudflare Tunnels** routing to separate applications
- ✅ **Unified Docker Compose** managing 4 containers
- ✅ Health checks ensuring services are ready before tunnels connect
- ✅ Automated container restarts and monitoring
- ✅ **Two working domains:**
  - 🌐 portfolio.gkverse.in (Next.js on port 4000)
  - 🌐 xchangehub.meelabs.site (React + Nginx on port 3000)

Your applications are now accessible via HTTPS with Cloudflare's DDoS protection, without exposing any ports directly to the internet!

---

## Quick Reference

**Start all services:**

```bash
cd ~ && docker compose up -d
```

**Stop all services:**

```bash
docker compose down
```

**Check status:**

```bash
docker compose ps
docker compose logs -f
```

**Update applications:**

```bash
cd ~/app/gkverse/portfolio && git pull
cd ~/app/meelabs/XChangeHub/frontend && git pull
cd ~ && docker compose up -d --build
```

**Tunnel IDs:**

- gkverse: `f523b141-d0c3-4dd4-afc7-6a2b4eec7f57`
- meelabs: `20810283-2c55-4d4c-b5fa-51f3af64f137`

```bash
docker ps
docker logs gkverse-portfolio
docker logs cloudflared
```

**Why:** Monitor container health and troubleshoot issues.

**What happens:**

- `docker ps`: Shows running containers
- `docker logs`: Displays container output and errors

**Successful cloudflared logs should show:**

```
INF Registered tunnel connection connIndex=0
INF Registered tunnel connection connIndex=1
INF Registered tunnel connection connIndex=2
INF Registered tunnel connection connIndex=3
```

### 5.8 Access Your Application

Visit your domain in a browser: `https://portfolio.gkverse.in`

**What happens:**

1. Browser connects to Cloudflare's edge network
2. Cloudflare routes through the tunnel to your server
3. Cloudflared forwards to your container on port 4000
4. Your application responds with content
5. Response travels back through tunnel with SSL encryption

---

## 6. Troubleshooting

### Common Issues and Solutions

#### Issue 1: 502 Bad Gateway

**Possible causes:**

- Cloudflared not running
- Wrong service name in config.yml
- Application not running on specified port

**Solution:**

```bash
docker ps  # Check if both containers are running
docker logs cloudflared  # Check tunnel status
docker logs gkverse-portfolio  # Check app logs
curl http://localhost:4000  # Test app locally
```

#### Issue 2: Permission Denied (Credentials File)

**Error:** `couldn't read tunnel credentials: permission denied`

**Solution:**

```bash
chmod 644 ~/.cloudflared/*.json
docker compose restart cloudflared
```

#### Issue 3: Container Name Mismatch

**Error:** Service can't connect to `portfolio` container

**Solution:** Ensure service name in `/etc/cloudflared/config.yml` matches container name:

```yaml
service: http://gkverse-portfolio:4000 # Must match container_name in docker-compose.yml
```

#### Issue 4: Tunnel Not Registering

**Symptoms:** Tunnel starts but doesn't register connections

**Solution:**

```bash
# Verify tunnel command
docker compose down
nano docker-compose.yml  # Ensure command is: tunnel --config /etc/cloudflared/config.yml run
docker compose up -d
```

#### Issue 5: Port Already in Use

**Error:** `port is already allocated`

**Solution:**

```bash
sudo lsof -i :4000  # Find what's using the port
docker compose down
docker compose up -d
```

### Useful Docker Commands

```bash
# View all containers (including stopped)
docker ps -a

# Stop all containers
docker compose down

# Rebuild and restart
docker compose up -d --build

# View logs
docker logs <container_name> -f

# Execute command in container
docker exec -it gkverse-portfolio sh

# Remove all stopped containers
docker container prune

# Remove unused images
docker image prune -a
```

### Update Your Application

```bash
cd ~/apps/gkverse/portfolio
git pull
docker compose down
docker compose up -d --build
```

---

## Security Best Practices

1. **Regular Updates**: Keep system and packages updated

   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. **Firewall Configuration**: Use UFW to restrict access

   ```bash
   sudo ufw default deny incoming
   sudo ufw default allow outgoing
   sudo ufw allow ssh
   sudo ufw enable
   ```

3. **Monitor Logs**: Regularly check application and system logs

   ```bash
   docker logs cloudflared
   sudo journalctl -u ssh
   ```

4. **Backup Credentials**: Store tunnel credentials safely
   - Backup `~/.cloudflared/` directory
   - Store tunnel ID securely

5. **Container Updates**: Regularly update container images
   ```bash
   docker compose pull
   docker compose up -d
   ```

---

## Additional Resources

- [Docker Documentation](https://docs.docker.com/)
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [DigitalOcean Documentation](https://docs.digitalocean.com/)
- [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices)

---

## Summary

You now have:

- ✅ Secure DigitalOcean droplet with SSH key authentication
- ✅ Non-root user with sudo privileges
- ✅ Docker environment for containerized applications
- ✅ Cloudflare Tunnel for secure, SSL-encrypted domain routing
- ✅ Automated container restarts and monitoring
- ✅ Ability to deploy multiple applications on different domains

Your applications are now accessible via HTTPS with Cloudflare's DDoS protection, without exposing any ports directly to the internet!
