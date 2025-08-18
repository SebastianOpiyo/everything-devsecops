#

# Complete Guide: Docker Deployment for Django Applications

This is a comprehensive tutorial on how to deploy a Django application using Docker and Docker Compose for developers who have never used Docker for deployment.

## Overview: What We'll Build

We'll create a containerized Django web application with this architecture:

```
Internet → Nginx Container (Port 80/443) → Django+Gunicorn Container (Port 8000) → PostgreSQL Database (Container/External)
```

## Part 1: Understanding Docker Concepts

### 1.1 What is Docker?

Docker is a containerization platform that packages applications and their dependencies into lightweight, portable containers.

**Key Concepts:**
- **Container**: A running instance of an image (like a virtual machine but lighter)
- **Image**: A blueprint for creating containers (like a template)
- **Dockerfile**: Instructions to build an image
- **Docker Compose**: Tool to define and run multi-container applications
- **Volume**: Persistent storage that survives container restarts

### 1.2 Why Use Docker for Django?

**Benefits:**
- **Consistency**: Same environment on development, staging, and production
- **Isolation**: No conflicts with system packages
- **Scalability**: Easy to scale up/down
- **Portability**: Runs anywhere Docker runs
- **Quick Setup**: One command deployment

**Components:**
- **Django Container**: Runs your Django app with Gunicorn
- **Nginx Container**: Handles HTTP requests and serves static files
- **Volumes**: Store persistent data (static files, media, logs)
- **Networks**: Allow containers to communicate



### Manual Setup vs Docker: Which is Simpler?

## **Docker Setup is SIMPLER** - Here's Why:

### 1. **Commands Comparison**

**Docker (3 commands):**
```bash
docker-compose build
docker-compose up -d
docker-compose exec web python manage.py migrate
```

**Manual (15+ commands):**
```bash
sudo apt install python3 python3-venv nginx
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install gunicorn
# ... create gunicorn config
# ... create systemd service
# ... configure nginx
# ... enable services
sudo systemctl enable my-created-service.service
sudo systemctl start my-created-service.service
sudo nginx -t
sudo systemctl restart nginx
```

### 2. **Environment Isolation**
- **Docker**: Completely isolated containers
- **Manual**: Shared system resources, potential conflicts

### 3. **Reproducibility**
- **Docker**: Same environment everywhere
- **Manual**: Different results on different servers

### 4. **Maintenance**
- **Docker**: `docker-compose restart`
- **Manual**: Multiple service restarts, permission fixes

### 5. **Deployment**
- **Docker**: Works identically on dev/staging/production
- **Manual**: Requires server-specific configuration

## **However, Manual Setup Has Advantages:**

### 1. **Resource Usage**
- **Manual**: Lower memory/CPU overhead
- **Docker**: Additional container overhead

### 2. **System Integration**
- **Manual**: Native systemd integration, better logging
- **Docker**: Additional abstraction layer

### 3. **Performance**
- **Manual**: Direct hardware access
- **Docker**: Slight performance penalty

### 4. **Learning**
- **Manual**: Understand each component deeply
- **Docker**: Abstracts complexity (good and bad)

## **Recommendation**

**For beginners or rapid deployment**: Use **Docker**
**For production systems requiring maximum performance**: Use **Manual setup**
**For learning purposes**: Try both, start with manual to understand, then move to Docker


## Part 2: Project Structure for Docker

### 2.1 Required Docker Files

```
/opt/my-django-app/
├── app/                          # Django apps
├── myproject/                    # Django project settings
├── static/                       # Static source files
├── requirements.txt              # Python dependencies
├── manage.py                     # Django management script
├── .env                         # Environment variables
├── Dockerfile                   # Instructions to build Django image
├── docker-compose.yml           # Multi-container configuration
├── nginx.conf                   # Nginx configuration
└── entrypoint.sh               # Container startup script
```

### 2.2 Environment Configuration

Create .env file:
```properties
# Django Settings
DEBUG=False
SECRET_KEY=your_super_secret_key_here_replace_with_random_string
DJANGO_SETTINGS_MODULE=myproject.settings

# Database Settings
DATABASE_NAME=your_database_name
DATABASE_USER=your_database_user
DATABASE_PASSWORD=your_secure_database_password
DATABASE_HOST=your_database_host
DATABASE_PORT=5432

# Allowed Hosts
ALLOWED_HOSTS=your-domain.com,www.your-domain.com,localhost,127.0.0.1

# Email Settings
EMAIL_HOST=your.email.server.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
DEFAULT_FROM_EMAIL=noreply@your-domain.com

# API Keys
GOOGLE_MAPS_API_KEY=your_google_maps_api_key_here
RECAPTCHA_PUBLIC_KEY=your_recaptcha_public_key_here
RECAPTCHA_PRIVATE_KEY=your_recaptcha_private_key_here
```

## Part 3: Creating the Dockerfile

### 3.1 Understanding Dockerfile

A Dockerfile contains instructions to build a Docker image. Each instruction creates a layer.

### 3.2 Django Application Dockerfile

Create Dockerfile:
```dockerfile
# Use official Python runtime as base image
FROM python:3.10-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV DEBIAN_FRONTEND=noninteractive

# Set work directory
WORKDIR /app

# Install system dependencies
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        postgresql-client \
        gcc \
        libc6-dev \
        libpq-dev \
        curl \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt /app/
RUN pip install --upgrade pip && \
    pip install -r requirements.txt && \
    pip install gunicorn

# Copy project files
COPY . /app/

# Create directories for static files and logs
RUN mkdir -p /app/staticfiles /app/media /app/logs

# Make entrypoint script executable
RUN chmod +x /app/entrypoint.sh

# Create non-root user
RUN adduser --disabled-password --gecos '' appuser && \
    chown -R appuser:appuser /app
USER appuser

# Expose port
EXPOSE 8000

# Run entrypoint script
ENTRYPOINT ["/app/entrypoint.sh"]
```

### 3.3 Entrypoint Script

