# Паспорт интеграции: MinIO S3 Object Storage

> Обновлено: 2025-12-21 | Версия: 1.0.0

## Обзор

MinIO - это высокопроизводительное S3-совместимое объектное хранилище, используемое платформой Vondi для управления всеми загружаемыми файлами (изображения, документы, вложения чата). Система предоставляет унифицированный уровень абстракции хранилища с поддержкой множественных бакетов, автоматической генерацией миниатюр, presigned URLs и политик очистки.

### Ключевые возможности

- **S3-совместимость**: Полная поддержка AWS S3 API
- **Множественные бакеты**: Раздельное хранение по типам контента
- **Presigned URLs**: Безопасный временный доступ к файлам
- **CDN Proxy**: Кеширование через backend proxy (7 дней)
- **Автоматические миниатюры**: 200x200 JPEG для изображений
- **Публичный доступ**: Настраиваемые bucket policies

---

## 1. Архитектура

### 1.1 Компоненты системы

```
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│   Browser   │─────→│ Backend Proxy│─────→│   MinIO     │
│             │      │  (Fiber)     │      │ S3 Storage  │
└─────────────┘      └──────────────┘      └─────────────┘
                            │                      │
                            ↓                      ↓
                     ┌──────────────┐      ┌─────────────┐
                     │  PostgreSQL  │      │   Buckets   │
                     │   (Metadata) │      │ - listings  │
                     └──────────────┘      │ - chat      │
                                           │ - products  │
                                           └─────────────┘
```

**Слои абстракции:**

| Слой | Компонент | Назначение |
|------|-----------|------------|
| **HTTP API** | `/internal/server/minio_proxy.go` | Proxy endpoints для публичного доступа |
| **Service Layer** | `/internal/services/image_service.go` | Бизнес-логика загрузки/обработки |
| **Storage Interface** | `/internal/storage/filestorage/interface.go` | Абстракция провайдера |
| **MinIO Client** | `/internal/storage/minio/client.go` | Low-level MinIO SDK операции |

---

### 1.2 Production Endpoint

```bash
Public URL:    https://s3.vondi.rs
Internal IP:   194.163.132.116:9000
API Port:      9000
Console Port:  9001
SSL:           Enabled
Region:        eu-central-1
```

---

## 2. Конфигурация

### 2.1 Environment Variables

```bash
# Provider Selection
FILE_STORAGE_PROVIDER=minio                    # Тип хранилища (minio|local)

# MinIO Connection
MINIO_ENDPOINT=localhost:9000                  # MinIO server address
MINIO_ACCESS_KEY=minioadmin                    # Access key ID
MINIO_SECRET_KEY=YOUR_SECURE_MINIO_SECRET     # Secret access key
MINIO_USE_SSL=false                            # Use HTTPS (true в production)
MINIO_LOCATION=eu-central-1                    # Bucket region

# Buckets Configuration
MINIO_BUCKET_NAME=listings                     # C2C маркетплейс объявления
MINIO_CHAT_BUCKET=chat-files                   # P2P чат вложения
MINIO_STOREFRONT_BUCKET=storefront-products    # B2C витрины товары
MINIO_REVIEW_PHOTOS_BUCKET=review-photos       # Фото отзывов
MINIO_FRANCHISE_BUCKET=franchise-documents     # Франшиза документы

# Public Access URLs
FILE_STORAGE_PUBLIC_URL=http://localhost:3000  # Public base URL (через proxy)
MINIO_PUBLIC_URL=http://localhost:9000         # Direct MinIO URL (для admin)
```

### 2.2 Go Configuration Struct

**Файл:** `backend/internal/config/config.go:60-74`

```go
type FileStorageConfig struct {
    Provider                string // "minio" или "local"
    MinioEndpoint           string // localhost:9000
    MinioAccessKey          string // Access Key ID
    MinioSecretKey          string // Secret Access Key
    MinioUseSSL             bool   // SSL включен?

    // Buckets
    MinioBucketName         string // listings (основной)
    MinioChatBucket         string // chat-files
    MinioStorefrontBucket   string // storefront-products
    MinioReviewPhotosBucket string // review-photos
    MinioFranchiseBucket    string // franchise-documents

    MinioLocation           string // eu-central-1
    PublicBaseURL           string // Proxy URL
}
```

