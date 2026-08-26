# DATABASE.md - Database Schema Reference

## Overview

- **Engine:** MySQL 8.0
- **Database Name:** `station_map`
- **Connection:** Docker container (host: `mysql`, port: `3306`)

---

## Table: `users`

```sql
CREATE TABLE users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role ENUM('admin', 'user') DEFAULT 'user',
    status ENUM('active', 'locked') DEFAULT 'active',
    remember_token VARCHAR(100) NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);
```

| Column | Type | Default | Description |
|--------|------|---------|-------------|
| id | BIGINT PK | auto | Primary key |
| name | VARCHAR(255) | - | User full name |
| email | VARCHAR(255) | - | Unique email (login) |
| password | VARCHAR(255) | - | Hashed password |
| role | ENUM | 'user' | `admin` or `user` |
| status | ENUM | 'active' | `active` or `locked` |

**Seed Data (Admin):**
```php
DB::table('users')->insert([
    'name' => 'Admin',
    'email' => 'admin@test.com',
    'password' => bcrypt('password'),
    'role' => 'admin',
    'status' => 'active',
]);
```

---

## Table: `stations`

```sql
CREATE TABLE stations (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    code VARCHAR(50) UNIQUE NULL,
    name VARCHAR(255) NOT NULL,
    latitude DECIMAL(10, 7) NOT NULL,
    longitude DECIMAL(10, 7) NOT NULL,
    status ENUM('active', 'deploying') DEFAULT 'deploying',
    owner_name VARCHAR(255) NULL,
    owner_phone VARCHAR(50) NULL,
    address TEXT NULL,
    extra_attributes JSON NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);
```

| Column | Type | Default | Description |
|--------|------|---------|-------------|
| id | BIGINT PK | auto | Primary key |
| code | VARCHAR(50) | NULL | Station code (e.g., TR-001) |
| name | VARCHAR(255) | - | Station name |
| latitude | DECIMAL(10,7) | - | GPS latitude |
| longitude | DECIMAL(10,7) | - | GPS longitude |
| status | ENUM | 'deploying' | `active` (green) or `deploying` (yellow) |
| owner_name | VARCHAR(255) | NULL | Land owner name |
| owner_phone | VARCHAR(50) | NULL | Land owner phone |
| address | TEXT | NULL | Station address |
| extra_attributes | JSON | NULL | Additional technical config |

---

## Table: `proposal_stations`

```sql
CREATE TABLE proposal_stations (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,
    latitude DECIMAL(10, 7) NOT NULL,
    longitude DECIMAL(10, 7) NOT NULL,
    owner_name VARCHAR(255) NOT NULL,
    owner_phone VARCHAR(50) NOT NULL,
    location_attributes JSON NULL,
    extra_attributes JSON NULL,
    images JSON NULL,
    documents JSON NULL,
    status ENUM('pending', 'approved', 'rejected') DEFAULT 'pending',
    admin_note TEXT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

| Column | Type | Default | Description |
|--------|------|---------|-------------|
| id | BIGINT PK | auto | Primary key |
| user_id | BIGINT FK | - | References `users.id` (CASCADE DELETE) |
| latitude | DECIMAL(10,7) | - | GPS latitude |
| longitude | DECIMAL(10,7) | - | GPS longitude |
| owner_name | VARCHAR(255) | - | Land owner name |
| owner_phone | VARCHAR(50) | - | Land owner phone |
| location_attributes | JSON | NULL | Dynamic location info (land type, road width, power grid...) |
| extra_attributes | JSON | NULL | Additional info |
| images | JSON | NULL | Array of image file paths |
| documents | JSON | NULL | Array of document file paths |
| status | ENUM | 'pending' | `pending`, `approved`, `rejected` |
| admin_note | TEXT | NULL | Admin rejection reason |

### JSON Column Examples

**location_attributes:**
```json
{
    "loai_mat_bang": "dat_nha_o",
    "do_rong_duong": "6m",
    "hien_trang_luoi_dien": "Da co tram bien ap"
}
```

**images:**
```json
[
    "proposals/images/img1.jpg",
    "proposals/images/img2.jpg"
]
```

---

## Relationships

```
users (1) ──────── (N) proposal_stations
   │                      │
   │ user_id (FK)         │
   └──────────────────────┘

proposal_stations (approve action) ──→ stations (copy data)
```

## Migration Commands

```bash
# Inside container: docker exec -it station-app bash

# Run all migrations
php artisan migrate

# Fresh start with seeders
php artisan migrate:fresh --seed

# Create new migration
php artisan make:migration create_stations_table
```

## Model Casts

```php
// ProposalStation.php
protected $casts = [
    'location_attributes' => 'array',
    'extra_attributes' => 'array',
    'images' => 'array',
    'documents' => 'array',
];

// Station.php
protected $casts = [
    'extra_attributes' => 'array',
];
```
