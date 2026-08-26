# KẾ HOẠCH TRIỂN KHAI HỆ THỐNG QUẢN LÝ VÀ ĐỀ XUẤT TRẠM

---

## 1. Môi trường & Kiến trúc Kỹ thuật

* **Containerization:** Docker Compose (3 services: app, nginx, mysql)
* **Backend & Admin Framework:** Laravel 11 + FilamentPHP v3 (Dựng Admin Panel tự động).
* **Database:** MySQL 8.0 (trong Docker container).
* **Frontend User:** Blade Template + Livewire/TailwindCSS + Leaflet.js (Bản đồ không cần API Key).
* **File Storage:** Local Disk (`storage/app/public`) lưu trữ hình ảnh và tài liệu đính kèm.
* **Excel Engine:** `maatwebsite/excel` hoặc `Filament Import/Export Actions`.

### 1.1. Kiến trúc Docker

```
┌─────────────────────────────────────────────────────────┐
│                    docker-compose.yml                   │
├─────────────┬─────────────┬─────────────────────────────┤
│   Service   │    Image    │           Port              │
├─────────────┼─────────────┼─────────────────────────────┤
│    app      │ PHP 8.2-FPM │    (internal: 9000)         │
│    nginx    │ nginx:alpine│    8080 → 80                │
│    mysql    │ mysql:8.0   │    3306 → 3306              │
└─────────────┴─────────────┴─────────────────────────────┘

Flow Request: Browser → Nginx:8080 → (proxy_pass) → PHP-FPM:9000 → Laravel → MySQL
```

### 1.2. Cấu trúc thư mục Docker

```
project-root/
├── docker-compose.yml
├── docker/
│   ├── Dockerfile          # Build PHP-FPM + extensions
│   ├── nginx/
│   │   └── default.conf    # Config Nginx reverse proxy
│   └── php/
│       └── php.ini         # PHP configuration
├── .env.docker             # Environment variables cho Docker
├── app/                    # Laravel source code (mounted volume)
├── ...
```

---

## 2. Cấu trúc Cơ sở Dữ liệu (Database Schema)

### Bảng 1: `users` (Quản lý tài khoản)

```php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->string('password');
    $table->enum('role', ['admin', 'user'])->default('user');
    $table->enum('status', ['active', 'locked'])->default('active');
    $table->rememberToken();
    $table->timestamps();
});

```

### Bảng 2: `stations` (Dữ liệu trạm chính thức)

```php
Schema::create('stations', function (Blueprint $table) {
    $table->id();
    $table->string('code')->nullable()->unique(); // Mã trạm (Ví dụ: TR-001)
    $table->string('name');                       // Tên trạm chính thức
    $table->decimal('latitude', 10, 7);
    $table->decimal('longitude', 10, 7);
    $table->enum('status', ['active', 'deploying'])->default('deploying'); // Xanh: active, Vàng: deploying
    $table->string('owner_name')->nullable();
    $table->string('owner_phone')->nullable();
    $table->text('address')->nullable();
    $table->json('extra_attributes')->nullable(); // Các thuộc tính cấu hình kỹ thuật bổ sung
    $table->timestamps();
});

```

### Bảng 3: `proposal_stations` (Dữ liệu trạm đề xuất)

```php
Schema::create('proposal_stations', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained('users')->onDelete('cascade');
    
    // Tọa độ & Thông tin liên hệ cơ bản
    $table->decimal('latitude', 10, 7);
    $table->decimal('longitude', 10, 7);
    $table->string('owner_name');
    $table->string('owner_phone');
    
    // Các trường thông tin động & File Upload
    $table->json('location_attributes')->nullable(); // Thông tin động về vị trí (Loại mặt bằng, độ rộng đường, hiện trạng lưới điện...)
    $table->json('extra_attributes')->nullable();    // Thông tin mở rộng khác
    $table->json('images')->nullable();              // Mảng đường dẫn ảnh chụp thực địa/mặt bằng
    $table->json('documents')->nullable();           // Mảng đường dẫn tệp giấy tờ liên quan (Sổ đỏ, hợp đồng...)
    
    // Trạng thái xử lý của Admin
    $table->enum('status', ['pending', 'approved', 'rejected'])->default('pending');
    $table->text('admin_note')->nullable();          // Phản hồi hoặc lý do từ chối của Admin
    $table->timestamps();
});

```

---

## 3. Quy trình & Luồng Nghiệp vụ