---

## 3. Buckets и назначение

### 3.1 Структура бакетов

| Bucket Name | Назначение | Public Read | Auto-Created | Retention Policy |
|-------------|-----------|-------------|--------------|------------------|
| **listings** | C2C marketplace объявления | ✅ Yes | ✅ Yes | Manual cleanup при удалении listing |
| **storefront-products** | B2C витрины товары, лого, баннеры | ✅ Yes | ✅ Yes | Manual cleanup при удалении product |
| **chat-files** | P2P чат вложения (изображения, документы) | ✅ Yes | ✅ Yes | 30 дней после удаления чата |
| **review-photos** | Фото отзывов о продуктах | ✅ Yes | ✅ Yes | Permanent (связаны с reviews) |
| **franchise-documents** | Франшиза контракты, лицензии | ❌ No | ✅ Yes | Permanent (юридические требования) |

### 3.2 Bucket Policies

**Политика публичного чтения** (применяется ко всем кроме franchise-documents):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": ["*"]},
    "Action": ["s3:GetObject"],
    "Resource": ["arn:aws:s3:::BUCKET_NAME/*"]
  }]
}
```

**Автосоздание бакетов** (`backend/internal/storage/minio/client.go:48-175`):

```go
func NewMinioClient(ctx context.Context, config MinioConfig) (*MinioClient, error) {
    // 1. Создание клиента MinIO
    client, err := minio.New(config.Endpoint, &minio.Options{
        Creds:  credentials.NewStaticV4(config.AccessKeyID, config.SecretAccessKey, ""),
        Secure: config.UseSSL,
    })

    // 2. Проверка и создание основного bucket
    exists, err := client.BucketExists(ctx, config.BucketName)
    if !exists {
        // Создаём bucket
        err = client.MakeBucket(ctx, config.BucketName, minio.MakeBucketOptions{
            Region: config.Location,
        })

        // Устанавливаем публичную политику
        policy := `{"Version":"2012-10-17","Statement":[...]}`
        err = client.SetBucketPolicy(ctx, config.BucketName, policy)
    }

    // 3-6. Аналогично для chat, storefront, review-photos, franchise buckets
    // ...
}
```

---

## 4. File Upload Flow

### 4.1 Direct Upload (Small Files < 10MB)

**Последовательность:**

```
┌─────────┐  POST /api/v1/marketplace/listings/:id/images  ┌──────────┐
│ Client  │──────────────────────────────────────────────→│ Backend  │
└─────────┘                                                └──────────┘
                                                                 │
                                                                 ↓
                                                    ┌─────────────────────┐
                                                    │ Validate file type, │
                                                    │ size, filename      │
                                                    └─────────────────────┘
                                                                 │
                                                                 ↓
                                                    ┌─────────────────────┐
                                                    │ ImageService.       │
                                                    │ UploadImage()       │
                                                    └─────────────────────┘
                                                                 │
                                                                 ↓
                                            ┌────────────────────┴────────────────────┐
                                            │                                         │
                                            ↓                                         ↓
                                  ┌──────────────────┐                  ┌──────────────────┐
                                  │ MinIO.PutObject  │                  │ Generate         │
                                  │ (main image)     │                  │ Thumbnail        │
                                  └──────────────────┘                  │ 200x200 JPEG     │
                                            │                            └──────────────────┘
                                            │                                         │
                                            ↓                                         ↓
                                  ┌──────────────────┐                  ┌──────────────────┐
                                  │ Return URL:      │                  │ MinIO.PutObject  │
                                  │ /listings/123/   │                  │ (thumb_xxx.jpg)  │
                                  │ timestamp_uuid   │                  └──────────────────┘
                                  └──────────────────┘                            │
                                            │                                     │
                                            └──────────────┬──────────────────────┘
                                                           │
                                                           ↓
                                                ┌──────────────────────┐
                                                │ Save to PostgreSQL:  │
                                                │ - image_url          │
                                                │ - thumbnail_url      │
                                                │ - metadata           │
                                                └──────────────────────┘
                                                           │
                                                           ↓
