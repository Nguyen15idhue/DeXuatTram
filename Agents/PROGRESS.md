# PROGRESS.md - Project Progress Tracker

> Updated: ___/___/______

## Overall Status: NOT STARTED

---

## Phase 0: Docker Setup

| Task | Status | Notes |
|------|--------|-------|
| Install Docker Desktop | ⬜ Not Started | |
| Create docker-compose.yml | ⬜ Not Started | |
| Create docker/Dockerfile | ⬜ Not Started | |
| Create docker/nginx/default.conf | ⬜ Not Started | |
| Create .env.docker | ⬜ Not Started | |
| Build containers | ⬜ Not Started | |
| Test http://localhost:8080 | ⬜ Not Started | |

---

## Phase 1: Laravel Setup (inside container)

| Task | Status | Notes |
|------|--------|-------|
| composer create-project | ⬜ Not Started | |
| php artisan key:generate | ⬜ Not Started | |
| php artisan storage:link | ⬜ Not Started | |
| Install FilamentPHP | ⬜ Not Started | |
| Install Breeze | ⬜ Not Started | |
| Install maatwebsite/excel | ⬜ Not Started | |
| Create public/markers/ | ⬜ Not Started | |

---

## Phase 2: Database

| Task | Status | Notes |
|------|--------|-------|
| Migration: users (add role, status) | ⬜ Not Started | |
| Model: User.php | ⬜ Not Started | |
| Migration: stations | ⬜ Not Started | |
| Model: Station.php | ⬜ Not Started | |
| Migration: proposal_stations | ⬜ Not Started | |
| Model: ProposalStation.php | ⬜ Not Started | |
| Seeder: AdminSeeder | ⬜ Not Started | |
| Run migrate:fresh --seed | ⬜ Not Started | |

---

## Phase 3: Admin Panel (FilamentPHP)

| Task | Status | Notes |
|------|--------|-------|
| UserResource - Form & Table | ⬜ Not Started | |
| UserResource - Lock/Unlock action | ⬜ Not Started | |
| StationResource - Form & Table | ⬜ Not Started | |
| StationResource - Import Excel | ⬜ Not Started | |
| StationResource - Export Excel | ⬜ Not Started | |
| ProposalStationResource - Form & Table | ⬜ Not Started | |
| ProposalStationResource - Approve action | ⬜ Not Started | |
| ProposalStationResource - Reject action | ⬜ Not Started | |
| ProposalStationResource - Export Excel | ⬜ Not Started | |

---

## Phase 4: Frontend (Map + Form)

| Task | Status | Notes |
|------|--------|-------|
| DashboardController | ⬜ Not Started | |
| Route /dashboard | ⬜ Not Started | |
| Leaflet CDN + Map container | ⬜ Not Started | |
| JS: Initialize map + 4 icons | ⬜ Not Started | |
| JS: Render official stations | ⬜ Not Started | |
| JS: Render my proposals (orange) | ⬜ Not Started | |
| JS: Render other proposals (gray) | ⬜ Not Started | |
| JS: Click map event | ⬜ Not Started | |
| Livewire: ProposalModal class | ⬜ Not Started | |
| Livewire: ProposalModal view | ⬜ Not Started | |
| Submit proposal form | ⬜ Not Started | |
| Checkbox filter for markers | ⬜ Not Started | |
| Personal proposal list | ⬜ Not Started | |

---

## Phase 5: Auth & Middleware

| Task | Status | Notes |
|------|--------|-------|
| CheckActiveUser middleware | ⬜ Not Started | |
| Register middleware | ⬜ Not Started | |
| Route permissions | ⬜ Not Started | |
| Test: user can't access /admin | ⬜ Not Started | |
| Test: locked user logout | ⬜ Not Started | |

---

## Phase 6: Testing

| Task | Status | Notes |
|------|--------|-------|
| Flow: Register → Login → Map | ⬜ Not Started | |
| Flow: Click map → Submit proposal | ⬜ Not Started | |
| Flow: Admin approve → Station created | ⬜ Not Started | |
| Flow: Admin reject → Status updated | ⬜ Not Started | |
| Upload images/documents | ⬜ Not Started | |
| Import/Export Excel | ⬜ Not Started | |

---

## Summary

| Phase | Status | Time Spent |
|-------|--------|------------|
| 0. Docker | ⬜ Not Started | 0 min |
| 1. Laravel Setup | ⬜ Not Started | 0 min |
| 2. Database | ⬜ Not Started | 0 min |
| 3. Admin Panel | ⬜ Not Started | 0 min |
| 4. Frontend | ⬜ Not Started | 0 min |
| 5. Auth | ⬜ Not Started | 0 min |
| 6. Testing | ⬜ Not Started | 0 min |
| **TOTAL** | ⬜ Not Started | **0 min** |