```
[ User ] --(Click vị trí trên Map)--> [ Form Đề Xuất ] --(Upload File & Điền TT)--> [ Bảng proposal_stations ]
                                                                                              |
                                                                                      (Admin kiểm tra)
                                                                                              |
[ Bảng stations ] <--(Copy dữ liệu sang & Đổi status: Active/Deploying)-- [ Action: Duyệt Đề Xuất ]

```

### 3.1. Phân hệ Người dùng (User Frontend)

1. **Đăng ký / Đăng nhập / Sửa profile:** Thông qua Laravel Breeze UI.
2. **Bản đồ tương tác (Leaflet.js):**
* **Hiển thị điểm:**
* Trạm chính thức (`stations`): Marker **Xanh** (`active`), Marker **Vàng** (`deploying`).
* Trạm đề xuất của **chính user đang đăng nhập** (`proposal_stations` where `user_id = auth()->id()`): Marker **Cam** (`pending`).
* Trạm đề xuất của **người dùng khác** (`proposal_stations` where `user_id != auth()->id()`): Marker **Xám** (`pending`).

> **Ghi chú màu sắc trên bản đồ:**
> | Màu | Ý nghĩa |
> |---|---|
> | 🟢 Xanh | Trạm đang hoạt động (`stations` - `active`) |
> | 🟡 Vàng | Trạm đang triển khai (`stations` - `deploying`) |
> | 🟠 Cam | Đề xuất của **bạn** |
> | ⚪ Xám | Đề xuất của **người khác** |


* **Tương tác click chọn tọa độ:** Click bất kỳ trên bản đồ $\rightarrow$ Lấy `lat`, `lng` $\rightarrow$ Mở Modal Form đề xuất.


3. **Gửi form đề xuất trạm:**
* Tọa độ tự động điền.
* Điền Họ tên chủ mặt bằng, Số điện thoại.
* Điền các trường thuộc tính vị trí động (`location_attributes`).
* Tải lên danh sách **Hình ảnh thực địa** và **Giấy tờ pháp lý**.


4. **Theo dõi & Lọc:**
* Danh sách trạm đề xuất cá nhân hiển thị dưới dạng Bảng/Card phía dưới Map.
* **Bộ lọc trên bản đồ (Checkbox):** Cho phép bật/tắt hiển thị từng loại marker:
  * ☑ Trạm hoạt động (Xanh)
  * ☑ Trạm triển khai (Vàng)
  * ☑ Đề xuất của tôi (Cam)
  * ☑ Đề xuất của người khác (Xám)



### 3.2. Phân hệ Quản trị (Admin Panel - FilamentPHP)

1. **Quản lý Tài khoản (`UserResource`):**
* Xem danh sách, tạo mới, chỉnh sửa, phân quyền (`role`).
* Nút thao tác nhanh: **Khóa / Mở khóa** tài khoản (`status`).


2. **Quản lý Trạm chính thức (`StationResource`):**
* CRUD thông tin trạm.
* **Import Excel:** Tải file `.xlsx` danh sách trạm $\rightarrow$ Chèn hàng loạt vào bảng `stations`.
* **Export Excel:** Xuất toàn bộ dữ liệu trạm chính thức ra file Excel theo bộ lọc.


3. **Quản lý Đề xuất trạm (`ProposalStationResource`):**
* Xem danh sách đề xuất từ toàn bộ người dùng.
* Hỗ trợ xem gallery ảnh thực địa và preview/tải về các tệp giấy tờ liên quan.
* **Action "Duyệt đề xuất":** Khi bấm Duyệt $\rightarrow$ Hệ thống tự chuyển dữ liệu sang bảng `stations` với trạng thái `deploying` hoặc `active`, đồng thời cập nhật `proposal_stations.status = 'approved'`.
* **Action "Từ chối đề xuất":** Khi bấm Từ chối $\rightarrow$ Hiển thị form nhập lý do (`admin_note`) $\rightarrow$ Cập nhật `proposal_stations.status = 'rejected'` và `admin_note`.
* **Export Excel:** Xuất danh sách đề xuất phục vụ báo cáo.



---

## 4. Kế hoạch Thực thi Chi tiết (Các bước cài đặt)

### Bước 1: Khởi tạo Docker + Laravel + Thư viện (45 phút)

#### 1.1. Tạo project Laravel

```bash
composer create-project laravel/laravel station-map
cd station-map
```

#### 1.2. Tạo cấu trúc Docker

```bash
mkdir -p docker/nginx docker/php
```

#### 1.3. File `docker-compose.yml`

