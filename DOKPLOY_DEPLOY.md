# Deploy DocuTranslate lên Dokploy

Hướng dẫn chi tiết cách deploy DocuTranslate lên [Dokploy](https://dokploy.com/) - một nền tảng PaaS self-hosted.

## Yêu cầu

- Dokploy đã được cài đặt và chạy trên server
- Truy cập được vào Dokploy Dashboard
- (Tùy chọn) Domain đã trỏ về server

## Cách 1: Deploy từ Docker Image (Khuyên dùng)

Đây là cách đơn giản và nhanh nhất.

### Bước 1: Tạo Project mới

1. Đăng nhập vào Dokploy Dashboard
2. Click **"Create Project"**
3. Đặt tên project (ví dụ: `docutranslate`)

### Bước 2: Tạo Service

1. Trong project vừa tạo, click **"Add Service"**
2. Chọn **"Docker"**
3. Đặt tên service (ví dụ: `docutranslate-app`)

### Bước 3: Cấu hình Docker Image

Trong tab **General**, cấu hình như sau:

| Trường | Giá trị |
|--------|---------|
| **Image** | `xunbu/docutranslate:latest` |
| **Registry** | Docker Hub (mặc định) |

> **Lưu ý:** Có thể sử dụng version cụ thể như `xunbu/docutranslate:v1.6.2`

### Bước 4: Cấu hình Port

Trong tab **Ports**, thêm mapping:

| Container Port | Protocol | Published |
|----------------|----------|-----------|
| `8010` | HTTP | Yes |

### Bước 5: Cấu hình Environment Variables (Tùy chọn)

Trong tab **Environment**, thêm các biến môi trường nếu cần:

```env
# Port (mặc định: 8010)
DOCUTRANSLATE_PORT=8010

# Bật proxy cho requests ra ngoài
DOCUTRANSLATE_PROXY_ENABLED=false

# Cache size (mặc định: 10)
DOCUTRANSLATE_CACHE_NUM=10

# Hugging Face endpoint (cho tính năng docling)
HF_ENDPOINT=https://hf-mirror.com
```

### Bước 6: Cấu hình Volume (Khuyên dùng)

Trong tab **Volumes**, mount volume để lưu output files:

| Host Path | Container Path | Mode |
|-----------|----------------|------|
| `/data/docutranslate/output` | `/app/output` | Read/Write |

### Bước 7: Cấu hình Command (QUAN TRỌNG - Đọc kỹ)

Trong tab **Advanced**, cấu hình như sau:

#### Cách A: Chỉ thêm Arguments (Nếu Dokploy hỗ trợ)

Nếu Dokploy có trường riêng cho **Arguments** hoặc **Args**:

```
--host 0.0.0.0 --cors
```

#### Cách B: Override Command hoàn toàn (Khuyên dùng)

Nếu Dokploy chỉ có trường **Command** và override toàn bộ, nhập đầy đủ:

```
uv run docutranslate -i --host 0.0.0.0 --cors
```

> **QUAN TRỌNG:**
> - Flag `-i` (interactive) là **BẮT BUỘC** để khởi động web server
> - `--host 0.0.0.0`: Cho phép truy cập từ tất cả interfaces (cần thiết cho container)
> - `--cors`: Bật CORS support (cần thiết nếu sử dụng domain riêng)
>
> **Nếu thiếu `-i`, ứng dụng sẽ không khởi động web server và bạn sẽ gặp lỗi 404!**

### Bước 8: Deploy

1. Click **"Deploy"** để bắt đầu
2. Đợi container được pull và start
3. Kiểm tra logs để đảm bảo không có lỗi

### Bước 9: Cấu hình Domain (Tùy chọn)

Trong tab **Domains**:

1. Click **"Add Domain"**
2. Nhập domain (ví dụ: `translate.yourdomain.com`)
3. Bật **HTTPS** nếu muốn SSL tự động
4. Click **"Save"**

---

## Cách 2: Deploy từ Git Repository

### Bước 1: Tạo Project và Service

1. Tạo project mới trong Dokploy
2. Chọn **"Add Service"** > **"Application"**
3. Chọn **"Git"** làm source

### Bước 2: Cấu hình Git Repository

| Trường | Giá trị |
|--------|---------|
| **Repository URL** | `https://github.com/xunbu/docutranslate.git` |
| **Branch** | `main` |
| **Build Path** | `/` (root) |

### Bước 3: Cấu hình Build

Dokploy sẽ tự động detect Dockerfile. Đảm bảo cấu hình:

| Trường | Giá trị |
|--------|---------|
| **Build Type** | Dockerfile |
| **Dockerfile Path** | `Dockerfile` |

### Bước 4: Build Arguments (Tùy chọn)

Nếu muốn cài version cụ thể:

```
DOC_VERSION=1.6.2
```

### Bước 5: Tiếp tục từ Bước 4 của Cách 1

Cấu hình Port, Environment, Volume, Command và Domain như hướng dẫn ở trên.

---

## Cấu hình nâng cao

### Health Check

Trong tab **Advanced** > **Health Check**:

```yaml
Test: ["CMD", "curl", "-f", "http://localhost:8010/"]
Interval: 30s
Timeout: 10s
Retries: 3
Start Period: 40s
```

### Resource Limits

Trong tab **Resources**, cấu hình giới hạn tài nguyên:

| Resource | Recommended |
|----------|-------------|
| **CPU Limit** | 2.0 |
| **Memory Limit** | 2GB |
| **Memory Reservation** | 512MB |

> **Lưu ý:** DocuTranslate với tính năng docling có thể cần nhiều RAM hơn khi xử lý PDF lớn.

### Restart Policy

Trong **Advanced** > **Restart Policy**:

- Chọn `unless-stopped` hoặc `always` cho production

---

## Sử dụng Docker Compose trong Dokploy

Nếu muốn sử dụng Docker Compose, tạo file `docker-compose.yml`:

```yaml
version: '3.8'

services:
  docutranslate:
    image: xunbu/docutranslate:latest
    container_name: docutranslate
    restart: unless-stopped
    ports:
      - "8010:8010"
    volumes:
      - ./output:/app/output
    environment:
      - DOCUTRANSLATE_PORT=8010
      - DOCUTRANSLATE_PROXY_ENABLED=false
      - DOCUTRANSLATE_CACHE_NUM=10
    command: ["--host", "0.0.0.0", "--cors"]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8010/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

Trong Dokploy:
1. Chọn **"Add Service"** > **"Compose"**
2. Paste nội dung trên vào
3. Click **"Deploy"**

---

## Troubleshooting

### Lỗi: 404 Not Found khi truy cập

**Đây là lỗi phổ biến nhất!**

**Nguyên nhân 1:** Thiếu flag `-i` trong command

**Giải pháp:**
1. Vào tab **Advanced** > **Command**
2. Đảm bảo command có dạng: `uv run docutranslate -i --host 0.0.0.0 --cors`
3. Flag `-i` là **BẮT BUỘC** để khởi động web server
4. Redeploy sau khi sửa

**Nguyên nhân 2:** Command bị override sai cách

**Giải pháp:**
- Kiểm tra logs của container: nếu thấy message "欢迎使用 DocuTranslate！请使用 '-i'..." nghĩa là thiếu flag `-i`
- Sửa command theo hướng dẫn ở Bước 7

**Nguyên nhân 3:** Domain/Proxy configuration sai

**Giải pháp:**
- Kiểm tra domain đã trỏ đúng về service
- Đảm bảo port mapping là `8010`
- Thử truy cập trực tiếp bằng IP:Port trước

### Lỗi: Container không start

**Nguyên nhân có thể:**
- Port 8010 đã được sử dụng

**Giải pháp:**
- Thay đổi port mapping hoặc environment variable `DOCUTRANSLATE_PORT`

### Lỗi: Không truy cập được từ browser (Connection refused)

**Nguyên nhân có thể:**
- Thiếu `--host 0.0.0.0` trong command

**Giải pháp:**
- Thêm `--host 0.0.0.0` vào phần Command trong Advanced settings

### Lỗi: CORS error trên frontend

**Nguyên nhân có thể:**
- Chưa bật CORS

**Giải pháp:**
- Thêm `--cors` vào command
- Hoặc sử dụng `--cors-regex ".*"` để cho phép tất cả origins

### Lỗi: Files output bị mất sau khi restart

**Nguyên nhân:**
- Chưa mount volume

**Giải pháp:**
- Mount `/app/output` đến một thư mục persistent trên host

### Lỗi: Out of memory khi xử lý PDF lớn

**Giải pháp:**
- Tăng Memory Limit trong Resource settings
- Khuyên dùng ít nhất 2GB RAM cho việc xử lý PDF phức tạp

---

## Kiểm tra deployment

Sau khi deploy thành công, truy cập:

- **Web UI:** `http://your-domain:8010` hoặc domain đã cấu hình
- **API Docs:** `http://your-domain:8010/docs` (Swagger UI)
- **Health:** `http://your-domain:8010/` (trả về 200 OK)

---

## Cập nhật version

### Với Docker Image:

1. Vào service trong Dokploy
2. Thay đổi image tag (ví dụ: `xunbu/docutranslate:v1.7.0`)
3. Click **"Redeploy"**

### Với Git Repository:

1. Push code mới lên repository
2. Trong Dokploy, click **"Redeploy"** hoặc bật **Auto Deploy** cho tự động

---

## Tài liệu tham khảo

- [Dokploy Documentation](https://docs.dokploy.com/)
- [DocuTranslate GitHub](https://github.com/xunbu/docutranslate)
- [DocuTranslate Docker Hub](https://hub.docker.com/r/xunbu/docutranslate)
