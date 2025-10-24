# 🌍 **Travel Memory — Full Deployment Guide**

A complete step-by-step guide to deploy the **Travel Memory** application — including backend setup, frontend configuration, load balancer, and domain integration using Cloudflare.

---

## 🖥️ **Backend Configuration**

### **Step 1 – Launch EC2 Instance**

**AMI (OS):**

* Ubuntu 22.04 LTS

**Instance Type:**

* t2.micro (Free Tier eligible)

**Key Pair:**

* Create or select an existing one (`.pem` file required for SSH access)

**Security Group Rules:**

| Type       | Port | Purpose                             |
| ---------- | ---- | ----------------------------------- |
| SSH        | 22   | For SSH access                      |
| HTTP       | 80   | Standard web traffic                |
| HTTPS      | 443  | Secure web traffic (if SSL enabled) |
| Custom TCP | 3001 | Backend API access                  |

> 💡 *Tip:*
> If the frontend and backend instances are within the same VPC but in different subnets, restrict inbound access on port 3001 to the frontend instance’s **private IP** for added security.

---

### **Step 2 – Connect to Your EC2 Instance**

Copy the **Public IP** or **Public DNS**, then connect via SSH:
  
```bash
ssh -i your-key.pem ubuntu@<EC2-Public-IP>
```
   
---

### **Step 3 – Update the System**

Keep the OS packages up to date:

```bash
sudo apt update && sudo apt upgrade -y
```

---

### **Step 4 – Install Dependencies**

**Node.js**

```bash
curl -s https://deb.nodesource.com/setup_18.x | sudo bash
sudo apt install nodejs -y
```

**Git**

```bash
sudo apt install git -y
```

**(Optional) Nginx**

```bash
sudo apt install nginx -y
```

**Verify**

```bash
node -v
npm -v
git --version
```

---

### **Step 5 – Clone the Repository**

```bash
cd /home/ubuntu
git clone https://github.com/UnpredictablePrashant/TravelMemory
```

---

### **Step 6 – Install Project Dependencies**

```bash
cd TravelMemory/backend
npm install
```

Create a `.env` file:

```bash
sudo nano .env
```

---

### **Step 7 – Configure Environment Variables**

```bash
PORT=3001
MONGO_URI=mongodb+srv://test:dRpB7CeQ3WHJct4f@cluster0.ofqf3gw.mongodb.net/TravelDb
```

---

### **Step 8 – Start the Server with PM2**

Install and configure **PM2** for process management:

```bash
sudo npm install -g pm2
pm2 start index.js --name "travelmemory-backend"
pm2 save
pm2 startup
```

---

## 💻 **Frontend Configuration**

### **Step 1 – Configure Nginx Reverse Proxy**

Edit the configuration file:

```bash
sudo nano /etc/nginx/nginx.conf
```

Paste the following:

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    server {
        listen 80;
        server_name _;
        root /var/www/travelmemory;
        index index.html;

        # Frontend routes
        location / {
            try_files $uri $uri/ /index.html;
        }

        # Backend API proxy
        location /api {
            proxy_pass http://localhost:3001;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
        }
    }
}
```

> ⚙️ *Notes:*
>
> * `root` → directory where your frontend build is copied (e.g., `/var/www/travelmemory`).
> * `proxy_pass` → backend’s private or public IP + port (e.g., `http://<backend-ip>:3001`).

---

### **Step 2 – Start Nginx**

```bash
sudo nginx -t
sudo systemctl start nginx
sudo systemctl enable nginx
```

---

### **Step 3 – Build Frontend**

```bash
cd ../frontend
npm install
```

Edit API base URL:

```bash
sudo nano src/url.js
```

---

### **Step 4 – Configure API URLs**

**Example src/url.js:**

```js
export const baseUrl = process.env.REACT_APP_BACKEND_URL || "/api/";
```