Create entrypoint.sh:
```bash
#!/bin/bash

# Exit on any error
set -e

echo "Starting Django application..."

# Wait for database to be ready
echo "Waiting for database..."
while ! nc -z $DATABASE_HOST $DATABASE_PORT; do
  sleep 0.1
done
echo "Database is ready!"

# Run Django management commands
echo "Running migrations..."
python manage.py migrate --noinput

echo "Collecting static files..."
python manage.py collectstatic --noinput

# Create superuser if it doesn't exist
echo "Creating superuser..."
python manage.py shell << EOF
from django.contrib.auth import get_user_model
User = get_user_model()
if not User.objects.filter(email='admin@your-domain.com').exists():
    User.objects.create_superuser('admin@your-domain.com', 'change_this_password')
    print('Superuser created')
else:
    print('Superuser already exists')
EOF

echo "Starting Gunicorn..."
exec gunicorn myproject.wsgi:application \
    --bind 0.0.0.0:8000 \
    --workers 4 \
    --timeout 30 \
    --keep-alive 2 \
    --max-requests 1000 \
    --max-requests-jitter 50 \
    --preload \
    --access-logfile /app/logs/gunicorn_access.log \
    --error-logfile /app/logs/gunicorn_error.log \
    --log-level info
```

## Part 4: Docker Compose Configuration

### 4.1 Understanding Docker Compose

Docker Compose defines and runs multi-container applications using a YAML file.

### 4.2 Docker Compose File

Create docker-compose.yml:
```yaml
version: '3.8'

services:
  # Django application with Gunicorn
  web:
    build: .
    restart: always
    volumes:
      - static_data:/app/staticfiles
      - media_data:/app/media
      - app_logs:/app/logs
    env_file:
      - .env
    environment:
      - DJANGO_ALLOWED_HOSTS=${ALLOWED_HOSTS}
    ports:
      - "127.0.0.1:8000:8000"  # Internal access only
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Nginx reverse proxy
  nginx:
    image: nginx:1.23-alpine
    restart: always
    ports:
      - "0.0.0.0:8443:8443"  # Public access
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - static_data:/app/staticfiles:ro
      - media_data:/app/media:ro
      - nginx_logs:/var/log/nginx
    depends_on:
      web:
        condition: service_healthy
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      timeout: 10s
      retries: 3

# Named volumes for persistent data
volumes:
  static_data:
    driver: local
  media_data:
    driver: local
  app_logs:
    driver: local
  nginx_logs:
    driver: local

# Custom network for service communication
networks:
  app-network:
    driver: bridge
```

### 4.3 Nginx Configuration

Create nginx.conf:
```nginx
upstream web {
    server web:8000;
}

server {
    listen 8443;
    server_name your-domain.com www.your-domain.com your.server.ip.address localhost;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;

    # Client settings
    client_max_body_size 100M;
    client_body_timeout 120s;
    client_header_timeout 120s;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/javascript
        application/xml+rss
        application/json;

    # Static files
    location /static/ {
        alias /app/staticfiles/;
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Media files
    location /media/ {
        alias /app/media/;
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Health check endpoint
    location /health/ {
        proxy_pass http://web;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        access_log off;
    }

    # Main application
    location / {
        proxy_pass http://web;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffers
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        proxy_busy_buffers_size 8k;
    }

    # Error pages
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
    
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

## Part 5: Docker Installation and Setup

### 5.1 Installing Docker on Ubuntu

```bash
# Update system packages
sudo apt update

# Install dependencies
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Add user to docker group (logout/login required)
sudo usermod -aG docker $USER
```

### 5.2 Verify Installation

```bash
# Check Docker version
docker --version

# Check Docker Compose version
docker-compose --version

# Test Docker installation
docker run hello-world
```

## Part 6: Deployment Process

### 6.1 Step-by-Step Deployment

```bash
# 1. Navigate to project directory
cd /opt/my-django-app

# 2. Build the Docker images
docker-compose build

# 3. Start the services
docker-compose up -d

# 4. Check service status
docker-compose ps

# 5. View logs
docker-compose logs -f

# Test the application
curl -I http://localhost:8443
```

### 6.2 Detailed Deployment Commands

```bash
# Build images without cache (clean build)
docker-compose build --no-cache

# Start services and follow logs
docker-compose up --build

# Start in background (detached mode)
docker-compose up -d

# Scale the web service (multiple containers)
docker-compose up --scale web=3 -d

# Stop all services
docker-compose down

# Stop and remove all data (CAUTION!)
docker-compose down -v
```

## Part 7: Container Management

### 7.1 Common Docker Commands

```bash
# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# List images
docker images

# Execute command in running container
docker-compose exec web python manage.py shell

# View container logs
docker-compose logs web
docker-compose logs nginx

# Follow logs in real-time
docker-compose logs -f web

# Restart a service
docker-compose restart web

# Stop a specific service
docker-compose stop web

# Start a specific service
docker-compose start web
```

### 7.2 Debugging Commands

```bash
# Access container shell
docker-compose exec web bash
docker-compose exec nginx sh

# Check container resource usage
docker stats

# Inspect container configuration
docker inspect my-django-app_web_1

# View container processes
docker-compose top

# Check service health
docker-compose ps
```

## Part 8: Data Management

### 8.1 Understanding Docker Volumes

Volumes persist data outside containers, surviving container restarts and removals.

```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect my-django-app_static_data

# Backup volume data
docker run --rm -v my-django-app_static_data:/data -v $(pwd):/backup alpine tar czf /backup/static_backup.tar.gz -C /data .

# Restore volume data
docker run --rm -v my-django-app_static_data:/data -v $(pwd):/backup alpine tar xzf /backup/static_backup.tar.gz -C /data
```

### 8.2 Database Operations

```bash
# Run Django migrations
docker-compose exec web python manage.py migrate

# Create superuser
docker-compose exec web python manage.py createsuperuser

# Collect static files
docker-compose exec web python manage.py collectstatic --noinput

