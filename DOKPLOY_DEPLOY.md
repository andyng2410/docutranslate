# Deploy DocuTranslate lên Dokploy

Hướng dẫn chi tiết cách deploy DocuTranslate lên [Dokploy](https://dokploy.com/) - một nền tảng PaaS self-hosted.

## Yêu cầu

- Dokploy đã được cài đặt và chạy trên server
- Truy cập được vào Dokploy Dashboard
- (Tùy chọn) Domain đã trỏ về server

---

## Cách 1: Build từ Git Repository với Docker Compose (Khuyên dùng)

Đây là cách tốt nhất vì bạn có toàn quyền kiểm soát quá trình build và cấu hình.

### Bước 1: Tạo Project mới

1. Đăng nhập vào Dokploy Dashboard
2. Click **"Create Project"**
3. Đặt tên project (ví dụ: `docutranslate`)

### Bước 2: Tạo Service Compose

1. Trong project vừa tạo, click **"Add Service"**
2. Chọn **"Compose"**
3. Đặt tên service (ví dụ: `docutranslate-app`)

### Bước 3: Cấu hình Git Repository

Trong tab **General** > **Provider**, chọn **Git** và cấu hình:

| Trường | Giá trị |
|--------|---------|
| **Repository URL** | `https://github.com/xunbu/docutranslate.git` |
| **Branch** | `main` |

> **Lưu ý:** Có thể sử dụng repository fork của bạn nếu muốn custom

### Bước 4: Cấu hình Compose

Trong tab **General**, đảm bảo:

| Trường | Giá trị |
|--------|---------|
| **Compose Path** | `docker-compose.yml` |

Repository đã có sẵn file `docker-compose.yml` với cấu hình đúng:

```yaml
version: '3.8'

services:
  docutranslate:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: docutranslate
    restart: unless-stopped
    ports:
      - "8010:8010"
    volumes:
      - docutranslate_output:/app/output
    environment:
      - DOCUTRANSLATE_PORT=8010
      - DOCUTRANSLATE_PROXY_ENABLED=false
      - DOCUTRANSLATE_CACHE_NUM=10
    # QUAN TRỌNG: Dùng entrypoint để đảm bảo -i flag được include
    entrypoint: ["uv", "run", "docutranslate", "-i", "--host", "0.0.0.0", "--cors"]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8010/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

volumes:
  docutranslate_output:
```

> **QUAN TRỌNG:**
> - `entrypoint` với flag `-i` là **BẮT BUỘC** để khởi động web server
> - `--host 0.0.0.0` cho phép truy cập từ bên ngoài container
> - `--cors` bật CORS support cho domain

### Bước 5: Deploy

1. Click **"Deploy"** để bắt đầu build
2. Đợi quá trình build hoàn tất (có thể mất 2-5 phút lần đầu)
3. Kiểm tra logs để đảm bảo không có lỗi

### Bước 6: Cấu hình Domain (Tùy chọn)

Trong tab **Domains**:

1. Click **"Add Domain"**
2. Nhập domain (ví dụ: `translate.yourdomain.com`)
3. Chọn container port: `8010`
4. Bật **HTTPS** nếu muốn SSL tự động
5. Click **"Save"**

---

## Cách 2: Build từ Git Repository (Application)

### Bước 1: Tạo Project và Service

1. Tạo project mới trong Dokploy
2. Chọn **"Add Service"** > **"Application"**
3. Đặt tên service

### Bước 2: Cấu hình Git Repository

Trong tab **General**, chọn **Git** và cấu hình:

| Trường | Giá trị |
|--------|---------|
| **Repository URL** | `https://github.com/xunbu/docutranslate.git` |
| **Branch** | `main` |
| **Build Path** | `/` (root) |

### Bước 3: Cấu hình Build

| Trường | Giá trị |
|--------|---------|
| **Build Type** | Dockerfile |
| **Dockerfile Path** | `Dockerfile` |

### Bước 4: Cấu hình Port

Trong tab **Ports**:

| Container Port | Protocol | Published |
|----------------|----------|-----------|
| `8010` | HTTP | Yes |

### Bước 5: Cấu hình Command (QUAN TRỌNG)

Trong tab **Advanced** > **Command**, nhập:

```
--host 0.0.0.0 --cors
```

> **Lưu ý:** Dockerfile đã có ENTRYPOINT với `-i`, chỉ cần thêm arguments

Nếu vẫn gặp lỗi 404, thử override hoàn toàn:

```
uv run docutranslate -i --host 0.0.0.0 --cors
```

### Bước 6: Deploy và cấu hình Domain

Tương tự như Cách 1.

---

## Cách 3: Deploy từ Docker Image có sẵn

Nếu không muốn build, có thể dùng image có sẵn trên Docker Hub.

### Bước 1: Tạo Service Docker

1. Tạo project mới
2. Chọn **"Add Service"** > **"Docker"**

### Bước 2: Cấu hình Image

| Trường | Giá trị |
|--------|---------|
| **Image** | `xunbu/docutranslate:latest` |

### Bước 3: Cấu hình Port

| Container Port | Protocol |
|----------------|----------|
| `8010` | HTTP |

### Bước 4: Cấu hình Command

Trong **Advanced** > **Command**:

```
--host 0.0.0.0 --cors
```

### Bước 5: Deploy

Click **"Deploy"** và đợi container start.

---

## Cấu hình nâng cao

### Environment Variables

| Variable | Mô tả | Mặc định |
|----------|-------|----------|
| `DOCUTRANSLATE_PORT` | Port server | `8010` |
| `DOCUTRANSLATE_PROXY_ENABLED` | Bật proxy | `false` |
| `DOCUTRANSLATE_CACHE_NUM` | Cache size | `10` |
| `HF_ENDPOINT` | Hugging Face endpoint | `https://hf-mirror.com` |

### Resource Limits (Khuyên dùng cho production)

| Resource | Giá trị |
|----------|---------|
| **CPU Limit** | 2.0 |
| **Memory Limit** | 2GB |
| **Memory Reservation** | 512MB |

### Health Check

```yaml
Test: ["CMD", "curl", "-f", "http://localhost:8010/"]
Interval: 30s
Timeout: 10s
Retries: 3
Start Period: 60s
```

---

## Troubleshooting

### Lỗi: 404 Not Found khi truy cập

**Đây là lỗi phổ biến nhất!**

**Nguyên nhân:** Thiếu flag `-i` trong command

**Giải pháp:**
1. Nếu dùng **Compose**: Đảm bảo có `entrypoint` với `-i` flag
2. Nếu dùng **Application/Docker**: Thêm vào Command:
   ```
   uv run docutranslate -i --host 0.0.0.0 --cors
   ```
3. Redeploy sau khi sửa

**Kiểm tra logs:** Nếu thấy message "欢迎使用 DocuTranslate！请使用 '-i'..." nghĩa là thiếu flag `-i`

### Lỗi: Connection refused

**Nguyên nhân:** Thiếu `--host 0.0.0.0`

**Giải pháp:** Thêm `--host 0.0.0.0` vào command/entrypoint

### Lỗi: CORS error

**Giải pháp:** Thêm `--cors` vào command/entrypoint

### Lỗi: Build failed

**Nguyên nhân có thể:**
- Thiếu memory khi build
- Network issue khi pull dependencies

**Giải pháp:**
- Tăng resource limits cho build
- Retry build

### Lỗi: Container restart loop

**Kiểm tra logs** để xem lỗi cụ thể. Thường do:
- Port conflict
- Missing environment variables
- Permission issues với volume

---

## Kiểm tra deployment

Sau khi deploy thành công, truy cập:

| URL | Mô tả |
|-----|-------|
| `http://your-domain:8010` | Web UI |
| `http://your-domain:8010/docs` | API Documentation (Swagger) |

---

## Cập nhật

### Với Compose/Git:

1. Code mới được push lên repository
2. Trong Dokploy, click **"Redeploy"**
3. Hoặc bật **Auto Deploy** cho tự động update khi có push

### Với Docker Image:

1. Thay đổi image tag (ví dụ: `xunbu/docutranslate:v1.7.0`)
2. Click **"Redeploy"**

---

## Tài liệu tham khảo

- [Dokploy Documentation](https://docs.dokploy.com/)
- [DocuTranslate GitHub](https://github.com/xunbu/docutranslate)
- [DocuTranslate Docker Hub](https://hub.docker.com/r/xunbu/docutranslate)