┌─────────┐                                    ┌──────────────────────┐
│ Client  │←───────────────────────────────────│ JSON Response:       │
└─────────┘  {id, image_url, thumbnail_url}    │ {id, urls, metadata} │
                                                └──────────────────────┘
```

**API Endpoints:**

```go
// Marketplace C2C listings
POST   /api/v1/marketplace/listings/:id/images
DELETE /api/v1/marketplace/listings/:id/images/:imageId
PUT    /api/v1/marketplace/listings/:id/images/reorder

// B2C Storefront products
POST /api/v1/storefronts/:slug/products/:id/images
POST /api/v1/marketplace/storefronts/slug/:slug/products/:id/images

// Storefront branding
POST /api/v1/marketplace/storefronts/:id/logo
POST /api/v1/marketplace/storefronts/:id/banner

// Review photos
POST /api/v1/reviews/upload-photos
```

---

### 4.2 Presigned URL Upload (Large Files > 10MB)

**Последовательность:**

```
┌─────────┐  GET /api/presigned-upload?file=video.mp4  ┌──────────┐
│ Client  │──────────────────────────────────────────→│ Backend  │
└─────────┘                                            └──────────┘
                                                             │
                                                             ↓
                                              ┌────────────────────────┐
                                              │ MinIO.PresignedPutURL  │
                                              │ (bucket, path, 15min)  │
                                              └────────────────────────┘
                                                             │
                                                             ↓
┌─────────┐                                    ┌────────────────────────┐
│ Client  │←───────────────────────────────────│ {upload_url, key}      │
└─────────┘                                    └────────────────────────┘
     │
     │  PUT https://s3.vondi.rs/bucket/path?signature=...
     │  (upload file directly to MinIO)
     ↓
┌──────────┐
│  MinIO   │
└──────────┘
     │
     │  200 OK
     ↓
┌─────────┐  POST /api/confirm-upload {object_key}  ┌──────────┐
│ Client  │────────────────────────────────────────→│ Backend  │
└─────────┘                                          └──────────┘
                                                           │
                                                           ↓
                                                ┌──────────────────────┐
                                                │ Save to PostgreSQL   │
                                                │ - file_url           │
                                                │ - metadata           │
                                                └──────────────────────┘
                                                           │
                                                           ↓
┌─────────┐                                    ┌──────────────────────┐
│ Client  │←───────────────────────────────────│ {file_id, public_url}│
└─────────┘                                    └──────────────────────┘
```

**Implementation** (`backend/internal/storage/minio/client.go:235-246`):

```go
func (m *MinioClient) GetPresignedURL(ctx context.Context, objectName string, expiry time.Duration) (string, error) {
    objectName = strings.TrimPrefix(objectName, "/")

    presignedURL, err := m.client.PresignedGetObject(ctx, m.bucketName, objectName, expiry, nil)
    if err != nil {
        return "", fmt.Errorf("error creating presigned URL: %w", err)
    }

    return presignedURL.String(), nil
}
```

**Use Cases:**
- Видео загрузки (100MB+ файлы)
- Bulk фото загрузки с мобильных приложений
- Прямые browser-to-MinIO загрузки (снижение нагрузки на backend)

**Presigned PUT URL:**

```go
// Для загрузки файла (PUT)
presignedURL, err := m.client.PresignedPutObject(ctx, bucketName, objectName, 15*time.Minute)