```yaml
version: '3.8'

services:
  # PHP-FPM Application
  app:
    build:
      context: .
      dockerfile: docker/Dockerfile
    container_name: station-app
    restart: unless-stopped
    working_dir: /var/www
    volumes:
      - .:/var/www
      - ./docker/php/php.ini:/usr/local/etc/php/conf.d/php.ini
    networks:
      - station-network
    depends_on:
      - mysql

  # Nginx Web Server
  nginx:
    image: nginx:alpine
    container_name: station-nginx
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - .:/var/www
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    networks:
      - station-network
    depends_on:
      - app

  # MySQL Database
  mysql:
    image: mysql:8.0
    container_name: station-mysql
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: station_map
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_USER: station_user
      MYSQL_PASSWORD: station_password
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - station-network

networks:
  station-network:
    driver: bridge

volumes:
  mysql-data:
```

#### 1.4. File `docker/Dockerfile`

```dockerfile
FROM php:8.2-fpm

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    libzip-dev \
    zip \
    unzip \
    libicu-dev \
    && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd zip intl \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Set working directory
WORKDIR /var/www

# Copy existing application directory contents
COPY . /var/www

# Install Laravel dependencies
RUN composer install --no-dev --optimize-autoloader --no-interaction

# Set permissions
RUN chown -R www-data:www-data /var/www/storage /var/www/bootstrap/cache

EXPOSE 9000
CMD ["php-fpm"]
```

#### 1.5. File `docker/nginx/default.conf`

```nginx
server {
    listen 80;
    server_name localhost;
    root /var/www/public;

    index index.php index.html;

    charset utf-8;
    client_max_body_size 64M;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }

    # Static files caching
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

#### 1.6. File `.env.docker`

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

# FilamentPHP
FILAMENT_PATH=admin
```

#### 1.7. Khởi động Docker & Cài đặt Laravel

```bash
# Build và khởi động containers
docker-compose up -d --build

# Vào container app
docker exec -it station-app bash

# Trong container - Tạo .env và generate key
cp .env.docker .env
php artisan key:generate

# Tạo storage link
php artisan storage:link

# Install FilamentPHP
composer require filament/filament:"^3.2" -W
php artisan filament:install --panels

# Install Breeze & Excel
composer require laravel/breeze --dev
php artisan breeze:install blade
composer require maatwebsite/excel

# Tạo thư mục markers
mkdir -p public/markers

# Truy cập: http://localhost:8080
```

#### 1.8. Các lệnh Docker thường dùng

```bash
# Khởi động tất cả services
docker-compose up -d

# Dừng tất cả services
docker-compose down

# Xem logs
docker-compose logs -f app
docker-compose logs -f nginx

# Vào container app
docker exec -it station-app bash

# Vào container mysql
docker exec -it station-mysql mysql -u station_user -p station_map

# Build lại khi thay đổi Dockerfile
docker-compose up -d --build

# Reset database
docker exec -it station-app php artisan migrate:fresh --seed
```

### Bước 2: Khai báo Migrations & Models (30 phút)

> **Lưu ý:** Thực hiện bên trong container `docker exec -it station-app bash`

* Tạo các migration theo cấu trúc ở **Mục 2**.
* Cấu hình `$casts` trong các Model tương ứng:

```php
// app/Models/ProposalStation.php
protected $casts = [
    'location_attributes' => 'array',
    'extra_attributes' => 'array',
    'images' => 'array',
    'documents' => 'array',
];

```

### Bước 3: Dựng Admin Panel với Filament (1.5 giờ)

> **Lưu ý:** Thực hiện bên trong container `docker exec -it station-app bash`

Sinh tự động 3 Resource chính:

```bash
php artisan make:filament-resource User
php artisan make:filament-resource Station
php artisan make:filament-resource ProposalStation

```

* **Cấu hình Upload File trên `ProposalStationResource`:**

```php
// Upload nhiều ảnh
Forms\Components\FileUpload::make('images')
    ->label('Ảnh chụp thực địa')
    ->multiple()
    ->image()
    ->directory('proposals/images'),

// Upload nhiều file tài liệu/s sổ đỏ
Forms\Components\FileUpload::make('documents')
    ->label('Giấy tờ liên quan')
    ->multiple()
    ->acceptedFileTypes(['application/pdf', 'image/*', 'application/msword'])
    ->directory('proposals/documents'),

// Nhập thuộc tính động vị trí (KeyValue / Key-Value Field)
Forms\Components\KeyValue::make('location_attributes')
    ->label('Thông tin vị trí bổ sung')
    ->keyLabel('Tiêu chí')
    ->valueLabel('Giá trị'),

```

* **Cấu hình Action Duyệt Đề Xuất trên `ProposalStationResource`:**

