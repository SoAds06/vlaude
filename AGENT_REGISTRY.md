# Agent Registry

Master list of all agents in this workspace.

| Agent | Folder | Goals | Skills | Heartbeat | Status |
|-------|--------|-------|--------|-----------|--------|
| (your first agent) | `agents/[name]/` | | | | Planning |
| Olta Danışmanı | `agents/olta-danismani/` | Doğru eşleşme %90+, sepet dönüşümü %25+ | CUSTOMER_INTAKE, PRODUCT_MATCH, CART_BUILDER | Event-driven (n8n webhook) + Haftalık inceleme | Active |
| Price Checker | `agents/price-checker/` | Fiyat rekabeti takibi, rakip fiyat tespiti | PRICE_CHECK, EMAIL_REPORT | Her gün 16:00 TRT (`0 13 * * *` UTC) | Active |
