# DigitalOcean Droplet Deployment Guide

## 💰 Pricing Overview

DigitalOcean Droplet pricing (as of 2024):

### Basic Droplets (Recommended for your ML pipeline)
- **$12/month**: 1 vCPU, 2 GB RAM, 50 GB SSD (Minimum for testing)
- **$18/month**: 2 vCPUs, 2 GB RAM, 60 GB SSD
- **$24/month**: 2 vCPUs, 4 GB RAM, 80 GB SSD ⭐ **Recommended starting point**
- **$48/month**: 4 vCPUs, 8 GB RAM, 160 GB SSD ⭐ **Recommended for production**

### Premium Droplets (NVMe SSD - Faster)
- **$14/month**: 1 vCPU, 2 GB RAM, 50 GB NVMe SSD
- **$28/month**: 2 vCPUs, 4 GB RAM, 80 GB NVMe SSD

**Recommendation**: Start with **$24/month (2 vCPU, 4 GB RAM)** or **$48/month (4 vCPU, 8 GB RAM)** due to:
- PyTorch models (Whisper, TTS)
- Large model files
- Multiple concurrent requests
- MongoDB database

---

## 🚀 Deployment Steps

### 1. Create DigitalOcean Droplet

1. Go to [DigitalOcean](https://www.digitalocean.com/)
2. Create a new Droplet
3. Choose:
   - **Image**: Ubuntu 22.04 LTS (recommended)
   - **Plan**: Basic, $24/month (4 GB RAM) or $48/month (8 GB RAM)
   - **Region**: Choose closest to your users
   - **Authentication**: SSH keys (recommended) or root password
4. After creation, note your **IP address** (e.g., `143.110.123.45`)

### 2. Connect to Your Droplet

```bash
ssh root@143.110.123.45
# or
ssh root@YOUR_IP_ADDRESS
```

### 3. Setup Server Environment

```bash
# Update system
apt update && apt upgrade -y

# Install Python 3.10+
apt install python3 python3-pip python3-venv -y

# Install system dependencies
apt install ffmpeg git curl -y

# Install MongoDB (or use MongoDB Atlas - recommended)
apt install -y mongodb

# Start MongoDB
systemctl start mongodb
systemctl enable mongodb
```

### 4. Upload Your Backend Code

**Option A: Using Git (Recommended)**
```bash
# Clone your repository
cd /var/www
git clone YOUR_REPO_URL vashafront-backend
cd vashafront-backend/backend
```

**Option B: Using SCP (from your local machine)**
```bash
# From your local machine
scp -r vashafront/backend root@143.110.123.45:/var/www/vashafront-backend
```

### 5. Setup Python Environment

```bash
cd /var/www/vashafront-backend/backend

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install --upgrade pip
pip install -r requirements.txt

# Install spaCy model
python -m spacy download en_core_web_lg
```

### 6. Configure Environment Variables

```bash
# Create .env file
nano .env
```

Add your configuration:
```env
MONGODB_URI=mongodb://localhost:27017/
SECRET_KEY=your-secret-key-here-min-32-chars
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password
```

**Important**: Generate a secure SECRET_KEY:
```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

### 7. Configure Firewall

```bash
# Allow SSH
ufw allow 22/tcp

# Allow HTTP/HTTPS (for future use)
ufw allow 80/tcp
ufw allow 443/tcp

# Allow your API port
ufw allow 8000/tcp

# Enable firewall
ufw enable

# Check status
ufw status
```

### 8. Run the Server

**For Testing (Temporary)**
```bash
cd /var/www/vashafront-backend/backend
source venv/bin/activate
uvicorn main:app --host 0.0.0.0 --port 8000
```

**For Production (Recommended - Using systemd)**
```bash
# Create systemd service
nano /etc/systemd/system/vashafront-backend.service
```

Add this content:
```ini
[Unit]
Description=Vashafront Backend API
After=network.target mongodb.service

[Service]
Type=simple
User=root
WorkingDirectory=/var/www/vashafront-backend/backend
Environment="PATH=/var/www/vashafront-backend/backend/venv/bin"
ExecStart=/var/www/vashafront-backend/backend/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start the service:
```bash
systemctl daemon-reload
systemctl enable vashafront-backend
systemctl start vashafront-backend
systemctl status vashafront-backend
```

### 9. Test Your Deployment

```bash
# From your local machine or browser
curl http://143.110.123.45:8000/docs

# Or visit in browser:
http://143.110.123.45:8000/docs
```

---

## ✅ Will `http://143.110.123.45:8000` Work?

**YES**, but only if:

1. ✅ **Server is bound to 0.0.0.0**: Use `uvicorn main:app --host 0.0.0.0 --port 8000`
   - ✅ Your code already supports this (see `FRONTEND_BACKEND_CONNECTION.md`)

2. ✅ **Firewall allows port 8000**: 
   ```bash
   ufw allow 8000/tcp
   ```

3. ✅ **MongoDB is running**: 
   ```bash
   systemctl status mongodb
   ```

4. ✅ **Environment variables are set**: Create `.env` file with required variables

5. ⚠️ **MongoDB Configuration**: Your code currently uses `mongodb://localhost:27017/`
   - For production, consider using **MongoDB Atlas** (free tier available)
   - Or ensure MongoDB is properly installed on the droplet

---

## 🔧 Important Considerations

### Resource Requirements

Your pipeline uses:
- **PyTorch** (~500MB-1GB+ RAM)
- **Whisper models** (~1-3GB RAM depending on size)
- **TTS models** (~500MB-1GB RAM)
- **MongoDB** (~200-500MB RAM)
- **System overhead** (~500MB)

**Minimum**: 4 GB RAM ($24/month)
**Recommended**: 8 GB RAM ($48/month) for better performance

### MongoDB Options

1. **Local MongoDB** (Simple, but uses droplet resources)
   ```bash
   apt install mongodb
   ```

2. **MongoDB Atlas** (Recommended - Free tier available)
   - Go to [MongoDB Atlas](https://www.mongodb.com/cloud/atlas)
   - Create free cluster (512 MB)
   - Get connection string
   - Update `MONGODB_URI` in `.env`

### Using IP Address vs Domain

- **IP Address**: `http://143.110.123.45:8000` ✅ Works immediately
- **Domain Name**: Optional, set up DNS pointing to your IP
  - Add A record: `api.yourdomain.com` → `143.110.123.45`

### Security Recommendations

1. **Don't use `--reload` in production**
2. **Use environment variables** for secrets (not hardcoded)
3. **Update CORS settings** in production:
   ```python
   allow_origins=["https://your-frontend-domain.com"]  # Instead of ["*"]
   ```
4. **Use HTTPS** with reverse proxy (Nginx + Let's Encrypt)
5. **Regular backups** of MongoDB data

---

## 📊 Monitoring & Logs

```bash
# View service logs
journalctl -u vashafront-backend -f

# Check server status
systemctl status vashafront-backend

# Monitor resources
htop
# or
free -h
df -h
```

---

## 🐛 Troubleshooting

### Server not accessible
```bash
# Check if server is running
systemctl status vashafront-backend

# Check firewall
ufw status

# Test from server itself
curl http://localhost:8000/docs
```

### Port already in use
```bash
# Find process using port 8000
lsof -i :8000
# or
netstat -tulpn | grep 8000

# Kill process (if needed)
kill -9 <PID>
```

### MongoDB connection issues
```bash
# Check MongoDB status
systemctl status mongodb

# Test MongoDB connection
mongo
# or
mongosh
```

### Out of memory
```bash
# Check memory usage
free -h

# Consider upgrading to 8GB plan or optimize models
```

---

## 💡 Next Steps

1. ✅ Test API endpoints: `http://YOUR_IP:8000/docs`
2. ✅ Update frontend to use: `http://YOUR_IP:8000`
3. ✅ Set up domain name (optional)
4. ✅ Configure HTTPS (optional but recommended)
5. ✅ Set up monitoring and backups

---

## 📝 Quick Reference

**Server Command**:
```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

**Service Management**:
```bash
systemctl start vashafront-backend
systemctl stop vashafront-backend
systemctl restart vashafront-backend
systemctl status vashafront-backend
```

**View Logs**:
```bash
journalctl -u vashafront-backend -f
```