```php
Tables\Actions\Action::make('approve')
    ->label('Duyệt trạm')
    ->icon('heroicon-o-check-circle')
    ->color('success')
    ->action(function (ProposalStation $record) {
        // 1. Tạo bản ghi trạm chính thức
        Station::create([
            'name' => 'Trạm đề xuất - ' . $record->owner_name,
            'latitude' => $record->latitude,
            'longitude' => $record->longitude,
            'status' => 'deploying',
            'owner_name' => $record->owner_name,
            'owner_phone' => $record->owner_phone,
            'extra_attributes' => array_merge($record->location_attributes ?? [], $record->extra_attributes ?? []),
        ]);

        // 2. Cập nhật trạng thái đề xuất
        $record->update(['status' => 'approved']);
    })
    ->requiresConfirmation();

// Action Từ chối đề xuất
Tables\Actions\Action::make('reject')
    ->label('Từ chối')
    ->icon('heroicon-o-x-circle')
    ->color('danger')
    ->form([
        Forms\Components\Textarea::make('admin_note')
            ->label('Lý do từ chối')
            ->required()
            ->rows(3),
    ])
    ->action(function (ProposalStation $record, array $data) {
        $record->update([
            'status' => 'rejected',
            'admin_note' => $data['admin_note'],
        ]);
    })
    ->requiresConfirmation();
```

### Bước 4: Tích hợp Bản đồ Leaflet.js phía Frontend (1.5 giờ)

Xây dựng file `resources/views/dashboard.blade.php`:

1. **Khai báo Leaflet CDN & Container Bản đồ:**

```html
<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

<div id="map" class="w-full h-[600px] rounded-lg"></div>

```

2. **JavaScript khởi tạo Map và Marker:**

```javascript
const map = L.map('map').setView([10.762622, 106.660172], 13); // Tọa độ mặc định

L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© OpenStreetMap contributors'
}).addTo(map);

// Khởi tạo các Icon màu
const greenIcon = L.icon({ iconUrl: '/markers/marker-green.png', iconSize: [25, 41] });
const yellowIcon = L.icon({ iconUrl: '/markers/marker-yellow.png', iconSize: [25, 41] });
const orangeIcon = L.icon({ iconUrl: '/markers/marker-orange.png', iconSize: [25, 41] });
const grayIcon = L.icon({ iconUrl: '/markers/marker-gray.png', iconSize: [25, 41] });

// Render Trạm chính thức từ Backend
const officialStations = @json($officialStations);
officialStations.forEach(station => {
    let icon = station.status === 'active' ? greenIcon : yellowIcon;
    L.marker([station.latitude, station.longitude], {icon: icon})
     .bindPopup(`<b>${station.name}</b><br>Trạng thái: ${station.status}`)
     .addTo(map);
});

// Render Trạm đề xuất CỦA BẠN (Marker Cam)
const myProposals = @json($myProposals);
myProposals.forEach(prop => {
    L.marker([prop.latitude, prop.longitude], {icon: orangeIcon})
     .bindPopup(`<b>Đề xuất của bạn</b><br>Chủ đất: ${prop.owner_name}`)
     .addTo(map);
});

// Render Trạm đề xuất CỦA NGƯỜI KHÁC (Marker Xám)
const otherProposals = @json($otherProposals);
otherProposals.forEach(prop => {
    L.marker([prop.latitude, prop.longitude], {icon: grayIcon})
     .bindPopup(`<b>Đề xuất từ người khác</b><br>Chủ đất: ${prop.owner_name}`)
     .addTo(map);
});

// Sự kiện Click lấy tọa độ
map.on('click', function(e) {
    document.getElementById('lat').value = e.latlng.lat;
    document.getElementById('lng').value = e.latlng.lng;
    openProposalModal(); // Hiển thị Modal Form đề xuất
});
```

3. **Modal Form Đề Xuất (Blade Component `resources/views/livewire/proposal-modal.blade.php`):**

