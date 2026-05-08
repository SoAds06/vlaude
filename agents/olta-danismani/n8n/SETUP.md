# n8n Entegrasyon Kurulum Kılavuzu

Platform: **n8n Cloud** | Müşteri verisi: **Typeform / Tally webhook** | AI: **Anthropic Claude API**

---

## Genel Mimari

```
Typeform/Tally (form)
    → n8n Webhook (müşteri verisi)
        → GitHub (blog kriterleri + ürün cache)
        → Claude API (akıllı eşleştirme)
        → T-Soft Sepet Linki
    → Müşteriye yanıt (e-posta / form teşekkür sayfası / WhatsApp)

n8n Schedule (her gün 09:00)
    → T-Soft XML Feed
        → GitHub Gist (products.json cache güncelle)
```

---

## Ön Hazırlık (1 kez yapılır)

### Adım 1 — vlaude Reposunu GitHub'a Push Et

Blog dosyaları ve konfigürasyon n8n Cloud'dan erişilebilir olmalı.

```bash
cd /Users/SoAds/Desktop/vlaude/vlaude
git remote add origin https://github.com/[KULLANICI_ADI]/vlaude.git
git push -u origin main
```

n8n şu dosyaları raw URL ile okuyacak:
```
https://raw.githubusercontent.com/[KULLANICI]/vlaude/main/agents/olta-danismani/knowledge/BLOG_INDEX.md
https://raw.githubusercontent.com/[KULLANICI]/vlaude/main/agents/olta-danismani/knowledge/blogs/spin-levrek-avi.md
```

### Adım 2 — Ürün Cache için GitHub Gist Oluştur

Ürün cache'i (products.json) GitHub Gist'te saklanır. Ücretsiz ve n8n'den kolayca okunur/yazılır.

1. https://gist.github.com adresine git
2. "New gist" tıkla
3. Dosya adı: `products.json`
4. İçerik: `{}` (boş, ilk sync dolduracak)
5. "Create secret gist" tıkla
6. URL'deki Gist ID'yi not al: `https://gist.github.com/[kullanici]/[GIST_ID]`

### Adım 3 — GitHub Personal Access Token Oluştur

Gist'e yazabilmek için token gerekli.

1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. "Generate new token (classic)"
3. Scope: sadece `gist` işaretle
4. Token'ı kopyala ve güvenli bir yere kaydet

### Adım 4 — n8n'de Credentials Ekle

n8n Cloud'da şu credential'ları oluştur:

**Anthropic:**
- Settings → Credentials → Add credential → Anthropic
- API Key: Anthropic Console'dan al

**GitHub (Gist için):**
- Add credential → GitHub
- Personal Access Token: Adım 3'teki token

---

## Workflow 1: Günlük Cache Sync

n8n'de yeni bir workflow oluştur, adı: `Olta Danışmanı — Günlük Cache Sync`

### Node 1: Schedule Trigger
- Trigger: Schedule
- Rule: Every Day at 09:00 (Europe/Istanbul timezone)

