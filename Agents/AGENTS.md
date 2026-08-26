# AGENTS.md - Station Map Project

## Project Overview

This is a **Laravel 11 + FilamentPHP v3** web application for managing and proposing station locations on an interactive map. The system runs in **Docker** with 3 containers: PHP-FPM, Nginx, MySQL.

## Tech Stack

| Component | Technology |
|-----------|------------|
| Backend | Laravel 11 |
| Admin Panel | FilamentPHP v3 |
| Auth | Laravel Breeze (Blade) |
| Frontend | Blade + Livewire + TailwindCSS |
| Map | Leaflet.js (OpenStreetMap, no API key) |
| Database | MySQL 8.0 (Docker) |
| Excel | maatwebsite/excel |
| Container | Docker Compose |

## Architecture

```
Browser → Nginx:8080 → PHP-FPM:9000 → Laravel → MySQL
```

## Key Files Reference

- **Database Schema:** See `Agents/DATABASE.md`
- **Docker Config:** See `Agents/DOCKER.md`
- **Commands:** See `Agents/COMMANDS.md`
- **Full Plan:** See `kehoach.md`

## Running Commands

All artisan commands must run inside the Docker container:

```bash
# Enter container
docker exec -it station-app bash

# Run artisan commands inside container
php artisan migrate:fresh --seed
php artisan make:filament-resource Station
```

## File Structure

```
project-root/
├── Agents/                    # AI assistant config
├── docker/
│   ├── Dockerfile
│   └── nginx/default.conf
├── app/
│   ├── Http/
│   │   ├── Controllers/
│   │   │   └── DashboardController.php
│   │   └── Middleware/
│   │       └── CheckActiveUser.php
│   ├── Models/
│   │   ├── User.php
│   │   ├── Station.php
│   │   └── ProposalStation.php
│   └── Filament/
│       └── Resources/
│           ├── UserResource.php
│           ├── StationResource.php
│           └── ProposalStationResource.php
├── database/
│   ├── migrations/
│   └── seeders/
├── resources/
│   ├── views/
│   │   ├── dashboard.blade.php
│   │   └── livewire/
│   │       └── proposal-modal.blade.php
│   └── css/
├── docker-compose.yml
├── .env.docker
├── kehoach.md
├── cac-buoc-lam.md
└── bao-cao-ket-qua.md
```

## Business Rules

### User Role
- Register/Login (Breeze)
- View map with 4 marker colors: Green (active), Yellow (deploying), Orange (my proposals), Gray (others' proposals)
- Click map → fill form → submit proposal
- View own proposals in list below map
- Filter map markers by type

### Admin Role
- CRUD Users (lock/unlock accounts)
- CRUD Stations (import/export Excel)
- CRUD Proposals (approve → create Station, reject with reason)
- View gallery/images preview

## Database Tables

1. `users` - id, name, email, password, role (admin|user), status (active|locked)
2. `stations` - id, code, name, latitude, longitude, status (active|deploying), owner_name, owner_phone, address, extra_attributes (json)
3. `proposal_stations` - id, user_id (FK), latitude, longitude, owner_name, owner_phone, location_attributes (json), extra_attributes (json), images (json), documents (json), status (pending|approved|rejected), admin_note

## Conventions

- Use Vietnamese for UI labels
- Use English for code/variables
- JSON columns for dynamic attributes
- FilamentPHP for admin CRUD
- Livewire for interactive frontend components
