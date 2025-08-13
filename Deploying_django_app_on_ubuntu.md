
This is a comprehensive tutorial on how we set up and deployed a Django application on Ubuntu server.

## **Overview: What We Built**

Suppose we created a production-ready Django web application with this architecture:

```
Internet → Nginx (Reverse Proxy) → Gunicorn (WSGI Server) OR Gunicorn + Uvicorn → Django Application → PostgreSQL Database
```

## **Part 1: Understanding the Components**

### **1.1 What Each Component Does**

- **Django**: The web framework that contains our application logic
- **Gunicorn**: A Python WSGI HTTP Server that runs Django / Uvicorn - Supports ASGI applications for async features
- **Nginx**: A web server that handles HTTP requests and serves static files
- **PostgreSQL**: The database server (hosted separately)
- **Systemd**: Linux service manager that keeps our application running

### **1.2 Why This Architecture?**

- **Django**: Handles business logic, URLs, views, templates
- **Gunicorn**: Handles multiple concurrent requests efficiently
- **Nginx**: Serves static files fast, handles SSL, load balancing
- **Separation**: Each component has a specific role, making it scalable and maintainable

## **Part 2: Initial Server Setup**

### **2.1 Server Information**
- **Server IP**: [Your server IP can be private or Public]
- **Domain**: your-domain.com
- **OS**: Ubuntu Server
- **Project Location**: your-project-directory

### **2.2 System Dependencies**

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install required packages
# We use postgesql-client for database access, you can install any other database client as needed
sudo apt install -y python3 python3-pip python3-venv nginx postgresql-client
sudo apt install -y gcc libc6-dev libpq-dev curl git
```

## **Part 3: Application Setup**

### **3.1 Project Structure**
```
/opt/your-app-directory/
├── app/                          # Django apps
│   ├── main/
│   ├── useraccount/
│   ├── inventory/
│   ├── inspection/
│   └── ...
├── your_app/                     # Django project settings
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── static/                       # Static source files
├── staticfiles/                  # Collected static files
├── media/                        # User uploads
├── logs/                         # Application logs
├── venv/                         # Python virtual environment
├── requirements.txt              # Python dependencies
├── manage.py                     # Django management script
├── .env.production              # Environment variables
├── gunicorn.conf.py             # Gunicorn configuration
└── entrypoint.sh                # Startup script
├── load_env.sh                  # Script to load environment variables
--> Note: Adjust paths and names as per your project structure
```

### **3.2 Python Virtual Environment**

```bash
# Create virtual environment
cd /opt/your-project-directory
python3 -m venv venv

# Activate virtual environment
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
pip install gunicorn
```

### **3.3 Environment Configuration**

Created .env.production with:
```properties
DEBUG=False
SECRET_KEY=somesecretkey975B28C6912F6A6AD9C6A3F3C231B
DATABASE_NAME=database_name_prod
DATABASE_USER=database_user_prod
DATABASE_PASSWORD=database_password_prod
DATABASE_HOST=domain_or_ip_of_database_server
DATABASE_PORT=5432
ALLOWED_HOSTS=your-domain.com,www.your-domain.com,127.0.0.1,localhost
ENVIRONMENT=production
```

## **Part 4: Gunicorn Configuration**

### **4.1 What is Gunicorn?**
Gunicorn (Green Unicorn) is a Python WSGI HTTP Server that:
- Handles multiple requests simultaneously
- Manages worker processes
- Provides better performance than Django's development server
- Interfaces between web server (Nginx) and Django application

What is Uvicorn?
- Uvicorn is an ASGI server that supports asynchronous applications, making it suitable for Django projects that use async features.
- It can be used alongside Gunicorn for better performance with async views.

### **4.2 Gunicorn Configuration File**

Create gunicorn.conf.py:
```python
import multiprocessing
import os
from pathlib import Path

# Load environment variables
def load_env_file(env_file_path):
    if os.path.exists(env_file_path):
        with open(env_file_path) as f:
            for line in f:
                if line.strip() and not line.startswith('#'):
                    key, value = line.strip().split('=', 1)
                    os.environ[key] = value

# Server configuration
bind = "127.0.0.1:8000"           # Internal only
workers = multiprocessing.cpu_count() * 2 + 1  # Worker processes
timeout = 30                       # Request timeout
keepalive = 2                     # Keep connections alive

# Logging
accesslog = "/opt/your-app-directory/logs/gunicorn_access.log"
errorlog = "/opt/your-app-directory/logs/gunicorn_error.log"
loglevel = "info"

