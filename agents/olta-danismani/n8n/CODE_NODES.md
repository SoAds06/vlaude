# n8n Code Node Scriptleri

Her script kendi node'una kopyala-yapıştır. n8n'de "Code" node, JavaScript kullanır.

---

## 1. XML Parser (İki Feed — Merge)
**Node:** Workflow 1 → Node 4
Node 2'nin adı "Feed Standard", Node 3'ün adı "Feed Variant" olmalı.

```javascript
// Test limiti — üretime geçince 9999 yap
const TEST_LIMIT = 20;

// XML'i ilk N item'dan sonra kes (parse süresini kısaltır)
function truncateXML(xml, n) {
  let pos = 0;
  for (let i = 0; i < n; i++) {
    const idx = xml.indexOf('</item>', pos);
    if (idx === -1) break;
    pos = idx + 7;
  }
  return pos > 0 ? xml.substring(0, pos) : xml;
}

// Feed 1: varyantsız ürünler (kamış, makine vb.)
const xml1 = truncateXML($('Feed Standard').first().json.data, TEST_LIMIT);
// Feed 2: varyantlı ürünler (misina, lider vb.)
const xml2 = truncateXML($('Feed Variant').first().json.data, TEST_LIMIT);

function parseXML(xml, forceNoVariant) {
  const items = [];
  const itemRegex = /<item>([\s\S]*?)<\/item>/g;
  let match;

  while ((match = itemRegex.exec(xml)) !== null) {
    const item = match[1];

    const getTag = (tag) => {
      const m = item.match(new RegExp(`<${tag}[^>]*><!\\[CDATA\\[([\\s\\S]*?)\\]\\]><\/${tag}>|<${tag}[^>]*>([^<]*)<\/${tag}>`));
      return m ? (m[1] || m[2] || '').trim() : '';
    };

    const getGTag = (tag) => {
      const m = item.match(new RegExp(`<g:${tag}[^>]*><!\\[CDATA\\[([\\s\\S]*?)\\]\\]><\/g:${tag}>|<g:${tag}[^>]*>([^<]*)<\/g:${tag}>`));
      return m ? (m[1] || m[2] || '').trim() : '';
    };

    const availability = getGTag('availability');
    if (availability !== 'in stock' && availability !== 'in_stock') continue;

    const priceStr = getGTag('price');
    const price = parseFloat(priceStr.replace(/[^0-9.]/g, '')) || 0;
    if (price === 0) continue;

    const rawId = getGTag('id');
    if (!rawId) continue;

    let product_id, subproduct_id;
    if (forceNoVariant) {
      // Feed Standard (kamış, makine): ID tiresiz, subproduct_id her zaman 0
      product_id = rawId;
      subproduct_id = '0';
    } else {
      // Feed Variant (misina, lider): ID formatı "productId-subproductId"
      if (rawId.includes('-')) {
        const dashIdx = rawId.indexOf('-');
        product_id = rawId.substring(0, dashIdx).trim();
        subproduct_id = rawId.substring(dashIdx + 1).trim();
      } else {
        product_id = rawId;
        subproduct_id = '0';
      }
    }

    items.push({
      product_id,
      subproduct_id,
      title: getGTag('title') || getTag('title'),
      color: getGTag('color'),
      price,
      availability: 'in stock',
      brand: getGTag('brand'),
      product_type: getGTag('product_type'),
      link: getGTag('link') || getTag('link'),
      description: getTag('description').substring(0, 300)
    });
  }
  return items;
}

const products1 = parseXML(xml1, true);   // varyantsız feed
const products2 = parseXML(xml2, false);  // varyantlı feed

// Birleştir — product_id+subproduct_id çiftiyle deduplicate
const seen = new Set();
const merged = [];
for (const p of [...products1, ...products2]) {
  const key = `${p.product_id}_${p.subproduct_id}`;
  if (!seen.has(key)) {
    seen.add(key);
    merged.push(p);
  }
}

return [{
  json: {
    last_updated: new Date().toISOString(),
    total_products: merged.length,
    products: merged
  }
}];
```

---

## 2. Form to Agent JSON
**Node:** Workflow 2 → Node 2

Typeform veya Tally'den gelen veriyi agent formatına çevirir.
Field isimlerini kendi formundaki isimlere göre güncelle.

