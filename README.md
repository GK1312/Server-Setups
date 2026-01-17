# Complete Guide: Deploy Applications on DigitalOcean with Cloudflare Tunnel

A comprehensive guide to setting up a secure DigitalOcean droplet and deploying multiple applications using Docker and Cloudflare Tunnel.

---

## Table of Contents
1. [Server Setup & Security](#1-server-setup--security)
2. [Install Required Dependencies](#2-install-required-dependencies)
3. [Configure Cloudflare Tunnel](#3-configure-cloudflare-tunnel)
4. [Setup Docker & Docker Compose](#4-setup-docker--docker-compose)
5. [Deploy Your Application](#5-deploy-your-application)
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

## 3. Configure Cloudflare Tunnel

### 3.1 Why Use Cloudflare Tunnel?

**Benefits:**
- **No port forwarding required**: Securely expose applications without opening firewall ports
- **DDoS protection**: Cloudflare's network protects your origin server
- **SSL/TLS encryption**: Automatic HTTPS for all domains
- **Multiple applications**: Route multiple domains/subdomains to different ports
- **Hide origin IP**: Your server's IP is never exposed publicly

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

### 3.4 Create a Tunnel

**Command:**
```bash
cloudflared tunnel create gkverse
```

**Why:** Creates a persistent named tunnel that can route traffic to your server. The tunnel has a unique ID.

**What happens:** Generates credentials file at `~/.cloudflared/<TUNNEL_ID>.json`. **Save this tunnel ID** - you'll need it later.

**Example output:**
```
Created tunnel gkverse with id f523b141-d0c3-4dd4-afc7-6a2b4eec7f57
```

### 3.5 Create DNS Routes

**Commands:**
```bash
cloudflared tunnel route dns gkverse portfolio.gkverse.in
cloudflared tunnel route dns gkverse api.gkverse.in
cloudflared tunnel route dns gkverse blog.gkverse.in
```

**Why:** Creates CNAME records in Cloudflare DNS that point your domains to the tunnel.

**What happens:** Each command creates a DNS record. Visit Cloudflare dashboard → DNS to verify the CNAME records were created.

### 3.6 Configure Tunnel Routing

**Commands:**
```bash
sudo mkdir -p /etc/cloudflared
sudo chown deploy:deploy /etc/cloudflared
sudo nano /etc/cloudflared/config.yml
```

**Configuration file:**
```yaml
tunnel: f523b141-d0c3-4dd4-afc7-6a2b4eec7f57
credentials-file: /home/nonroot/.cloudflared/f523b141-d0c3-4dd4-afc7-6a2b4eec7f57.json

ingress:
  - hostname: portfolio.gkverse.in
    service: http://gkverse-portfolio:4000
  
  - hostname: api.gkverse.in
    service: http://localhost:5000
  
  - hostname: blog.gkverse.in
    service: http://localhost:80
  
  - service: http_status:404
```

**Why:** This configuration maps your domains to specific local services/ports. Cloudflare tunnel reads this to route incoming traffic.

**Configuration explanation:**
- `tunnel`: Your tunnel ID from step 3.4
- `credentials-file`: Path to tunnel credentials (inside Docker container)
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
ssh deploy@YOUR_SERVER_IP
```

### 4.3 Install Docker Compose Plugin

**Command:**
```bash
sudo apt install docker-compose-plugin
```

**Why:** Docker Compose manages multi-container applications using YAML configuration files.

**What happens:** Installs Docker Compose V2 (integrated into Docker CLI).

**Verify installation:**
```bash
docker --version
docker compose version
```

---

## 5. Deploy Your Application

### 5.1 Setup Project Structure

**Commands:**
```bash
mkdir -p ~/apps/gkverse/portfolio
cd ~/apps/gkverse/portfolio
git clone YOUR_REPOSITORY_URL .
```

**Why:** Organized directory structure makes it easier to manage multiple applications.

**What happens:** Creates directory structure and clones your repository into it.

### 5.2 Create Dockerfile

**Command:**
```bash
nano Dockerfile
```

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

**Why multi-stage build:**
- **Build stage**: Installs all dependencies and builds the application
- **Production stage**: Copies only built files, creating a smaller final image
- Reduces image size by excluding build tools and dev dependencies

**What happens:** This file defines how Docker builds your application image.

### 5.3 Create Docker Compose Configuration

**Command:**
```bash
nano docker-compose.yml
```

**Docker Compose content:**
```yaml
networks:
  tunnel:
    driver: bridge

services:
  portfolio:
    build: .
    container_name: gkverse-portfolio
    restart: always
    networks:
      - tunnel
    expose:
      - "4000"

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    command: tunnel --config /etc/cloudflared/config.yml run
    networks:
      - tunnel
    volumes:
      - /home/deploy/.cloudflared/f523b141-d0c3-4dd4-afc7-6a2b4eec7f57.json:/home/nonroot/.cloudflared/f523b141-d0c3-4dd4-afc7-6a2b4eec7f57.json:ro
      - /etc/cloudflared/config.yml:/etc/cloudflared/config.yml:ro
    restart: unless-stopped
    depends_on:
      - portfolio
```

**Why this configuration:**
- **networks**: Creates a shared network so containers can communicate by name
- **expose vs ports**: `expose` only makes port available to other containers (more secure than `ports` which exposes to host)
- **volumes**: Mounts config and credentials from host into container
- **depends_on**: Ensures portfolio starts before cloudflared
- **restart policies**: Automatically restarts containers if they crash or server reboots

**What happens:** Docker Compose orchestrates both your app and cloudflared tunnel in a single network.

### 5.4 Handle Project Structure Issues

**If your repository has nested folders:**
```bash
mv Portfolio/* . 
mv Portfolio/.* . 2>/dev/null
rmdir Portfolio
```

**Why:** Dockerfile expects files in the current directory. Nested folders cause build failures.

**What happens:** Moves all files (including hidden ones) to current directory and removes empty folder.

### 5.5 Build and Run Containers

**Command:**
```bash
docker compose up -d --build
```

**Flags explained:**
- `up`: Creates and starts containers
- `-d`: Detached mode (runs in background)
- `--build`: Forces rebuild of images

**What happens:** Docker builds your application image and starts both containers.

**If you encounter yarn.lock errors:**
```bash
sudo corepack enable
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

**What happens:** If successful, you'll see HTML output from your application.

### 5.7 Check Container Status

**Commands:**
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
service: http://gkverse-portfolio:4000  # Must match container_name in docker-compose.yml
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
