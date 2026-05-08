# Olta Danışmanı — Heartbeat

## Tetiklenme Modeli
Bu agent **event-driven** çalışır — cron değil. n8n'den gelen her müşteri isteği bir döngü başlatır.

### n8n Entegrasyonu
```
n8n (Webhook/Form node)
    → Claude API (HTTP Request node)
        POST body: { "agent": "olta-danismani", "musteri_verisi": { ... } }
    → Olta Danışmanı HEARTBEAT döngüsü çalışır
    → Sonuç JSON'u n8n'e döner
    → n8n müşteriye iletir (e-posta, WhatsApp, chatbot yanıtı vb.)
```

### Zorunlu Günlük Döngü — Cache Sync
Her gün **saat 09:00** (n8n Schedule node ile tetiklenir):
1. CACHE_SYNC skill'ini çalıştır → `data/cache/products.json` güncelle
2. Sync log'unu oku — anormal değişiklik var mı? (>%5 ürün değişimi → insan flag et)
3. Önceki günün log dosyasını özetle
4. MEMORY.md'yi güncelle (pattern varsa)

---

## Her Tetiklemede (Müşteri İsteği)

### 1. Müşteri Verisini Al ve Doğrula
- n8n'den gelen JSON'u oku ve normalize et
- Zorunlu/opsiyonel alan listesi ve default değerler: `skills/CUSTOMER_INTAKE.md` Adım 1-2
- Eksik zorunlu alan varsa → hata JSON'u döndür, döngü durur

**Skill:** `CUSTOMER_INTAKE.md`

### 2. Cache ve Blog Yazılarını Hazırla (bağımsız — aynı anda)
- **Cache:** `data/cache/products.json` dosyasını oku. Geçerlilik ve stale kontrolü: `PRODUCT_MATCH.md` Adım 0.
- **Blog:** `knowledge/BLOG_INDEX.md` dosyasını oku → müşteri profiline uyan yazıları belirle → "Ürün Seçim Kriterleri" bölümlerini çıkar
- Her ikisi hazır olunca PRODUCT_MATCH başlar
- Eşleşen blog yazısı yoksa → hata döndür, döngüyü durdur

**Skills:** `CACHE_SYNC.md` (gerekirse) + `BLOG_READER.md`

### 3. Ürün Eşleştir
- Blog kriterlerini cache ürünlerine uygula
- Kriterlere uymayan ürünleri ele
- Marka ve bütçe filtresi uygula

**Skill:** `PRODUCT_MATCH.md`

### 5. Sepet Linki Oluştur
- Ana öneri için T-Soft sepet linki oluştur
- Varsa bütçe dostu alternatif için ikinci link oluştur

**Skill:** `CART_BUILDER.md`

### 6. Log Yaz
- `outputs/YYYY-MM-DD_log.md` dosyasına satır ekle:
  ```
  | HH:MM | disiplin | ortam | bütçe | toplam | sonuç |
  ```
  `sonuç` değerleri: `✓ link` | `✗ blog_yok` | `✗ butce_yetersiz` | `✗ eksik_alan` | `✗ cache_hatasi`

### 7. JSON Döndür
- n8n'e sonuç JSON'unu ver (Output Contract formatında)

---

## Karar Ağacı — Hangi Skill Çalışır?

```
Tetikleme geldi
    │
    ├─ Müşteri verisi eksik (zorunlu alan) → CUSTOMER_INTAKE → hata döndür
    │
    ├─ Veri tam → BLOG_READER çalıştır
    │       │
    │       ├─ Eşleşen blog yazısı yok → hata: "blog_yok" döndür (insan ekleme yapmalı)
    │       │
    │       └─ Kriterler oluştu → PRODUCT_MATCH çalıştır
    │               │
    │               ├─ Bütçeye uyan ürün bulunamadı → hata: "bütçe yetersiz" döndür
    │               │
    │               └─ Ürünler bulundu → CART_BUILDER çalıştır → JSON döndür
    │
    └─ Günlük döngü (sabah 09:00) → CACHE_SYNC + log özeti + MEMORY.md güncelle
```

---

## Haftalık İnceleme (Her Pazartesi 09:00)

### 1. Veri Topla
- `outputs/` klasöründeki haftalık log dosyalarını oku
- Kaç istek geldi, kaç sepet linki oluşturuldu

### 2. Hedeflere Göre Değerlendir

Log formatı (`sonuç` sütunu): `✓ link` | `✗ blog_yok` | `✗ butce_yetersiz` | `✗ eksik_alan` | `✗ cache_hatasi`

| Metrik | Hedef | Bu Hafta | Durum |
|--------|-------|----------|-------|
| Toplam istek | — | [log satırı say] | Bilgi |
| Hata oranı (tüm ✗) | <%10 | [✗ satır / toplam] | OK / Uyarı |
| En çok sorulan disiplin | — | [log'dan en sık değer] | Bilgi |
| En çok tükenen kategori | — | [CACHE_SYNC sync_log'undan] | Bilgi |

**Manuel güncelleme gerektiren metrikler (insan girer — T-Soft panelinden):**
| Sepet dönüşümü | >%25 | [manuel] |
| Ortalama sepet değeri | Bütçenin %80-100'ü | [manuel] |

### 3. Pattern Analizi
- Hangi disiplin + ortam kombinasyonu en çok soruluyor?
- Hangi kategoride stok sıkça yetersiz kalıyor?
- Hangi bütçe aralığı en yaygın?

### 4. Hafızayı Güncelle
- Onaylanan kalıpları `MEMORY.md`'ye ekle

### 5. Journal'a Yaz
- Toplam istek sayısı
- Hata oranı
- Stok sorunları varsa flag et (insan onayı gerekiyor)

---

## Eskalasyon Kuralları
- Aynı kategoride 3+ ardışık "stok yok" hatası → journal'a yaz, insan flag et
- Hata oranı haftada >%20 → insan onayı iste (n8n flow bozuk olabilir)
- Google Shopping XML feed'e ulaşılamazsa → log yaz, retry yapma — hata döndür
- MEMORY.md'de çelişkili pattern varsa → temizlemek için insan onayı iste
