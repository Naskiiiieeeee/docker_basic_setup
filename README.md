# Docker Laravel Development Setup - Complete Cheat Sheet

## 📋 Project Overview
Complete Docker development environment para sa Laravel applications na may:
- **PHP 8.2 FPM** - Application server
- **Nginx** - Web server
- **MySQL 8.0** - Database
- **Redis** - Caching/Sessions
- **phpMyAdmin** - Database management

## 🗂️ Project Structure
```
project-root/
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── nginx/
│   └── conf.d/
│       └── default.conf
├── public/
│   └── index.php
├── storage/
├── vendor/
├── node_modules/
└── .env
```

## 🐳 Docker Files Setup

### 1. Dockerfile
```dockerfile
# Use PHP 8.2 FPM
FROM php:8.2-fpm

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    libpng-dev \
    libonig-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    locales \
    libxml2-dev \
    libicu-dev \ 
    zip \
    unzip \
    git \
    curl \
    nodejs \
    npm \
    jpegoptim \
    optipng \
    pngquant \
    gifsicle \
    vim \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install pdo_mysql mbstring exif intl bcmath gd

# Get latest Composer
COPY --from=composer:2.7 /usr/bin/composer /usr/bin/composer

# Set working directory
WORKDIR /var/www/html

# Copy app files (with correct ownership)
COPY --chown=www-data:www-data . /var/www/html

# Permissions
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html

# Install PHP dependencies
RUN composer install --no-dev --optimize-autoloader

# Build frontend assets
RUN npm install && npm run build

# Expose port for PHP-FPM
EXPOSE 9000

CMD ["php-fpm"]
```

### 2. docker-compose.yml
```yaml
version: '3.8'

services:
  jmca-app:
    container_name: jmca-app
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - .:/var/www/html
      - /var/www/html/vendor
      - /var/www/html/node_modules
    environment:
      - APP_ENV=local
      - APP_DEBUG=true
      - APP_URL=http://localhost:8080
      - DB_CONNECTION=mysql
      - DB_HOST=jmca-db
      - DB_PORT=3306
      - DB_DATABASE=jmca_database
      - DB_USERNAME=jmca_user
      - DB_PASSWORD=jmca_password
    depends_on:
      - jmca-db
    networks:
      - jmca-network

  jmca-web:
    image: nginx:stable-alpine
    container_name: jmca-web
    ports:
      - "8080:80"
    volumes:
      - .:/var/www/html
      - ./nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - jmca-app
    networks:
      - jmca-network

  jmca-db:
    container_name: jmca-db
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: jmca_database
      MYSQL_USER: jmca_user
      MYSQL_PASSWORD: jmca_password
    volumes:
      - jmca_mysql_data:/var/lib/mysql
    networks:
      - jmca-network

  jmca-phpmyadmin:
    container_name: jmca-phpmyadmin
    image: phpmyadmin/phpmyadmin
    restart: always
    ports:
      - "8081:80"
    environment:
      PMA_HOST: jmca-db
      PMA_PORT: 3306
      MYSQL_ROOT_PASSWORD: root_password
    depends_on:
      - jmca-db
    networks:
      - jmca-network

  jmca-redis:
    container_name: jmca-redis
    image: redis:alpine
    ports:
      - "6379:6379"
    networks:
      - jmca-network

volumes:
  jmca_mysql_data:

networks:
  jmca-network:
```

