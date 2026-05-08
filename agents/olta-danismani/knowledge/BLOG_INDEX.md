# Blog Yazısı Dizini

Agent her müşteri isteğinde bu dizini okur ve müşteri profiline uyan yazıları seçer.

**Kural:** Bu dizinde olmayan bir yazı agent tarafından okunmaz.
**Kural:** Yeni blog yazısı ekledikten sonra buraya mutlaka bir satır ekle.
**Kural:** Silinen veya taşınan yazıyı buradan da kaldır.

---

## Dizin

| Dosya | Başlık | Disiplin | Hedef Balık | Ortam | Deneyim |
|-------|--------|----------|-------------|-------|---------|
| blogs/spin-levrek-avi.md | Spin At-Çek Tekniği ile Levrek Avı | spin | levrek | kıyı, açık deniz, nehir ağzı | tümü |

---

## Nasıl Satır Eklenir?

```
| blogs/spin-kamis-secimi.md | Spin Avında Kamış Seçimi Rehberi | spin | levrek, alabalık | kıyı, göl | tümü |
| blogs/sazancilik-baslangic.md | Sazancılığa Başlangıç Kılavuzu | sazancılık | sazan | göl, baraj | beginner |
| blogs/levrek-avciligi.md | Kıyıdan Levrek Avcılığı | spin, predator | levrek | kıyı | intermediate, advanced |
```

---

## Eşleştirme Mantığı

Agent şu öncelik sırasıyla yazı seçer:

1. `disiplin` + `hedef_balik` + `ortam` üçü de eşleşen yazı → **birincil kaynak**
2. `disiplin` + `ortam` eşleşen yazı → **destekleyici kaynak**
3. `disiplin` tek başına eşleşen yazı → **genel rehber**
4. `deneyim = "genel"` veya `"tümü"` olan yazılar → her profilde okunabilir

Eşleşen yazı yoksa → agent öneri yapamaz, insan flag eder.
