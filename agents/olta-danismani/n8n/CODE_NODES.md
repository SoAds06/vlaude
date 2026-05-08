# n8n Code Node Scriptleri

Her script kendi node'una kopyala-yapıştır. n8n'de "Code" node, JavaScript kullanır.

---

## 1. XML Parser (İki Feed — Merge)
**Node:** Workflow 1 → Node 4
Node 2'nin adı "Feed Standard", Node 3'ün adı "Feed Variant" olmalı.

```javascript
// Feed 1: varyantsız ürünler (kamış, makine vb.)
const xml1 = $('Feed Standard').first().json.data;
// Feed 2: varyantlı ürünler (misina, lider vb.)
const xml2 = $input.first().json.data;

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
      product_id = rawId;
      subproduct_id = '0';
    } else {
      const itemGroupId = getGTag('item_group_id');
      const isVariant = itemGroupId && itemGroupId !== rawId;
      product_id = isVariant ? itemGroupId : rawId;
      subproduct_id = isVariant ? rawId : '0';
    }

    items.push({
      product_id,
      subproduct_id,
      title: getTag('title'),
      price,
      availability: 'in stock',
      brand: getGTag('brand'),
      product_type: getGTag('product_type'),
      link: getTag('link'),
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

// Groq/OpenAI, Gemini ve Anthropic yanıt formatlarını destekle
let rawText = '';
if (response.choices && response.choices[0]) {
  // Groq / OpenAI formatı
  rawText = response.choices[0].message.content || '';
} else if (response.candidates && response.candidates[0]) {
  // Gemini formatı
  rawText = response.candidates[0].content.parts[0].text || '';
} else if (response.content && response.content[0]) {
  // Anthropic formatı
  rawText = response.content[0].text || '';
} else if (response.text) {
  rawText = response.text;
} else {
  rawText = JSON.stringify(response);
}

// JSON parse et (Claude bazen açıklama ekleyebilir)
let result;
try {
  // Sadece JSON bloğunu bul
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

// Sepet linki yoksa oluştur
if (!result.sepet_linki && result.oneriler && result.oneriler.length > 0) {
  const parts = result.oneriler
    .map(o => `count:1;product_id:${o.product_id};subproduct_id:${o.subproduct_id || '0'}`)
    .join('-');
  result.sepet_linki = `https://${SITE_DOMAIN}/srv/service/cart/create-cart-from-url/${parts}`;
}

return [{ json: result }];
```

---

## Notlar

- Tüm scriptlerde `throw new NodeOperationError($node, mesaj)` kullanıldığında n8n workflow'u durdurur ve hata mesajını döndürür.
- `SITE_DOMAIN_BURAYA` yazan yeri kendi domain'inle değiştir.
- Typeform/Tally form alan isimlerini Node 2'de kendi formuna göre güncelle.
- Cache Gist ID ve GitHub kullanıcı adını SETUP.md'deki talimatlara göre doldur.