# Performance
preload_app = True                # Load app before workers fork
max_requests = 1000              # Restart workers after requests
```

### **4.3 Why These Settings?**
- **bind 127.0.0.1:8000**: Only accepts local connections (Nginx will proxy)
- **workers**: Multiple processes handle concurrent requests
- **preload_app**: Improves memory usage and startup time
- **max_requests**: Prevents memory leaks by restarting workers

## **Part 5: Systemd Service Setup**

### **5.1 What is Systemd?**
Systemd manages services on Linux:
- Starts/stops services
- Automatically restarts on failure
- Manages dependencies
- Provides logging

### **5.2 Service Configuration**

Create your-app-name.service:
```ini
[Unit]
Description=Your_App_Name Gunicorn daemon
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/your-app-directory
ExecStart=/bin/bash -c 'cd /opt/your-app-directory && source venv/bin/activate && source load_env.sh && gunicorn --config gunicorn.conf.py your_app_name.wsgi:application'
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
TimeoutStopSec=5
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

### **5.3 Service Management Commands**
```bash
# Enable service to start on boot
sudo systemctl enable your-app-name.service

# Start service
sudo systemctl start your-app-name.service

# Check status
sudo systemctl status your-app-name.service

# View logs
sudo journalctl -u your-app-name.service -f
```

## **Part 6: Nginx Configuration**

### **6.1 What is Nginx?**
Nginx is a web server and reverse proxy that:
- Handles HTTP/HTTPS requests from internet
- Serves static files efficiently (CSS, JS, images)
- Proxies dynamic requests to Gunicorn
- Provides SSL termination
- Handles compression and caching

### **6.2 Nginx Configuration**

Create your-app-name:
```nginx
upstream your_app_name {
    server 127.0.0.1:8000;        # Gunicorn backend
}

server {
    listen 80;                     # HTTP
    listen 8443;                   # Custom HTTPS port, can be changed to a standard port like 443
    server_name my-domain-name.org www.my-domain-name.org [your-public-ip] localhost;

    client_max_body_size 100M;     # Allow large file uploads

    #SSL configuration (optional)
    # Uncomment and configure if using SSL, also edit based on your type of SSL certificate
    # listen 443 ssl;
    # ssl_certificate /etc/letsencrypt/live/my-domain-name.org/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/my-domain-name.org/privkey.pem;
    # ssl_protocols TLSv1.2 TLSv1.3;

    # Static files (served directly by Nginx)
    location /static/ {
        alias /opt/your-app-directory/staticfiles/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Media files (served directly by Nginx)
    location /media/ {
        alias /opt/your-app-directory/media/;
        expires 30d;
    }

    # Application requests (proxied to Gunicorn)
    location / {
        proxy_pass http://your_app_name;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### **6.3 Why This Configuration?**
- **Static/Media locations**: Nginx serves files directly (faster than Django)
- **Proxy headers**: Django needs to know the original client IP
- **Timeouts**: Handle long-running requests
- **upstream**: Load balancing (can add more Gunicorn instances)

## **Part 7: Django Production Settings**

### **7.1 Key Production Settings**

```python
# Security
DEBUG = False
ALLOWED_HOSTS = ['my-domain-name.org', '<my-public-ip/my-private-ip>', 'localhost']

# Static files
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

# Database
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DATABASE_NAME'),
        'USER': os.getenv('DATABASE_USER'),
        'PASSWORD': os.getenv('DATABASE_PASSWORD'),
        'HOST': os.getenv('DATABASE_HOST'),
        'PORT': os.getenv('DATABASE_PORT'),
    }
}
```

### **7.2 Static Files Management**

```bash
# Collect static files
python manage.py collectstatic --noinput

# This copies all static files from apps to staticfiles/ directory
# Nginx serves these files directly for better performance
```

## **Part 8: Deployment Process**

### **8.1 Step-by-Step Deployment**

```bash
# 1. Prepare application
cd /opt/your-app-directory
source venv/bin/activate
source load_env.sh

# 2. Run Django setup
python manage.py migrate
python manage.py collectstatic --noinput

# 3. Test Gunicorn manually
gunicorn --config gunicorn.conf.py your_app_name.wsgi:application

# 4. Set up systemd service
sudo systemctl enable your_app_name.service
sudo systemctl start your_app_name.service

# 5. Configure and start Nginx
sudo ln -s /etc/nginx/sites-available/your_app_name /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx

# 6. Open firewall ports
sudo ufw allow 80/tcp
sudo ufw allow 8443/tcp
```

### **8.2 Testing the Deployment**

```bash
# Test Gunicorn is running
sudo systemctl status your_app_name.service

