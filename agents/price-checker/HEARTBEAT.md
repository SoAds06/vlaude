# Price Checker Heartbeat

## Schedule
Her gün Türkiye saati (TRT = UTC+3) ile **16:00**'da çalışır.
Cron ifadesi: `0 13 * * *` (UTC)

## Each Cycle

### 1. Read Context
- `MEMORY.md` dosyasını oku — önceki döngülerden kalıpları kontrol et
- `data/marka.csv` dosyasını oku — güncel marka listesini al
- `data/rivals.csv` dosyasını oku — rakip URL listesini al
- Bugünün `journal/entries/` klasörüne bak — varsa öncelikli sinyal var mı?

### 2. Assess State
- `marka.csv` ve `rivals.csv` dosyaları mevcut ve okunabilir mi?
  - Hayır → İnsan müdahalesi iste, journal'a yaz, dur.
- Bugün için `outputs/YYYY-MM-DD_price-report.md` zaten oluşturulmuş mu?
  - Evet → Zaten çalıştırılmış, tekrar çalıştırma.
- Devam et → PRICE_CHECK skill'ini çalıştır.

### 3. Execute Skill

**Her zaman çalışan tek döngü:**
```
PRICE_CHECK → EMAIL_REPORT
```

Karar ağacı:
- Her gün varsayılan olarak: PRICE_CHECK çalıştır → tamamlanınca EMAIL_REPORT çalıştır
- PRICE_CHECK başarısız olursa: hatayı logla, EMAIL_REPORT'u ÇALIŞTIIRMA, journal'a yaz
- EMAIL_REPORT başarısız olursa: rapor dosyasını `outputs/` klasöründe bırak, journal'a yaz

### 4. Log to Journal
Her döngü sonunda `journal/entries/YYYY-MM-DD_HHMM.md` dosyasına yaz:
- Kaç marka işlendi
- Kaç ürün eşleşti / eşleşmedi
- E-posta gönderildi mi
- Dikkat çekici fiyat farkları (varsa)

## Weekly Review (Her Pazartesi)

### 1. Gather Data
- Son 7 günün `outputs/` klasöründeki rapor dosyalarını oku

### 2. Score Against Targets
| Metric | Target | Bu Hafta | Durum |
|--------|--------|----------|-------|
| Günlük rapor başarısı | 7/7 | ? | ? |
| Ürün eşleşme oranı | %80 | ? | ? |
| E-posta teslimatı | 7/7 | ? | ? |

### 3. Analyze Wins and Misses
- **Başarılar:** Hangi markalar her gün eşleşiyor? → MEMORY.md'ye yaz.
- **Sorunlar:** Hangi rakip siteler tutarsız? Hangi markalar eşleşmiyor? → MEMORY.md'ye yaz.

### 4. Update Memory
Onaylanan kalıpları `MEMORY.md`'nin ilgili bölümlerine ekle.

### 5. Log Weekly Summary to Journal
- İşlenen marka sayısı
- Ortalama eşleşme oranı
- En büyük fiyat farkı bulgusu
- Gelecek hafta için öneriler

## Escalation Rules
- 2+ gün art arda rapor üretilememesi → İnsan müdahalesi gerekir
- Bir rakip site tamamen erişilemez → MEMORY.md'ye not düş, insana bildir
- `marka.csv` veya `rivals.csv` yoksa → Dur, insanı bilgilendir
- E-posta her gün başarısız oluyorsa → İnsanı bilgilendir