```javascript
const body = $input.first().json.body || $input.first().json;

// Typeform formatı: body.form_response.answers
// Tally formatı: body.data.fields
// Düz webhook: body doğrudan

// Aşağıdaki mapping'i formun field isimlerine göre düzenle:
const agentData = {
  av_disiplini: (body.av_disiplini || body['Av Disiplini'] || '').toLowerCase().trim(),
  hedef_balik: (body.hedef_balik || body['Hedef Balık'] || '').toLowerCase().trim(),
  ortam: (body.ortam || body['Ortam'] || '').toLowerCase().trim(),
  marka_tercihi: (body.marka_tercihi || body['Marka Tercihi'] || 'fark etmez').toLowerCase().trim(),
  butce_min: parseInt(body.butce_min || body['Minimum Bütçe'] || '0', 10),
  butce_max: parseInt(body.butce_max || body['Maksimum Bütçe'] || '0', 10),
  deneyim: (body.deneyim || body['Deneyim'] || 'intermediate').toLowerCase().trim()
};

// Deneyim normalize
if (!['beginner', 'intermediate', 'advanced'].includes(agentData.deneyim)) {
  agentData.deneyim = 'intermediate';
}

// Marka normalize
if (!agentData.marka_tercihi || agentData.marka_tercihi === 'fark etmez' || agentData.marka_tercihi === '') {
  agentData.marka_tercihi = 'fark etmez';
}

return [{ json: agentData }];
```

---

## 3. Validate Fields
**Node:** Workflow 2 → Node 3

```javascript
const data = $input.first().json;
const required = ['av_disiplini', 'ortam', 'butce_max'];
const missing = required.filter(f => !data[f] || data[f] === '');

if (missing.length > 0) {
  // Hata durumunda workflow'u durdur
  throw new NodeOperationError(
    $node,
    JSON.stringify({
      hata: 'eksik_alan',
      mesaj: `Lütfen şu bilgileri doldurun: ${missing.join(', ')}`,
      eksik_alanlar: missing
    })
  );
}

if (data.butce_max <= 0) {
  throw new NodeOperationError(
    $node,
    JSON.stringify({
      hata: 'gecersiz_butce',
      mesaj: 'Geçerli bir bütçe giriniz.'
    })
  );
}

return $input.all();
```

---

## 4. Find Blog Files
**Node:** Workflow 2 → Node 5

```javascript
const indexContent = $input.first().json.data;
const profile = $('Form to Agent JSON').first().json;

// Tablo satırlarını parse et
const lines = indexContent
  .split('\n')
  .filter(l => l.startsWith('|') && !l.includes('---') && !l.includes('Dosya') && !l.includes('henüz'));

const entries = lines.map(line => {
  const cols = line.split('|').map(c => c.trim()).filter(Boolean);
  if (cols.length < 6) return null;
  return {
    file: cols[0].replace(/`/g, '').trim(),
    baslik: cols[1].trim(),
    disiplin: cols[2].trim(),
    hedef_balik: cols[3].trim(),
    ortam: cols[4].trim(),
    deneyim: cols[5].trim()
  };
}).filter(Boolean);

const discipline = profile.av_disiplini || '';
const customerOrtam = profile.ortam || '';
const customerHedef = profile.hedef_balik || '';

// Ortam eşleştirme yardımcısı
const ortamMatches = (blogOrtam, customerOrtam) => {
  const ortamlar = blogOrtam.split(',').map(o => o.trim().toLowerCase());
  const customer = customerOrtam.toLowerCase();
  return ortamlar.some(o =>
    o === customer ||
    o === 'genel' ||
    (o === 'nehir ağzı' && customer === 'nehir') ||
    (o === 'nehir' && customer === 'nehir ağzı')
  );
};

// Seviye 1: disiplin + hedef balık + ortam
let matched = entries.filter(e =>
  e.disiplin.toLowerCase().includes(discipline) &&
  ortamMatches(e.ortam, customerOrtam) &&
  (customerHedef === '' || e.hedef_balik.toLowerCase().includes(customerHedef))
);

// Seviye 3: sadece disiplin (Seviye 1 bulunamazsa)
if (matched.length === 0) {
  matched = entries.filter(e =>
    e.disiplin.toLowerCase().includes(discipline) ||
    e.deneyim.toLowerCase() === 'tümü' ||
    e.deneyim.toLowerCase() === 'genel'
  );
}

if (matched.length === 0) {
  throw new NodeOperationError(
    $node,
    JSON.stringify({
      hata: 'blog_yok',
      mesaj: `${discipline} / ${customerOrtam} için henüz rehber yazı bulunmuyor. Lütfen daha sonra tekrar deneyin.`
    })
  );
}

