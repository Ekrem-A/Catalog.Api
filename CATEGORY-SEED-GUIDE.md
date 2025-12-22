# 📦 Kategori Seed Kılavuzu

## 🎯 Kategori Yapısı

Toplam **17 ana kategori** ve **170+ alt kategori** içerir.

### Ana Kategoriler:
1. Kişisel Bilgisayarlar
2. Bilgisayar Bileşenleri
3. Çevre Birimleri
4. Baskı Çözümleri
5. Kurumsal Ürünler
6. Ağ Ürünleri
7. Telefonlar
8. Barkod Ürünleri
9. Güvenlik Ürünleri
10. Tüketici Elektroniği
11. Ofis & Tüketim Ürünleri
12. Yazılımlar
13. Outlet

---

## 🚀 Kategorileri Ekle

### Tek Komut - Migration İle Otomatik!

```bash
# Migration çalıştır - kategoriler otomatik eklenecek
dotnet ef database update --project Catalog.Infrastructure --startup-project Catalog.Api
```

**Hepsi bu kadar!** 170 kategori otomatik eklenir. ✅

---

## 📋 Category Entity Değişiklikleri

### ✅ Yeni Yapı:
```csharp
public class Category : BaseEntity
{
    public string Name { get; set; }
    public string Slug { get; set; }
    public Guid? ParentId { get; set; }  // ← Parent kategori
    public int DisplayOrder { get; set; }
    public string? Description { get; set; }
    
    // Navigation Properties
    public virtual Category? Parent { get; set; }
    public virtual ICollection<Category> Children { get; set; }
    public virtual ICollection<Product> Products { get; set; }
}
```

### ❌ Kaldırılan Alanlar:
- `Level` (string)
- `ParentName` (string)
- `ParentId` (int) → `ParentId` (Guid?) olarak değişti

---

## 🔍 Kategori Sorgulama

### Tüm Kategorileri Getir
```bash
curl http://localhost:5000/api/categories
```

### Sadece Ana Kategoriler
```csharp
var mainCategories = categories.Where(c => c.ParentId == null);
```

### Bir Kategorinin Alt Kategorileri
```csharp
var subCategories = categories.Where(c => c.ParentId == parentId);
```

### Hiyerarşik Yapı (Parent > Child)
```csharp
// Ana kategori
var kisiselBilgisayarlar = categories.First(c => c.Name == "Kişisel Bilgisayarlar");

// Alt kategorileri
var altKategoriler = categories.Where(c => c.ParentId == kisiselBilgisayarlar.Id);
// Sonuç: Notebook, Notebook Aksesuar, Tablet, ...
```

---

## 📊 Kategori Örnekleri

### Kişisel Bilgisayarlar (Ana Kategori)
- Notebook
- Notebook Aksesuar
- Tablet
- All In One Pc
- Masaüstü PC
- Mini PC
- Tablet Aksesuar

### Bilgisayar Bileşenleri (Ana Kategori)
- Anakartlar
- Bellekler
- Ekran Kartları
- Hard Diskler
- İşlemciler
- Bilgisayar Kasası
- Optik Sürücüler
- Soğutma /Overclock
- Diğer Kartlar
- Kart Okuyucular
- Kablolar ve Adaptörler

### Güvenlik Ürünleri (Ana Kategori)
- IP / AHD Kameralar
- NVR / AHD Kayıt Cihazları
- Hırsız Alarm Sistemleri
- İnterkom Sistemleri
- Geçiş Kontrol Sistemleri
- CCTV Kablo
- Switch / Adaptör / AP
- ... (20 alt kategori)

---

## 🛠️ Seed Data Özellikleri

### Slug Oluşturma
Her kategori için otomatik slug oluşturulur:

**Örnekler:**
```
"Kişisel Bilgisayarlar" → "kisisel-bilgisayarlar"
"Bilgisayar Kasası" → "bilgisayar-kasasi"
"IP / AHD Kameralar" → "ip-ahd-kameralar"
"Soğutma /Overclock" → "sogutma-overclock"
```

### Display Order
Kategoriler eklenme sırasına göre otomatik sıralanır (0'dan başlar).

### Default Değerler
- `IsActive`: true
- `CreatedAt`: UTC şu an
- `Description`: null (sonradan eklenebilir)

---

## 🔄 Yeniden Seed (Re-seed)

Eğer kategorileri yeniden eklemek isterseniz:

### 1. Mevcut Kategorileri Sil
```sql
-- SQL ile tüm kategorileri sil
DELETE FROM categories;
```

### 2. Seed Endpoint'ini Tekrar Çağır
```bash
curl -X POST http://localhost:5000/api/categories/seed
```

---

## 🧪 Test

### Kategori Sayısını Kontrol Et
```bash
curl http://localhost:5000/api/categories | jq 'length'
```

### Ana Kategori Sayısı
```bash
curl http://localhost:5000/api/categories | jq '[.[] | select(.parentId == null)] | length'
```

### Bir Ana Kategorinin Alt Kategorileri
```bash
# Önce ana kategori ID'sini al
curl http://localhost:5000/api/categories | jq '.[] | select(.name == "Kişisel Bilgisayarlar")'

# Alt kategorileri getir
curl http://localhost:5000/api/categories | jq '[.[] | select(.parentId == "PARENT_ID_BURAYA")]'
```

---

## 📝 Notlar

### ⚠️ Önemli
- Seed işlemi **sadece bir kez** çalışır
- Eğer kategoriler zaten varsa hata verir
- Migration'ı çalıştırmayı unutmayın!

### ✅ Avantajlar
- 170+ kategori otomatik eklenir
- Hiyerarşik yapı (parent-child)
- SEO-friendly slug'lar
- Türkçe karakter desteği

### 🔄 Güncellemeler
Kategorileri güncellemek için:
1. `CategorySeedData.cs` dosyasını düzenle
2. Migration oluştur (gerekirse)
3. Kategorileri sil ve yeniden seed et

---

## 🐛 Sorun Giderme

### "Categories already exist" Hatası
**Çözüm:** Önce kategorileri silin
```sql
DELETE FROM categories;
```

### Migration Hatası
**Çözüm:** Migration'ı yeniden oluşturun
```bash
dotnet ef migrations remove --project Catalog.Infrastructure --startup-project Catalog.Api
dotnet ef migrations add UpdateCategoryHierarchy --project Catalog.Infrastructure --startup-project Catalog.Api
dotnet ef database update --project Catalog.Infrastructure --startup-project Catalog.Api
```

### Foreign Key Hatası (Products)
**Çözüm:** Önce products tablosunu boşaltın
```sql
DELETE FROM products;
DELETE FROM categories;
```

---

## 🎯 Sonraki Adımlar

1. ✅ Migration çalıştır
2. ✅ Seed endpoint'ini çağır
3. ✅ Kategorileri kontrol et
4. ⏭️ Ürünleri kategorilere atayın
5. ⏭️ Frontend'de kategori menüsü oluşturun