# Load fixtures
docker-compose exec web python manage.py loaddata fixture.json

# Create database backup
docker-compose exec web python manage.py dumpdata > backup.json
```

## Part 9: Environment Management

### 9.1 Multiple Environments

Create environment-specific files:

**docker-compose.override.yml** (development):
```yaml
version: '3.8'

services:
  web:
    environment:
      - DEBUG=True
    volumes:
      - .:/app
    ports:
      - "8000:8000"
```

**docker-compose.prod.yml** (production):
```yaml
version: '3.8'

services:
  web:
    restart: always
    environment:
      - DEBUG=False
  
  nginx:
    restart: always
```

### 9.2 Environment Commands

```bash
# Development
docker-compose up

# Production
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Staging
docker-compose -f docker-compose.yml -f docker-compose.staging.yml up -d
```

## Part 10: SSL/HTTPS Setup with Docker

### 10.1 SSL with Let's Encrypt

Create `docker-compose.ssl.yml`:
```yaml
version: '3.8'

services:
  certbot:
    image: certbot/certbot
    volumes:
      - certbot_data:/etc/letsencrypt
      - certbot_www:/var/www/certbot
    command: certonly --webroot --webroot-path=/var/www/certbot --email admin@your-domain.com --agree-tos --no-eff-email -d your-domain.com

  nginx:
    volumes:
      - certbot_data:/etc/letsencrypt:ro
      - certbot_www:/var/www/certbot:ro

volumes:
  certbot_data:
  certbot_www:
```

### 10.2 SSL Nginx Configuration

Update nginx.conf for HTTPS:
```nginx
server {
    listen 80;
    server_name your-domain.com;
    
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    
    location / {
        return 301 https://$server_name$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen 8443 ssl http2;
    server_name your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
    
    # SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    # ... rest of configuration
}
```

## Part 11: Monitoring and Logging

### 11.1 Container Monitoring

```bash
# Real-time container stats
docker stats

# Container resource usage
docker-compose top

# Service status
docker-compose ps

# Health checks
docker inspect --format='{{.State.Health.Status}}' my-django-app_web_1
```

### 11.2 Log Management

```bash
# View all logs
docker-compose logs

# Follow specific service logs
docker-compose logs -f web
docker-compose logs -f nginx

# View last N lines
docker-compose logs --tail=50 web

# Filter logs by timestamp
docker-compose logs --since="2025-01-01T00:00:00Z" web

# Save logs to file
docker-compose logs web > app.log
```

### 11.3 Log Rotation

Add to docker-compose.yml:
```yaml
services:
  web:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

## Part 12: Performance Optimization

### 12.1 Docker Optimization

```dockerfile
# Multi-stage build for smaller images
FROM python:3.10-slim as builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

FROM python:3.10-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
```

### 12.2 Docker Compose Optimization

```yaml
services:
  web:
    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '1.0'
          memory: 1G
    
    # Health checks
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
```

## Part 13: Backup and Recovery

### 13.1 Application Backup

```bash
# Create backup script
cat > backup.sh << 'EOF'
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/$DATE"

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup volumes
docker run --rm -v my-django-app_static_data:/data -v $BACKUP_DIR:/backup alpine tar czf /backup/static.tar.gz -C /data .
docker run --rm -v my-django-app_media_data:/data -v $BACKUP_DIR:/backup alpine tar czf /backup/media.tar.gz -C /data .

# Backup database
docker-compose exec -T web python manage.py dumpdata > $BACKUP_DIR/database.json

echo "Backup completed: $BACKUP_DIR"
EOF

chmod +x backup.sh
```

### 13.2 Recovery Process

```bash
# Restore from backup
DATE="20250813_120000"  # Backup date
BACKUP_DIR="/backups/$DATE"

# Stop services
docker-compose down

# Restore volumes
docker run --rm -v my-django-app_static_data:/data -v $BACKUP_DIR:/backup alpine tar xzf /backup/static.tar.gz -C /data
docker run --rm -v my-django-app_media_data:/data -v $BACKUP_DIR:/backup alpine tar xzf /backup/media.tar.gz -C /data

# Start services
docker-compose up -d

# Restore database
docker-compose exec -T web python manage.py loaddata < $BACKUP_DIR/database.json
```

## Part 14: Troubleshooting Common Issues

### 14.1 Container Won't Start

```bash
# Check container logs
docker-compose logs web

# Check container status
docker-compose ps

# Inspect container
docker inspect my_django_app_web_1

# Try manual start
docker-compose run --rm web bash
```

### 14.2 Network Issues

```bash
# Check network connectivity
docker-compose exec web ping nginx
docker-compose exec nginx ping web

# Inspect network
docker network ls
docker network inspect my-django-app_app-network

# Check port bindings
docker port my-django-app_nginx_1
```

### 14.3 Volume Issues

```bash
# Check volume mounts
docker-compose exec web ls -la /app/staticfiles/
docker-compose exec nginx ls -la /app/staticfiles/

# Check volume permissions
docker-compose exec web id
docker-compose exec web ls -la /app/

# Recreate volumes
docker-compose down -v
docker-compose up -d
```

## Part 15: Production Deployment Checklist

### 15.1 Pre-Deployment

- [ ] Environment variables configured
- [ ] Secret keys generated
- [ ] Database connection tested
- [ ] SSL certificates obtained
- [ ] Domain DNS configured
- [ ] Firewall rules set
- [ ] Backup strategy implemented

### 15.2 Deployment Commands

```bash
# 1. Clone repository
git clone <repository-url> /opt/my-django-app
cd /opt/my-django-app

# 2. Configure environment
cp .env.example .env
nano .env

# 3. Build and deploy
docker-compose build
docker-compose up -d

# 4. Initialize database
docker-compose exec web python manage.py migrate
docker-compose exec web python manage.py collectstatic --noinput

# 5. Create superuser
docker-compose exec web python manage.py createsuperuser

# 6. Test deployment
curl -I http://localhost:8443
```

### 15.3 Post-Deployment

```bash
# Check all services are running
docker-compose ps

# Monitor logs
docker-compose logs -f

# Test application functionality
curl -I http://your-domain.com:8443

# Set up monitoring
# Set up log rotation
# Configure automatic backups
```

## Summary: Docker vs Manual Setup

### Docker Advantages
1. **Simplicity**: One command deployment
2. **Consistency**: Same environment everywhere
3. **Isolation**: No system conflicts
4. **Scalability**: Easy horizontal scaling
5. **Portability**: Runs on any Docker host
6. **Rollback**: Easy version management

### Docker Commands Summary

```bash
# Basic operations
docker-compose build          # Build images
docker-compose up -d          # Start services
docker-compose down           # Stop services
docker-compose ps             # Check status
docker-compose logs -f        # View logs

# Management
docker-compose exec web bash  # Access container
docker-compose restart web    # Restart service
docker-compose pull           # Update images
docker system prune           # Clean up
```

### Access URLs

After deployment, your application will be accessible at:
- **HTTP**: `http://your-domain.com:8443`
- **IP**: `http://your.server.ip.address:8443`
- **Local**: `http://localhost:8443`

This Docker setup provides a **production-ready**, **scalable**, and **easily maintainable** Django application deployment that's perfect for developers new to containerization.

Similar code found with 2 license types

SebastianOpiyo: Give me an improved version that creates a container for postgresql and use gunicorn and uvicorn together to improve processing

GitHub Copilot: # Complete Docker Guide: Django with PostgreSQL, Gunicorn + Uvicorn

This is an enhanced version that includes PostgreSQL container and combines Gunicorn with Uvicorn workers for improved async processing.

## Overview: Enhanced Architecture

```
Internet → Nginx Container → Django+Gunicorn+Uvicorn Container → PostgreSQL Container
                                        ↓
                                Redis Container (Cache/Sessions)
```

## Part 1: Enhanced Project Structure

```
/opt/my-django-app/
├── app/                          # Django apps
├── myproject/                    # Django project settings
├── static/                       # Static source files
├── requirements.txt              # Python dependencies
├── manage.py                     # Django management script
├── .env                         # Environment variables
├── Dockerfile                   # Instructions to build Django image
├── docker-compose.yml           # Multi-container configuration
├── docker-compose.prod.yml      # Production overrides
├── nginx.conf                   # Nginx configuration
├── entrypoint.sh               # Container startup script
├── wait-for-it.sh              # Database wait script
└── gunicorn.conf.py            # Gunicorn configuration
```

## Part 2: Enhanced Environment Configuration

Create .env file:
```properties
# Django Settings
DEBUG=False
SECRET_KEY=your_super_secret_key_here_replace_with_random_string
DJANGO_SETTINGS_MODULE=myproject.settings

# Database Settings (Now using containerized PostgreSQL)
POSTGRES_DB=my_app_db
POSTGRES_USER=my_app_user
POSTGRES_PASSWORD=secure_password_here
DATABASE_HOST=postgres
DATABASE_PORT=5432

# Redis Settings
REDIS_URL=redis://redis:6379/0
CACHE_URL=redis://redis:6379/1
SESSION_REDIS_URL=redis://redis:6379/2

# Allowed Hosts
ALLOWED_HOSTS=your-domain.com,www.your-domain.com,127.0.0.1,localhost,your.server.ip

# Email Settings
EMAIL_HOST=your.email.server.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
DEFAULT_FROM_EMAIL=noreply@your-domain.com

# API Keys
GOOGLE_MAPS_API_KEY=your_google_maps_api_key_here
RECAPTCHA_PUBLIC_KEY=your_recaptcha_public_key_here
RECAPTCHA_PRIVATE_KEY=your_recaptcha_private_key_here

# Performance Settings
GUNICORN_WORKERS=4
GUNICORN_WORKER_CONNECTIONS=1000
GUNICORN_MAX_REQUESTS=1000
GUNICORN_TIMEOUT=30
```

## Part 3: Enhanced Requirements

Create requirements.txt:
```txt
Django>=4.2.0,<5.0
psycopg2-binary>=2.9.5
redis>=4.5.0
django-redis>=5.2.0
gunicorn>=21.2.0
uvicorn[standard]>=0.24.0
uvloop>=0.19.0
httptools>=0.6.0
django-environ>=0.11.0
django-extensions>=3.2.0
django-filters>=23.0
django-widget-tweaks>=1.4.0
django-crispy-forms>=2.0
crispy-bootstrap5>=0.7
django-axes>=6.1.0
django-recaptcha>=3.0.0
sweetify>=2.3.0
viewflow>=2.2.0
Pillow>=10.0.0
celery>=5.3.0
django-celery-beat>=2.5.0
django-celery-results>=2.5.0
whitenoise>=6.6.0
```

## Part 4: Enhanced Dockerfile

Create Dockerfile:
```dockerfile
# Multi-stage build for optimized image
FROM python:3.11-slim as builder

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV DEBIAN_FRONTEND=noninteractive

# Install system dependencies for building
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    g++ \
    libc6-dev \
    libpq-dev \
    libffi-dev \
    libssl-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt /tmp/
RUN pip install --upgrade pip && \
    pip install --user -r /tmp/requirements.txt

# Production stage
FROM python:3.11-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV DEBIAN_FRONTEND=noninteractive
ENV PATH=/root/.local/bin:$PATH

# Install runtime dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    postgresql-client \
    netcat-openbsd \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy Python dependencies from builder stage
COPY --from=builder /root/.local /root/.local

# Set work directory
WORKDIR /app

# Copy wait script
COPY wait-for-it.sh /app/
RUN chmod +x /app/wait-for-it.sh

# Copy project files
COPY . /app/

# Create directories
RUN mkdir -p /app/staticfiles /app/media /app/logs

# Create non-root user
RUN adduser --disabled-password --gecos '' appuser && \
    chown -R appuser:appuser /app

# Make scripts executable
RUN chmod +x /app/entrypoint.sh

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8000/health/ || exit 1

# Run entrypoint script
ENTRYPOINT ["/app/entrypoint.sh"]
```

## Part 5: Enhanced Gunicorn Configuration

Create gunicorn.conf.py:
```python
import multiprocessing
import os

# Server socket
bind = "0.0.0.0:8000"
backlog = 2048

# Worker processes - using uvicorn workers for async support
workers = int(os.getenv('GUNICORN_WORKERS', multiprocessing.cpu_count() * 2 + 1))
worker_class = "uvicorn.workers.UvicornWorker"
worker_connections = int(os.getenv('GUNICORN_WORKER_CONNECTIONS', 1000))
timeout = int(os.getenv('GUNICORN_TIMEOUT', 30))
keepalive = 2

# Restart workers after this many requests
max_requests = int(os.getenv('GUNICORN_MAX_REQUESTS', 1000))
max_requests_jitter = 50

# Logging
accesslog = "/app/logs/gunicorn_access.log"
errorlog = "/app/logs/gunicorn_error.log"
loglevel = "info"
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s" %(D)s'

# Process naming
proc_name = 'my_django_app_gunicorn'

# Server mechanics
daemon = False
pidfile = '/tmp/gunicorn.pid'
user = None
group = None
tmp_upload_dir = None

# SSL (if needed)
keyfile = None
certfile = None

# Performance
preload_app = True
enable_stdio_inheritance = True

# Uvicorn specific settings
uvicorn_kwargs = {
    "loop": "uvloop",
    "http": "httptools",
    "lifespan": "on",
    "access_log": True,
    "use_colors": False,
}

# Worker class specific settings
def post_fork(server, worker):
    """Called just after a worker has been forked."""
    server.log.info("Worker spawned (pid: %s)", worker.pid)

def pre_fork(server, worker):
    """Called just prior to forking the worker subprocess."""
    pass

def when_ready(server):
    """Called just after the server is started."""
    server.log.info("Server is ready. Spawning workers")

def worker_int(worker):
    """Called just after a worker exited on SIGINT or SIGQUIT."""
    worker.log.info("worker received INT or QUIT signal")

def on_exit(server):
    """Called just before exiting Gunicorn."""
    server.log.info("Server is shutting down")
```

## Part 6: Enhanced Entrypoint Script

Create entrypoint.sh:
```bash
#!/bin/bash

# Exit on any error
set -e

echo "=== Starting Django Application ==="

# Load environment variables
if [ -f .env ]; then
    export $(cat .env | grep -v '^#' | xargs)
fi

# Wait for PostgreSQL
echo "Waiting for PostgreSQL..."
./wait-for-it.sh ${DATABASE_HOST}:${DATABASE_PORT} --timeout=60 --strict -- echo "PostgreSQL is ready"

# Wait for Redis
echo "Waiting for Redis..."
./wait-for-it.sh redis:6379 --timeout=30 --strict -- echo "Redis is ready"

# Database operations
echo "Running database migrations..."
python manage.py migrate --noinput

echo "Creating cache table..."
python manage.py createcachetable || echo "Cache table might already exist"

# Static files
echo "Collecting static files..."
python manage.py collectstatic --noinput --clear

# Create superuser if it doesn't exist
echo "Checking for superuser..."
python manage.py shell << EOF
import os
from django.contrib.auth import get_user_model

User = get_user_model()
email = os.environ.get('DJANGO_SUPERUSER_EMAIL', 'admin@your-domain.com')
password = os.environ.get('DJANGO_SUPERUSER_PASSWORD', 'change_this_password')

if not User.objects.filter(email=email).exists():
    User.objects.create_superuser(email, password)
    print(f'Superuser created: {email}')
else:
    print(f'Superuser already exists: {email}')
EOF

# Validate Django configuration
echo "Validating Django configuration..."
python manage.py check --deploy

echo "=== Starting Gunicorn with Uvicorn workers ==="

# Start Gunicorn with Uvicorn workers
exec gunicorn myproject.asgi:application \
    --config /app/gunicorn.conf.py \
    --bind 0.0.0.0:8000 \
    --workers ${GUNICORN_WORKERS:-4} \
    --worker-class uvicorn.workers.UvicornWorker \
    --worker-connections ${GUNICORN_WORKER_CONNECTIONS:-1000} \
    --timeout ${GUNICORN_TIMEOUT:-30} \
    --keep-alive 2 \
    --max-requests ${GUNICORN_MAX_REQUESTS:-1000} \
    --max-requests-jitter 50 \
    --preload \
    --access-logfile /app/logs/gunicorn_access.log \
    --error-logfile /app/logs/gunicorn_error.log \
    --log-level info \
    --capture-output \
    --enable-stdio-inheritance
```

## Part 7: Wait-for-it Script

Create `wait-for-it.sh`:
```bash
#!/usr/bin/env bash
# wait-for-it.sh

WAITFORIT_cmdname=${0##*/}

echoerr() { if [[ $WAITFORIT_QUIET -ne 1 ]]; then echo "$@" 1>&2; fi }

usage()
{
    cat << USAGE >&2
Usage:
    $WAITFORIT_cmdname host:port [-s] [-t timeout] [-- command args]
    -h HOST | --host=HOST       Host or IP under test
    -p PORT | --port=PORT       TCP port under test
                                Alternatively, you specify the host and port as host:port
    -s | --strict               Only execute subcommand if the test succeeds
    -q | --quiet                Don't output any status messages
    -t TIMEOUT | --timeout=TIMEOUT
                                Timeout in seconds, zero for no timeout
    -- COMMAND ARGS             Execute command with args after the test finishes
USAGE
    exit 1
}

wait_for()
{
    if [[ $WAITFORIT_TIMEOUT -gt 0 ]]; then
        echoerr "$WAITFORIT_cmdname: waiting $WAITFORIT_TIMEOUT seconds for $WAITFORIT_HOST:$WAITFORIT_PORT"
    else
        echoerr "$WAITFORIT_cmdname: waiting for $WAITFORIT_HOST:$WAITFORIT_PORT without a timeout"
    fi
    WAITFORIT_start_ts=$(date +%s)
    while :
    do
        if [[ $WAITFORIT_ISBUSY -eq 1 ]]; then
            nc -z $WAITFORIT_HOST $WAITFORIT_PORT
            WAITFORIT_result=$?
        else
            (echo -n > /dev/tcp/$WAITFORIT_HOST/$WAITFORIT_PORT) >/dev/null 2>&1
            WAITFORIT_result=$?
        fi
        if [[ $WAITFORIT_result -eq 0 ]]; then
            WAITFORIT_end_ts=$(date +%s)
            echoerr "$WAITFORIT_cmdname: $WAITFORIT_HOST:$WAITFORIT_PORT is available after $((WAITFORIT_end_ts - WAITFORIT_start_ts)) seconds"
            break
        fi
        sleep 1
    done
    return $WAITFORIT_result
}

wait_for_wrapper()
{
    # In order to support SIGINT during timeout: http://unix.stackexchange.com/a/57692
    if [[ $WAITFORIT_QUIET -eq 1 ]]; then
        timeout $WAITFORIT_BUSYTIMEFLAG $WAITFORIT_TIMEOUT $0 --quiet --child --host=$WAITFORIT_HOST --port=$WAITFORIT_PORT --timeout=$WAITFORIT_TIMEOUT &
    else
        timeout $WAITFORIT_BUSYTIMEFLAG $WAITFORIT_TIMEOUT $0 --child --host=$WAITFORIT_HOST --port=$WAITFORIT_PORT --timeout=$WAITFORIT_TIMEOUT &
    fi
    WAITFORIT_PID=$!
    trap "kill -INT -$WAITFORIT_PID" INT
    wait $WAITFORIT_PID
    WAITFORIT_RESULT=$?
    if [[ $WAITFORIT_RESULT -ne 0 ]]; then
        echoerr "$WAITFORIT_cmdname: timeout occurred after waiting $WAITFORIT_TIMEOUT seconds for $WAITFORIT_HOST:$WAITFORIT_PORT"
    fi
    return $WAITFORIT_RESULT
}

# process arguments
while [[ $# -gt 0 ]]
do
    case "$1" in
        *:* )
        WAITFORIT_hostport=(${1//:/ })
        WAITFORIT_HOST=${WAITFORIT_hostport[0]}
        WAITFORIT_PORT=${WAITFORIT_hostport[1]}
        shift 1
        ;;
        --child)
        WAITFORIT_CHILD=1
        shift 1
        ;;
        -q | --quiet)
        WAITFORIT_QUIET=1
        shift 1
        ;;
        -s | --strict)
        WAITFORIT_STRICT=1
        shift 1
        ;;
        -h)
        WAITFORIT_HOST="$2"
        if [[ $WAITFORIT_HOST == "" ]]; then break; fi
        shift 2
        ;;
        --host=*)
        WAITFORIT_HOST="${1#*=}"
        shift 1
        ;;
        -p)
        WAITFORIT_PORT="$2"
        if [[ $WAITFORIT_PORT == "" ]]; then break; fi
        shift 2
        ;;
        --port=*)
        WAITFORIT_PORT="${1#*=}"
        shift 1
        ;;
        -t)
        WAITFORIT_TIMEOUT="$2"
        if [[ $WAITFORIT_TIMEOUT == "" ]]; then break; fi
        shift 2
        ;;
        --timeout=*)
        WAITFORIT_TIMEOUT="${1#*=}"
        shift 1
        ;;
        --)
        shift
        WAITFORIT_CLI=("$@")
        break
        ;;
        --help)
        usage
        ;;
        *)
        echoerr "Unknown argument: $1"
        usage
        ;;
    esac
done

if [[ "$WAITFORIT_HOST" == "" || "$WAITFORIT_PORT" == "" ]]; then
    echoerr "Error: you need to provide a host and port to test."
    usage
fi

WAITFORIT_TIMEOUT=${WAITFORIT_TIMEOUT:-15}
WAITFORIT_STRICT=${WAITFORIT_STRICT:-0}
WAITFORIT_CHILD=${WAITFORIT_CHILD:-0}
WAITFORIT_QUIET=${WAITFORIT_QUIET:-0}

# Check to see if timeout is from busybox?
WAITFORIT_TIMEOUT_PATH=$(type -p timeout)
WAITFORIT_TIMEOUT_PATH=$(realpath $WAITFORIT_TIMEOUT_PATH 2>/dev/null || echo $WAITFORIT_TIMEOUT_PATH)

WAITFORIT_BUSYTIMEFLAG=""
if [[ $WAITFORIT_TIMEOUT_PATH =~ "busybox" ]]; then
    WAITFORIT_ISBUSY=1
    # Check if busybox timeout uses -t flag
    # (recent Alpine versions don't support -t anymore)
    if timeout &>/dev/stdout | grep -q -e '-t '; then
        WAITFORIT_BUSYTIMEFLAG="-t"
    fi
else
    WAITFORIT_ISBUSY=0
fi

if [[ $WAITFORIT_CHILD -gt 0 ]]; then
    wait_for
    WAITFORIT_RESULT=$?
    exit $WAITFORIT_RESULT
else
    if [[ $WAITFORIT_TIMEOUT -gt 0 ]]; then
        wait_for_wrapper
        WAITFORIT_RESULT=$?
    else
        wait_for
        WAITFORIT_RESULT=$?
    fi
fi

if [[ $WAITFORIT_CLI != "" ]]; then
    if [[ $WAITFORIT_RESULT -ne 0 && $WAITFORIT_STRICT -eq 1 ]]; then
        echoerr "$WAITFORIT_cmdname: strict mode, refusing to execute subprocess"
        exit $WAITFORIT_RESULT
    fi
    exec "${WAITFORIT_CLI[@]}"
else
    exit $WAITFORIT_RESULT
fi
```

## Part 8: Enhanced Docker Compose

Create docker-compose.yml:
```yaml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-my_app_db}
      POSTGRES_USER: ${POSTGRES_USER:-my_app_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-secure_password_here}
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8 --lc-collate=C --lc-ctype=C"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init-db.sql:ro
    ports:
      - "127.0.0.1:5432:5432"  # Only for debugging
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-my_app_user} -d ${POSTGRES_DB:-my_app_db}"]
      interval: 10s
      timeout: 5s
      retries: 5
    command: >
      postgres
      -c shared_preload_libraries=pg_stat_statements
      -c pg_stat_statements.track=all
      -c max_connections=200
      -c shared_buffers=256MB
      -c effective_cache_size=1GB
      -c maintenance_work_mem=64MB
      -c checkpoint_completion_target=0.7
      -c wal_buffers=16MB
      -c default_statistics_target=100

  # Redis Cache
  redis:
    image: redis:7-alpine
    restart: always
    volumes:
      - redis_data:/data
    ports:
      - "127.0.0.1:6379:6379"  # Only for debugging
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    command: >
      redis-server
      --appendonly yes
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru

  # Django application with Gunicorn + Uvicorn
  web:
    build: .
    restart: always
    volumes:
      - static_data:/app/staticfiles
      - media_data:/app/media
      - app_logs:/app/logs
    env_file:
      - .env
    environment:
      - DATABASE_HOST=postgres
      - REDIS_URL=redis://redis:6379/0
    ports:
      - "127.0.0.1:8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 90s
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M

  # Celery Worker for background tasks
  celery:
    build: .
    restart: always
    volumes:
      - app_logs:/app/logs
      - media_data:/app/media
    env_file:
      - .env
    environment:
      - DATABASE_HOST=postgres
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    command: celery -A myproject worker -l info --concurrency=4
    healthcheck:
      test: ["CMD", "celery", "-A", "myproject", "inspect", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  # Celery Beat for scheduled tasks
  celery-beat:
    build: .
    restart: always
    volumes:
      - app_logs:/app/logs
      - celery_beat_data:/app/celerybeat-schedule
    env_file:
      - .env
    environment:
      - DATABASE_HOST=postgres
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    command: celery -A myproject beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler

  # Nginx reverse proxy
  nginx:
    image: nginx:1.25-alpine
    restart: always
    ports:
      - "0.0.0.0:80:80"
      - "0.0.0.0:443:443"
      - "0.0.0.0:8443:8443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - static_data:/app/staticfiles:ro
      - media_data:/app/media:ro
      - nginx_logs:/var/log/nginx
      - certbot_data:/etc/letsencrypt:ro
      - certbot_www:/var/www/certbot:ro
    depends_on:
      web:
        condition: service_healthy
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "nginx", "-t"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Certbot for SSL certificates
  certbot:
    image: certbot/certbot
    restart: "no"
    volumes:
      - certbot_data:/etc/letsencrypt
      - certbot_www:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"

# Named volumes for persistent data
volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  static_data:
    driver: local
  media_data:
    driver: local
  app_logs:
    driver: local
  nginx_logs:
    driver: local
  celery_beat_data:
    driver: local
  certbot_data:
    driver: local
  certbot_www:
    driver: local

# Custom network
networks:
  app-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

## Part 9: Production Override

Create `docker-compose.prod.yml`:
```yaml
version: '3.8'

services:
  web:
    environment:
      - DJANGO_DEBUG=False
      - GUNICORN_WORKERS=8
      - GUNICORN_WORKER_CONNECTIONS=2000
    deploy:
      resources:
        limits:
          cpus: '4.0'
          memory: 4G
        reservations:
          cpus: '2.0'
          memory: 2G
      replicas: 2

  postgres:
    environment:
      - POSTGRES_SHARED_BUFFERS=512MB
      - POSTGRES_EFFECTIVE_CACHE_SIZE=2GB
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '1.0'
          memory: 1G

  redis:
    command: >
      redis-server
      --appendonly yes
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --tcp-keepalive 300

  celery:
    command: celery -A facilityinspection worker -l info --concurrency=8 --max-tasks-per-child=1000
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
```

## Part 10: Enhanced Nginx Configuration

Create nginx.conf:
```nginx
# Upstream servers
upstream web {
    least_conn;
    server web:8000 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

# Rate limiting
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

# Cache configuration
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g 
                inactive=60m use_temp_path=off;

# HTTP to HTTPS redirect
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;
    
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
    
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS server
server {
    listen 443 ssl http2;
    listen 8443 ssl http2;
    server_name your-domain.com www.your-domain.com;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
    
    # Modern SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets off;
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/my-app.domain.com/chain.pem;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline';" always;

    # Client settings
    client_max_body_size 100M;
    client_body_timeout 120s;
    client_header_timeout 120s;
    client_body_buffer_size 16k;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_comp_level 6;
    gzip_types
        application/atom+xml
        application/javascript
        application/json
        application/ld+json
        application/manifest+json
        application/rss+xml
        application/vnd.geo+json
        application/vnd.ms-fontobject
        application/x-font-ttf
        application/x-web-app-manifest+json
        application/xhtml+xml
        application/xml
        font/opentype
        image/bmp
        image/svg+xml
        image/x-icon
        text/cache-manifest
        text/css
        text/plain
        text/vcard
        text/vnd.rim.location.xloc
        text/vtt
        text/x-component
        text/x-cross-domain-policy;

    # Static files with aggressive caching
    location /static/ {
        alias /app/staticfiles/;
        expires 1y;
        add_header Cache-Control "public, immutable";
        add_header Vary Accept-Encoding;
        access_log off;
        
        # Serve pre-compressed files if available
        gzip_static on;
        
        # Security for static files
        location ~* \.(js|css)$ {
            add_header X-Content-Type-Options nosniff;
        }
    }

    # Media files with moderate caching
    location /media/ {
        alias /app/media/;
        expires 30d;
        add_header Cache-Control "public";
        access_log off;
        
        # Security for media files
        location ~* \.(jpg|jpeg|png|gif|ico|svg|webp)$ {
            add_header X-Content-Type-Options nosniff;
        }
    }

    # Health check endpoint (no rate limiting)
    location /health/ {
        proxy_pass http://web;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        access_log off;
    }

    # API endpoints with rate limiting
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        
        proxy_pass http://web;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # API specific timeouts
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }

    # Login endpoints with strict rate limiting
    location ~ ^/(login|auth)/ {
        limit_req zone=login burst=5 nodelay;
        
        proxy_pass http://web;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket support (if needed)
    location /ws/ {
        proxy_pass http://web;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # Main application with caching for static content
    location / {
        # Cache GET requests for static content
        set $no_cache 0;
        if ($request_method != GET) {
            set $no_cache 1;
        }
        if ($http_cookie ~* "sessionid|csrftoken") {
            set $no_cache 1;
        }
        
        proxy_cache my_cache;
        proxy_cache_bypass $no_cache;
        proxy_no_cache $no_cache;
        proxy_cache_valid 200 302 10m;
        proxy_cache_valid 404 1m;
        proxy_cache_use_stale error timeout invalid_header updating http_500 http_502 http_503 http_504;
        
        proxy_pass http://web;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;
        
        # Connection settings
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffers
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        proxy_busy_buffers_size 8k;
        proxy_temp_file_write_size 8k;
    }

    # Error pages
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
    
    location = /50x.html {
        root /usr/share/nginx/html;
    }

    # Logs
    access_log /var/log/nginx/my_app_access.log combined;
    error_log /var/log/nginx/my_app_error.log warn;
}
```

## Part 11: Enhanced Django Settings

Update your Django settings to support the new configuration:

```python
# settings.py additions

# Async support
ASGI_APPLICATION = 'facilityinspection.asgi.application'

# Database with connection pooling
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('POSTGRES_DB'),
        'USER': env('POSTGRES_USER'),
        'PASSWORD': env('POSTGRES_PASSWORD'),
        'HOST': env('DATABASE_HOST'),
        'PORT': env('DATABASE_PORT'),
        'CONN_MAX_AGE': 60,
        'OPTIONS': {
            'connect_timeout': 10,
            'options': '-c default_transaction_isolation=read_committed'
        }
    }
}

# Redis configuration
REDIS_URL = env('REDIS_URL', default='redis://redis:6379/0')

# Cache configuration
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': env('CACHE_URL', default='redis://redis:6379/1'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            'CONNECTION_POOL_KWARGS': {
                'max_connections': 50,
                'retry_on_timeout': True,
            }
        },
        'KEY_PREFIX': 'my_app',
        'TIMEOUT': 300,
    }
}

