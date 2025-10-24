Travel Memory
Backend Configuration

Step 1. Launch EC2 Instance
AMI (OS):
 Ubuntu (e.g., Ubuntu 22.04 LTS)
Instance Type:
 t2.micro (Free tier eligible)
Key Pair:
 Create or select an existing one (.pem file required for SSH access)
Security Group Configuration:
Allow SSH (Port 22) – for connecting to the instance via SSH


Allow HTTP (Port 80) – for standard web traffic


Allow HTTPS (Port 443) – for secure web traffic (if SSL enabled)


Allow Custom TCP (Port 3001) – required for backend API access


Note:
 If the frontend and backend EC2 instances are within the same VPC but different subnets, you can enhance security by allowing inbound access only from the frontend instance’s private IP on the port where the backend server runs (usually 3000 or 3001).


Step 2. Connect to the Instance
After launch, copy the Public IP or Public DNS.

Use terminal:
ssh -i your-key.pem ubuntu@<EC2-Public-IP>


Now inside our EC2 server (remote Linux computer).



Step 3. Update the System
Always first update Ubuntu packages:
sudo apt update && sudo apt upgrade -y

This ensures all software dependencies are up to date.

Step 4. Install Required Dependencies
Depending on our project stack, install the basics:
Node.js (for backend)
curl -s https://deb.nodesource.com/setup_18.x | sudo bash
sudo apt install nodejs -y

Git (to clone your repo)
sudo apt install git -y

(Optional) Nginx (for reverse proxy)
sudo apt install nginx -y

We can check:
node -v
npm -v
git --version


Step 5. Clone Our Project Repository
Move to our home directory:
cd /home/ubuntu

Clone our code:
git clone https://github.com/UnpredictablePrashant/TravelMemory


Step 6. Go to Our Project Folder and Install Dependencies
Example (if our backend is inside a folder):
cd TravelMemory/backend
npm install
This installs all Node.js packages defined in package.json.

# Create .env file
sudo nano .env


Step 7: Configure Environment Variables
# .env file content
PORT=3001
MONGO_URI=mongodb+srv://test:dRpB7CeQ3WHJct4f@cluster0.ofqf3gw.mongodb.net/TravelDb





Step 9: PM2 Setup for Process Management
# Install PM2 globally
sudo npm install -g pm2

# Start backend with PM2
pm2 start index.js --name "travelmemory-backend"

# Save PM2 configuration
pm2 save
pm2 startup






Frontend Configuration
Step 1: Configure Nginx Reverse Proxy
sudo nano /etc/nginx/nginx.conf
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

Note:
The root path should point to the directory where our frontend build is copied (e.g., /var/www/travelmemory).