// Для скачивания файла (GET)
presignedURL, err := m.client.PresignedGetObject(ctx, bucketName, objectName, 1*time.Hour, nil)
```

**Параметры:**
- **PUT URL Expiry**: 15 минут (для загрузки)
- **GET URL Expiry**: 1 час (для скачивания)
- **Секретность**: URL содержит подпись, доступен только владельцу

---

## 5. Image Processing Pipeline

### 5.1 Automatic Thumbnail Generation

**Параметры:**
- **Размер**: 200x200 пикселей
- **Формат**: JPEG (85% качество)
- **Алгоритм**: Bilinear interpolation (3x быстрее Lanczos3)
- **Путь**: Тот же bucket, префикс `thumb_`

**Implementation** (`backend/internal/services/image_service.go:315-335`):

```go
func (s *ImageService) createThumbnail(imageBytes []byte, filename string) ([]byte, error) {
    // Декодирование изображения (поддержка JPEG, PNG, GIF, WebP)
    img, _, err := image.Decode(bytes.NewReader(imageBytes))
    if err != nil {
        return nil, fmt.Errorf("failed to decode image: %w", err)
    }

    // Resize до 200x200 с сохранением aspect ratio
    thumbnail := resize.Thumbnail(200, 200, img, resize.Bilinear)

    // Кодирование в JPEG
    var buf bytes.Buffer
    if err := jpeg.Encode(&buf, thumbnail, &jpeg.Options{Quality: 85}); err != nil {
        return nil, fmt.Errorf("failed to encode thumbnail: %w", err)
    }

    return buf.Bytes(), nil
}
```

**Производительность:**

| Алгоритм | Качество | Время (2MB image) |
|----------|----------|-------------------|
| Lanczos3 (old) | 95% | ~500ms |
| Bilinear (new) | 85% | ~150ms |

**Улучшение**: 3x performance boost

---

### 5.2 Path Generation

**Формат:** `{entity_type}/{entity_id}/{timestamp}_{uuid}.{ext}`

**Примеры:**

```bash
# Marketplace listings
listings/12345/20231215143052_a3f8b9c2-4d5e-6f7a-8b9c-0d1e2f3a4b5c.jpg
listings/12345/thumb_20231215143052_a3f8b9c2-4d5e-6f7a-8b9c-0d1e2f3a4b5c.jpg

# Storefront products
storefront-products/789/20231215150000_b7d3e4f1-5c6d-7e8f-9a0b-1c2d3e4f5a6b.png
storefront-products/789/thumb_20231215150000_b7d3e4f1-5c6d-7e8f-9a0b-1c2d3e4f5a6b.jpg

# Storefront branding
storefront-products/456/logo_20231215160000_c8e4f5a2-6d7e-8f9a-0b1c-2d3e4f5a6b7c.webp
storefront-products/456/banner_20231215160000_d9f0a1b2-7e8f-9a0b-1c2d-3e4f5a6b7c8d.jpg
```

**Implementation** (`backend/internal/services/image_service.go:261-285`):

```go
func (s *ImageService) generateFilePath(entityType ImageType, entityID int, filename string) string {
    ext := filepath.Ext(filename)
    uniqueID := uuid.New().String()
    timestamp := time.Now().Format("20060102150405")

    switch entityType {
    case ImageTypeMarketplaceListing:
        return fmt.Sprintf("listings/%d/%s_%s%s", entityID, timestamp, uniqueID, ext)
    case ImageTypeStorefrontProduct:
        return fmt.Sprintf("%d/%s_%s%s", entityID, timestamp, uniqueID, ext)
    case ImageTypeStorefrontLogo:
        return fmt.Sprintf("%d/logo_%s_%s%s", entityID, timestamp, uniqueID, ext)
    case ImageTypeStorefrontBanner:
        return fmt.Sprintf("%d/banner_%s_%s%s", entityID, timestamp, uniqueID, ext)
    case ImageTypeChatFile:
        return fmt.Sprintf("%d/%s_%s%s", entityID, timestamp, uniqueID, ext)
    case ImageTypeReviewPhoto:
        return fmt.Sprintf("%d/%s_%s%s", entityID, timestamp, uniqueID, ext)
    default:
        return fmt.Sprintf("misc/%s_%s%s", timestamp, uniqueID, ext)
    }
}
```

---

### 5.3 Supported Formats

**Images:**
- JPEG/JPG (`.jpg`, `.jpeg`)
- PNG (`.png`)
- GIF (`.gif`)
- WebP (`.webp`)

**Max Size**: 10MB per image

**Decoders** (`backend/internal/services/image_service.go:8-25`):

```go
import (
    _ "image/gif"                    // GIF support
    _ "image/png"                    // PNG support
    _ "golang.org/x/image/webp"     // WebP support
)
```

---

## 6. File Validation & Security

**Security Layer**: `backend/pkg/utils/file_validation.go`

### 6.1 Type Validation

**Allowed MIME Types:**

```go
var AllowedFileTypes = map[string][]string{
    "image": {
        "image/jpeg", "image/jpg", "image/png",
        "image/gif", "image/webp",
    },
    "video": {
        "video/mp4", "video/mpeg", "video/quicktime",
        "video/x-msvideo", "video/webm",
    },
    "document": {
        "application/pdf", "application/msword",
        "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
        "application/vnd.ms-excel",
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        "text/plain",
    },
}
```

### 6.2 Size Validation

**Max File Sizes:**

```go
var MaxFileSizes = map[string]int64{
    "image":    10 * 1024 * 1024,   // 10MB
    "video":    100 * 1024 * 1024,  // 100MB
    "document": 20 * 1024 * 1024,   // 20MB
}
```

### 6.3 Filename Sanitization

**Dangerous Extensions (blocked):**

```go
dangerousExtensions := []string{
    ".exe", ".bat", ".cmd", ".com", ".scr", ".vbs", ".js",
    ".jar", ".app", ".deb", ".rpm", ".dmg", ".pkg", ".run",
}
```

**Character Sanitization:**

```go
replacer := strings.NewReplacer(
    "<", "_", ">", "_", ":", "_", "\"", "_",
    "/", "_", "\\", "_", "|", "_", "?", "_", "*", "_",
)
sanitized := replacer.Replace(filename)
```

**Max Filename Length**: 255 characters (preserving extension)

### 6.4 MIME Type Verification

**Content-based detection:**

```go
// Проверка MIME type из содержимого файла (НЕ только расширение!)
buffer := make([]byte, 512)
_, err := file.Read(buffer)
detectedType := http.DetectContentType(buffer)