// En iyi eşleşen dosyayı al (Seviye 1 öncelikli)
const bestMatch = matched[0].file;

return [{ json: { matched_file: bestMatch, profile } }];
```

---

## 5. Extract Criteria
**Node:** Workflow 2 → Node 7

```javascript
const blogContent = $input.first().json.data;

// "## Ürün Seçim Kriterleri" bölümünü çıkar
// Şablon: bu bölüm frontmatter'dan hemen sonra, "---" ayırıcısından önce
const startMarker = '## Ürün Seçim Kriterleri';
const endMarker = '\n---\n';

const start = blogContent.indexOf(startMarker);
if (start === -1) {
  // Eski format: bölüm dosyanın sonundaysa
  return [{ json: { criteria_text: blogContent } }];
}

const end = blogContent.indexOf(endMarker, start);
const criteriaSection = end > start
  ? blogContent.substring(start, end)
  : blogContent.substring(start);

return [{ json: { criteria_text: criteriaSection.trim() } }];
```

---

## 6. Filter Products
**Node:** Workflow 2 → Node 9

```javascript
let cache;
try {
  cache = $input.first().json;
  if (typeof cache === 'string') cache = JSON.parse(cache);
} catch(e) {
  // Cache yoksa veya bozuksa boş dön
  return [{ json: { filtered_products: {}, warning: 'Cache okunamadı' } }];
}

const profile = $('Find Blog Files').first().json.profile;
const products = cache.products || [];
const budgetMax = profile.butce_max || 999999;
const brandPref = (profile.marka_tercihi || 'fark etmez').toLowerCase();

// Stokta olan ve bütçe içinde kalan ürünler
const inBudget = products.filter(p =>
  p.availability === 'in stock' && p.price <= budgetMax
);

// Kategorize et (anahtar kelime eşleştirme)
const match = (p, keywords) =>
  keywords.some(kw => (p.title + ' ' + p.product_type).toLowerCase().includes(kw));

const liderKeywords = ['fluorokar', 'lider', 'şok misina', 'leader', 'fluoro'];
const misinaKeywords = ['misina', 'örme ip', 'braid', 'monofilament'];
const excludeLider = (p) => !liderKeywords.some(kw => (p.title + ' ' + p.product_type).toLowerCase().includes(kw));

const categories = {
  olta_kamisi: inBudget.filter(p => match(p, ['kamış', 'kamis', 'rod', 'feeder', 'fly rod', 'surf kamış'])),
  makine: inBudget.filter(p => match(p, ['makine', 'reel', 'baitrunner', 'baitcast', 'olta makinesi'])),
  misina: inBudget.filter(p => match(p, misinaKeywords) && excludeLider(p)),
  lider: inBudget.filter(p => match(p, liderKeywords)),
  igne_aksesuar: inBudget.filter(p => match(p, ['iğne', 'igne', 'jig', 'kaşık', 'kanca', 'hook', 'lure', 'soft bait']))
};

// Marka filtresi uygula (fark etmez değilse)
if (brandPref !== 'fark etmez') {
  Object.keys(categories).forEach(cat => {
    const brandFiltered = categories[cat].filter(p =>
      (p.brand || '').toLowerCase().includes(brandPref)
    );
    // Marka filtresi sonuç verirse uygula, yoksa orijinali koru
    if (brandFiltered.length > 0) {
      categories[cat] = brandFiltered;
    }
  });
}

// Her kategoriden max 15 ürün al (token tasarrufu)
const trimmed = {};
Object.keys(categories).forEach(cat => {
  trimmed[cat] = categories[cat].slice(0, 15).map(p => ({
    product_id: p.product_id,
    subproduct_id: p.subproduct_id,
    title: p.title,
    color: p.color || '',
    price: p.price,
    brand: p.brand || '',
    link: p.link || '',
    description: (p.description || '').substring(0, 150)
  }));
});

return [{ json: { filtered_products: trimmed, profile } }];
```

---

## 7. Parse Response
**Node:** Workflow 2 → Node 11

```javascript
const SITE_DOMAIN = 'www.sihirliolta.com';

const response = $input.first().json;

// Gerçek ürün listesini al — Groq ID hallucination'ını düzeltmek için
const filteredProducts = $('Filter Products').first().json.filtered_products || {};

// product_id → ürün verisi lookup tablosu
const productMap = {};
Object.values(filteredProducts).forEach(list => {
  list.forEach(p => { productMap[String(p.product_id)] = p; });
});

