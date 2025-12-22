# 🐳 Docker Compose Guide

## 📋 Servis Mimarisi

Bu proje **Azure SQL Database** kullanır. Local development için sadece Redis ve Kafka container'ları çalıştırılır.

### Servisler

```
┌─────────────────────────────────────────┐
│         Azure SQL Database              │  ← Cloud (Azure)
│     (ekomyserver.database.windows.net)  │
└─────────────────────────────────────────┘
              ↑
              │ SQL Connection
              │
┌─────────────────────────────────────────┐
│         Catalog API (Container)         │
└─────────────────────────────────────────┘
       ↓                    ↓
┌─────────────┐      ┌─────────────┐
│    Redis    │      │    Kafka    │
│ (Container) │      │ (Container) │
└─────────────┘      └─────────────┘
```

---

## 🚀 Kullanım

### 1. Development (Tüm Servisler)

```bash
# Tüm servisleri başlat (API + Redis + Kafka)
docker-compose up -d

# Logları izle
docker-compose logs -f catalog-api

# Sadece infrastructure servisleri (Redis + Kafka)
docker-compose up -d redis kafka zookeeper

# API'yi local çalıştır
cd Catalog.Api
dotnet run
```

### 2. Production

```bash
# .env dosyası oluştur
cp .env.example .env

# Environment variables'ları düzenle
nano .env

# Production ile başlat
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## 📦 Servis Detayları

### Catalog API
- **Port:** 5000
- **URL:** http://localhost:5000
- **Swagger:** http://localhost:5000/swagger
- **Health:** http://localhost:5000/health
- **Database:** Azure SQL (Cloud)

### Redis
- **Port:** 6379
- **Host:** localhost:6379
- **Data:** Persistent volume (`redis_data`)

### Kafka
- **Port (Internal):** 9092
- **Port (External):** 29092
- **Host:** localhost:29092
- **Topics:** Auto-created

### Zookeeper
- **Port:** 2181
- **Required by:** Kafka

---

## 🔧 Yapılandırma

### Environment Variables

**Development:**
```bash
# appsettings.Development.json kullanılır
# API Key ve Rate Limiting disabled
# Azure SQL bağlantısı appsettings'den gelir
```

**Production:**
```bash
# docker-compose.prod.yml + .env dosyası
export AZURE_SQL_CONNECTION_STRING="Server=tcp:..."
export ORDER_SERVICE_API_KEY="strong-key-here"
export BASKET_SERVICE_API_KEY="strong-key-here"
```

### Azure SQL Connection String

```bash
# Format
Server=tcp:SERVER_NAME.database.windows.net,1433;
Initial Catalog=DATABASE_NAME;
Persist Security Info=False;
User ID=USERNAME;
Password=PASSWORD;
MultipleActiveResultSets=False;
Encrypt=True;
TrustServerCertificate=False;
Connection Timeout=30;
```

**Mevcut Connection:**
```
Server: ekomyserver.database.windows.net
Database: CatalogService_db
User: Ekrem
```

---

## 📊 Servis Yönetimi

### Tüm Servisleri Başlat
```bash
docker-compose up -d
```

### Sadece Infrastructure (Redis + Kafka)
```bash
docker-compose up -d redis kafka zookeeper
```

### Durumu Kontrol Et
```bash
docker-compose ps
```

### Logları İzle
```bash
# Tüm servisler
docker-compose logs -f

# Sadece API
docker-compose logs -f catalog-api

# Sadece Kafka
docker-compose logs -f kafka
```

### Servisleri Durdur
```bash
docker-compose down
```

### Servisleri Durdur + Volume'leri Sil
```bash
docker-compose down -v
```

---

## 🔍 Health Check

### API Health Check
```bash
curl http://localhost:5000/health
```

**Başarılı Response:**
```json
{
  "status": "Healthy",
  "duration": "00:00:00.1234567",
  "entries": {
    "database": {
      "status": "Healthy",
      "description": "Azure SQL Database connected"
    }
  }
}
```

### Redis Health Check
```bash
docker exec -it catalog-redis redis-cli ping
# Çıktı: PONG
```

### Kafka Health Check
```bash
docker exec -it catalog-kafka kafka-topics --bootstrap-server localhost:9092 --list
```

---

## 🐛 Sorun Giderme

### API Azure SQL'e Bağlanamıyor

**Kontrol:**
1. Firewall kuralları Azure Portal'da açık mı?
2. Connection string doğru mu?
3. Database var mı?

```bash
# Azure SQL Firewall - Client IP ekle
az sql server firewall-rule create \
  --resource-group YOUR_RG \
  --server ekomyserver \
  --name AllowMyIP \
  --start-ip-address YOUR_IP \
  --end-ip-address YOUR_IP
