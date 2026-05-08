# Skill: Cache Senkronizasyonu (CACHE_SYNC)

## Hedef Bağlantısı
→ "Yanıt hızı" KPI'sını besler. Her müşteri isteğinde XML fetch yapmak yerine local cache'den okur.

## Tetiklenme Koşulları
- Her gün **09:00** — n8n Schedule node (zamanlama `HEARTBEAT.md`'de tanımlı)
- `products.json` yoksa — HEARTBEAT veya PRODUCT_MATCH tarafından çağrılır, bekletir (blocking)
- `last_updated` > 24 saat — HEARTBEAT veya PRODUCT_MATCH tarafından çağrılır, arka planda (non-blocking)

## Giriş
`data/imports/FEED_CONFIG.md` — feed URL'i

## Çıkış
`data/cache/products.json` — güncel ürün cache'i
`data/cache/sync_log.md` — sync özeti

---

## Cache Dosyası Formatı

`data/cache/products.json`:
```json
{
  "last_updated": "2026-05-08T09:00:00",
  "feed_url": "https://[domain]/googlebase.xml",
  "total_products": 1247,
  "products": [
    {
      "id": "12345",
      "title": "Shimano Catana FD Spin Kamışı 270cm",
      "price": 890.0,
      "availability": "in stock",
      "brand": "Shimano",
      "product_type": "Balıkçılık > Olta Kamışı > Spin Kamışı",
      "link": "https://[domain]/urun/shimano-catana-fd-270",
      "description": "Spin avı için ideal, 7-35g..."
    }
  ]
}
```

---

## Süreç

### Adım 1 — Cache Geçerliliğini Kontrol Et

`data/cache/products.json` dosyasını oku:
- Dosya yoksa → **Tam Sync** yap (ilk kurulum)
- `last_updated` değeri 24 saatten eskiyse → **Artımlı Sync** yap
- `last_updated` 24 saatten yeniyse → sync yapma, çık

```
şu_an - last_updated > 24 saat?
    Evet → Artımlı Sync
    Hayır → Çık (cache geçerli)
    Dosya yok → Tam Sync
```

---

### SEÇENEK A — Tam Sync (İlk Kurulum)

1. `FEED_CONFIG.md`'den feed URL'ini al
2. HTTP GET ile XML feed'i indir
3. Tüm `<item>` elementlerini parse et
4. Her item için şu alanları çıkar:
   - `g:id` → `id`
   - `title` → `title`
   - `g:price` → `price` (sayı olarak: `"890.00 TRY"` → `890.0`)
   - `g:availability` → `availability`
   - `g:brand` → `brand`
   - `g:product_type` → `product_type`
   - `link` → `link`
   - `description` → `description` (ilk 300 karakter yeterli)
5. Tüm ürünleri `products.json`'a yaz
6. `last_updated` = şu anki zaman damgası
7. Sync log'una yaz

---

### SEÇENEK B — Artımlı Sync (Günlük)

Sadece değişen ürünleri günceller. Tam download yapılır ama yazma minimumdur.

1. Feed URL'den XML'i indir (tam download — ağ trafiği aynı, ama yazma az)
2. Mevcut cache'i `id → ürün` sözlüğü olarak hafızaya al
3. Yeni feed'deki her `<item>` için:

```
yeni_urun = parse(item)
eski_urun = cache.get(yeni_urun.id)

eğer eski_urun yoksa:
    → YENİ ürün: cache'e ekle, değişiklik sayacını artır

eğer eski_urun varsa:
    değişti_mi = (
        eski_urun.price != yeni_urun.price VEYA
        eski_urun.availability != yeni_urun.availability
    )
    eğer değişti_mi:
        → GÜNCELLE: cache'deki kaydı yeni değerlerle değiştir, sayacı artır
    değilse:
        → Dokunma
```

4. Cache'deki ama yeni feed'de olmayan ürünler:
   - `availability` = `"out of stock"` yap (silme — geçmiş isteklerde referans olabilir)

5. Güncellenmiş cache'i `products.json`'a yaz
6. `last_updated` = şu anki zaman damgası

---

### Adım Son — Sync Log'u Güncelle

`data/cache/sync_log.md` dosyasına satır ekle:

```markdown
| 2026-05-08 09:00 | Artımlı | 1247 toplam | 12 güncellendi | 3 yeni | 0 silindi |
```

---

## Hata Durumları

| Durum | Davranış |
|-------|----------|
| Feed URL erişilemiyor | Mevcut cache'i koru, log'a yaz, insan flag et |
| Feed XML bozuk (parse hatası) | Mevcut cache'i koru, log'a yaz |
| Cache dosyası bozuk | Tam sync yap (yeniden oluştur) |
| Disk yazma hatası | Log'a yaz, insan flag et |

**Kritik kural:** Herhangi bir hata durumunda mevcut `products.json` **asla silinmez veya üzerine yazılmaz** — önce yeni veri doğrulanır, sonra dosya güncellenir.

---

## Kalite Çıtası
- Sync sonrası `products.json` geçerli JSON formatında olmalı
- `last_updated` her sync'te güncellenmiş olmalı
- Artımlı sync, tam sync'e kıyasla toplam ürünün %5'inden fazlasını değiştirmemeli (fazlaysa insan flag et — feed yapısı değişmiş olabilir)
- Sync log her sync sonrası bir satır içermeli
