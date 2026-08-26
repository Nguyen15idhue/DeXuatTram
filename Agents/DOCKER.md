# DOCKER.md - Docker Configuration Reference

## Services

| Service | Container Name | Image | Port | Purpose |
|---------|---------------|-------|------|---------|
| app | station-app | PHP 8.2-FPM | 9000 (internal) | Laravel backend |
| nginx | station-nginx | nginx:alpine | 8080 → 80 | Web server |
| mysql | station-mysql | mysql:8.0 | 3306 → 3306 | Database |

## Network

- **Network Name:** `station-network`
- **Driver:** bridge
- All services connected to same network

## Volumes

| Volume | Container Path | Purpose |
|--------|---------------|---------|
| . (source) | /var/www | Laravel source code (bind mount) |
| php.ini | /usr/local/etc/php/conf.d/php.ini | PHP configuration |
| nginx conf | /etc/nginx/conf.d/default.conf | Nginx configuration |
| mysql-data | /var/lib/mysql | MySQL data persistence |

## Files

```
project-root/
├── docker-compose.yml          # Main Docker config
├── docker/
│   ├── Dockerfile              # PHP-FPM image build
│   ├── nginx/
│   │   └── default.conf        # Nginx reverse proxy config
│   └── php/
│       └── php.ini             # PHP settings
└── .env.docker                 # Environment variables
```

## Environment Variables (.env.docker)

```env
APP_NAME="Station Map"
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost:8080

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=station_map
DB_USERNAME=station_user
DB_PASSWORD=station_password

SESSION_DRIVER=file
CACHE_DRIVER=file
QUEUE_CONNECTION=sync

FILAMENT_PATH=admin
```

## Common Commands

```bash
# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# Build/rebuild containers
docker-compose up -d --build

# View logs
docker-compose logs -f app
docker-compose logs -f nginx
docker-compose logs -f mysql

# Enter app container
docker exec -it station-app bash

# Enter MySQL container
docker exec -it station-mysql mysql -u station_user -p station_map

# Run Laravel commands inside container
docker exec -it station-app php artisan migrate:fresh --seed
docker exec -it station-app php artisan key:generate
docker exec -it station-app php artisan storage:link

# Check container status
docker ps

# Restart single service
docker-compose restart app
docker-compose restart nginx
```

## Port Mapping

| Service | Container Port | Host Port | URL |
|---------|---------------|-----------|-----|
| nginx | 80 | 8080 | http://localhost:8080 |
| mysql | 3306 | 3306 | localhost:3306 |

## Dockerfile Details

- **Base Image:** php:8.2-fpm
- **PHP Extensions:** pdo_mysql, mbstring, exif, pcntl, bcmath, gd, zip, intl
- **Composer:** Installed via multi-stage copy
- **Working Directory:** /var/www
- **Permissions:** storage/ and bootstrap/cache/ owned by www-data

## Troubleshooting

```bash
# If containers won't start
docker-compose down -v  # Remove volumes too
docker-compose up -d --build

# Check PHP extensions
docker exec -it station-app php -m

# Check MySQL connection
docker exec -it station-app php artisan tinker
>>> DB::connection()->getPdo();

# View container resources
docker stats
```