// Kategori → filtered_products anahtarı eşleşmesi
const CAT_KEY = {
  'Olta Kamışı': 'olta_kamisi',
  'Makine': 'makine',
  'Misina': 'misina',
  'Lider': 'lider',
  'Yem': 'igne_aksesuar'
};
const NON_VARIANT_CATEGORIES = ['Olta Kamışı', 'Makine'];

// Groq yanıtını parse et
let rawText = '';
if (response.choices && response.choices[0]) {
  rawText = response.choices[0].message.content || '';
} else if (response.candidates && response.candidates[0]) {
  rawText = response.candidates[0].content.parts[0].text || '';
} else if (response.content && response.content[0]) {
  rawText = response.content[0].text || '';
} else if (response.text) {
  rawText = response.text;
} else {
  rawText = JSON.stringify(response);
}

let result;
try {
  const jsonMatch = rawText.match(/\{[\s\S]*\}/);
  if (!jsonMatch) throw new Error('JSON bulunamadı');
  result = JSON.parse(jsonMatch[0]);
} catch(e) {
  return [{
    json: {
      hata: 'parse_hatasi',
      mesaj: 'Öneri oluşturulurken hata oluştu. Lütfen tekrar deneyin.',
      ham_yanit: rawText.substring(0, 500)
    }
  }];
}

// Groq'un döndürdüğü her product_id'yi gerçek listeyle doğrula
// Hallucinated ID → kategorideki ilk gerçek ürünle değiştir
if (result.oneriler && result.oneriler.length > 0) {
  result.oneriler = result.oneriler.map(o => {
    const pid = String(o.product_id || '');
    if (!productMap[pid]) {
      // Hallucinated ID — gerçek listeden düzelt
      const catKey = CAT_KEY[o.kategori];
      const realList = catKey ? (filteredProducts[catKey] || []) : [];
      if (realList.length > 0) {
        const real = realList[0];
        o.product_id = real.product_id;
        o.subproduct_id = real.subproduct_id || '0';
        o.urun_adi = real.title;
        o.urun_linki = real.link || '';
        o.fiyat = real.price;
      }
    } else {
      // Gerçek ID — link ve subproduct_id'yi de güncelle
      o.urun_linki = productMap[pid].link || o.urun_linki || '';
    }
    // Kamış ve Makine her zaman subproduct_id:0
    if (NON_VARIANT_CATEGORIES.includes(o.kategori)) {
      o.subproduct_id = '0';
    }
    return o;
  });

  // Sepet linkini doğrulanmış ID'lerden oluştur
  const parts = result.oneriler
    .filter(o => o.product_id && o.product_id !== 'kriter_uyumlu_urun_yok')
    .map(o => {
      const subId = NON_VARIANT_CATEGORIES.includes(o.kategori) ? '0' : (o.subproduct_id || '0');
      return `count:1;product_id:${o.product_id};subproduct_id:${subId}`;
    })
    .join('-');
  if (parts) {
    result.sepet_linki = `https://${SITE_DOMAIN}/srv/service/cart/create-cart-from-url/${parts}`;
  }
}

return [{ json: result }];
```

---

## 8. Test Modu — Mock Response (Groq Olmadan)
**Node:** Workflow 2 → Node 11 yerini alır (test sırasında)

Groq'u devre dışı bırakmak için:
1. n8n'de Groq HTTP Request node'unu **disable** et (sağ tık → Disable)
2. Parse Response (Node 11) kodunu bu kodla değiştir
3. Gerçek teste geçince Groq'u tekrar enable et, Node 11'i orijinal kodla geri yükle

```javascript
const SITE_DOMAIN = 'www.sihirliolta.com';
const filteredProducts = $('Filter Products').first().json.filtered_products || {};
const profile = $('Filter Products').first().json.profile || {};

const NON_VARIANT_CATEGORIES = ['Olta Kamışı', 'Makine'];

const CATEGORY_MAP = [
  { key: 'olta_kamisi',   label: 'Olta Kamışı' },
  { key: 'makine',        label: 'Makine' },
  { key: 'misina',        label: 'Misina' },
  { key: 'lider',         label: 'Lider' },
  { key: 'igne_aksesuar', label: 'Yem / Aksesuar' }
];

const oneriler = [];
let toplam = 0;