### 3. nginx/conf.d/default.conf
```nginx
server {
    listen 80;
    index index.php index.html;
    server_name localhost;
    root /var/www/html/public;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass ccms-app:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

### 4. .dockerignore
```
node_modules
vendor
.git
.env
.dockerignore
Dockerfile
docker-compose.yml
README.md
storage/logs/*
storage/framework/cache/*
storage/framework/sessions/*
storage/framework/views/*
public/storage
```

## 🚀 Setup Process

### Step 1: Create Directory Structure
```bash
# Create nginx config directory
mkdir -p nginx/conf.d

# Create Laravel directories (if not exists)
mkdir -p storage/logs
mkdir -p storage/framework/{cache,sessions,views}
mkdir -p public
```

### Step 2: Create Files
1. Create `Dockerfile` sa project root
2. Create `docker-compose.yml` sa project root  
3. Create `nginx/conf.d/default.conf`
4. Create `.dockerignore` sa project root

### Step 3: Build and Run
```bash
# Build containers
docker-compose build

# Start all services
docker-compose up -d

# Check running containers
docker-compose ps
```

## 🔧 Essential Commands

### Container Management
```bash
# Start services
docker-compose up -d

# Stop services
docker-compose down

# Rebuild containers
docker-compose build --no-cache

# View logs
docker-compose logs -f [service-name]

# Restart specific service
docker-compose restart jmca-app
```

### Laravel Commands
```bash
# Execute commands inside app container
docker-compose exec jmca-app php artisan migrate
docker-compose exec jmca-app php artisan key:generate
docker-compose exec jmca-app composer install
docker-compose exec jmca-app npm install

# Access container shell
docker-compose exec jmca-app bash
```

### Database Management
```bash
# MySQL commands
docker-compose exec jmca-db mysql -u jmca_user -p jmca_database

# Import database
docker-compose exec -T jmca-db mysql -u jmca_user -p jmca_password jmca_database < backup.sql

# Export database
docker-compose exec jmca-db mysqldump -u jmca_user -p jmca_password jmca_database > backup.sql
```

## 🌐 Access Points

| Service | URL | Port |
|---------|-----|------|
| **Laravel App** | http://localhost:8080 | 8080 |
| **phpMyAdmin** | http://localhost:8081 | 8081 |
| **MySQL** | localhost:3306 | 3306 |
| **Redis** | localhost:6379 | 6379 |

## 🔑 Default Credentials

### Database
- **Host**: jmca-db (internal) / localhost (external)
- **Database**: jmca_database
- **Username**: jmca_user
- **Password**: jmca_password
- **Root Password**: root_password

### phpMyAdmin
- **Server**: jmca-db
- **Username**: jmca_user / root
- **Password**: jmca_password / root_password

## 🛠️ Troubleshooting

### Common Issues

#### Port Already in Use
```bash
# Check what's using the port
netstat -tulpn | grep :8080

# Kill process using port
sudo kill -9 <PID>
```

#### Permission Issues
```bash
# Fix storage permissions
docker-compose exec jmca-app chown -R www-data:www-data /var/www/html/storage
docker-compose exec jmca-app chmod -R 755 /var/www/html/storage
```

#### Database Connection Issues
```bash
# Check if database is ready
docker-compose exec jmca-db mysql -u root -proot_password -e "SHOW DATABASES;"

# Restart database service
docker-compose restart jmca-db
```

#### Clear Laravel Caches
```bash
docker-compose exec jmca-app php artisan cache:clear
docker-compose exec jmca-app php artisan config:clear
docker-compose exec jmca-app php artisan route:clear
docker-compose exec jmca-app php artisan view:clear
```

### Container Health Checks
```bash
# Check container status
docker-compose ps

# Check container resource usage
docker stats

# View container details
docker inspect jmca-app
```

## 📝 Development Workflow

### 1. Daily Startup
```bash
cd project-directory
docker-compose up -d
```

### 2. Making Changes
- Edit files normally in host machine
- Changes are automatically synced via volumes
- Restart services kung may config changes

### 3. Database Changes
```bash
# Run migrations
docker-compose exec jmca-app php artisan migrate

# Seed database
docker-compose exec jmca-app php artisan db:seed
```

### 4. Frontend Changes
```bash
# Install new packages
docker-compose exec jmca-app npm install

# Build assets
docker-compose exec jmca-app npm run dev
# or for production
docker-compose exec jmca-app npm run build
```

### 5. PHP Dependencies
```bash
# Install new packages
docker-compose exec jmca-app composer install

# Add new package
docker-compose exec jmca-app composer require package-name
```

## 🔄 Environment Variables

### Laravel .env Setup
```env
APP_NAME=JMCA
APP_ENV=local
APP_KEY=base64:your-generated-key
APP_DEBUG=true
APP_URL=http://localhost:8080

DB_CONNECTION=mysql
DB_HOST=ccms-db
DB_PORT=3306
DB_DATABASE=jmca_database
DB_USERNAME=jmca_user
DB_PASSWORD=jmca_password

CACHE_DRIVER=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=redis

REDIS_HOST=ccms-redis
REDIS_PASSWORD=null
REDIS_PORT=6379
```

## 🚦 Production Considerations

### Production docker-compose.yml Changes
```yaml
# Remove port mappings for db and redis
# Add restart policies
# Use production environment variables
# Remove development volumes
```

### Security
- Change default passwords
- Remove phpMyAdmin in production
- Use secrets keys for sensitive data
- Enable SSL/TLS

## 📚 Additional Resources

- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Reference](https://docs.docker.com/compose/)
- [Laravel Documentation](https://laravel.com/docs)
- [Nginx Configuration](https://nginx.org/en/docs/)

---

**Note**: Please Update the credentials and configuration based on specific requirements of your project. This setup optimized the development environment.