The proxy_pass should point to our backend private IP and port (e.g., http://<backend-private/public-ip>:3001/).


If running locally, we can test using http://localhost:3001.

Step 2: Start Nginx
# Test nginx configuration
sudo nginx -t

# Start nginx
sudo systemctl start nginx
sudo systemctl enable nginx


Step 3: Build Frontend
cd ../frontend

# Install dependencies
npm install

# Update API URLs
sudo nano src/url.js



Step 4: Configure API URLs
In the frontend project, all API requests are made using a base URL defined inside the src/url.js file.
Example:
export const baseUrl = process.env.REACT_APP_BACKEND_URL || "http://api.shivamassociate.store";

The baseUrl tells the frontend where to send API requests.
 It can vary depending on the environment setup:
Local Development: http://localhost:3001
 Used when backend runs locally.


Private Network (Internal Access): http://<backend-private-ip>:3001
 Used when frontend and backend are within the same VPC or subnet.


Production Domain: http://api.shivamassociate.store
 Used when backend is deployed behind a load balancer or DNS domain.


When using Nginx Reverse Proxy, we do not expose backend ports (like 3000 or 3001) to the public.
Instead, Nginx handles API routing internally through a relative path such as /api.
In this case, update your src/url.js file as:
export const baseUrl = process.env.REACT_APP_BACKEND_URL || "/api/";

This way:
Frontend sends requests to /api/...


Nginx forwards them internally to http://<backend-private-ip>:3001


Backend processes the request and returns the response through the same route


This approach keeps backend ports hidden, improves security, and ensures a clean integration between frontend and backend.



Step 5: Build and Deploy Frontend
# Build React application
npm run build

# Create deployment directory
sudo mkdir -p /var/www/travelmemory
sudo cp -r build/* /var/www/travelmemory/

# Set permissions
sudo chown -R nginx:nginx /var/www/travelmemory
sudo chmod -R 755 /var/www/travelmemory







Load Balancer Setup
Step 1: Create Application Load Balancer
Navigate to EC2 → Load Balancers → Create Load Balancer
Configure:
Name: travel-memory-lb
Scheme: internet-facing
IP address type: ipv4
Listeners: HTTP (80) and HTTPS (443)
Security Groups: Use previously created security group
Configure Target Groups:
Name: saurabh-travelmemory-targetgroup
Target type: instances
Protocol: HTTP
Port: 80
Health check path: /
Register Targets: Add all EC2 instances





Step 2: Configure Health Checks
# Health check settings
Protocol: HTTP
Port: 80
Path: /health-check (create this endpoint in your backend)
Timeout: 5 seconds
Interval: 30 seconds
Unhealthy threshold: 2
Healthy threshold: 5

















Domain Setup with Cloudflare
Goal:
Make our AWS application accessible using a custom domain (e.g. www.travelmemory.com and api.travelmemory.com) instead of the AWS public IP or load balancer URL.

Step-by-Step Configuration

Step 1: Buy & Add Our Domain to Cloudflare
Buy a domain from any registrar (GoDaddy, Namecheap, Google Domains, Hostinger, etc.).
 Example:

 travelmemory.com
Go to https://dash.cloudflare.com → Log in.


Click “Add a site” → Enter our domain name:

 travelmemory.com
 ⚠️ Do NOT enter our EC2 public domain like
 ec2-13-234-45-78.ap-south-1.compute.amazonaws.com
 because Cloudflare only accepts domains we own.



Choose the Free plan → Continue.


Cloudflare will automatically scan for DNS records. We can skip or continue — we’ll add our own records next.


Cloudflare shows us two nameservers (for example):

abby.ns.cloudflare.com
nolan.ns.cloudflare.com
Go to our domain registrar (where we bought the domain) → open DNS / Nameserver settings → replace the old nameservers with these Cloudflare ones.




Wait 5–30 minutes (sometimes up to 24 hours) for propagation.
 Once verified, our domain will show “Active” in Cloudflare.




Step 2: Add DNS Records in Cloudflare
Now we connect our domain to AWS infrastructure.
Go to our domain → DNS → Records tab → click “Add Record”.
We’ll add two records:

🟢 (A) Frontend (React App on EC2) — A Record
Type
Name
IPv4 Address
Proxy Status
A
@ or www
<EC2 Public IPv4>
✅ Proxied (orange cloud)

Example:
Type: A
Name: www
IPv4: 13.234.45.78
Proxy: OF(GREY cloud)

This makes  http://shivamassociate.store  point to our frontend hosted on EC2 (with Nginx serving /var/www/travelmemory/frontend/build).

🟣 (B) Backend (Node.js via Load Balancer) — CNAME Record
Type
Name
Target
Proxy Status
CNAME
api
<Load Balancer DNS>
DNS Only (gray cloud)

Example:
Type: CNAME
Name: api
Target: myapp-lb-155231221.ap-south-1.elb.amazonaws.com
Proxy: OFF

This connects api.travelmemory.com → AWS Load Balancer → our backend instances.




Step 4: Update Frontend Environment
If your React app calls APIs, update your .env file (on EC2):
REACT_APP_API_URL=https://api.shivamassociate.store

Then rebuild your frontend and restart Nginx:
npm run build
sudo systemctl restart nginx




 Step 5: Test the Setup
Now verify:
Test
Expected Result
Visit http://shivamassociate.store
Loads our frontend React app
Visit http://api.shivamassociate.store/health
Hits our Node.js backend (via Load Balancer)






🧭 Visual Overview
[ User Browser ]
       ↓
   Cloudflare (DNS)
       ↓
 |────────────|
 |  A Record  |   CNAME       
 | (Frontend) | (Backend)     |
 ↓            ↓
EC2 (IP)   AWS Load Balancer
                 ↓
           EC2 Backend Instances


















