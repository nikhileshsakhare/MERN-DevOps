## Deploy MERN-E-Commerce-Store on Amazon Linux EC2

### Step 1 — Connect & Update

```bash
<ssh command from ec2 connect option>
sudo yum update -y
```

---

### Step 2 — Install Node.js

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install --lts
node -v && npm -v
```

---

### Step 3 — Install Git & Clone the Repo

```bash
sudo yum install git -y
git clone https://github.com/nikhileshsakhare/MERN-DevOps.git
cd MERN-DevOps
```

---

### Step 4 — Install MongoDB

```bash
sudo tee /etc/yum.repos.d/mongodb-org-7.0.repo << 'EOF'
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/amazon/2023/mongodb-org/7.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-7.0.asc
EOF

sudo yum install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod
```

### Step 5 — Install Dependencies

```bash
# Root
cd ~/MERN-DevOps
npm install

# Backend
cd backend && npm install && cd ..

# Frontend
cd frontend && npm install && cd ..
```

---

### Step 6 — Build the Frontend

```bash
cd ~/MERN-DevOps/frontend
npm run build
```

---

### Step 7 — Start Backend with PM2

```bash
npm install -g pm2
cd ~/MERN-DevOps/backend
pm2 start index.js --name "backend"
pm2 save
pm2 startup
```

---

### Step 8 — Install & Configure Nginx

```bash
sudo yum install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
<runn the command of nginx-conf.txt>
```

---

### Step 9 — Test & Reload Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Step 10 — Verify Everything is Running

```bash
curl http://localhost:5000/api/products   # should return JSON
curl http://localhost/api/products        # should return same JSON
pm2 status                                # backend: online
sudo systemctl status mongod              # active (running)
sudo systemctl status nginx               # active (running)
```