# Session configuration
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'
SESSION_COOKIE_AGE = 3600  # 1 hour

# Celery configuration
CELERY_BROKER_URL = env('REDIS_URL', default='redis://redis:6379/0')
CELERY_RESULT_BACKEND = env('REDIS_URL', default='redis://redis:6379/0')
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'UTC'
CELERY_BEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'

# Logging configuration
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
            'style': '{',
        },
    },
    'handlers': {
        'file': {
            'level': 'INFO',
            'class': 'logging.FileHandler',
            'filename': '/app/logs/django.log',
            'formatter': 'verbose',
        },
        'console': {
            'level': 'INFO',
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
    },
    'root': {
        'handlers': ['console', 'file'],
        'level': 'INFO',
    },
    'loggers': {
        'django': {
            'handlers': ['console', 'file'],
            'level': 'INFO',
            'propagate': False,
        },
    },
}
```

## Part 12: Deployment Commands

### 12.1 Initial Deployment

```bash
# Clone and setup
git clone <repository-url> /opt/my_app
cd /opt/my_app

# Configure environment
cp .env.example .env
nano .env

# Build and deploy
docker-compose build
docker-compose up -d

# Check status
docker-compose ps
docker-compose logs -f
```

### 12.2 Production Deployment

```bash
# Production deployment
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build

