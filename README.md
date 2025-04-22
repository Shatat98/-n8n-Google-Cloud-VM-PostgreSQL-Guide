# 🌐 n8n + PostgreSQL on Google Cloud VM (Beginner-Friendly Guide)

This guide helps you self-host **n8n** on a **Google Cloud VM** using **Docker + Docker Compose**, and migrate from default SQLite (filesystem) storage to **PostgreSQL** for improved performance and scalability.


🇸🇦 [اضغط هنا لقراءة الدليل باللغة العربية](README.ar.md)
---

## ✅ Requirements
Before you begin, make sure you have:

- A Google Cloud account
- A domain name (e.g., `yourdomain.com`)
- Ability to set DNS A records
- Basic familiarity with using terminal & SSH

---

## 🚀 Step 1: Create Google Cloud VM

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Navigate to `Compute Engine > VM instances`
3. Click **Create Instance** and configure as follows:
   - **Name**: `n8n-vm`
   - **Region**: choose one near you
   - **Machine type**: `e2-micro` (Free Tier)
   - **Boot disk**: Debian 12
   - ✅ Enable HTTP & HTTPS traffic
4. Click **Create**

### 🌐 Link your domain (A record)
> ⚠️ Go to your domain registrar and point an **A record** for your domain (e.g., `ai.yourdomain.com`) to the **external IP** of your VM.

---

## 🧰 Step 2: Prepare the VM

SSH into your VM and run the following:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install docker.io docker-compose nginx certbot python3-certbot-nginx -y
sudo systemctl start docker
sudo systemctl enable docker
```

---

## 📦 Step 3: Create Project Directory & Docker Compose

```bash
mkdir n8n-setup && cd n8n-setup
nano docker-compose.yml
```

Paste the following:

```yaml
version: '3.7'
services:
  postgres:
    image: postgres:15
    restart: always
    environment:
      POSTGRES_USER: n8n_user
      POSTGRES_PASSWORD: YOUR_PASSWORD
      POSTGRES_DB: n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data

  n8n:
    image: n8nio/n8n
    restart: unless-stopped
    ports:
      - 5678:5678
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n_user
      - DB_POSTGRESDB_PASSWORD=YOUR_PASSWORD
      - N8N_HOST=ai.yourdomain.com
      - WEBHOOK_URL=https://ai.yourdomain.com/
      - WEBHOOK_TUNNEL_URL=https://ai.yourdomain.com/
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - postgres

volumes:
  postgres_data:
  n8n_data:
```

> ⚠️ **Important:** Replace all instances of:
> - `YOUR_PASSWORD` → with a strong secure password  
> - `ai.yourdomain.com` → with your actual domain or subdomain  

---

## 🟢 Step 4: Start the containers

```bash
sudo docker-compose up -d
```

Check that both `n8n` and `postgres` containers are running:
```bash
sudo docker ps
```

---

## 🔐 Step 5: Configure NGINX Reverse Proxy + SSL

```bash
sudo nano /etc/nginx/sites-available/n8n
```

Paste:

```nginx
server {
    server_name ai.yourdomain.com;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

> ⚠️ Make sure to replace `ai.yourdomain.com` with your actual domain

Enable it:

```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

Enable SSL:

```bash
sudo certbot --nginx -d ai.yourdomain.com
```

---

## 🧪 Step 6: Access n8n

Visit: `https://ai.yourdomain.com`  
You should see the **n8n setup screen**.

---

## 🔍 Verify PostgreSQL Usage

```bash
sudo docker exec -it CONTAINER_ID env | grep DB_
```

You should see values for:
```
DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=postgres
...
```

---

## 🔄 How to Update Later

```bash
sudo docker-compose down
sudo docker pull n8nio/n8n
sudo docker-compose up -d
```

Your data is kept in Docker volumes (`n8n_data`, `postgres_data`).

---


## 🛠 Tips

- Always use strong passwords and avoid using defaults in production.
- For A record setup: go to your domain registrar (e.g. GoDaddy, Namecheap) → DNS → add `A` record → point to VM's external IP.

  ---

### 🧑‍💻 Made with ❤️ by Ahmed Shatat
