# 🔐 Catalog API Security Guide

## Güvenlik Katmanları

Bu API aşağıdaki güvenlik katmanlarına sahiptir:

### ✅ 1. API Key Authentication
- Tüm endpoint'ler API Key gerektirir (Health check hariç)
- Header: `X-API-Key: your-api-key`
- Her servis için ayrı API Key

### ✅ 2. Rate Limiting
- IP bazlı rate limiting
- Varsayılan: 100 request / 60 saniye
- Response headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

### ✅ 3. Security Headers
- X-Content-Type-Options: nosniff
- X-Frame-Options: DENY
- X-XSS-Protection: 1; mode=block
- Content-Security-Policy
- HSTS (HTTPS'te)

### ✅ 4. CORS Policy
- Sadece belirlenen origin'lerden erişim
- Credential support

### ✅ 5. HTTPS Enforcement
- Production'da zorunlu HTTPS

### ✅ 6. Request Size Limit
- Maksimum request boyutu: 10MB

---

## 🔧 Konfigürasyon

### appsettings.json

```json
{
  "ApiKeySettings": {
    "Enabled": true,
    "HeaderName": "X-API-Key",
    "ValidKeys": [
      {
        "Key": "your-order-service-key-here",
        "ServiceName": "OrderService",
        "Description": "Order Service API Key"
      },
      {
        "Key": "your-basket-service-key-here",
        "ServiceName": "BasketService",
        "Description": "Basket Service API Key"
      }
    ]
  },
  "RateLimitSettings": {
    "Enabled": true,
    "MaxRequestsPerWindow": 100,
    "WindowSeconds": 60
  },
  "CorsSettings": {
    "AllowedOrigins": [
      "https://yourdomain.com"
    ]
  }
}
```

---

## 🚀 Kullanım

### 1. Development (Güvenlik Kapalı)

```bash
# appsettings.Development.json'da güvenlik disabled
dotnet run --environment=Development
```

**Test Request:**
```bash
curl http://localhost:5000/api/products
```

### 2. Production (Güvenlik Açık)

**Test Request:**
```bash
curl https://api.yourdomain.com/api/products \
  -H "X-API-Key: your-order-service-key-here"
```

---

## 📋 API Key Oluşturma

### Güvenli API Key Üretme (PowerShell)

```powershell
# 32 byte rastgele key
$bytes = New-Object byte[] 32
[Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
[Convert]::ToBase64String($bytes)
```

### Güvenli API Key Üretme (Bash/Linux)

```bash
# 32 byte rastgele key
openssl rand -base64 32
```

### Güvenli API Key Üretme (C#)

```csharp
using System.Security.Cryptography;

var randomBytes = new byte[32];
using var rng = RandomNumberGenerator.Create();
rng.GetBytes(randomBytes);
var apiKey = Convert.ToBase64String(randomBytes);
Console.WriteLine(apiKey);
```

---

## 🔍 Response Codes

### Başarılı
- `200 OK` - İstek başarılı
- `201 Created` - Kaynak oluşturuldu
- `204 No Content` - İşlem başarılı, içerik yok

### Client Hataları
- `400 Bad Request` - Geçersiz istek
- `401 Unauthorized` - API Key geçersiz veya eksik
- `404 Not Found` - Kaynak bulunamadı
- `429 Too Many Requests` - Rate limit aşıldı

### Server Hataları
- `500 Internal Server Error` - Sunucu hatası

---

## 🛡️ Best Practices

### 1. API Key Yönetimi

**✅ YAPILMASI GEREKENLER:**
- API Key'leri environment variables'da sakla
- API Key'leri düzenli olarak rotate et
- Her servis için farklı API Key kullan
- API Key'leri log'lama

**❌ YAPILMAMASI GEREKENLER:**
- API Key'leri kod içinde hardcode etme
- API Key'leri Git'e commit etme
- Aynı key'i tüm servisler için kullanma
- API Key'leri client-side'da kullanma

### 2. Rate Limiting

```csharp
// Response Headers
X-RateLimit-Limit: 100        // Limit
X-RateLimit-Remaining: 95     // Kalan
X-RateLimit-Reset: 2024-01-01T12:00:00Z  // Reset zamanı
```

Rate limit aşıldığında:
```json
{
  "error": "Too Many Requests",
  "message": "Rate limit exceeded. Maximum 100 requests per 60 seconds.",
  "retryAfter": 60
}
```

### 3. Error Handling

API Key eksik:
```json
{
  "error": "Unauthorized",
  "message": "API Key is missing"
}
```

API Key geçersiz:
```json
{
  "error": "Unauthorized",
  "message": "Invalid API Key"
}
```

---

## 🧪 Test Etme

### Postman Collection

```json
{
  "name": "Catalog API",
  "request": {
    "method": "GET",
    "header": [
      {
        "key": "X-API-Key",
        "value": "{{api_key}}",
        "type": "text"
      }
    ],
    "url": {
      "raw": "{{base_url}}/api/products",
      "host": ["{{base_url}}"],
      "path": ["api", "products"]
    }
  }
}
```

### cURL Examples

**Get All Products:**
```bash
curl -X GET "https://api.yourdomain.com/api/products" \
  -H "X-API-Key: your-api-key"
```

**Create Product:**
```bash
curl -X POST "https://api.yourdomain.com/api/products" \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Samsung Galaxy S24",
    "brandId": "guid-here",
    "price": 999.99,
    "stockQuantity": 50
  }'
```

**Rate Limit Test:**
```bash
# 101 request gönder (limit aşımı için)
for i in {1..101}; do
  curl -X GET "https://api.yourdomain.com/api/products" \
    -H "X-API-Key: your-api-key"
done
```

---

## 🔐 Production Deployment

### 1. Azure Key Vault ile API Key Yönetimi

```csharp
// Program.cs
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{keyVaultName}.vault.azure.net/"),
    new DefaultAzureCredential());
```

### 2. Environment Variables

```bash
# Docker
docker run -e ApiKeySettings__ValidKeys__0__Key="your-key" catalog-api

# Kubernetes
kubectl create secret generic catalog-api-keys \
  --from-literal=order-service-key='your-key'
```

### 3. SSL Certificate

```bash
# Let's Encrypt
certbot certonly --standalone -d api.yourdomain.com
```

---

## 📊 Monitoring

### Log Formatı

```
[2024-01-01 12:00:00] INFO: API request from service: OrderService
[2024-01-01 12:00:01] WARNING: Rate limit exceeded for IP: 192.168.1.1
[2024-01-01 12:00:02] WARNING: Invalid API Key attempted. IP: 192.168.1.2
```

### Application Insights Query

```kusto
requests
| where customDimensions.ServiceName == "OrderService"
| summarize count() by bin(timestamp, 1h)
```

---

## ⚠️ Güvenlik Kontrol Listesi

### Deployment Öncesi

- [ ] API Key'ler güçlü ve unique mi?
- [ ] API Key'ler environment variables'da mı?
- [ ] HTTPS zorunlu mu?
- [ ] CORS doğru yapılandırıldı mı?
- [ ] Rate limiting aktif mi?
- [ ] Security headers eklendi mi?
- [ ] Logging aktif mi?
- [ ] Error messages hassas bilgi içermiyor mu?

### Düzenli Kontroller

- [ ] API Key rotation (her 90 günde bir)
- [ ] Log analizi (şüpheli aktivite)
- [ ] Rate limit ayarları güncel mi?
- [ ] SSL certificate geçerli mi?

---

## 🆘 Sorun Giderme

### "API Key is missing" hatası
```bash
# Header'ı kontrol et
curl -v https://api.yourdomain.com/api/products \
  -H "X-API-Key: your-key"
```

### "Invalid API Key" hatası
- appsettings.json'da key'in doğru olduğunu kontrol et
- Whitespace veya line break olmadığından emin ol

### Rate limit sürekli aşılıyor
```json
// appsettings.json - Limit artır
{
  "RateLimitSettings": {
    "MaxRequestsPerWindow": 200,
    "WindowSeconds": 60
  }
}
```

---

## 📚 Ek Kaynaklar

- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [Microsoft API Security Best Practices](https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-security)
- [Rate Limiting Strategies](https://cloud.google.com/architecture/rate-limiting-strategies-techniques)

