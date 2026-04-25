# Price Checker Rules

## CAN
- `data/marka.csv` ve `data/rivals.csv` dosyalarını okuyabilir
- `www.sihirliolta.com` adresine bağlanıp ürün ve fiyat verisi çekebilir
- `rivals.csv` içindeki her rakip URL'ye bağlanıp fiyat arayabilir
- Ürün eşleştirmesi için ürün adı, model numarası veya barkod kullanabilir
- `outputs/` klasörüne rapor dosyaları yazabilir
- `journal/entries/` klasörüne günlük kayıt yazabilir
- `MEMORY.md` dosyasını güncelleyebilir
- sergen@sihirliolta.com adresine e-posta gönderebilir

## CANNOT
- `knowledge/` klasörüne asla yazamaz
- Başka ajanların `MEMORY.md` dosyasını değiştiremez
- Rakip sitelerde oturum açamaz, form dolduramaz veya sepete ürün ekleyemez
- Fiyatları tahmin edemez veya uydurамaz — her fiyat gerçek sayfa verisinden gelmelidir
- Aynı günün raporunu üzerine yazamaz — sadece yeni dosya oluşturur
- Kullanıcı adına herhangi bir satın alma işlemi yapamaz
- `marka.csv` veya `rivals.csv` dosyalarını değiştiremez

## Handoff Conditions (İnsan devreye girer)
- `marka.csv` veya `rivals.csv` bulunamıyor ya da boş
- sihirliolta.com sitesi 2+ gün erişilemez durumdaysa
- E-posta gönderimi 2+ gün art arda başarısız oluyorsa
- Tüm rakip siteler için eşleşme oranı %20'nin altına düşüyorsa
- Bir rakip site yapısı tamamen değişmiş ve ürün bulunamıyorsa

## Eşleştirme Mantığı
- Ürün eşleştirmesi şu sırayla denenecek: barkod/SKU → model numarası → tam ürün adı → kısmi ürün adı
- Eşleşme bulunamazsa satır "Bulunamadı" olarak işaretlenir (fiyat boş bırakılır)
- Birden fazla eşleşme varsa en yakın ürün adı eşleşmesi seçilir

## Veri Kalitesi
- Fiyatlar TL cinsinden alınır; para birimi farklıysa not düşülür
- Fiyat bilgisi sayfadan okunamazsa "Okunamadı" yazılır, tahmin yapılmaz
- Çekilen fiyatın tarihi/saati rapora eklenir (tutarsızlık için referans)
