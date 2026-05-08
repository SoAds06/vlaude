# Skill: Sepet Linki Oluşturma (CART_BUILDER)

## Hedef Bağlantısı
→ "Sepet dönüşümü" ve "Yanıt hızı" KPI'larını besler.

## Giriş
PRODUCT_MATCH skill'inden çıkan öneri listesi.

## Çıkış
T-Soft sepet linki (ana + alternatif) ve tam sonuç JSON'u.

---

## Süreç

### Adım 1 — T-Soft Sepet Link Formatını Uygula

T-Soft'ta birden fazla ürünü sepete ekleyen link formatı:

```
https://[SITE_DOMAIN]/sepet?ekle=[urun_id1]:[adet1],[urun_id2]:[adet2],...
```

**Örnek:**
```
https://balikcidunyasi.com/sepet?ekle=12345:1,12346:1,12347:1,12348:1
```

Her ürün için adet = 1 (varsayılan). Misina için adet müşteri tercihi yoksa 1.

**Not:** `SITE_DOMAIN` ve `CART_BASE_URL` değerleri `data/imports/FEED_CONFIG.md` dosyasından okunur.

### Adım 2 — Ana Sepet Linkini Oluştur

1. PRODUCT_MATCH çıktısındaki `urun_id` değerlerini al
2. Her birini `urun_id:1` formatında virgülle birleştir
3. Base URL'e ekle
4. URL encode gerekiyorsa uygula (Türkçe karakter varsa)

```
urun_idleri = ["12345", "12346", "12347", "12348"]
link = "https://[domain]/sepet?ekle=" + ",".join([id + ":1" for id in urun_idleri])
```

### Adım 3 — Alternatif Sepet Linki Oluştur (varsa)

PRODUCT_MATCH "bütçe dostu alternatif" versiyonu döndürdüyse:
- Aynı format ile ikinci bir link oluştur
- Ana linkten en az %15 daha ucuz olmalı

### Adım 4 — Link Doğrulaması
- URL geçerli formatında mı? (köşeli parantez, boşluk veya encode edilmemiş karakter var mı?)
- Toplam ürün sayısı 4–10 arasında mı?
- PRODUCT_MATCH çıktısında 4 zorunlu kategori mevcut olmalı (olta, makine, misina, lider/iğne) — eksikse PRODUCT_MATCH hatası olarak değerlendir, işlemi durdur.

Kontrol başarısızsa → hata döndür, log yaz.

### Adım 5 — Sonuç JSON'unu Hazırla

```json
{
  "musteri_profili": {
    "av_disiplini": "spin",
    "ortam": "kıyı",
    "marka_tercihi": "shimano",
    "butce_max": 2500,
    "deneyim": "intermediate"
  },
  "oneriler": [
    {
      "kategori": "Olta Kamışı",
      "urun_adi": "Shimano Catana FD 270M",
      "urun_id": "12345",
      "fiyat": 890,
      "neden": "Spin avı için ideal 270cm / 7-35g güç aralığı."
    },
    {
      "kategori": "Makine",
      "urun_adi": "Shimano Sienna 3000",
      "urun_id": "12346",
      "fiyat": 650,
      "neden": "3000 beden, orta kullanıcıya uygun fiyat/performans."
    },
    {
      "kategori": "Misina",
      "urun_adi": "Shimano Kairiki 0.20mm 150m",
      "urun_id": "12347",
      "fiyat": 280,
      "neden": "Örme ip, spin avı için ideal duyarlılık."
    },
    {
      "kategori": "İğne & Aksesuar",
      "urun_adi": "Owner Cultiva Jig Başlık Seti",
      "urun_id": "12348",
      "fiyat": 180,
      "neden": "Levrek avı için standart jig kurşun seti."
    }
  ],
  "toplam_fiyat": 2000,
  "butce_max": 2500,
  "sepet_linki": "https://[domain]/sepet?ekle=12345:1,12346:1,12347:1,12348:1",
  "alternatif_sepet_linki": null,
  "not": "Tüm ürünler stokta mevcut. Shimano markası tercihlerinize göre seçildi."
}
```

### Adım 6 — Log Satırı Yaz

`outputs/YYYY-MM-DD_log.md` dosyasına ekle:
```markdown
| HH:MM | spin | kıyı | shimano | 2500 TL | 2000 TL | ✓ link oluşturuldu |
```

---

## T-Soft Sepet Link Alternatifleri

T-Soft versiyonuna göre link formatı farklılık gösterebilir. Önce ana formatı dene, çalışmıyorsa alternatifleri sırayla dene:

| Format | URL |
|--------|-----|
| Ana format | `/sepet?ekle=ID1:1,ID2:1` |
| Alternatif 1 | `/sepet/ekle?urunler=ID1,ID2` |
| Alternatif 2 | `/urun/sepeteekle?id=ID1&id=ID2` |

**Not:** Hangi format çalıştığını ilk başarılı kullanımda `MEMORY.md`'ye yaz.

---

## Kalite Çıtası
- Link açıldığında ürünler direkt sepete eklenmiş olmalı (test edilmeli)
- JSON formatı n8n'in beklediği şemayla uyumlu olmalı
- `neden` alanları müşteri diline uygun, teknik ama anlaşılır olmalı
- Stokta olmayan ürün ID'si asla linke girmez