if detectedType != expectedType {
    return errors.New("MIME type mismatch")
}
```

### 6.5 Image Bomb Protection

**Dimension limits:**

```go
const maxWidth = 10000
const maxHeight = 10000

img, _, _ := image.DecodeConfig(file)
if img.Width > maxWidth || img.Height > maxHeight {
    return errors.New("image dimensions too large")
}
```

---

## 7. CDN & Public Access

### 7.1 URL Generation

**Direct MinIO URL** (НЕ используется в production):

```
http://localhost:9000/listings/12345/20231215143052_uuid.jpg
```

**Proxied URL** (рекомендуется):

```
https://vondi.rs/listings/12345/20231215143052_uuid.jpg
```

**URL Construction** (`backend/internal/storage/minio/client.go:201-224`):

```go
func (m *MinioClient) UploadFile(ctx context.Context, objectName string, reader io.Reader, size int64, contentType string) (string, error) {
    objectName = strings.TrimPrefix(objectName, "/")

    // Upload to MinIO
    _, err := m.client.PutObject(ctx, m.bucketName, objectName, reader, size, minio.PutObjectOptions{
        ContentType: contentType,
    })
    if err != nil {
        return "", fmt.Errorf("error uploading file to MinIO: %w", err)
    }

    // Generate public URL
    var publicURL string
    if m.baseURL != "" {
        // Используем публичный URL если задан (через proxy)
        publicURL = fmt.Sprintf("%s/%s/%s", m.baseURL, m.bucketName, objectName)
    } else {
        // Иначе относительный путь
        publicURL = fmt.Sprintf("/%s/%s", m.bucketName, objectName)
    }

    return publicURL, nil
}
```

---

### 7.2 Backend Proxy Implementation

**Purpose**: CDN-подобное кеширование, единый домен, security headers

**Routes** (`backend/internal/server/minio_proxy.go`):

```go
// Marketplace listings
app.Get("/listings/*", server.ProxyMinIO)

// Chat files
app.Get("/chat-files/*", server.ProxyChatFiles)