# Scale services
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d --scale web=3 --scale celery=2

# Monitor
docker-compose -f docker-compose.yml -f docker-compose.prod.yml logs -f
```

### 12.3 SSL Setup

```bash
# Initial certificate
docker-compose run --rm certbot certonly --webroot --webroot-path=/var/www/certbot --email admin@kplc.co.ke --agree-tos --no-eff-email -d my_app.com -d www.my_app.com

# Reload nginx
docker-compose exec nginx nginx -s reload
```

## Part 13: Monitoring and Management

### 13.1 Performance Monitoring

```bash
# Container stats
docker stats

# Database performance
docker-compose exec postgres psql -U your_db_user -d your_database -c "SELECT * FROM pg_stat_activity;"

# Redis monitoring
docker-compose exec redis redis-cli monitor

# Application logs
docker-compose logs -f web
docker-compose exec web tail -f /app/logs/gunicorn_access.log
```

### 13.2 Scaling Commands

```bash
# Scale web servers
docker-compose up -d --scale web=5

# Scale celery workers
docker-compose up -d --scale celery=3

# Update without downtime
docker-compose build web
docker-compose up -d --no-deps web
```

## Summary: Enhanced Features

This enhanced Docker setup provides:

1. **PostgreSQL Container**: Fully containerized database with optimized settings
2. **Gunicorn + Uvicorn**: Async-capable WSGI server for better performance
3. **Redis**: Caching and session storage
4. **Celery**: Background task processing
5. **Enhanced Nginx**: Advanced caching, rate limiting, and security
6. **SSL/HTTPS**: Automated certificate management
7. **Health Checks**: Comprehensive service monitoring
8. **Horizontal Scaling**: Easy service scaling
9. **Performance Optimization**: Tuned for production workloads
10. **Security**: Multiple layers of protection

**Access URLs:**
- **HTTPS**: `https://my_app_domain.com`
- **Custom Port**: `https://my_app_domain.com:port`

This setup provides enterprise-grade deployment with high availability, performance, and security.

Similar code found with 6 license types