```html
<!-- Modal Overlay -->
<div x-show="showModal" x-cloak class="fixed inset-0 z-50 flex items-center justify-center bg-black/50">
    <div class="bg-white rounded-lg shadow-xl w-full max-w-lg mx-4 max-h-[90vh] overflow-y-auto">
        <div class="p-6">
            <h3 class="text-lg font-bold mb-4">Đề xuất trạm mới</h3>
            <form wire:submit="submitProposal">
                @csrf
                <!-- Tọa độ (readonly) -->
                <div class="grid grid-cols-2 gap-4 mb-4">
                    <div>
                        <label class="block text-sm font-medium">Vĩ độ (Latitude)</label>
                        <input type="text" id="lat" wire:model="latitude" readonly
                               class="mt-1 block w-full rounded-md border-gray-300 bg-gray-100">
                    </div>
                    <div>
                        <label class="block text-sm font-medium">Kinh độ (Longitude)</label>
                        <input type="text" id="lng" wire:model="longitude" readonly
                               class="mt-1 block w-full rounded-md border-gray-300 bg-gray-100">
                    </div>
                </div>

                <!-- Họ tên chủ mặt bằng -->
                <div class="mb-4">
                    <label class="block text-sm font-medium">Họ tên chủ mặt bằng *</label>
                    <input type="text" wire:model="owner_name" required
                           class="mt-1 block w-full rounded-md border-gray-300">
                </div>

                <!-- Số điện thoại -->
                <div class="mb-4">
                    <label class="block text-sm font-medium">Số điện thoại *</label>
                    <input type="text" wire:model="owner_phone" required
                           class="mt-1 block w-full rounded-md border-gray-300">
                </div>

                <!-- Thông tin động (location_attributes) -->
                <div class="mb-4">
                    <label class="block text-sm font-medium">Loại mặt bằng</label>
                    <select wire:model="location_attributes.loai_mat_bang"
                            class="mt-1 block w-full rounded-md border-gray-300">
                        <option value="">-- Chọn --</option>
                        <option value="dat_nha_o">Đất nhà ở</option>
                        <option value="dat_cong_nghiep">Đất công nghiệp</option>
                        <option value="dat_trong">Đất trống</option>
                        <option value="nha_may">Nhà máy</option>
                    </select>
                </div>

                <div class="mb-4">
                    <label class="block text-sm font-medium">Độ rộng đường tiếp cận</label>
                    <input type="text" wire:model="location_attributes.do_rong_duong"
                           class="mt-1 block w-full rounded-md border-gray-300">
                </div>

                <div class="mb-4">
                    <label class="block text-sm font-medium">Hiện trạng lưới điện</label>
                    <input type="text" wire:model="location_attributes.hien_trang_luoi_dien"
                           class="mt-1 block w-full rounded-md border-gray-300">
                </div>

                <!-- Upload ảnh thực địa -->
                <div class="mb-4">
                    <label class="block text-sm font-medium">Ảnh chụp thực địa</label>
                    <input type="file" wire:model="images" multiple accept="image/*"
                           class="mt-1 block w-full">
                    @if($images)
                        <div class="mt-2 flex gap-2 flex-wrap">
                            @foreach($images as $image)
                                <img src="{{ $image->temporaryUrl() }}" class="w-20 h-20 object-cover rounded">
                            @endforeach
                        </div>
                    @endif
                </div>

                <!-- Upload giấy tờ -->
                <div class="mb-4">
                    <label class="block text-sm font-medium">Giấy tờ liên quan (PDF, Word)</label>
                    <input type="file" wire:model="documents" multiple
                           accept=".pdf,.doc,.docx"
                           class="mt-1 block w-full">
                </div>

                <!-- Buttons -->
                <div class="flex justify-end gap-2 mt-6">
                    <button type="button" wire:click="closeModal"
                            class="px-4 py-2 bg-gray-300 rounded hover:bg-gray-400">
                        Hủy
                    </button>
                    <button type="submit"
                            class="px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700">
                        Gửi đề xuất
                    </button>
                </div>
            </form>
        </div>
    </div>
</div>
```

### Bước 5: Phân quyền & Middleware khóa tài khoản (30 phút)

> **Lưu ý:** Thực hiện bên trong container `docker exec -it station-app bash`

1. **Middleware kiểm tra tài khoản bị khóa (`app/Http/Middleware/CheckActiveUser.php`):**

```php
public function handle(Request $request, Closure $next)
{
    if (auth()->check() && auth()->user()->status === 'locked') {
        auth()->logout();
        return redirect()->route('login')->withErrors(['email' => 'Tài khoản của bạn đã bị khóa.']);
    }
    return $next($request);
}

```

2. **Đăng ký Middleware** vào `bootstrap/app.php` hoặc nhóm Web Routes.

---

## 5. Mở rộng về sau & Bảo trì

* **Thêm trường động mới vào Form đề xuất:** Chỉ cần bổ sung các cặp Key-Value tương ứng vào cột `location_attributes` hoặc `extra_attributes` phía Frontend mà **không cần sửa lại schema database**.
* **Lưu trữ tệp tin:** Tất cả ảnh và file giấy tờ được phân thư mục tự động theo dạng `storage/app/public/proposals/{images|documents}/`. Khi đưa lên server production thực tế, chỉ cần thực hiện sao lưu (backup) thư mục `storage` và CSDL MySQL.