# Test Nginx is running
sudo systemctl status nginx

# Test application responds
curl -I http://localhost:8443
curl -I http://102.215.40.133:8443
```

## **Part 9: SSL/HTTPS Setup (Optional)**

```bash
# SSL configuration (optional)
# Uncomment and configure if using SSL, also edit based on your type of SSL certificate
# listen 443 ssl;
# ssl_certificate /etc/letsencrypt/live/my-domain-name.org/fullchain.pem;
# ssl_certificate_key /etc/letsencrypt/live/my-domain-name.org/privkey.pem;
# ssl_protocols TLSv1.2 TLSv1.3;

### **9.1 Installing SSL Certificate**

```bash
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Get SSL certificate
sudo certbot --nginx -d my-domain-name.org -d www.my-domain-name.org

# Set up auto-renewal
sudo crontab -e
# Add: 0 */12 * * * /usr/bin/certbot renew --quiet --reload-nginx
```

## **Part 10: Monitoring and Maintenance**

### **10.1 Log Files to Monitor**

```bash
# Application logs
tail -f /opt/your-app-directory/logs/gunicorn_access.log
tail -f /opt/your-app-directory/logs/gunicorn_error.log

# Nginx logs
tail -f /var/log/nginx/your_app_name_access.log
tail -f /var/log/nginx/your_app_name_error.log

# System logs
sudo journalctl -u your_app_name.service -f
sudo journalctl -u nginx.service -f
```

### **10.2 Common Maintenance Tasks**

```bash
# Restart application
sudo systemctl restart your_app_name.service

# Reload Nginx configuration
sudo systemctl reload nginx

# Update application
cd /opt/your-app-directory
git pull
```bash
# Restart application
sudo systemctl restart your_app_name.service

# Reload Nginx configuration
sudo systemctl reload nginx

# Update application
cd /opt/your-app-directory
git pull
source venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py collectstatic --noinput
sudo systemctl restart your_app_name.service

# Check disk space
df -h
du -sh /opt/your-app-directory/logs/
```

## **Part 11: Troubleshooting Common Issues**

### **11.1 Application Won't Start**
```bash
# Check logs
sudo journalctl -u your_app_name.service -n 50

# Test manually
cd /opt/your-app-directory
source venv/bin/activate
python manage.py check
gunicorn --config gunicorn.conf.py your_app_name.wsgi:application
```

### **11.2 Static Files Not Loading**
```bash
# Recollect static files
python manage.py collectstatic --noinput --clear

# Check permissions
sudo chown -R www-data:www-data /opt/your-app-directory/staticfiles/

# Test Nginx can access files
sudo -u www-data ls -la /opt/your-app-directory/staticfiles/
```

### **11.3 Database Connection Issues**
```bash
# Test database connection
python manage.py check --database default

# Test from PostgreSQL client
psql -h <db_host> -p 5432 -U <db_user> -d <db_name>
```

## **Part 12: Performance Optimization**

### **12.1 Gunicorn Optimization**
- **Workers**: `(CPU cores * 2) + 1`
- **Worker Class**: `sync` for CPU-bound, `gevent` for I/O-bound
- **Max Requests**: Restart workers to prevent memory leaks

### **12.2 Nginx Optimization**
- **Gzip Compression**: Reduce bandwidth
- **Static File Caching**: Set long expiry headers
- **Connection Keep-Alive**: Reduce connection overhead

### **12.3 Django Optimization**
- **Database Connection Pooling**: Reuse connections
- **Caching**: Redis/Memcached for session/query caching
- **Static File CDN**: Serve static files from CDN

## **Summary: What We Accomplished**

1. **Deployed** a Django application on Ubuntu server
2. **Configured** Gunicorn as WSGI server
3. **Set up** Nginx as reverse proxy and static file server
4. **Created** systemd service for automatic startup and monitoring
5. **Secured** the application with proper permissions and firewall
6. **Made** the application accessible via domain and IP address
7. **Prepared** for SSL/HTTPS implementation
8. **Established** logging and monitoring

The application is now accessible at:
- **HTTP**: `http://my-domain-org:<port/optional>`
- **IP**: `http://<your_server_ip>:<port>`

This setup provides a **production-ready**, **scalable**, and **maintainable** Django application deployment.

## **Conclusion**
This guide provides a comprehensive overview of deploying a Django application on an Ubuntu server. By following these steps, you can ensure your application is robust, secure, and ready for production use. Remember to regularly monitor and maintain your application to keep it running smoothly. Happy coding!
```