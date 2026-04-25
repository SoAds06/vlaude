# Skill: E-posta Raporu (EMAIL_REPORT)

## Purpose
Günlük fiyat karşılaştırma raporunu formatlar ve sergen@sihirliolta.com adresine e-posta olarak gönderir.

## Serves Goals
- Zamanlama (16:00 TRT'de tamamlanma)
- Günlük rapor üretimi

## Inputs
- Bugünün `outputs/YYYY-MM-DD_price-report.md` dosyası (PRICE_CHECK skill'i tarafından üretilmiş)

## Process

### Aşama 1 — Raporu Oku ve Özetle
1. `outputs/YYYY-MM-DD_price-report.md` dosyasını oku.
2. Şu özet istatistikleri hesapla:
   - İşlenen marka sayısı
   - Toplam karşılaştırılan ürün sayısı
   - Başarılı eşleşme oranı (%)
   - En büyük fiyat avantajı (sihirliolta.com daha ucuz olan ürün)
   - En büyük fiyat dezavantajı (sihirliolta.com daha pahalı olan ürün)

### Aşama 2 — E-posta İçeriğini Hazırla

**Konu:** `[sihirliolta.com] Günlük Fiyat Raporu — YYYY-MM-DD`

**Gövde:**

```
Merhaba,

YYYY-MM-DD tarihli günlük fiyat karşılaştırma raporu aşağıdadır.

📊 ÖZET
• İşlenen marka: X
• Karşılaştırılan ürün: X
• Eşleşme oranı: %X
• En büyük avantaj: [Ürün Adı] — rakipten X TL ucuz ([Rakip])
• En büyük dezavantaj: [Ürün Adı] — rakipten X TL pahalı ([Rakip])

📋 DETAY TABLO

[outputs/YYYY-MM-DD_price-report.md içindeki tam tabloyu buraya yapıştır]

---
Bu rapor Price Checker Agent tarafından otomatik olarak oluşturulmuştur.
Çalışma saati: YYYY-MM-DD 16:00 TRT
```

### Aşama 3 — E-postayı Gönder
3. Aşağıdaki komutu çalıştır:

```bash
mail -s "[sihirliolta.com] Günlük Fiyat Raporu — $(date +%Y-%m-%d)" \
     -a "Content-Type: text/plain; charset=UTF-8" \
     sergen@sihirliolta.com < /tmp/price_report_email.txt
```

Alternatif (mail komutu yoksa) — curl ile SendGrid API:
```bash
curl -X POST https://api.sendgrid.com/v3/mail/send \
  -H "Authorization: Bearer $SENDGRID_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "personalizations": [{"to": [{"email": "sergen@sihirliolta.com"}]}],
    "from": {"email": "agent@sihirliolta.com", "name": "Price Checker"},
    "subject": "[sihirliolta.com] Günlük Fiyat Raporu — YYYY-MM-DD",
    "content": [{"type": "text/html", "value": "[HTML_CONTENT]"}]
  }'
```

4. Gönderim başarılıysa journal'a yaz: "E-posta gönderildi — [tarih saat]".
5. Gönderim başarısız olursa: hatayı journal'a yaz, rapor dosyasını `outputs/` klasöründe bırak.

## Outputs
- E-posta: sergen@sihirliolta.com adresine gönderilir
- Journal kaydı: gönderim durumu

## Quality Bar
- E-posta konusu her zaman tarih içerir
- Gövdede hem özet istatistikler hem tam tablo bulunur
- Gönderim başarısız olsa bile rapor dosyası silinmez

## Tools
- Bash — mail komutu veya curl ile e-posta gönderimi

## Integration
PRICE_CHECK skill'inden sonra çalışır.
PRICE_CHECK başarısız olmuşsa bu skill çalıştırılmaz.

## Yapılandırma Notu
E-posta göndermek için sistemde `mail` komutu veya `SENDGRID_API_KEY` ortam değişkeni tanımlı olmalıdır.
Hangisi mevcut olduğunu `which mail` ile kontrol et; yoksa SendGrid yolunu kullan.