| Environment | baseUrl Example                                                      | Description   |
| ----------- | -------------------------------------------------------------------- | ------------- |
| Local       | [http://localhost:3001](http://localhost:3001)                       | Local backend |
| Internal    | http://<backend-private-ip>:3001                                     | Within VPC    |
| Production  | [http://api.shivamassociate.store](http://api.shivamassociate.store) | Public domain |

> 🧠 When using Nginx, `/api/` proxies internally to `http://<backend-ip>:3001` keeping backend ports hidden.

---

### **Step 5 – Deploy Frontend**

```bash
npm run build
sudo mkdir -p /var/www/travelmemory
sudo cp -r build/* /var/www/travelmemory/
sudo chown -R nginx:nginx /var/www/travelmemory
sudo chmod -R 755 /var/www/travelmemory
```

---

## ⚙️ **Load Balancer Setup**

### **Step 1 – Create an Application Load Balancer**

In AWS → EC2 → **Load Balancers** → Create:

| Setting           | Value                            |
| ----------------- | -------------------------------- |
| Name              | travel-memory-lb                 |
| Scheme            | Internet-facing                  |
| Listeners         | HTTP (80), HTTPS (443)           |
| IP Type           | IPv4                             |
| Security Groups   | Use existing SG                  |
| Target Group      | saurabh-travelmemory-targetgroup |
| Protocol / Port   | HTTP / 80                        |
| Health Check Path | `/`                              |

---

### **Step 2 – Health Check Configuration**

| Setting             | Value           |
| ------------------- | --------------- |
| Protocol            | HTTP            |
| Port                | 80              |
| Path                | `/health-check` |
| Timeout             | 5 s             |
| Interval            | 30 s            |
| Unhealthy Threshold | 2               |
| Healthy Threshold   | 5               |

> ✅ Make sure your backend exposes a `/health-check` endpoint.

---

## 🌐 **Domain Setup with Cloudflare**

### **Goal**

Use custom domains (e.g.,
`www.travelmemory.com` and `api.travelmemory.com`) instead of AWS IPs.

---

### **Step 1 – Add Domain to Cloudflare**

1. Purchase a domain (GoDaddy, Namecheap, Hostinger etc.).
2. Log in to [Cloudflare Dashboard](https://dash.cloudflare.com).
3. Click **“Add a site”** → enter your domain (e.g. `travelmemory.com`).
4. Choose the **Free Plan** → Continue.
5. Update your domain registrar’s nameservers to Cloudflare’s (e.g. `abby.ns.cloudflare.com`, `nolan.ns.cloudflare.com`).
6. Wait 5–30 min for DNS propagation.

---

### **Step 2 – Add DNS Records**

| Type      | Name         | Target                | Proxy      | Purpose              |
| --------- | ------------ | --------------------- | ---------- | -------------------- |
| **A**     | `@` or `www` | `<EC2 Public IP>`     | 🟠 Proxied | Frontend (React App) |
| **CNAME** | `api`        | `<Load Balancer DNS>` | ⚪ DNS Only | Backend (Node.js)    |

**Example:**

* A Record → `www` → `13.234.45.78`
* CNAME Record → `api` → `myapp-lb-155231221.ap-south-1.elb.amazonaws.com`

---

### **Step 3 – Update Frontend Environment**

On your frontend EC2:

```bash
REACT_APP_API_URL=https://api.shivamassociate.store
```

Then rebuild and restart Nginx:

```bash
npm run build
sudo systemctl restart nginx
```

---

### **Step 4 – Testing**

| Test                                      | Expected Result                    |
| ----------------------------------------- | ---------------------------------- |
| `http://shivamassociate.store`            | Loads React Frontend               |
| `http://api.shivamassociate.store/health` | Backend responds via Load Balancer |

 <img width="1280" height="800" alt="Screenshot 2025-10-24 at 10 38 30 AM" src="https://github.com/user-attachments/assets/38dee53d-66cb-47be-b041-b841350c8baf" />

  <img width="1280" height="800" alt="Screenshot 2025-10-24 at 10 39 12 AM" src="https://github.com/user-attachments/assets/0adcfaa1-7da5-487a-acc2-c47869c080f3" />

  <img width="1280" height="800" alt="Screenshot 2025-10-24 at 10 39 30 AM" src="https://github.com/user-attachments/assets/7f23d36f-620c-4509-96cc-7b72b4587580" />

  <img width="1280" height="800" alt="Screenshot 2025-10-24 at 10 39 38 AM" src="https://github.com/user-attachments/assets/c4e85695-3139-4582-9b33-c047b79cd474" />

---

## 🧭 **Visual Overview**

```
[ User Browser ]
       ↓
   Cloudflare (DNS)
       ↓
 ┌────────────┬──────────────┐
 │  A Record  │   CNAME      │
 │ (Frontend) │ (Backend)    │
 └──────┬─────┴──────┬───────┘
        ↓             ↓
     EC2 (Frontend)   AWS Load Balancer
                          ↓
                  EC2 Backend Instances
```

---

### ✅ **Deployment Completed**

You now have:

* Backend running on EC2 + PM2
* Frontend served via Nginx
* Application Load Balancer for scaling
* Cloudflare DNS for clean domains

---

