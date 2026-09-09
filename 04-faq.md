# FAQ: Tracking Cromo (GTM + Stape + GA4 + Meta + Google Ads)

> Perguntas rápidas sobre o modelo padrão. Base: containers web + server da pasta `teste`.

---

## Setup / Arquitetura

**1. Preciso criar tags de Google Ads no GTM?**
Não criamos tags AW novas. As tags existentes **ficam**: `GAds - Conversion Linker` e `Tag do Google AW-...` são mantidas (decisão: não quebram nada; o Conversion Linker cuida do `gclid` e coexiste com o linker do GA4). Conversões do Ads são **importadas do GA4** (favoritar ✩ no GA4 → Google Ads → Metas → Conversões → Novo → Importar → GA4). 24-48h para processar. Só preencher Conversion ID/Label real na `Tag do Google AW-...` se houver campanha ativa.

**2. Por que `transport_url`/Stape?**
A tag `GA4 - Config` aponta `transport_url={{Stape URL}}`. Assim todo evento GA4 (site + motor) passa pelo **GTM Server** antes de chegar ao GA4. Isso permite o server reenviar para Meta (CAPI) com os mesmos IDs e dados de usuário (IP, UA, cookies first-party).

**3. O que é o client GA4 no server?**
O **client GA4** (client type `gaaw_client`) é o "portão de entrada" do server: interpreta requests GA4, gera cookie **FPID** (first-party, `cookieDomain=auto`, maxAge 63072000) e dispara o trigger `TR \| GA4 Client`. Mantém a identidade do usuário entre domínios (site ↔ motor) sem depender do `_ga` third-party.

**4. O mesmo GTM web roda no site E no motor?**
Sim. O mesmo container (ex.: `GTM-MV3D22C` no Pedra) é instalado no WordPress do site e no Omnibees (MyHotel → Scripts de Rastreamento). Os triggers separam o que dispara em cada domínio via `Page URL`/`Page Hostname` + variáveis `Hotel Domain` e `Booking Engine Domain`.

---

## Eventos / Parâmetros

**5. `search` e `view_search_results` são o mesmo evento?**
O motor Omnibees emite **um único push** `ecommerceSearch`. A partir dele criamos **duas tags GA4** (`search` + `view_search_results`) e **duas Meta** (`Search` + `ViewContent`) para cobrir busca e resultados juntos (jornada real do motor).

**6. Quais parâmetros as tags GA4 do motor enviam?**
As 7 tags GA4 do motor (`search`, `view_search_results`, `add_to_cart`, `remove_from_cart`, `begin_checkout`, `add_payment_info`, `purchase`) usam **`sendEcommerceData=true`**: o GTM lê o dataLayer e envia sozinho `value`, `currency`, `items[]` (item_id, item_name, price, quantity) e `transaction_id` no purchase. **Não** adicionar esses parâmetros à mão (duplicaria).

**7. E os parâmetros na Meta (Pixel)?**
As tags HTML Meta usam variáveis **DLV** (Data Layer v2):
- `content_name` = `{{DLV - Room Name}}` (`ecommerce.items[0].item_name`)
- `content_ids` = `[{{DLV - Room ID}}]` (`ecommerce.items[0].item_id`)
- `value` = `{{DLV - Final Price}}` (`ecommerce.value`)
- `currency` = `{{DLV - Currency}}` (`ecommerce.currency`)
- `transaction_id` = `{{DLV - Transaction ID}}` (`ecommerce.transaction_id`) no Purchase
- `search_string` = `{{DLV - Search String}}` (JS que monta "checkin X, checkout Y, N adultos..." do último `ecommerceSearch`)

**8. Por que no browser usamos eventos custom para contato (trackCustom `Contact`, `ContactPhone`)? E no server?**
- **Browser (Pixel):** o `fbq('track')` é para eventos padrão da Meta (Purchase, Lead, etc.). WhatsApp/telefone são **eventos customizados** de clique, então usamos `fbq('trackCustom', 'Contact', ...)` e `fbq('trackCustom', 'ContactPhone', ...)` para diferenciar os canais sem ocupar um evento padrão.
- **Server (CAPI):** o WhatsApp usa `eventNameStandard=Contact` (esse evento existe na lista padrão do template). O **telefone usa `eventNameCustom=ContactPhone`**, espelhando o browser.
- **E o dedup?** Meta deduplica browser+server por `event_name` + `event_id`. Alinhado: WhatsApp (browser `Contact` ↔ server `Contact`) e Phone (browser `ContactPhone` ↔ server `ContactPhone`), ambos com o mesmo `event_id` → conta 1× cada.