for (const { key, label } of CATEGORY_MAP) {
  const list = filteredProducts[key] || [];
  if (list.length === 0) continue;

  // Rastgele ürün seç
  const p = list[Math.floor(Math.random() * list.length)];
  const subId = NON_VARIANT_CATEGORIES.includes(label) ? '0' : (p.subproduct_id || '0');

  oneriler.push({
    kategori: label,
    urun_adi: p.title,
    product_id: p.product_id,
    subproduct_id: subId,
    urun_linki: p.link || '',
    fiyat: p.price,
    eslesen_kriterler: ['[TEST MODU]'],
    neden: 'Test modu — Groq devre dışı, rastgele seçildi.'
  });
  toplam += p.price;
}

const parts = oneriler
  .map(o => `count:1;product_id:${o.product_id};subproduct_id:${o.subproduct_id}`)
  .join('-');

return [{
  json: {
    musteri_profili: profile,
    oneriler,
    toplam_fiyat: toplam,
    sepet_linki: parts ? `https://${SITE_DOMAIN}/srv/service/cart/create-cart-from-url/${parts}` : null,
    alternatif_sepet_linki: null,
    dogrulanamayan_kriterler: [],
    not: 'TEST MODU — Groq devre dışı, ürünler rastgele seçildi.'
  }
}];
```

---

## 9. Statik Test Verisi (HTTP Request Olmadan)

**Node:** WF2 → HTTP Request yerine "Statik Ürünler" adında bir Code node koy, çıktısını doğrudan Filter Products'a bağla.

Bu node canlı XML yerine dosyadan ayıklanan 20+20 ürünü döndürür. Test sırasında HTTP Request ve XML Parser node'larını devre dışı bırakıp bunu kullan.

```javascript
const products = [
  {"product_id":"19477","subproduct_id":"0","title":"Seaguar Seabass Flash Green 8X 150 Mt İp Misina","color":"","price":1089,"availability":"in stock","brand":"Seaguar","product_type":"BALIK AVI MALZEMELERİ > İP VE MİSİNALAR > ÖRGÜ İPLER","link":"https://www.sihirliolta.com/seaguar-seabass-flash-green-8x-150-mt-ip-misina","description":"Seaguar Seabass Flash Green 8X 150 Mt İp Misina"},
  {"product_id":"19497","subproduct_id":"0","title":"Bauer Jig Pro 4.7 Cm 10 Gr Jig Yem","color":"","price":191.88,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-47-cm-10-gr-jig-yem","description":"Bauer Jig Pro 10 Gr Jig Yem"},
  {"product_id":"19498","subproduct_id":"0","title":"Bauer Jig Pro 6 Cm 20 Gr Jig Yem","color":"","price":206.64,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-6-cm-20-gr-jig-yem","description":"Bauer Jig Pro 20 Gr Jig Yem"},
  {"product_id":"19499","subproduct_id":"0","title":"Bauer Jig Pro 7 Cm 30 Gr Jig Yem","color":"","price":221.4,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-7-cm-30-gr-jig-yem","description":"Bauer Jig Pro 30 Gr Jig Yem"},
  {"product_id":"19500","subproduct_id":"0","title":"Bauer Jig Pro 7.5 Cm 40 Gr Jig Yem","color":"","price":236.16,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-75-cm-40-gr-jig-yem","description":"Bauer Jig Pro 40 Gr Jig Yem"},
  {"product_id":"20156","subproduct_id":"0","title":"Daiwa Gekkabijin Ajing Beam Kiwami 5 Cm Silikon Yem","color":"","price":292.01,"availability":"in stock","brand":"Daiwa","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > SİLİKON YEM","link":"https://www.sihirliolta.com/daiwa-gekkabijin-ajing-beam-5-cm-silikon-yem","description":"Daiwa Gekkabijin Ajing Beam 5 Cm Silikon Yem"},
  {"product_id":"21167","subproduct_id":"0","title":"BlueBlue Conifer 17 Cm 190 Gr Jig Yem","color":"","price":1725,"availability":"in stock","brand":"BlueBlue","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/blueblue-conifer-17-cm-190-gr-jig-yem","description":"BlueBlue Conifer 17 Cm 190 Gr Jig Yem"},
  {"product_id":"21168","subproduct_id":"0","title":"BlueBlue Conifer 19 Cm 250 Gr Jig Yem","color":"","price":2014.8,"availability":"in stock","brand":"BlueBlue","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/blueblue-conifer-19-cm-250-gr-jig-yem","description":"BlueBlue Conifer 19 Cm 250 Gr Jig Yem"},
  {"product_id":"17905","subproduct_id":"0","title":"Shimano Dialuna 23 305 Cm 10-50 Gr Spin Olta Kamışı","color":"","price":21754,"availability":"in stock","brand":"Shimano","product_type":"BALIK AVI MALZEMELERİ > OLTA KAMIŞLARI > SPIN KAMIŞLARI","link":"https://www.sihirliolta.com/shimano-dialuna-23-305-cm-10-50-gr-spin-olta-kamisi","description":"Shimano Dialuna 23 305 Cm 10-50 Gr Spin Olta Kamışı"},
  {"product_id":"21748","subproduct_id":"0","title":"NS Black Hole Boca Casting & S.L.J 7 Cm 40 Gr Jig Yem","color":"","price":439.12,"availability":"in stock","brand":"NS Black Hole","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/ns-black-hole-boca-casting-slj-7-cm-40-gr-jig-yem","description":"NS Black Hole Boca Casting & S.L.J 7 Cm 40 Gr Jig Yem"},
  {"product_id":"17342","subproduct_id":"0","title":"Ryuji Assist PE Line Beyaz Asist İpi","color":"","price":304.41,"availability":"in stock","brand":"Ryuji","product_type":"BALIK AVI MALZEMELERİ > İP VE MİSİNALAR > ASSIST İPLERİ","link":"https://www.sihirliolta.com/ryuji-assist-pe-line-beyaz-asist-ipi","description":"Ryuji Assist PE Line Beyaz Asist İpi"},
  {"product_id":"20090","subproduct_id":"0","title":"Ima No: 6 Gunkichi 4 Cm 5 Gr Jig Yem","color":"","price":293.2,"availability":"in stock","brand":"Ima","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/ima-no-6-gunkichi-4-cm-5-gr-jig-yem","description":"Ima No: 6 Gunkichi 4 Cm 5 Gr Jig Yem"},
  {"product_id":"2514","subproduct_id":"0","title":"Remixon GR-1807 Olta Zili","color":"","price":12.32,"availability":"in stock","brand":"Remixon","product_type":"BALIK AVI MALZEMELERİ > OLTA APARATLARI > ALARM | ZİL | FOSFOR > Olta Zili","link":"https://www.sihirliolta.com/remixon-gr-1807-olta-zili","description":"Remixon GR-1807 Olta Zili"},
  {"product_id":"2515","subproduct_id":"0","title":"Remixon GR-1809 Olta Zili","color":"","price":8.99,"availability":"in stock","brand":"Remixon","product_type":"BALIK AVI MALZEMELERİ > OLTA APARATLARI > ALARM | ZİL | FOSFOR > Olta Zili","link":"https://www.sihirliolta.com/remixon-gr-1809-olta-zili","description":"Remixon GR-1809 Olta Zili"},
  {"product_id":"20091","subproduct_id":"0","title":"Ima No: 7 Gunkichi 4.5 Cm 7 Gr Jig Yem","color":"","price":244.47,"availability":"in stock","brand":"Ima","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/ima-no-7-gunkichi-45-cm-7-gr-jig-yem","description":"Ima No: 7 Gunkichi 4.5 Cm 7 Gr Jig Yem"},
  {"product_id":"18127","subproduct_id":"0","title":"Daiwa Samurai R 9.2 Cm 40 Gr Jig Yem","color":"","price":451.58,"availability":"in stock","brand":"Daiwa","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/daiwa-samurai-r-92-cm-40-gr-jig-yem","description":"Daiwa Samurai R 9.2 Cm 40 Gr Jig Yem"},
  {"product_id":"17427","subproduct_id":"0","title":"Bauer Aston 420 Cm 100-250 Gr Surf Olta Kamışı","color":"","price":3301.32,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > OLTA KAMIŞLARI > SURF KAMIŞLAR","link":"https://www.sihirliolta.com/bauer-aston-420-cm-100-250-gr-surf-olta-kamisi","description":"Bauer Aston 420 Cm 100-250 Gr Surf Olta Kamışı"},
  {"product_id":"17555","subproduct_id":"0","title":"Shimano Nexave FI 2500 HG Spin Olta Makinesi","color":"","price":4314,"availability":"in stock","brand":"Shimano","product_type":"BALIK AVI MALZEMELERİ > OLTA MAKİNELERİ > SPIN MAKİNELER","link":"https://www.sihirliolta.com/shimano-nexave-fi-2500-hg-spin-olta-makinesi","description":"Shimano Nexave FI 2500 HG Spin Olta Makinesi"},
  {"product_id":"17319","subproduct_id":"0","title":"Ryuji Bubble Bait 8 Cm 12 Gr Maket Yem","color":"","price":300.92,"availability":"in stock","brand":"Ryuji","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > MAKET YEM","link":"https://www.sihirliolta.com/ryuji-bubble-bait-8-cm-12-gr-maket-yem","description":"Ryuji Bubble Bait 8 Cm 12 Gr Maket Yem"},
  {"product_id":"14478","subproduct_id":"0","title":"Daiwa Kohga Alpha 150 Gr Slider Yem","color":"","price":960.15,"availability":"in stock","brand":"Daiwa","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > TAI RUBBER & SLIDER JIG","link":"https://www.sihirliolta.com/daiwa-kohga-alpha-150-gr-slider-yem","description":"Daiwa Kohga Alpha 150 Gr Slider Yem"},
  {"product_id":"19500","subproduct_id":"57044","title":"Bauer Jig Pro 7.5 Cm 40 Gr Jig Yem","color":"Zebra Glow","price":236.16,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-75-cm-40-gr-jig-yem","description":"Bauer Jig Pro 40 Gr Jig Yem"},
  {"product_id":"19498","subproduct_id":"57025","title":"Bauer Jig Pro 6 Cm 20 Gr Jig Yem","color":"Blue Pink Zebra","price":206.64,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-6-cm-20-gr-jig-yem","description":"Bauer Jig Pro 20 Gr Jig Yem"},
  {"product_id":"19498","subproduct_id":"57026","title":"Bauer Jig Pro 6 Cm 20 Gr Jig Yem","color":"Zebra Glow","price":206.64,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-6-cm-20-gr-jig-yem","description":"Bauer Jig Pro 20 Gr Jig Yem"},
  {"product_id":"19498","subproduct_id":"57027","title":"Bauer Jig Pro 6 Cm 20 Gr Jig Yem","color":"Orange Zebra Glow","price":206.64,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-6-cm-20-gr-jig-yem","description":"Bauer Jig Pro 20 Gr Jig Yem"},
  {"product_id":"19499","subproduct_id":"57028","title":"Bauer Jig Pro 7 Cm 30 Gr Jig Yem","color":"Blue Zebra","price":221.4,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-7-cm-30-gr-jig-yem","description":"Bauer Jig Pro 30 Gr Jig Yem"},
  {"product_id":"19499","subproduct_id":"57029","title":"Bauer Jig Pro 7 Cm 30 Gr Jig Yem","color":"Orange","price":221.4,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-7-cm-30-gr-jig-yem","description":"Bauer Jig Pro 30 Gr Jig Yem"},
  {"product_id":"19499","subproduct_id":"57030","title":"Bauer Jig Pro 7 Cm 30 Gr Jig Yem","color":"Green Zebra","price":221.4,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-7-cm-30-gr-jig-yem","description":"Bauer Jig Pro 30 Gr Jig Yem"},
  {"product_id":"19498","subproduct_id":"57023","title":"Bauer Jig Pro 6 Cm 20 Gr Jig Yem","color":"Rainbow","price":206.64,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-6-cm-20-gr-jig-yem","description":"Bauer Jig Pro 20 Gr Jig Yem"},
  {"product_id":"19498","subproduct_id":"57022","title":"Bauer Jig Pro 6 Cm 20 Gr Jig Yem","color":"Pink Zebra","price":206.64,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-6-cm-20-gr-jig-yem","description":"Bauer Jig Pro 20 Gr Jig Yem"},
  {"product_id":"19498","subproduct_id":"57021","title":"Bauer Jig Pro 6 Cm 20 Gr Jig Yem","color":"Green Zebra","price":206.64,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-6-cm-20-gr-jig-yem","description":"Bauer Jig Pro 20 Gr Jig Yem"},
  {"product_id":"19498","subproduct_id":"57020","title":"Bauer Jig Pro 6 Cm 20 Gr Jig Yem","color":"Orange","price":206.64,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-6-cm-20-gr-jig-yem","description":"Bauer Jig Pro 20 Gr Jig Yem"},
  {"product_id":"19497","subproduct_id":"57018","title":"Bauer Jig Pro 4.7 Cm 10 Gr Jig Yem","color":"Orange Zebra Glow","price":191.88,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-47-cm-10-gr-jig-yem","description":"Bauer Jig Pro 10 Gr Jig Yem"},
  {"product_id":"19498","subproduct_id":"57019","title":"Bauer Jig Pro 6 Cm 20 Gr Jig Yem","color":"Blue Zebra","price":206.64,"availability":"in stock","brand":"Bauer","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > JIG YEM","link":"https://www.sihirliolta.com/bauer-jig-pro-6-cm-20-gr-jig-yem","description":"Bauer Jig Pro 20 Gr Jig Yem"},
  {"product_id":"18690","subproduct_id":"56979","title":"Ryuji V8 8 Cm 5.5 Gr Silikon Yem","color":"Black Back","price":214.25,"availability":"in stock","brand":"Ryuji","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > SİLİKON YEM","link":"https://www.sihirliolta.com/ryuji-v8-8-cm-55-gr-silikon-yem","description":"Ryuji V8 8 Cm 5.5 Gr Silikon Yem"},
  {"product_id":"18690","subproduct_id":"56977","title":"Ryuji V8 8 Cm 5.5 Gr Silikon Yem","color":"Red Head White Silver","price":214.25,"availability":"in stock","brand":"Ryuji","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > SİLİKON YEM","link":"https://www.sihirliolta.com/ryuji-v8-8-cm-55-gr-silikon-yem","description":"Ryuji V8 8 Cm 5.5 Gr Silikon Yem"},
  {"product_id":"18690","subproduct_id":"56976","title":"Ryuji V8 8 Cm 5.5 Gr Silikon Yem","color":"White Silver","price":214.25,"availability":"in stock","brand":"Ryuji","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > SİLİKON YEM","link":"https://www.sihirliolta.com/ryuji-v8-8-cm-55-gr-silikon-yem","description":"Ryuji V8 8 Cm 5.5 Gr Silikon Yem"},
  {"product_id":"18690","subproduct_id":"56973","title":"Ryuji V8 8 Cm 5.5 Gr Silikon Yem","color":"Chartreuse Glow","price":214.25,"availability":"in stock","brand":"Ryuji","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > SİLİKON YEM","link":"https://www.sihirliolta.com/ryuji-v8-8-cm-55-gr-silikon-yem","description":"Ryuji V8 8 Cm 5.5 Gr Silikon Yem"},
  {"product_id":"18790","subproduct_id":"57080","title":"Ryuji Dog Strike 9 Cm 11 Gr Su Üstü Maket Yem","color":"Bone","price":317.12,"availability":"in stock","brand":"Ryuji","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > MAKET YEM","link":"https://www.sihirliolta.com/ryuji-dog-strike-9-cm-11-gr-su-ustu-maket-yem","description":"Ryuji Dog Strike 9 Cm 11 Gr Su Üstü Maket Yem"},
  {"product_id":"18790","subproduct_id":"57079","title":"Ryuji Dog Strike 9 Cm 11 Gr Su Üstü Maket Yem","color":"Rainbow","price":317.12,"availability":"in stock","brand":"Ryuji","product_type":"BALIK AVI MALZEMELERİ > SUNİ YEMLER > MAKET YEM","link":"https://www.sihirliolta.com/ryuji-dog-strike-9-cm-11-gr-su-ustu-maket-yem","description":"Ryuji Dog Strike 9 Cm 11 Gr Su Üstü Maket Yem"},
  {"product_id":"19518","subproduct_id":"57052","title":"Lucky John V-Snap Klips","color":"1","price":146.7,"availability":"in stock","brand":"Lucky John","product_type":"BALIK AVI MALZEMELERİ > OLTA APARATLARI > KLİPS | FIRDÖNDÜ > Klips","link":"https://www.sihirliolta.com/lucky-john-v-snap-klips","description":"Lucky John V-Snap Klips"}
];

return [{ json: { last_updated: "2026-05-16", total_products: products.length, products } }];
```

> **Kullanım:** WF2'de Feed Standard + Feed Variant HTTP Request nodelarını ve XML Parser'ı (Node 4) devre dışı bırak. "Statik Ürünler" adında yeni bir Code node ekle, bu kodu yapıştır, çıktısını Filter Products'a bağla. Gist Cache HTTP Request'ini de bypass et.

---

## Notlar

- Tüm scriptlerde `throw new NodeOperationError($node, mesaj)` kullanıldığında n8n workflow'u durdurur ve hata mesajını döndürür.
- `SITE_DOMAIN_BURAYA` yazan yeri kendi domain'inle değiştir.
- Typeform/Tally form alan isimlerini Node 2'de kendi formuna göre güncelle.
- Cache Gist ID ve GitHub kullanıcı adını SETUP.md'deki talimatlara göre doldur.
