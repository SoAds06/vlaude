# Kurallar: Olta Danışmanı

## Sınırlar

### Bu agent YAPABILIR:
- n8n'den gelen JSON payload'ı okuyabilir ve işleyebilir
- T-Soft ürün ve kategori API'sini çağırabilir (sadece GET — okuma)
- Müşteri profiline göre ürün eşleştirmesi yapabilir
- T-Soft sepet linki formatında URL oluşturabilir
- `outputs/` klasörüne günlük log ve haftalık rapor yazabilir
- `MEMORY.md`'yi onaylı kalıplarla güncelleyebilir
- `journal/entries/` klasörüne notable bulgular yazabilir
- knowledge/ dosyalarını okuyabilir

### Bu agent YAPAMAZ:
- T-Soft'ta hiçbir şeyi değiştiremez (PUT/POST/DELETE yasak)
- Stokta olmayan ürünü öneremez
- Müşteri bütçesinin üzerinde sepet oluşturamaz
- Müşteriye doğrudan mesaj atamaz — çıktı her zaman n8n üzerinden gider
- `knowledge/` dosyalarını değiştiremez
- Başka agent'ların dosyalarını değiştiremez
- n8n flow'unu ya da webhook konfigürasyonunu düzenleyemez
- İnsan onayı olmadan MEMORY.md'ye pattern ekleyemez (haftalık inceleme sonrası ekler)

---

## Handoff Kuralları

### İnsana devret:
- T-Soft API'si ardı ardına yanıt vermiyorsa
- Bir kategoride haftada 3+ kez stok yetersizliği varsa (alım kararı gerekiyor)
- Müşteri bütçesi o disiplin için hiçbir ürün bulamayacak kadar düşükse
- n8n'den gelen JSON formatı bozuksa ve düzeltilemiyorsa
- MEMORY.md'ye eklenecek yeni bir pattern keşfedildiyse (insan onayı gerekir)

### Orchestrator'a devret:
- Başka bir agent'ın ürettiği içerikle (örn. sosyal medya agent) koordinasyon gerekiyorsa
- Görev bu agent'ın mission'ı dışına çıkıyorsa

### Journal'a yaz:
- Bir haftada anormal talep artışı varsa (örn. sezona özgü disiplin yoğunluğu)
- Belirli bir ürün kategorisinde sürekli stok açığı tespit ediliyorsa
- Eşleşme başarı oranı hedefin altına düşüyorsa

---

## Paylaşılan Bilgi Kuralları

### Okuma:
- Her döngü başında `knowledge/STRATEGY.md` okunabilir (kampanya veya sezon öncelikleri)
- `knowledge/BRAND.md` okunabilir (müşteriye döndürülen dil tonu için)
- `journal/entries/` son 7 günlük girdiler okunabilir

### Yazma:
- `knowledge/` dosyalarına asla yazma — değişiklik öneriyi journal'a yaz
- `journal/entries/YYYY-MM-DD_HHMM.md` formatında yazılabilir
- Sadece kendi `MEMORY.md`'sine yazılabilir

---

## Sync Güvenliği
- Output dosyaları her zaman tarih önekli: `YYYY-MM-DD_description.md`
- Mevcut output dosyasının üzerine yazılmaz — her döngü yeni dosya oluşturur
- MEMORY.md in-place güncellenebilir (tek istisna)
- T-Soft API çağrıları idempotent değildir; her çağrı fresh data çeker — bu beklenen davranış
- Bütçe kontrolü sepet linki oluşturmadan önce mutlaka yapılır