```

### Kafka Başlamıyor

**Çözüm:**
```bash
# Zookeeper'ı önce başlat
docker-compose up -d zookeeper

# 10 saniye bekle
sleep 10

# Kafka'yı başlat
docker-compose up -d kafka
```

### Redis Connection Hatası

```bash
# Redis container'ını yeniden başlat
docker-compose restart redis

# Logs kontrol et
docker-compose logs redis
```

### Port Çakışması

**1433 portu kullanımda (Local SQL Server):**
```bash
# Local SQL Server'ı durdur
net stop MSSQLSERVER

# Veya farklı port kullan (gerek yok, Azure SQL kullanıyoruz)
```

**6379 portu kullanımda:**
```bash
# docker-compose.yml'de port değiştir
ports:
  - "6380:6379"  # Host:Container
```

---

## 🔄 Migration

### Otomatik Migration
API başladığında otomatik migrate edilir (`Program.cs`'de yapılandırıldı).

### Manuel Migration
```bash
# Migration oluştur
dotnet ef migrations add MigrationName \
  --project Catalog.Infrastructure \
  --startup-project Catalog.Api

# Database'i güncelle
dotnet ef database update \
  --project Catalog.Infrastructure \
  --startup-project Catalog.Api

# Migration'ı geri al
dotnet ef database update PreviousMigrationName \
  --project Catalog.Infrastructure \
  --startup-project Catalog.Api
```

---

## 📈 Production Deployment

### 1. Azure Container Apps (Önerilen)
```bash
# Docker image build
docker build -t catalog-api:latest -f Catalog.Api/Dockerfile .

# Azure Container Registry'ye push
az acr login --name yourregistry
docker tag catalog-api:latest yourregistry.azurecr.io/catalog-api:latest
docker push yourregistry.azurecr.io/catalog-api:latest

# Container App deploy
az containerapp create \
  --name catalog-api \
  --resource-group YOUR_RG \
  --image yourregistry.azurecr.io/catalog-api:latest \
  --environment YOUR_ENVIRONMENT
```

### 2. Docker Compose Production
```bash
# Environment variables oluştur
cp .env.example .env
nano .env

# Production'da başlat
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## 🧪 Test

### API Test
```bash
# Health check
curl http://localhost:5000/health

# Get products
curl http://localhost:5000/api/products

# Create product (API Key gerekirse)
curl -X POST http://localhost:5000/api/products \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-api-key" \
  -d '{
    "name": "Test Product",
    "brandId": "guid-here",
    "price": 99.99,
    "stockQuantity": 10
  }'
```

### Redis Test
```bash
# Redis'e bağlan
docker exec -it catalog-redis redis-cli

# Komutlar
> PING
> KEYS *
> GET key-name
> SET test-key "test-value"
> EXIT
```

### Kafka Test
```bash
# Topics listele
docker exec -it catalog-kafka kafka-topics \
  --bootstrap-server localhost:9092 \
  --list

# Consumer başlat
docker exec -it catalog-kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic product-events \
  --from-beginning
```

---

## 📝 Notlar

### Azure SQL vs Local SQL

✅ **Azure SQL Avantajları:**
- Managed service (backup, patching otomatik)
- High availability
- Scalable
- Security features (firewall, encryption)
- Geo-replication

❌ **Local SQL Dezavantajları:**
- Manuel backup
- Resource intensive
- Development için yeterli ama production için değil

### Redis Kullanımı

**Cache Keys:**
```
products:all
products:{id}
categories:all
brands:all
```

**Cache Expiration:**
- Product list: 5 dakika
- Product detail: 10 dakika
- Categories: 30 dakika

### Kafka Topics

**Auto-created Topics:**
- `product-created`
- `product-updated`
- `product-deleted`
- `product-price-changed`
- `product-stock-updated`

---

## 🔐 Güvenlik

### Development
```bash
# API Key: Disabled
# Rate Limiting: Disabled
# CORS: Allow All
```

### Production
```bash
# API Key: Enabled (docker-compose.prod.yml)
# Rate Limiting: Enabled (100 req/min)
# CORS: Specific origins only
# HTTPS: Required
```

### API Keys Oluşturma

```bash
# PowerShell
$bytes = New-Object byte[] 32
[Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
[Convert]::ToBase64String($bytes)

# Bash
openssl rand -base64 32
```

---

## 📚 Ek Kaynaklar

- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Azure SQL Database](https://learn.microsoft.com/en-us/azure/azure-sql/)
- [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/)
- [Redis Documentation](https://redis.io/docs/)
- [Kafka Documentation](https://kafka.apache.org/documentation/)

