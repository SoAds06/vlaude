# Skill: Müşteri Verisi Alma (CUSTOMER_INTAKE)

## Hedef Bağlantısı
→ "Doğru ürün eşleşmesi" KPI'sini besler. Temiz veri girmeden doğru öneri çıkmaz.

## Giriş
n8n'den gelen JSON payload.

## Çıkış
Normalize edilmiş müşteri profili (iç format) veya hata mesajı.

---

## Süreç

### Adım 1 — Zorunlu Alanları Kontrol Et
Aşağıdaki alanlar eksikse döngü durdur ve hata döndür:

| Alan | Beklenen değerler |
|------|-------------------|
| `av_disiplini` | spin, kıl iğne, sazancılık, predator, sinek avı, surf casting |
| `ortam` | kıyı, açık deniz, göl, nehir, nehir ağzı, baraj |
| `butce_max` | pozitif sayı (TL) |

**Hata formatı:**
```json
{
  "hata": "eksik_alan",
  "mesaj": "Av disiplini belirtilmemiş. Lütfen tekrar gönderin.",
  "eksik_alanlar": ["av_disiplini"]
}
```

### Adım 2 — Opsiyonel Alanları Doldur (Default)
| Alan | Eksikse Default |
|------|----------------|
| `hedef_balik` | av disiplininden çıkar (bkz. eşleştirme tablosu) |
| `marka_tercihi` | "fark etmez" |
| `deneyim` | "intermediate" |
| `butce_min` | 0 |

### Adım 3 — Disiplin → Hedef Balık Eşleştirmesi (hedef_balik eksikse)
| Disiplin | Varsayılan Hedef Balık |
|----------|----------------------|
| spin | levrek, alabalık |
| kıl iğne | istavrit, çinekop, palamut |
| sazancılık | sazan |
| predator | levrek, turna |
| sinek avı | alabalık, kızılkanat |
| surf casting | kefal, levrek, tekir |

### Adım 4 — Değerleri Normalize Et
- `av_disiplini` → küçük harf, Türkçe karakter korunur
- `butce_min` ve `butce_max` → integer
- `marka_tercihi` → küçük harf (Shimano → shimano, Fark etmez → fark etmez)
- `deneyim` → `beginner | intermediate | advanced` (diğer değerleri "intermediate" say)

### Adım 5 — Müşteri Profili Oluştur
```json
{
  "av_disiplini": "spin",
  "hedef_balik": ["levrek", "alabalık"],
  "ortam": "kıyı",
  "marka_tercihi": "shimano",
  "butce_min": 500,
  "butce_max": 2000,
  "deneyim": "intermediate"
}
```
Bu profil PRODUCT_MATCH skill'ine geçer.

---

## Kalite Çıtası
- Zorunlu alanların tamamı dolu veya açık hata mesajı dönmeli
- Normalize edilmiş profil tutarlı olmalı (büyük/küçük harf karışıklığı yok)
- Hata mesajları müşteri yönlendirilebilir dilde yazılmış olmalı