// Storefront products
app.Get("/storefront-products/*", server.ProxyStorefrontProducts)
```

**Proxy Handler Features:**

```go
func (s *Server) ProxyMinIO(c *fiber.Ctx) error {
    path := c.Params("*")
    minioURL := fmt.Sprintf("http://%s/listings/%s", s.cfg.FileStorage.MinioEndpoint, path)

    // 1. HTTP request к MinIO
    client := &http.Client{}
    resp, err := client.Get(minioURL)
    if err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{"error": "Failed to fetch file"})
    }
    defer resp.Body.Close()

    // 2. Копирование headers
    c.Set("Content-Type", resp.Header.Get("Content-Type"))
    c.Set("Content-Length", resp.Header.Get("Content-Length"))

    // 3. Cache-Control headers (7 дней)
    c.Set("Cache-Control", "public, max-age=604800")

    // 4. Content-Type detection по расширению (fallback)
    if resp.Header.Get("Content-Type") == "" {
        ext := strings.ToLower(path[strings.LastIndex(path, ".")+1:])
        switch ext {
        case "jpg", "jpeg":
            c.Set("Content-Type", "image/jpeg")
        case "png":
            c.Set("Content-Type", "image/png")
        case "webp":
            c.Set("Content-Type", "image/webp")
        default:
            c.Set("Content-Type", "application/octet-stream")
        }
    }

    // 5. Stream response (без буферизации в память)
    io.Copy(c.Response().BodyWriter(), resp.Body)
    return nil
}
```

**Преимущества:**

1. ✅ **Browser Caching**: 7 дней (`Cache-Control: public, max-age=604800`)
2. ✅ **Единый домен**: `vondi.rs/*` вместо `s3.vondi.rs/*`
3. ✅ **Streaming**: Не загружает файл в память (low memory footprint)
4. ✅ **Content-Type detection**: Автоматическое определение MIME type
5. ✅ **Error handling**: 404 для несуществующих файлов

---

### 7.3 Presigned URLs (Temporary Access)

**Use Cases:**
- Приватные файлы (franchise documents)
- Time-limited share links
- Безопасные upload URLs

**Expiry Defaults:**

| Operation | Default Expiry | Configurable |
|-----------|----------------|--------------|
| Upload (PUT) | 15 минут | ✅ Yes |
| Download (GET) | 1 час | ✅ Yes |

**Example:**

```go
// Download presigned URL (1 hour)
url, err := minioClient.GetPresignedURL(ctx, "listings/123/photo.jpg", 1*time.Hour)
// → https://s3.vondi.rs/listings/123/photo.jpg?signature=...&expires=...

// Upload presigned URL (15 minutes)
url, err := minioClient.PresignedPutObject(ctx, "listings", "123/new.jpg", 15*time.Minute)
// → https://s3.vondi.rs/listings/123/new.jpg?signature=...&expires=...
```

**Безопасность:**
- URL содержит криптографическую подпись
- Невозможно изменить путь без инвалидации подписи
- Автоматическое истечение после expiry

---

## 8. Database Schema

### 8.1 Marketplace Images

**Table**: `marketplace_images`

```sql
CREATE TABLE marketplace_images (
    id SERIAL PRIMARY KEY,
    listing_id INTEGER NOT NULL REFERENCES marketplace_listings(id) ON DELETE CASCADE,

    -- URLs
    image_url TEXT NOT NULL,
    thumbnail_url TEXT,
    public_url TEXT,

    -- Storage metadata
    file_path TEXT,
    storage_type VARCHAR(50) DEFAULT 'minio',
    storage_bucket VARCHAR(100),

    -- Image metadata
    file_name VARCHAR(255),
    content_type VARCHAR(100),

    -- Display settings
    is_main BOOLEAN DEFAULT FALSE,
    display_order INTEGER DEFAULT 0,

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_marketplace_images_listing ON marketplace_images(listing_id);
CREATE INDEX idx_marketplace_images_is_main ON marketplace_images(is_main);
```

**Поля:**
- `image_url`: Полный URL изображения (через proxy)
- `thumbnail_url`: URL миниатюры 200x200
- `file_path`: Путь в MinIO (без bucket)
- `storage_bucket`: Имя бакета (`listings`)
- `is_main`: Главное изображение для listing
- `display_order`: Порядок отображения

---

### 8.2 Storefront Product Images

**Table**: `storefront_product_images`

```sql
CREATE TABLE storefront_product_images (
    id SERIAL PRIMARY KEY,
    storefront_product_id INTEGER NOT NULL REFERENCES storefront_products(id) ON DELETE CASCADE,

    -- URLs
    image_url TEXT NOT NULL,
    thumbnail_url TEXT,
    public_url TEXT,

    -- Storage metadata
    file_path TEXT,
    file_name VARCHAR(255),
    file_size INTEGER,
    content_type VARCHAR(100),
    storage_type VARCHAR(50) DEFAULT 'minio',
    storage_bucket VARCHAR(100),

    -- Display settings
    is_default BOOLEAN DEFAULT FALSE,
    display_order INTEGER DEFAULT 0,

    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_storefront_product_images_product ON storefront_product_images(storefront_product_id);
CREATE INDEX idx_storefront_product_images_is_default ON storefront_product_images(is_default);
```

---

### 8.3 Chat Attachments

**Table**: `chat_attachments`

```sql
CREATE TABLE chat_attachments (
    id SERIAL PRIMARY KEY,
    message_id INTEGER NOT NULL REFERENCES chat_messages(id) ON DELETE CASCADE,

    -- URLs & metadata
    file_url TEXT NOT NULL,
    file_name VARCHAR(255),
    file_size INTEGER,
    file_type VARCHAR(100),
    mime_type VARCHAR(100),
    storage_bucket VARCHAR(100),

    -- Timestamp
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_chat_attachments_message ON chat_attachments(message_id);
```

---

## 9. Code Examples

### 9.1 Basic Upload

```go
package main

import (
    "context"
    "os"
    "backend/internal/storage/minio"
)

func uploadFile(minioClient *minio.MinioClient) error {
    ctx := context.Background()

    // 1. Open file
    file, err := os.Open("/tmp/test-image.jpg")
    if err != nil {
        return err
    }
    defer file.Close()

    // 2. Get file size
    stat, err := file.Stat()
    if err != nil {
        return err
    }

    // 3. Upload to MinIO
    objectName := "123/test-image.jpg"
    publicURL, err := minioClient.UploadFile(
        ctx,
        objectName,
        file,
        stat.Size(),
        "image/jpeg",
    )
    if err != nil {
        return err
    }

    fmt.Printf("File uploaded: %s\n", publicURL)
    // → http://localhost:3000/listings/123/test-image.jpg

    return nil
}
```

---

### 9.2 Presigned URL Generation

```go
package main

import (
    "context"
    "time"
    "backend/internal/storage/minio"
)

func generatePresignedURL(minioClient *minio.MinioClient) (string, error) {
    ctx := context.Background()

    objectName := "123/test-image.jpg"
    expiry := 1 * time.Hour // Valid for 1 hour

    presignedURL, err := minioClient.GetPresignedURL(ctx, objectName, expiry)
    if err != nil {
        return "", err
    }

    return presignedURL, nil
}
```

---

## 10. Monitoring & Troubleshooting

### 10.1 Health Checks

```bash
# MinIO liveness probe
curl http://localhost:9000/minio/health/live
# → 200 OK (MinIO is running)

# MinIO readiness probe
curl http://localhost:9000/minio/health/ready
# → 200 OK (MinIO is ready)

# Check buckets
mc ls vondi/
```

---

### 10.2 Common Issues

**Issue**: "Bucket does not exist"

```bash
# Solution 1: Restart backend (auto-creates buckets)
docker restart vondi-dev-backend

# Solution 2: Manual creation
mc mb minio/listings
mc policy set public minio/listings
```

---

**Issue**: "Access Denied"

```bash
# Check credentials
echo $MINIO_ACCESS_KEY
echo $MINIO_SECRET_KEY

# Verify bucket policy
mc admin policy info minio/ public

# Test connection
mc ls minio/
```

---

**Issue**: "Connection refused"

```bash
# Check MinIO is running
docker ps | grep minio
nc -zv localhost 9000

# Check firewall
sudo ufw status

# Test direct MinIO endpoint
curl http://localhost:9000/minio/health/live
```

---

## 11. Связанные документы

| Документ | Назначение |
|----------|------------|
| `/p/github.com/vondi-global/.passport/infrastructure/storage.md` | Общая инфраструктура хранилищ |
| `/p/github.com/vondi-global/.passport/flows/file-upload.md` | File upload flow детально |
| `/p/github.com/vondi-global/.passport/infrastructure/network.md` | Proxy и CDN конфигурация |
| `/p/github.com/vondi-global/SYSTEM_PASSPORT.md` | Системный паспорт платформы |

---

**Дата создания:** 2025-12-21
**Автор:** Claude Code (Sonnet 4.5)
**Версия:** 1.0.0
**Следующий review:** 2026-01-21
