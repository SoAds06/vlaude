# Skill: Ürün Eşleştirme (PRODUCT_MATCH)

## Hedef Bağlantısı
→ "Doğru ürün eşleşmesi" ve "Ortalama sepet değeri" KPI'larını besler.

## Giriş
1. CUSTOMER_INTAKE'ten gelen normalize edilmiş müşteri profili
2. BLOG_READER'dan gelen kriterler sözlüğü

**Bu skill BLOG_READER çalışmadan başlamaz.**
Blog kriterlerinin kapsamadığı hiçbir ürün kararı verilmez.

## Çıkış
Her kategori için seçilmiş ürün listesi (CART_BUILDER'a geçer).

---

## Veri Kaynağı: Local Cache

CACHE_SYNC skill'inin günde bir kez oluşturduğu `data/cache/products.json` dosyasından okunur.

### Cache Geçerlilik Kontrolü (Adım 0)
- `products.json` var mı? → Yoksa CACHE_SYNC tetikle, tamamlanınca devam et
- `last_updated` 24 saatten eski mi? → CACHE_SYNC'i arka planda tetikle (bu isteği bloklamaz)
- Tamsa → direkt oku

**`g:id` değeri T-Soft ürün ID'siyle aynıdır — sepet linkinde bu kullanılır.**

---

## Süreç

### Adım 1 — Cache'i Oku

1. `data/cache/products.json` dosyasını oku
2. `availability = "in stock"` olmayanları ele
3. Kalan ürünleri belleğe al

```
Cache yok → CACHE_SYNC → başarısızsa:
{ "hata": "cache_hatasi", "mesaj": "Ürün veritabanı şu an güncellenemiyor." }
```

### Adım 2 — Kategori Sınıflandırması

Ürünleri `product_type` ve `title` alanlarına göre iç kategoriye eşleştir:

| İç Kategori | Aranacak Kelimeler |
|-------------|-------------------|
| olta_kamisi | kamış, kamis, rod, feeder, fly rod, surf kamış |
| makine | makine, olta makinesi, baitrunner, baitcast, reel |
| misina | misina, örme ip, braid, monofilament |
| lider | fluorokarbon, lider, şok misina, leader, fluoro |
| igne_aksesuar | iğne, igne, jig, kaşık, kanca, hook, lure, soft bait |
| genel_aksesuar | çanta, kutu, gözlük, şamandıra, yem, aksesuar |

Eşleşme bulunamazsa → "bilinmeyen" kategoride bırak, eşleştirmede kullanma.

### Adım 3 — Blog Kriterlerini Uygula

BLOG_READER'dan gelen `kriterler` sözlüğünü al.

Her iç kategori için o kategorinin kriterlerini `title` ve `description` alanlarına karşılaştır:

**Nasıl uygulanır:**
- Kriter bir özelliği zorunlu kılıyorsa ("Fast aksiyon olmalı") → bu kelime veya eşdeğeri ürün başlığında/açıklamasında geçmiyorsa ürünü ele
- Kriter bir şeyi yasaklıyorsa ("baitcasting makine önerilmez") → bu kelime geçen ürünleri ele
- Kriter bir aralık veriyorsa ("270-300cm") → başlıkta bu aralıkla uyumlu sayı/ifade aranır
- Kriter mevcut cache verisiyle doğrulanamıyorsa (örn. "5+1 bilyeli sürgü" ama ürün açıklamasında yok) → kriteri yok say, ürünü eleme; bu kriteri `dogrulanamayan_kriterler` listesine ekle — çıktı JSON'undaki `not` alanına yansıtılır.

**Kritik kural:** Kriter listesinde geçmeyen hiçbir ek filtre uygulanmaz.
Kendi yorumunla ürün eleme — sadece blog kriterleri geçerlidir.

### Adım 4 — Marka Filtresi
- `marka_tercihi` "fark etmez" değilse → `brand` alanını filtrele
- O kategoride tercih edilen markadan ürün yoksa → filtreyi kaldır, nota ekle

### Adım 5 — Bütçe Kontrolü ve Seçim

**Seçim sırası:**
1. Her zorunlu kategoriden (olta + makine + misina + lider + iğne/yem) en az 1 ürün seç
2. Toplam fiyatı hesapla
3. Bütçe altında kalıyorsa genel aksesuar ekle (opsiyonel)
4. Bütçe aşılıyorsa fiyata göre sıralayıp en yakın uygun ürüne geç

**Bütçe hiç tutmuyorsa:**
```json
{
  "hata": "butce_yetersiz",
  "mesaj": "Bu disiplin için minimum ekipman [X] TL. Bütçenizi artırabilir misiniz?",
  "minimum_butce": 0
}
```

### Adım 6 — "Neden" Açıklamasını Blog'dan Türet

Her seçilen ürün için `neden` alanını doldururken:
- İlgili blog kriterinden direkt al veya parafraz et
- Kendi yorumunu ekleme — blog ne diyorsa onu yaz
- Teknik terimi parantez içinde açıkla (blog'da geçiyorsa)
- 1-2 cümle yeterli

**Örnek:**
> Blog kriteri: "Fast aksiyon kıyı levrek avında hassas titreşim iletimi sağlar"
> Neden alanı: "Fast aksiyon (titreşim iletimi yüksek) bu kamış kıyıdan levrek avı için blog rehberimizde önerilen özelliğe sahip."

### Adım 7 — Çıktı Formatı

```json
{
  "oneriler": [
    {
      "kategori": "Olta Kamışı",
      "urun_adi": "Shimano Catana FD Spin 270M",
      "urun_id": "12345",
      "urun_linki": "https://[domain]/urun/shimano-catana-fd-270",
      "fiyat": 890,
      "eşleşen_kriterler": ["270-300cm arası", "Fast aksiyon"],
      "neden": "Kıyı levrek spin avı için hazırladığımız rehberde önerilen 270cm / fast aksiyon özelliklerine sahip."
    }
  ],
  "toplam_fiyat": 2100,
  "butce_max": 2500,
  "kaynak_yazilar": ["Spin Avında Kamış Seçimi", "Levrek Avcılığı Rehberi"],
  "not": "Marka tercihiniz Shimano için tüm ürünler eşleşti."
}
```

`eşleşen_kriterler` alanı: bu ürünün hangi blog kriterini karşıladığını gösterir.

---

## Cache Fallback

**Cache yok** (`products.json` mevcut değil):
→ CACHE_SYNC tetikle ve bekle → başarısızsa hata döndür.

**Cache eski** (`last_updated` > 24 saat):
→ CACHE_SYNC'i arka planda tetikle (bu isteği bloklamaz) → mevcut cache ile devam et.

---

## Kalite Çıtası
- Stokta olmayan ürün seçilmez
- Toplam fiyat `butce_max` geçmez
- Her seçim için `neden` blog içeriğinden türetilmiş olur
- Blog kriterleri dışından hiçbir kural eklenmez
- En az 5 zorunlu kategori (olta, makine, misina, lider, iğne/yem) dolu olmadan CART_BUILDER'a geçilmez
- Eşleşen blog yoksa döngü durur, öneri yapılmaz
