# How to Deploy React App on VPS + Nginx (Traditional Method, No CI/CD)

This guide walks through deploying a React application on a VPS using Nginx as a web server and reverse proxy, without continuous integration/deployment pipelines.

---

## 1. Connect to VPS, Update System, Install Nginx, and Setup Firewall

Nginx listens on specific ports. Configuring the firewall allows your website to accept incoming requests.

### Common Firewall Ports

| Port | Purpose |
|------|---------|
| **80** | HTTP (Nginx) |
| **443** | HTTPS (secure website) |
| **22** | SSH |
| **3000 / 5000** | Backend (Node.js / API) |

```bash
# Connect to VPS
ssh user@your-vps-ip

# Update system
sudo apt update && sudo apt upgrade -y

# Install Nginx
sudo apt install nginx -y

# Configure firewall
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
sudo ufw enable
```

---

## 2. Create Directory for React App

```bash
sudo mkdir -p /var/www/react-app
sudo chown -R $USER:$USER /var/www/react-app
```

### Directory Structure Explanation

- `/var` → Variable data (files that change over time, logs, cache, websites)
- `/var/www` → Conventional directory for web content
- Changing ownership avoids permission errors when uploading files

---

## 3. Upload Build Files to the Directory

### Build Your React App Locally

```bash
npm run build
```

Upload the contents of `build/` (or `dist/` for Vite) to `/var/www/react-app`

### Upload Options

- **WinSCP** → Drag & drop GUI interface
- **scp** → Command line secure copy
- **rsync** → Command line synchronization

```bash
# Example using scp
scp -r build/* user@your-vps-ip:/var/www/react-app/

# Example using rsync
rsync -avz build/ user@your-vps-ip:/var/www/react-app/
```

---

## 4. Create Nginx Server Block

A server block defines how Nginx handles requests for a specific website/app.

### Why We Need Server Blocks

- To host multiple websites, domains, or apps on a single VPS
- Without it, Nginx serves only the default site

### Analogy

- **Firewall** → Main gate
- **Nginx** → Receptionist
- **Server block** → Instructions: "If someone asks for example.com, send them to Room A"

### Basic Server Block Example

```nginx
server {
    listen 80;                    # Listen on port 80 (HTTP)
    server_name example.com;      # Domain name

    root /var/www/react-app;      # Serve files from this directory
    index index.html;             # Default file

    location / {
        try_files $uri /index.html;  # SPA routing: fallback to index.html
    }
}
```

### Explanation of Each Line

- `server {}` → Defines one site/app
- `listen 80;` → Accept HTTP requests on port 80
- `server_name example.com;` → Domain(s) this config applies to
- `root /var/www/react-app;` → Directory with app files
- `index index.html;` → Default file to serve
- `location / { try_files $uri /index.html; }` → SPA routing for React

### Create the Server Block

```bash
sudo nano /etc/nginx/sites-available/react-app
```

Paste the server block configuration above, then save and exit.

---

## 5. Enable the Site

Nginx loads only configs linked in `/etc/nginx/sites-enabled/`

```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/react-app /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

### Optional: Remove Default Site

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo systemctl reload nginx
```

---

## 6. Adding Reverse Proxy (for Backend APIs)

### What is a Reverse Proxy?

A reverse proxy (like Nginx) sits between clients and backend servers:
- Receives client requests
- Forwards to backend
- Returns the response

**The client never connects directly to the backend.**

### Request Flow

```
Client → Nginx (reverse proxy) → Backend (Node.js) → Nginx → Client
```

### Analogy

- **Customer** → Orders at counter
- **Staff (Nginx)** → Sends order to kitchen (backend)
- **Staff** → Brings food (response) back
- Customer never enters kitchen

### Why Use Reverse Proxy for React + Node.js

**Setup:**
- Frontend: React app served by Nginx
- Backend: Node.js/Express API running on `localhost:5000`

**Problem without reverse proxy:**
- Frontend must call `http://SERVER_IP:5000/api/users`
- Exposes backend port publicly
- Complicates HTTPS setup
- Causes potential CORS issues

**Solution with reverse proxy:**
- Client calls `https://example.com/api/users`
- Nginx forwards internally to `http://localhost:5000`
- Cleaner, secure, unified domain

### Nginx Config with Reverse Proxy

```nginx
server {
    listen 80;
    server_name example.com;

    root /var/www/react-app;
    index index.html;

    # Serve React app
    location / {
        try_files $uri /index.html;
    }

    # Proxy API requests to Node.js backend
    location /api {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Benefits

- ✅ Hide backend port
- ✅ Unified domain (no CORS issues)
- ✅ Easier HTTPS setup
- ✅ Better security and control

---

## 7. Install SSL (HTTPS)

### What SSL Means

- **SSL** (Secure Sockets Layer) encrypts traffic between client and server
- Website URL changes from `http://` → `https://`
- Padlock appears in browser
- Nginx listens on port 443 for secure traffic

### Why SSL is Important

- ✅ Encrypts sensitive data (login info, forms, API calls)
- ✅ Prevents eavesdropping/man-in-the-middle attacks
- ✅ Needed for SEO and browser trust
- ✅ Required by modern frontend frameworks (some APIs require HTTPS)

### How to Install SSL (Let's Encrypt + Certbot)

#### Step 1: Install Certbot and Nginx Plugin

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

#### Step 2: Ensure Domain Points to VPS IP

Update DNS A record: `example.com → VPS IP`

⚠️ SSL cannot be issued if domain does not resolve to the server

#### Step 3: Obtain SSL Certificate and Configure Nginx

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

**What Certbot does:**
- `-d example.com` → Primary domain
- `-d www.example.com` → Optional subdomain
- Verifies domain ownership
- Creates SSL certificate
- Modifies Nginx config to listen on 443
- Optionally redirects HTTP (port 80) → HTTPS

#### Step 4: Test Auto-Renewal

```bash
sudo certbot renew --dry-run
```

Certificates expire every 90 days but auto-renew via cron job.

---

## ✅ Summary

1. **Connect to VPS**, update, install Nginx, configure firewall
2. **Create** `/var/www/react-app` and set ownership
3. **Upload** React build files
4. **Create** Nginx server block in `/etc/nginx/sites-available/`
5. **Enable** the site (link to `sites-enabled`)
6. **Add reverse proxy** for backend APIs (optional)
7. **Install SSL** (HTTPS) with Let's Encrypt / Certbot

Your React app is now **live, secure, and accessible** via your domain! 🚀

---

## Troubleshooting Tips

### Nginx Won't Start
```bash
sudo nginx -t  # Check for syntax errors
sudo systemctl status nginx  # Check service status
```

### 502 Bad Gateway (Backend API)
- Ensure backend is running on the correct port
- Check firewall rules
- Verify proxy_pass URL in Nginx config

### Site Not Loading
- Check DNS propagation: `dig example.com`
- Verify Nginx is running: `sudo systemctl status nginx`
- Check Nginx error logs: `sudo tail -f /var/log/nginx/error.log`

### SSL Issues
- Ensure domain points to correct IP
- Check port 80 and 443 are open in firewall
- Review Certbot logs: `sudo certbot certificates`

---

## Additional Resources

- [Nginx Documentation](https://nginx.org/en/docs/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)
- [Digital Ocean Nginx Guides](https://www.digitalocean.com/community/tags/nginx)

---

**Date:** January 3, 2026  
**Topic:** React Deployment  
**Tags:** #react #nginx #vps #deployment #ssl #reverse-pr