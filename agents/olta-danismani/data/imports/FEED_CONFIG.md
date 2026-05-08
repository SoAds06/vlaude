# Google Shopping XML Feed Konfigürasyonu

Agent buradan okur, asla değiştirmez.

## Zorunlu Alan

```
FEED_URL_STANDARD=https://sihirliolta.com/xml/googleshopping.com.php
FEED_URL_VARIANT=https://sihirliolta.com/xml/googlevariant.com.php
SITE_DOMAIN=https://www.sihirliolta.com
CART_BASE_URL=https://www.sihirliolta.com/srv/service/cart/create-cart-from-url/
```

## Feed URL'ini Nasıl Bulursun?

T-Soft panelinde:
1. Pazaryeri & Entegrasyonlar → Google Shopping
2. "Feed URL" veya "XML Dosyası" linkini kopyala
3. Yukarıdaki `FEED_URL` alanına yapıştır

Alternatif yaygın T-Soft feed URL formatları:
```
https://[domain]/googlebase.xml
https://[domain]/feed/google.xml
https://[domain]/urunler.xml
```

Tarayıcıda açıp XML görebiliyorsan URL doğrudur.

## Sepet Link Formatı (T-Soft)

```
https://[domain]/sepet?ekle=[urun_id1]:1,[urun_id2]:1,[urun_id3]:1
```

İlk başarılı test sonrası hangi formatın çalıştığını agents/olta-danismani/MEMORY.md'ye not düş.

## Feed Güncelleme Sıklığı

T-Soft genellikle feed'i her gün otomatik günceller.
Stok değişiklikleri anlık yansımayabilir — genellikle 1-4 saat gecikme olabilir.
Bu kabul edilebilir: agent her tetiklenişte feed'i taze çeker.

## Önemli Not: Ürün ID'si

Feed'deki `<g:id>` değeri T-Soft'taki ürün ID'siyle aynı olmalıdır.
Sepet linkinde bu ID kullanılır: `/sepet?ekle=g:id:1`

Test yöntemi:
1. Feed'den herhangi bir `<g:id>` değeri al (örn. 12345)
2. Tarayıcıda aç: `https://[domain]/sepet?ekle=12345:1`
3. Ürün sepete ekleniyorsa ID eşleşmesi doğrudur
