# COMMANDS.md - Common Commands Reference

## Docker Commands

```bash
# Start/Stop
docker-compose up -d                    # Start all containers
docker-compose down                     # Stop all containers
docker-compose up -d --build            # Rebuild and start

# Enter containers
docker exec -it station-app bash        # Enter PHP-FPM container
docker exec -it station-mysql mysql -u station_user -p station_map  # Enter MySQL

# Logs
docker-compose logs -f app              # Follow app logs
docker-compose logs -f nginx            # Follow nginx logs
docker-compose logs -f mysql            # Follow mysql logs

# Status
docker ps                               # List running containers
docker stats                            # Resource usage
```

## Laravel Artisan (inside container)

```bash
# Must run inside: docker exec -it station-app bash

# Database
php artisan migrate                     # Run migrations
php artisan migrate:fresh               # Drop all and re-run
php artisan migrate:fresh --seed        # Drop, re-run, and seed
php artisan make:migration create_xxx_table  # Create migration

# Models & Resources
php artisan make:model Station -m       # Create model with migration
php artisan make:filament-resource Station  # Create Filament resource
php artisan make:controller DashboardController  # Create controller
php artisan make:livewire ProposalModal  # Create Livewire component
php artisan make:seeder AdminSeeder     # Create seeder

# Setup
php artisan key:generate                # Generate APP_KEY
php artisan storage:link                # Create storage symlink
php artisan serve                       # Start dev server (not needed in Docker)

# Filament
php artisan filament:install --panels   # Install Filament admin panel

# Breeze
php artisan breeze:install blade        # Install Breeze with Blade

# Cache
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear
```

## Composer Commands (inside container)

```bash
# Must run inside: docker exec -it station-app bash

composer install                        # Install dependencies
composer install --no-dev               # Production install
composer update                         # Update dependencies
composer require filament/filament:"^3.2" -W  # Install Filament
composer require laravel/breeze --dev   # Install Breeze
composer require maatwebsite/excel      # Install Excel
```

## NPM Commands (inside container, if needed)

```bash
npm install
npm run dev
npm run build
```

## Git Commands (on host)

```bash
git init
git add .
git commit -m "Initial commit"
git remote add origin <url>
git push -u origin main
```

## Quick Start Sequence

```bash
# 1. Create project
composer create-project laravel/laravel station-map
cd station-map

# 2. Create Docker files (copy from kehoach.md)
# docker-compose.yml, docker/Dockerfile, docker/nginx/default.conf, .env.docker

# 3. Build and start
docker-compose up -d --build

# 4. Enter container and setup
docker exec -it station-app bash
cp .env.docker .env
php artisan key:generate
php artisan storage:link

# 5. Install packages
composer require filament/filament:"^3.2" -W
php artisan filament:install --panels
composer require laravel/breeze --dev
php artisan breeze:install blade
composer require maatwebsite/excel

# 6. Create database objects
php artisan make:migration create_stations_table
php artisan make:migration create_proposal_stations_table
php artisan make:model Station -m
php artisan make:model ProposalStation -m
php artisan make:seeder AdminSeeder

# 7. Run migrations
php artisan migrate:fresh --seed

# 8. Create Filament resources
php artisan make:filament-resource User
php artisan make:filament-resource Station
php artisan make:filament-resource ProposalStation

# 9. Create frontend
php artisan make:controller DashboardController
php artisan make:livewire ProposalModal

# 10. Exit container and test
exit
# Open http://localhost:8080
```
