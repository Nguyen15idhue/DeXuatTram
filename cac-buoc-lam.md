# CÁC BƯỚC CẦN LÀM - CHECKLIST TRIỂN KHAI

---

## BƯỚC 0: Cài đặt Docker (~15 phút)

- [ ] Cài Docker Desktop cho Windows: https://docs.docker.com/desktop/install/windows-install/
- [ ] Cài Docker Compose (kèm theo Docker Desktop)
- [ ] Kiểm tra: `docker --version` và `docker-compose --version`

**Kết quả:** Docker Desktop chạy OK, lệnh `docker ps` hoạt động

---

## BƯỚC 1: Khởi tạo Docker + Laravel + Thư viện (~45 phút)

### 1.1. Tạo project Laravel
- [ ] `composer create-project laravel/laravel station-map`
- [ ] `cd station-map`

### 1.2. Tạo cấu trúc Docker
- [ ] `mkdir -p docker/nginx docker/php`
- [ ] Tạo file `docker-compose.yml` (3 services: app, nginx, mysql)
- [ ] Tạo file `docker/Dockerfile` (PHP 8.2-FPM + extensions)
- [ ] Tạo file `docker/nginx/default.conf` (reverse proxy config)
- [ ] Tạo file `.env.docker` (environment variables)

### 1.3. Build & khởi động Docker
- [ ] `docker-compose up -d --build`
- [ ] Kiểm tra: `docker ps` thấy 3 containers chạy (station-app, station-nginx, station-mysql)
- [ ] Truy cập http://localhost:8080 thấy trang mặc định

### 1.4. Cài đặt Laravel bên trong container
- [ ] `docker exec -it station-app bash`
- [ ] `cp .env.docker .env`
- [ ] `php artisan key:generate`
- [ ] `php artisan storage:link`
- [ ] Cài FilamentPHP: `composer require filament/filament:"^3.2" -W`
- [ ] Cài Filament Panel: `php artisan filament:install --panels`
- [ ] Cài Breeze: `composer require laravel/breeze --dev` → `php artisan breeze:install blade`
- [ ] Cài Excel: `composer require maatwebsite/excel`
- [ ] Tạo thư mục markers: `mkdir -p public/markers`
- [ ] Thoát container: `exit`

**Kết quả:** Truy cập http://localhost:8080 → thấy trang login Laravel, Docker 3 containers chạy OK

---

## BƯỚC 2: Database - Migrations & Models (~30 phút)

> **Lưu ý:** Thực hiện bên trong container `docker exec -it station-app bash`

### 2.1. Bảng `users` (sửa Migration có sẵn)
- [ ] Sửa `database/migrations/xxxx_create_users_table.php`: thêm cột `role` (enum) và `status` (enum)
- [ ] Cập nhật `app/Models/User.php`: thêm `$fillable`, `$casts` cho role/status

### 2.2. Bảng `stations`
- [ ] Tạo migration: `php artisan make:migration create_stations_table`
- [ ] Viết schema: id, code, name, latitude, longitude, status, owner_name, owner_phone, address, extra_attributes, timestamps
- [ ] Tạo Model: `app/Models/Station.php` với `$casts = ['extra_attributes' => 'array']`

### 2.3. Bảng `proposal_stations`
- [ ] Tạo migration: `php artisan make:migration create_proposal_stations_table`
- [ ] Viết schema: id, user_id (FK), latitude, longitude, owner_name, owner_phone, location_attributes, extra_attributes, images, documents, status, admin_note, timestamps
- [ ] Tạo Model: `app/Models/ProposalStation.php` với `$casts` cho location_attributes, extra_attributes, images, documents

### 2.4. Seed dữ liệu mẫu
- [ ] Tạo Seeder admin default: `php artisan make:seeder AdminSeeder`
- [ ] Insert admin: name=admin, email=admin@test.com, password=bcrypt, role=admin, status=active
- [ ] Chạy: `php artisan migrate:fresh --seed`

**Kết quả:** Database có 3 bảng + 1 user admin default, `php artisan tinker` query được

---

## BƯỚC 3: Admin Panel - FilamentPHP (~1.5 giờ)

> **Lưu ý:** Thực hiện bên trong container `docker exec -it station-app bash`

### 3.1. UserResource
- [ ] `php artisan make:filament-resource User`
- [ ] Cấu hình Form: name, email, password, role (select), status (select)
- [ ] Cấu hình Table: columns (name, email, role, status), filters, actions
- [ ] Thêm Action nhanh: **Khóa/Mở khóa** tài khoản

### 3.2. StationResource
- [ ] `php artisan make:filament-resource Station`
- [ ] Cấu hình Form: code, name, latitude, longitude, status, owner_name, owner_phone, address, extra_attributes (KeyValue)
- [ ] Cấu hình Table: columns đầy đủ, search, filters theo status
- [ ] Thêm **Import Excel** action (dùng `Filament\Actions\ImportAction` hoặc `maatwebsite/excel`)
- [ ] Thêm **Export Excel** action

### 3.3. ProposalStationResource
- [ ] `php artisan make:filament-resource ProposalStation`
- [ ] Cấu hình Form: latitude, longitude, owner_name, owner_phone, location_attributes (KeyValue), extra_attributes (KeyValue), images (FileUpload multiple), documents (FileUpload multiple), status, admin_note
- [ ] Cấu hình Table: columns (user_id, owner_name, owner_phone, status, created_at), filters
- [ ] Thêm **Action "Duyệt"**: tạo Station mới từ ProposalStation + update status=approved
- [ ] Thêm **Action "Từ chối"**: form nhập admin_note + update status=rejected
- [ ] Thêm **Export Excel** action
- [ ] Cấu hình xem gallery ảnh preview trong Table/Form

