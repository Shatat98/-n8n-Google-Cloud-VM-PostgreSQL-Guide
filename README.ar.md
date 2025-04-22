# 🌐 دليل تثبيت n8n مع PostgreSQL على Google Cloud VM (للمبتدئين)

هذا الدليل يرشدك خطوة بخطوة لتثبيت **n8n** على خادم **Google Cloud VM** باستخدام **Docker + Docker Compose**، وتحديث قاعدة البيانات من SQLite الافتراضية إلى **PostgreSQL**.

---

## ✅ المتطلبات

قبل أن تبدأ تأكد من توفر التالي:

- حساب Google Cloud Platform
- اسم نطاق Domain (مثل: `yourdomain.com`)
- إمكانية تعديل إعدادات DNS لإضافة A Record
- معرفة أساسية باستخدام الطرفية (Terminal) و SSH

---

## ☁️ الخطوة 1: إنشاء Google Cloud VM

1. اذهب إلى [Google Cloud Console](https://console.cloud.google.com/)
2. ادخل إلى: Compute Engine → VM instances
3. اضغط على **Create Instance** واملأ البيانات التالية:
   - **الاسم**: `n8n-vm`
   - **المنطقة**: الأقرب إليك
   - **النوع**: `e2-micro` أو `e2-small`
   - **النظام**: Debian 12
   - ✅ فعل خيار السماح بـ HTTP و HTTPS

4. بعد الإنشاء، انسخ **IP الخارجي External IP**

---

## 🌐 ربط النطاق (A Record)

> ⚠️ توجه إلى لوحة تحكم النطاق الخاص بك (مثل GoDaddy أو Namecheap)، ثم أضف A Record يشير إلى الـ IP الخارجي لخادمك، على سبيل المثال:
```
Type: A
Name: ai
Value: [ضع هنا IP الخادم]
```

---

## 🧰 الخطوة 2: تجهيز الخادم

اتصل بـ SSH ثم نفذ:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install docker.io docker-compose nginx certbot python3-certbot-nginx -y
sudo systemctl enable --now docker
```

---

## 📁 الخطوة 3: إنشاء مجلد المشروع وكتابة docker-compose

```bash
mkdir n8n-setup && cd n8n-setup
nano docker-compose.yml
```

الصق التالي:

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

> ⚠️ **تنبيه:** يجب استبدال:
> - `YOUR_PASSWORD` بكلمة سر آمنة
> - `ai.yourdomain.com` بالنطاق الفعلي الخاص بك

---

## 🚀 الخطوة 4: تشغيل الحاويات

```bash
sudo docker-compose up -d
```

تأكد أن كل الحاويات تعمل:

```bash
sudo docker ps
```

---

## 🔐 الخطوة 5: إعداد NGINX و SSL

```bash
sudo nano /etc/nginx/sites-available/n8n
```

الصق:

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

ثم فعّل الإعدادات:

```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

لإضافة SSL:

```bash
sudo certbot --nginx -d ai.yourdomain.com
```

---

## 🌍 الخطوة 6: الدخول إلى n8n

افتح متصفحك وادخل على:
```
https://ai.yourdomain.com
```

إذا ظهرت صفحة إعداد n8n فأنت على الطريق الصحيح 🎉

---

## ✅ الخطوة 7: التأكد من PostgreSQL

```bash
sudo docker exec -it [اسم حاوية n8n] env | grep DB_
```

---

## 🔄 تحديث n8n مستقبلاً

```bash
sudo docker-compose down
sudo docker pull n8nio/n8n
sudo docker-compose up -d
```

---

## 🧑‍💻 إعداد الدليل بواسطة أحمد شتات
