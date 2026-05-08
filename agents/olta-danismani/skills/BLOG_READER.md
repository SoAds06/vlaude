# Skill: Blog Okuyucu (BLOG_READER)

## Hedef Bağlantısı
→ "Doğru ürün eşleşmesi" KPI'sını besler.
Bu skill çalışmadan PRODUCT_MATCH başlamaz. Tüm ürün kararları burada oluşturulan kriterlere dayanır.

## Giriş
CUSTOMER_INTAKE'ten gelen normalize edilmiş müşteri profili.

## Çıkış
Müşteri profiline uygun blog yazılarından çıkarılmış ürün seçim kriterleri sözlüğü.
Bu sözlük PRODUCT_MATCH'e geçer.

---

## Temel Kural

> Agent yalnızca blog yazılarında açıkça belirtilen kriterlere göre ürün seçer.
> Blog yazısında geçmeyen bir kriter varsayılmaz, tahmin edilmez, dışarıdan eklenmez.
> Eşleşen yazı yoksa öneri yapılmaz.

---

## Süreç

### Adım 1 — BLOG_INDEX.md'yi Oku

`knowledge/BLOG_INDEX.md` dosyasını oku. Dizindeki tüm satırları al.

### Adım 2 — Müşteri Profiline Uyan Yazıları Seç

Müşteri profili: `{ disiplin, hedef_balik, ortam, deneyim }`

Şu öncelik sırasıyla yazı seç:

**Ortam eşleştirme notu:** Blog satırındaki ortam hücresi virgülle ayrılmış olabilir (örn. "kıyı, açık deniz, nehir ağzı"). Müşterinin `ortam` değeri bu listeden herhangi biriyle eşleşiyorsa eşleşme var sayılır. Ayrıca `"nehir ağzı"` → `"nehir"` olarak da eşleşir.

**Seviye 1 — Tam eşleşme (birincil kaynak):**
- `disiplin` AND `hedef_balik` AND `ortam` üçü de eşleşiyor → bu yazıyı oku

**Seviye 2 — Kısmi eşleşme (destekleyici kaynak):**
- `disiplin` AND `ortam` eşleşiyor ama `hedef_balik` farklı → destekleyici oku
- `disiplin` AND `hedef_balik` eşleşiyor ama `ortam` farklı → destekleyici oku

**Seviye 3 — Genel rehber:**
- Sadece `disiplin` eşleşiyor → genel rehber olarak oku
- `deneyim = "tümü"` veya `"genel"` olan yazılar → her zaman oku

**Eşleşen yazı yoksa:**
```json
{
  "hata": "blog_yok",
  "mesaj": "Bu disiplin/ortam kombinasyonu için henüz rehber yazı bulunmuyor. Lütfen ilgili blog yazısını ekleyin.",
  "profil": { "disiplin": "...", "ortam": "...", "hedef_balik": "..." }
}
```
→ Döngüyü durdur, PRODUCT_MATCH çalıştırma.

### Adım 3 — Seçilen Yazıları Oku

1. Seviye 1 yazıyı oku ve "Ürün Seçim Kriterleri" bölümünü çıkar.
2. **Kapsam kontrolü:** Seviye 1 kriterleri şu beş kategoriyi kapsıyor mu?
   `olta_kamisi`, `makine`, `misina`, `lider`, `igne_aksesuar`
   - Tüm beşi varsa → Seviye 2-3 okumayı atla, Adım 6'ya geç.
   - Eksik kategori(ler) varsa → yalnızca eksik kategoriler için Seviye 2-3 yazıları oku.
3. Seviye 2-3'ten alınan kriterler yalnızca Seviye 1'in kapsamadığı kategoriler için geçerlidir.

### Adım 4 — "Ürün Seçim Kriterleri" Bölümünü Çıkar

Her yazıdaki `## Ürün Seçim Kriterleri` bölümünü oku. Dışındaki içeriği atla.

Her kategori için kriterleri listele:

```
olta_kamisi_kriterleri:
  - "270-300cm arası"
  - "Fast aksiyon"
  - "7-35g güç aralığı"
  - "Ağır jigging kamışı uygun değil"

makine_kriterleri:
  - "2500-4000 beden"
  - "Baitcasting önerilmez"

misina_kriterleri:
  - "Örme ip (braid) tercih edilmeli"
  - "0.14-0.20mm"

igne_kriterleri:
  - "Jig başlık + soft bait"
  - "5-15g"
```

### Adım 5 — Çakışan Kriterleri Çöz

Birden fazla yazı okunduysa aynı kriter farklı değerler içerebilir.
Çakışma varsa: **Seviye 1 yazının kriteri geçerlidir.** Seviye 2-3 yalnızca eksik kriterleri tamamlar.

Örnek:
- Seviye 1 yazı: "270-300cm kamış"
- Seviye 2 yazı: "240-270cm kamış"
→ "270-300cm" geçerli.

### Adım 6 — Çıktı: Kriterler Sözlüğü

```json
{
  "kaynak_yazilar": [
    {
      "seviye": 1,
      "dosya": "blogs/spin-kamis-secimi.md",
      "baslik": "Spin Avında Kamış Seçimi"
    },
    {
      "seviye": 3,
      "dosya": "blogs/genel-ekipman-rehberi.md",
      "baslik": "Balıkçılık Ekipmanı Seçim Rehberi"
    }
  ],
  "kriterler": {
    "olta_kamisi": [
      "270-300cm arası",
      "Fast veya Extra Fast aksiyon",
      "7-35g güç aralığı",
      "Ağır jigging kamışı (80g+) uygun değil"
    ],
    "makine": [
      "2500-4000 beden spin makinesi",
      "Baitcasting makine beginner için önerilmez"
    ],
    "misina": [
      "Örme ip (braid) tercih edilmeli",
      "0.14-0.20mm kalınlık",
      "Fluoro karbon şok misina bağlanmalı"
    ],
    "igne_aksesuar": [
      "Jig başlık + soft bait kombinasyonu",
      "5-15g jig başlık ağırlığı"
    ],
    "genel": [
      "Ucuz giriş segment makinelerden kaçın"
    ]
  }
}
```

Bu sözlük direkt olarak PRODUCT_MATCH'e geçer.

---

## Kalite Çıtası
- Kriterler yalnızca blog yazılarından gelir — dışarıdan eklenmez
- Her kriter hangi yazıdan geldiği takip edilebilir olmalı
- Eşleşen yazı yoksa PRODUCT_MATCH başlamaz
- Seviye 1 yazı her zaman zorunlu; Seviye 2-3 tamamlayıcıdır