**9. O que é `{{Event ID}}`?**
Variável JS que gera `evt_<timestamp>_<random>` **uma vez por disparo de tag**. É incluída em quase todas as tags (GA4 via `eventSettingsTable`/config, Meta via `eventID`) e viaja ao server, que **lê** e reutiliza via Automap (dropdown "Event ID" das CAPI fica vazio de propósito).

**10. Por que não recriar `event_id` no server?**
Se o server gerasse outro ID, browser e server teriam IDs diferentes e a Meta contaria **2 eventos** (dedup quebrado). A variável server `Event ID` (tipo `ed`) **apenas lê** `event_id` vindo do request. É o core do dedup.

---

## Triggers

**11. Por que `TR \| Click - Botao Reservar` agora é LINK_CLICK?**
Antes era FORM_SUBMISSION filtrando `Click URL` (que não existe em form) e **nunca disparava**. Agora é **Link Click** com `{{Click URL}}` contém `reservar`. Se o site usar outro texto/URL, ajustar o filtro.

**12. Filtros dos triggers usam valores fixos?**
Não. Os triggers de domínio usam variáveis: `Pageview - Site` e `Form - Submit` filtram `{{Hotel Domain}}`, `PageView - Motor` filtra `{{Booking Engine Domain}}`. Contatos usam substring fixa esperada: `whatsapp` (cobre `api.whatsapp.com` e `wa.me`), `tel:`, `maps`, `waze`.

---

## Variáveis

**13. `{{Currency}}` vs `{{DLV - Currency}}`: qual usar?**
- `Currency` = constante (ex.: `BRL`), usada em AddToCart/RemoveFromCart (essas tags têm `currency` top-level no dataLayer).
- `DLV - Currency` = Data Layer v2 lendo `ecommerce.currency` (checkout/purchase **não** têm `currency` top-level). Foi removida a constante `Currency` duplicada no modelo atual; seguir com `DLV - Currency` nos novos eventos.

**14. Por que tantas variáveis `[v]` (promotion_name, discount, value, items...)?**
São **variáveis automáticas do template GA4 Advanced** (criadas pelo template, não por nós). **Não deletar**: o template referencia por nome.

**15. `Meta Pixel ID` agora é usada?**
Sim. A tag `Meta - Config (Pixel Base)` agora inicializa com `{{Meta Pixel ID}}` (antes era `__PIXEL_ID__` hardcoded no HTML). Preencher a constante por cliente.

---

## Google Ads (importação)

**16. Quais eventos importar no Ads?**
Favoritar ✩ no GA4: `purchase` (Principal), `begin_checkout`, `add_payment_info`, `add_to_cart`, `search`, `click_whatsapp`, `click_phone`, `generate_lead_form` (secundárias, conforme a conta).

**17. Quanto tempo leva?**
24-48h. O Ads importa **dados processados** do GA4, não tempo real. Se o evento não aparece no import, conferir se está favoritado e se o GA4 já processou dados dele.

**18. E se outra agência usa tags AW no GTM?**
Não mexer nas tags alheias (fora do escopo). Nossas tags AW existentes (`GAds - Conversion Linker`, `Tag do Google AW-...`) ficam, e as conversões importadas do GA4 convivem com as deles; se houver duplicidade de conversão, avaliar com o cliente.

---

## Erros comuns

**19. Evento duplicado no Meta/GA4.**
Checar `Event ID`: mesma tag disparando 2× (triggers sobrepostos) ou `event_id` recriado no server. Cada evento = 1 trigger = 1 tag → nunca duplicar fogo. No Meta, conferir também se browser e server usam **o mesmo nome de evento** (ex.: phone `ContactPhone` nos dois, ver pergunta 8).

**20. Import do JSON falha com "campo inválido".**
GTM Server não aceita JSON gerado por **PowerShell** (corrompe emojis nos nomes: `dYY¦`). Regenerar com **Node.js** (`JSON.stringify`), mantendo `customTemplate`/`client` do export do destino. Nomes limpos, sem BOM, sem `folder`/`folderId`, sem placeholder de token real.

**21. Tags GA4 do motor sem `value`/`currency`.**
O `sendEcommerceData` está `false` na tag (herança do Aretê). Deixar `true` nas 7 tags do motor e `false` nas tags do site (eventos de interação não têm ecommerce) e no `page_view_motor` (é pageview).

**22. Evento de site não dispara (ex.: telefone).**
Conferir se o site **tem o elemento** (ex.: link `tel:`). O trigger só dispara se o clique existir. WhatsApp: verificar se a URL contém `whatsapp` (o trigger cobre `api.whatsapp.com` e `wa.me`).

---

*Para mais detalhes técnicos: `docs/standard-tracking-model.md`, `containers/PEDRA LAGUNA/mapa-eventos-pedra-laguna.md`.*