**Kết quả:** Truy cập `/admin` → login admin → thấy 3 Resources, CRUD hoạt động, Import/Export Excel OK

---

## BƯỚC 4: Frontend - Bản đồ Leaflet.js + Form Đề Xuất (~1.5 giờ)

> **Lưu ý:** Thực hiện bên trong container `docker exec -it station-app bash`

### 4.1. Dashboard Controller & Route
- [ ] Tạo Controller: `php artisan make:controller DashboardController`
- [ ] Viết method `index()`: query `$officialStations`, `$myProposals`, `$otherProposals` → compact view
- [ ] Cấu hình Route: `Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard')`

### 4.2. Blade View - Bản đồ
- [ ] Sửa `resources/views/dashboard.blade.php`
- [ ] Thêm Leaflet CDN (CSS + JS)
- [ ] Thêm `<div id="map">` với TailwindCSS styling
- [ ] Viết JS khởi tạo map, tile layer, 4 icon màu
- [ ] Render markers: officialStations (green/yellow), myProposals (orange), otherProposals (gray)
- [ ] Thêm sự kiện `map.on('click')` lấy tọa độ

### 4.3. Modal Form Đề Xuất (Livewire)
- [ ] Tạo Livewire Component: `php artisan make:livewire ProposalModal`
- [ ] Viết class: properties (latitude, longitude, owner_name, owner_phone, location_attributes, images, documents)
- [ ] Viết method `submitProposal()`: validate + create ProposalStation + close modal
- [ ] Viết method `openModal()` / `closeModal()`
- [ ] Tạo view Blade: modal overlay form với đầy đủ trường
- [ ] Tích hợp Livewire vào Dashboard view

### 4.4. Bộ lọc bản đồ
- [ ] Thêm checkbox filter UI (4 loại marker)
- [ ] Viết JS toggle hiển thị/ẩn marker layer theo checkbox

### 4.5. Danh sách đề xuất cá nhân
- [ ] Hiển thị bảng/Card danh sách proposal_stations của user bên dưới map
- [ ] Thêm filter theo status (pending, approved, rejected)

**Kết quả:** User login → thấy bản đồ với 4 màu marker → click bản đồ → mở modal → điền form → submit → marker cam hiện trên map → thấy danh sách bên dưới

---

## BƯỚC 5: Middleware & Phân quyền (~30 phút)

> **Lưu ý:** Thực hiện bên trong container `docker exec -it station-app bash`

### 5.1. Middleware kiểm tra tài khoản khóa
- [ ] Tạo `app/Http/Middleware/CheckActiveUser.php`
- [ ] Kiểm tra `auth()->user()->status === 'locked'` → logout + redirect login
- [ ] Đăng ký Middleware vào `bootstrap/app.php`

### 5.2. Phân quyền Route
- [ ] Route user: `/dashboard`, `/profile` → group middleware `auth`
- [ ] Route admin: `/admin/*` → group middleware `auth` + kiểm tra role admin (Filament tự xử lý)
- [ ] Trang proposal chỉ user đã đăng nhập mới truy cập được

### 5.3. Kiểm tra phân quyền
- [ ] Test: user thường không vào được `/admin`
- [ ] Test: user bị khóa → logout tự động
- [ ] Test: admin vào được `/admin` và quản lý tất cả

**Kết quả:** Phân quyền hoạt động đúng, user bị khóa không đăng nhập được

---

## BƯỚC 6: Test tổng thể & Fix bug (~30 phút)

> **Lưu ý:** Truy cập http://localhost:8080 để test. Dùng `docker-compose logs -f app` để xem logs lỗi.

- [ ] Test flow User: Đăng ký → Login → Xem bản đồ → Click map → Điền form → Gửi đề xuất → Xem danh sách
- [ ] Test flow Admin: Login → Xem danh sách đề xuất → Duyệt/Từ chối → Quản lý trạm → Import/Export Excel
- [ ] Test phân quyền: User không vào admin, user bị khóa logout
- [ ] Test upload ảnh/giấy tờ: upload thành công, xem được preview
- [ ] Fix các bug phát sinh

**Kết quả:** Hệ thống hoạt động hoàn chỉnh trên Docker, không lỗi nghiêm trọng

---

## TỔNG THỜI GIAN DỰ KIẾN: ~4-5.5 giờ

| Bước | Thời gian |
|------|-----------|
| Bước 0: Cài Docker | 15 phút |
| Bước 1: Docker + Laravel | 45 phút |
| Bước 2: Database | 30 phút |
| Bước 3: Admin Panel | 1.5 giờ |
| Bước 4: Frontend Map | 1.5 giờ |
| Bước 5: Middleware | 30 phút |
| Bước 6: Test | 30 phút |

### Các lệnh Docker thường dùng

```bash
# Khởi động
docker-compose up -d

# Dừng
docker-compose down

# Vào container app
docker exec -it station-app bash

# Xem logs
docker-compose logs -f app
docker-compose logs -f nginx

# Build lại
docker-compose up -d --build

# Reset database
docker exec -it station-app php artisan migrate:fresh --seed
```
