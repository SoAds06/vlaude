# Price Checker Memory

## GOREV TALIMATLARI (her calisma icin gecerli, degistirme)
- **Kategori kisitlamasi:** Sadece olta kamisi ve olta makinesi kategorilerindeki urunleri sec. Diger kategorileri atla.
- **Kategori URL'leri:** https://www.sihirliolta.com/?s=MARKA&product_cat=olta-kamisi ve https://www.sihirliolta.com/?s=MARKA&product_cat=olta-makinesi
- **Canli fiyat:** Her secilen urunun kendi urun sayfasina gir (URL'yi ac) ve fiyati o sayfadan oku. Arama listesindeki fiyati kullanma.
- **Rakip fiyat:** Rakip sitede urun bulunduktan sonra o urunun kendi sayfasina gir, fiyati sayfadan oku.
- **Indirimli fiyat:** Indirimli fiyat varsa onu al.
- **Cikti formati:** Sonuclari markdown tablo VE Excel'e yapistirilabiir CSV metin olarak yaz (virgullu, kod blogu olmadan).
- **GitHub push yok:** Dosya yazma veya GitHub'a push yapma. Sonuclari sadece bu session'da goster.

<!-- Bu dosya ajan tarafından doldurulur. Başlangıçta boş bırakılır. -->
<!-- Haftalık review sırasında onaylanan kalıplar buraya eklenir. -->

## Güvenilir Rakip Siteler
<!-- Tutarlı şekilde veri çekilebilenler -->
- Henüz doğrulanmış güvenilir site yok (ilk döngü).

## Sorunlu Rakip Siteler
<!-- Erişim sorunu, yapı değişikliği veya düşük eşleşme oranı olanlar -->
- **balkanlarav.com.tr:** Daiwa ürünü taşımıyor (2026-04-25 itibarıyla tüm Daiwa aramalarında 0 sonuç).
- **oltamuhendisi.com:** Shimano Kairiki stoğu var ama fiyat arama sayfasında gösterilmiyor — doğrudan ürün URL'si gerekiyor.
- Her iki site de sihirliolta.com ürün yelpazesiyle düşük örtüşme gösteriyor (%17 eşleşme, 2026-04-25).

## Ürün Eşleştirme Kalıpları
<!-- Hangi markada hangi eşleştirme yöntemi en iyi çalışıyor -->
- Shimano Kairiki serisi: Renk varyantı farklı olsa bile seri adı + uzunluk (örn. "Kairiki 300mt") ile kısmi eşleşme mümkün.
- Spesifik model numaraları (Nasci FD 1000, Bassterra A, Harrier Jigging CA) rakip sitelerde arama ile bulunamıyor.

## Dikkat Çekici Fiyat Farkları
<!-- Sürekli büyük fark gözlemlenen marka/ürün/rakip kombinasyonları -->
- **Shimano Kairiki 300mt:** balkanlarav.com.tr 2.173 TL vs sihirliolta.com 3.306 TL — sihirliolta %34 daha pahalı (2026-04-25). Takip edilmeli.

## Genel Notlar
- rivals.csv gözden geçirilmeli: mevcut rakipler sihirliolta.com ile örtüşen ürün yelpazesi taşımıyor.
- İlk döngü: 2026-04-25 16:00 TRT.