### Node 2: HTTP Request — XML Feed Çek
- Method: GET
- URL: `[FEED_URL]` (FEED_CONFIG.md'den)
- Response Format: Text
- Timeout: 30000ms

### Node 3: Code — XML Parse Et
Script: bkz. `CODE_NODES.md` → "XML Parser"

### Node 4: HTTP Request — Gist'e Yaz
- Method: PATCH
- URL: `https://api.github.com/gists/[GIST_ID]`
- Authentication: GitHub (Predefined Credential Type)
- Body (JSON):
```json
{
  "files": {
    "products.json": {
      "content": "={{ JSON.stringify($json) }}"
    }
  }
}
```

---

## Workflow 2: Müşteri İsteği

n8n'de yeni bir workflow oluştur, adı: `Olta Danışmanı — Müşteri İsteği`

### Node 1: Webhook
- HTTP Method: POST
- Path: `olta-danismani`
- Response Mode: "Using Respond to Webhook Node"
- Bu URL'i kopyala → Typeform/Tally'e yapıştır

### Node 2: Code — Form Alanlarını Dönüştür
Typeform/Tally'den gelen form verilerini agent JSON formatına çevirir.
Script: bkz. `CODE_NODES.md` → "Form to Agent JSON"

### Node 3: Code — Zorunlu Alan Kontrolü
Script: bkz. `CODE_NODES.md` → "Validate Fields"

### Node 4: HTTP Request — BLOG_INDEX.md Oku
- Method: GET
- URL: `https://raw.githubusercontent.com/[KULLANICI]/vlaude/main/agents/olta-danismani/knowledge/BLOG_INDEX.md`
- Response Format: Text

### Node 5: Code — Eşleşen Blog Dosyalarını Bul
Script: bkz. `CODE_NODES.md` → "Find Blog Files"

### Node 6: HTTP Request — Blog Dosyasını Oku
- Method: GET
- URL: `={{ 'https://raw.githubusercontent.com/[KULLANICI]/vlaude/main/agents/olta-danismani/knowledge/' + $json.matched_file }}`
- Response Format: Text

### Node 7: Code — Kriterleri Çıkar
Script: bkz. `CODE_NODES.md` → "Extract Criteria"

### Node 8: HTTP Request — Ürün Cache'ini Oku
- Method: GET
- URL: `https://gist.githubusercontent.com/[KULLANICI]/[GIST_ID]/raw/products.json`
- Response Format: JSON
- On Error: Continue (cache yoksa boş döner)

### Node 9: Code — Ürünleri Filtrele
Script: bkz. `CODE_NODES.md` → "Filter Products"

### Node 10: Anthropic — Claude'u Çağır
- Model: `claude-sonnet-4-6`
- System Prompt: bkz. `SYSTEM_PROMPT.md` (tam metni kopyala)
- User Message:
```
Müşteri profili:
={{ JSON.stringify($('Form to Agent JSON').first().json, null, 2) }}

Blog kriterleri:
={{ $('Extract Criteria').first().json.criteria_text }}

Stok (ön filtrelenmiş):
={{ JSON.stringify($('Filter Products').first().json.filtered_products, null, 2) }}
```
- Max Tokens: 2000

### Node 11: Code — Yanıtı Parse Et ve Sepet Linki Oluştur
Script: bkz. `CODE_NODES.md` → "Parse Response"

### Node 12: Respond to Webhook
- Response Body: `={{ JSON.stringify($json) }}`
- Response Code: 200
- Response Headers: `Content-Type: application/json`

---

## Typeform / Tally Form Yapısı

Formda şu alanlar olmalı (field name = n8n'de kullanılan key):

| Form Sorusu | Field Name / Variable |
|-------------|----------------------|
| Av disiplini (spin, sazancılık...) | `av_disiplini` |
| Hedef balık | `hedef_balik` |
| Nerede avlanıyorsunuz? | `ortam` |
| Marka tercihi | `marka_tercihi` |
| Maksimum bütçe (TL) | `butce_max` |
| Deneyim seviyesi | `deneyim` |

Typeform'da "Hidden fields" veya "Variable names" ayarından bu isimleri kullan.

---

## Hata Durumu Yönetimi

Workflow 2'de hata durumlarını ele almak için Node 3 (Validate) ve Node 5 (Find Blog) sonralarına **Error Trigger** veya **IF node** ekle:

```
IF hata var:
    → Respond to Webhook (hata JSON'u)
    {
      "hata": "...",
      "mesaj": "Müşteriye gösterilecek mesaj"
    }
```

---

## Deployment Checklist

- [ ] vlaude repo GitHub'da yayında
- [ ] Gist oluşturuldu, ID not alındı
- [ ] GitHub token oluşturuldu
- [ ] n8n Anthropic credential eklendi
- [ ] n8n GitHub credential eklendi
- [ ] Workflow 1 (Cache Sync) oluşturuldu ve test edildi
- [ ] Workflow 1 aktif (Schedule çalışıyor)
- [ ] Workflow 2 (Müşteri İsteği) oluşturuldu
- [ ] Typeform/Tally webhook URL'e bağlandı
- [ ] Uçtan uca test: form doldur → sepet linki al → T-Soft'ta kontrol et
- [ ] `FEED_CONFIG.md`'deki SITE_DOMAIN ve CART_BASE_URL dolu
