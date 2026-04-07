# Deploy to DigitalOcean Droplet — Step-by-Step Guide

This guide walks you through deploying the **Province du Sud-Kivu** static website to a DigitalOcean Droplet using Nginx as the web server.

---

## Prerequisites

Before you begin, make sure you have:

- A [DigitalOcean](https://www.digitalocean.com/) account
- A local machine with `git`, `ssh`, and `rsync` installed
- A registered domain name pointed to DigitalOcean's nameservers (optional but recommended for HTTPS)

---

## Step 1 — Create a Droplet

1. Log in to your [DigitalOcean Control Panel](https://cloud.digitalocean.com/).
2. Click **Create → Droplets**.
3. Choose the following settings:
   - **Image**: Ubuntu 22.04 LTS (x64)
   - **Plan**: Basic — Shared CPU — $6/month (1 GB RAM, 25 GB SSD) is sufficient for a static site
   - **Datacenter region**: Pick the region closest to your primary audience (e.g., Frankfurt or Amsterdam for Central Africa)
   - **Authentication**: Select **SSH Key** and add your public key (`~/.ssh/id_rsa.pub`). If you don't have one, generate it with:
     ```bash
     ssh-keygen -t ed25519 -C "your_email@example.com"
     ```
4. Give the Droplet a hostname (e.g., `sudkivu-web`).
5. Click **Create Droplet**.
6. Note the **public IPv4 address** shown in the dashboard (e.g., `203.0.113.10`). You will use it throughout this guide.

---

## Step 2 — Connect to the Droplet via SSH

```bash
ssh root@<YOUR_DROPLET_IP>
```

Accept the host fingerprint prompt when connecting for the first time.

---

## Step 3 — Update the Server and Install Nginx

```bash
# Update package lists and upgrade installed packages
apt update && apt upgrade -y

# Install Nginx
apt install -y nginx

# Enable Nginx to start automatically on reboot
systemctl enable nginx

# Start Nginx now
systemctl start nginx

# Verify Nginx is running
systemctl status nginx
```

At this point, visiting `http://<YOUR_DROPLET_IP>` in a browser should display the default Nginx welcome page.

---

## Step 4 — Configure the Firewall

```bash
# Allow OpenSSH so you don't lose your SSH connection
ufw allow OpenSSH

# Allow HTTP and HTTPS traffic
ufw allow 'Nginx Full'

# Enable the firewall
ufw enable

# Verify the rules
ufw status
```

---

## Step 5 — Create the Web Root Directory

```bash
# Create the directory that will hold the website files
mkdir -p /var/www/sudkivu

# Give the Nginx user ownership
chown -R www-data:www-data /var/www/sudkivu

# Set appropriate permissions
chmod -R 755 /var/www/sudkivu
```

---

## Step 6 — Configure Nginx for the Site

Create a new Nginx server block (virtual host):

```bash
nano /etc/nginx/sites-available/sudkivu
```

Paste the following configuration, replacing `sudkivu.cd` with your actual domain (or remove the `server_name` line if you are using only the IP address):

```nginx
server {
    listen 80;
    server_name sudkivu.cd www.sudkivu.cd;

    root /var/www/sudkivu;
    index index.html;

    # Serve files; return 404 for anything not found
    location / {
        try_files $uri $uri/ =404;
    }

    # Long-term caching for static assets
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|pdf|mp4|svg|woff|woff2|ttf)$ {
        expires 30d;
        add_header Cache-Control "public, no-transform";
    }

    # Deny access to hidden files (e.g., .git, .env)
    location ~ /\. {
        deny all;
    }
}
```

Save and close the file (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

Enable the site and reload Nginx:

```bash
# Create a symbolic link to enable the site
ln -s /etc/nginx/sites-available/sudkivu /etc/nginx/sites-enabled/

# Test the Nginx configuration for syntax errors
nginx -t

# Reload Nginx to apply the new configuration
systemctl reload nginx
```

---

## Step 7 — Deploy the Website Files

On your **local machine**, clone the repository (if not already done) and sync the files to the Droplet:

```bash
# Clone the repository
git clone https://github.com/cerccongo/sudkivu.git
cd sudkivu

# Sync all files to the Droplet, excluding Git and GitHub metadata
rsync -avz --progress \
  --exclude='.git/' \
  --exclude='.github/' \
  --exclude='*.mp4' \
  ./ root@<YOUR_DROPLET_IP>:/var/www/sudkivu/
```

> **Tip:** The `--exclude='*.mp4'` flag skips large video files to speed up the transfer. Remove it if you need video files on the server.

After the sync completes, verify the site is live:

```bash
curl -I http://<YOUR_DROPLET_IP>
```

You should see `HTTP/1.1 200 OK`.

---

## Step 8 — Configure DNS (Domain Name)

If you have a domain name (e.g., `sudkivu.cd`):

1. In your DNS provider's control panel, add an **A record**:
   - **Name / Host**: `@` (for the root domain)
   - **Value / Points to**: `<YOUR_DROPLET_IP>`
   - **TTL**: 3600 (or lower for faster propagation)
2. Add a second **A record** for `www`:
   - **Name / Host**: `www`
   - **Value / Points to**: `<YOUR_DROPLET_IP>`

DNS propagation can take up to 48 hours but usually completes within a few minutes.

---

## Step 9 — Enable HTTPS with Let's Encrypt (Recommended)

Once DNS is propagated and your domain resolves to the Droplet, install Certbot:

```bash
# Install Certbot and its Nginx plugin
apt install -y certbot python3-certbot-nginx

# Obtain and install the SSL certificate (replace with your domain)
certbot --nginx -d sudkivu.cd -d www.sudkivu.cd
```

Follow the prompts. Certbot will:
- Automatically modify your Nginx configuration to serve traffic over HTTPS
- Set up automatic renewal via a systemd timer

Verify auto-renewal works:

```bash
certbot renew --dry-run
```

After this step, your site will be accessible at `https://sudkivu.cd`.

---

## Step 10 — Update the Site (Manual)

To push new content to the server after making changes locally:

```bash
cd sudkivu

# Pull the latest changes from GitHub
git pull origin main

# Sync the updated files to the Droplet
rsync -avz --progress \
  --exclude='.git/' \
  --exclude='.github/' \
  ./ root@<YOUR_DROPLET_IP>:/var/www/sudkivu/
```

No Nginx restart is needed — static files are served directly from disk.

---

## Step 11 — Automate Deployments with GitHub Actions (Optional)

You can configure GitHub Actions to automatically deploy to the Droplet on every push to `main`.

### 1. Add repository secrets

In your GitHub repository, go to **Settings → Secrets and variables → Actions** and add:

| Secret name  | Value                                                       |
|--------------|-------------------------------------------------------------|
| `DO_HOST`    | Your Droplet's public IP address (e.g., `203.0.113.10`)    |
| `DO_SSH_KEY` | Contents of your **private** SSH key (`~/.ssh/id_rsa` or `~/.ssh/id_ed25519`) |

### 2. Create the workflow file

Create `.github/workflows/deploy-droplet.yml` in the repository:

```yaml
name: Deploy to DigitalOcean Droplet

on:
  push:
    branches: ["main"]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy via rsync over SSH
        uses: easingthemes/ssh-deploy@v5.1.2
        with:
          SSH_PRIVATE_KEY: ${{ secrets.DO_SSH_KEY }}
          REMOTE_HOST: ${{ secrets.DO_HOST }}
          REMOTE_USER: root
          TARGET: /var/www/sudkivu/
          EXCLUDE: ".git/, .github/"
```

Every push to `main` will now automatically sync files to the Droplet.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `ssh: connect to host ... port 22: Connection refused` | Check that the Droplet is running and that port 22 is open in the firewall (`ufw allow OpenSSH`) |
| Nginx shows **403 Forbidden** | Run `chmod -R 755 /var/www/sudkivu` and verify `chown -R www-data:www-data /var/www/sudkivu` |
| Nginx shows **502 Bad Gateway** | Check `nginx -t` for config errors, then `systemctl reload nginx` |
| SSL certificate fails | Ensure DNS is fully propagated and port 80 is open before running Certbot |
| `rsync` fails with permission error | Make sure the SSH key added to the Droplet matches the one on your local machine |
| Changes not visible after sync | Clear your browser cache or do a hard refresh (`Ctrl+Shift+R`) |

---

## Quick Reference

```bash
# SSH into the Droplet
ssh root@<YOUR_DROPLET_IP>

# Check Nginx status
systemctl status nginx

# Reload Nginx after config changes
systemctl reload nginx

# View Nginx error log
tail -f /var/log/nginx/error.log

# View Nginx access log
tail -f /var/log/nginx/access.log

# Renew SSL certificate manually
certbot renew
```

---

*Province du Sud-Kivu — Gouvernement Provincial — RDC*
