# Skill: Fiyat Karşılaştırması (PRICE_CHECK)

## Purpose
sihirliolta.com'dan marka bazında rastgele 3 ürün seçip rakip sitelerdeki aynı ürünlerin fiyatlarıyla karşılaştırır ve bir markdown tablosu üretir.

## Serves Goals
- Fiyat rekabeti takibi
- Rakip fiyat tespiti

## Inputs
- `data/marka.csv` — `brandname` sütununda marka adları
- `data/rivals.csv` — `links` sütununda rakip site URL'leri
- `https://www.sihirliolta.com` — canlı ürün ve fiyat verisi (web scraping)
- `MEMORY.md` — önceki döngülerde tespit edilen sorunlu siteler / kalıplar

## Process

### Aşama 1 — Veri Kaynaklarını Hazırla
1. `data/marka.csv` dosyasını oku; `brandname` sütunundaki tüm marka adlarını bir listeye al.
2. `data/rivals.csv` dosyasını oku; `links` sütunundaki tüm rakip URL'leri bir listeye al.
3. `MEMORY.md`'deki "Sorunlu Rakip Siteler" bölümünü oku — bu siteler için zaman kaybetme, "Erişim Sorunu" yaz.

### Aşama 2 — sihirliolta.com'dan Ürün Seç
**Sadece şu iki kategori:** olta kamışı ve olta makinesi. Bu kategoriler dışındaki ürünleri seçme.

Her marka için:
4. Önce kategori sayfalarına git:
   - Olta kamışı: `https://www.sihirliolta.com/olta-kamisi/?s=[marka_adı]` veya `https://www.sihirliolta.com/?s=[marka_adı]&product_cat=olta-kamisi`
   - Olta makinesi: `https://www.sihirliolta.com/olta-makinesi/?s=[marka_adı]` veya `https://www.sihirliolta.com/?s=[marka_adı]&product_cat=olta-makinesi`
   - Çalışmıyorsa `https://www.sihirliolta.com/?s=[marka_adı]` ile genel arama yap, sadece kamış ve makine olanları filtrele.
5. Listeden rastgele 3 ürün seç (stokta olan, olta kamışı veya olta makinesi kategorisinde olan).
6. Her seçilen ürünün **kendi ürün sayfasına** gir (URL'ye tıkla).
7. Ürün sayfasından şunları kaydet:
   - Ürün adı (tam, sayfadaki başlık)
   - SKU / model numarası (varsa, ürün sayfasından)
   - Canlı fiyat: sayfadaki güncel satış fiyatı (TL) — indirimli fiyat varsa onu al
   - Ürün sayfası URL'si

Eğer bir markada 3'ten az olta kamışı/makinesi ürünü bulunursa: mevcut kadarı al, eksik satırları "Ürün Bulunamadı" olarak işaretle.

### Aşama 3 — Rakip Sitelerde Ürün Ara
Her rakip URL × her seçili ürün kombinasyonu için:
8. Rakip sitenin arama sayfasına git: `[rakip_url]/?s=[ürün_adı]` veya `[rakip_url]/search?q=[ürün_adı]`.
9. Arama sonuçlarında şu sırayla eşleştir:
   a. SKU / model numarası eşleşmesi (en güvenilir)
   b. Tam ürün adı eşleşmesi
   c. Kısmi ürün adı eşleşmesi (marka + model)
10. Eşleşen ürünü bulduktan sonra **ürünün kendi sayfasına gir** ve sayfadan canlı fiyatı oku (TL). İndirimli fiyat varsa onu al.
11. Eşleşme bulunamazsa: "Bulunamadı" yaz, fiyat boş bırak.
12. Fiyat okunamazsa: "Okunamadı" yaz, tahmin yapma.

### Aşama 4 — Tabloyu Oluştur
13. Aşağıdaki formatta bir markdown tablosu oluştur:

```markdown
| Marka | Ürün Adı | sihirliolta.com | [Rakip 1 Adı] | [Rakip 2 Adı] | ... |
|-------|----------|-----------------|---------------|---------------|-----|
| Shimano | Shimano XYZ 3000 | 1.250 TL | 1.180 TL | Bulunamadı | ... |
```

- Her markanın 3 ürünü art arda sıralanır
- Markalar alfabetik sırayla listelenir
- Fiyatlar `X.XXX,XX TL` formatında yazılır
- Tablonun altına veri çekim tarihi ve saati eklenir

### Aşama 5 — Raporu Kaydet
14. Raporu `outputs/YYYY-MM-DD_price-report.md` dosyasına yaz.
15. Dosya başlığına şunları ekle: tarih, çalışma saati (TRT), işlenen marka sayısı, toplam ürün sayısı, eşleşme oranı.

## Outputs
- `outputs/YYYY-MM-DD_price-report.md` — tam fiyat karşılaştırma tablosu

## Quality Bar
- En az %80 ürün için rakip fiyat verisi bulunmalı
- Hiçbir fiyat tahmin veya uydurma ile doldurulmamalı
- Tablo en az 2 rakip kolonu içermeli (rivals.csv'de en az 2 URL olduğu varsayılır)
- Rapor dosyası varsa üzerine yazılmaz — yeni dosya oluşturulur

## Tools
- WebFetch — sihirliolta.com ve rakip sitelerdeki sayfaları çekmek için
- WebSearch — ürün bulunamazsa rakip sitede arama yapmak için (ek yöntem)

## Integration
Tamamlandığında EMAIL_REPORT skill'ini tetikler.
Eşleşme oranı %50'nin altındaysa bunu rapora ve journal'a özellikle not düşer.
