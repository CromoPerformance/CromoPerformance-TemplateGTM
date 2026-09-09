# Mapa de Correspondência: GTM <-> GA4 <-> Meta <-> Google Ads

> Base: modelo padrão Cromo (Omnibees). Cobre o que existe no modelo.
> Nomes: 🟧 = GA4, 🟦 = Meta. Origem: `(Site)` = página do hotel, `(Omnibees)` = motor de reservas.

---

## 1. Correspondência por evento

Cada evento da jornada tem: trigger web, tag GA4 (browser), pixel Meta (browser), tag GA4 Server e tag Meta CAPI (server).

### Motor (Omnibees), e-commerce

| Evento dataLayer | Trigger (web) | Tag GA4 (browser) | Tag Meta (browser) | GA4 Server (reeenvia) | Meta CAPI (server) |
|---|---|---|---|---|---|
| `ecommerceSearch` | `TR \| Search` | `🟧 GA4 - search` | `🟦 Meta - Search` | GA4 Server | `🟦 Meta CAPI - Search` |
| `ecommerceSearch` (mesmo push) | `TR \| Event - view_search_results` | `🟧 GA4 - view_search_results` | `🟦 Meta - ViewContent` | GA4 Server | `🟦 Meta CAPI - ViewContent` |
| `add_to_cart` | `TR \| Add to Cart` | `🟧 GA4 - add_to_cart` | `🟦 Meta - AddToCart` | GA4 Server | `🟦 Meta CAPI - AddToCart` |
| `remove_from_cart` | `TR \| Remove from Cart` | `🟧 GA4 - remove_from_cart` | `🟦 Meta - RemoveFromCart` | GA4 Server | `🟦 Meta CAPI - RemoveFromCart` |
| `ecommerceCheckout` + label `Passo 1` | `TR \| Omnibees - Checkout Step 1` | `🟧 GA4 - begin_checkout` | `🟦 Meta - InitiateCheckout` | GA4 Server | `🟦 Meta CAPI - InitiateCheckout` |
| `ecommerceCheckout` + label `Passo 2` | `TR \| Omnibees - Checkout Step 2` | `🟧 GA4 - add_payment_info` | `🟦 Meta - AddPaymentInfo` | GA4 Server | `🟦 Meta CAPI - AddPaymentInfo` |
| `ecommercePurchase` | `TR \| Purchase` | `🟧 GA4 - purchase` | `🟦 Meta - Purchase` | GA4 Server | `🟦 Meta CAPI - Purchase` |
| PageView do motor | `TR \| PageView - Motor` | `🟧 GA4 - page_view_motor` | `🟦 Meta - PageViewMotor` | GA4 Server | `🟦 Meta CAPI - PageViewMotor` |

> ⚠️ Busca e resultados são o **mesmo** push (`ecommerceSearch`): geram `search` + `view_search_results` no GA4 e `Search` + `ViewContent` na Meta.

### Site (página do hotel)

| Ação | Trigger (web) | Tag GA4 (browser) | Tag Meta (browser) | Meta CAPI (server) |
|---|---|---|---|---|
| PageView do site | `TR \| Pageview - Site` | `🟧 GA4 - page_view_site` | `🟦 Meta - PageViewSite` | `🟦 Meta CAPI - PageViewSite` |
| Clique WhatsApp | `TR \| Click - WhatsApp` | `🟧 GA4 - click_whatsapp` | `🟦 Meta - Contact WhatsApp` (trackCustom `Contact`) | `🟦 Meta CAPI - Contact (WhatsApp)` |
| Clique Telefone (`tel:`) | `TR \| Click - Phone` | `🟧 GA4 - click_phone` | `🟦 Meta - Contact Phone` (trackCustom `ContactPhone`) | `🟦 Meta CAPI - Contact (Phone)` |
| Clique Mapa (`maps`) | `TR \| Click - Mapa 1` | `🟧 GA4 - click_mapa` | `🟦 Meta - FindLocation` | `🟦 Meta CAPI - FindLocation` |
| Clique Mapa (`waze`) | `TR \| Click - Mapa 2` | `🟧 GA4 - click_mapa` (mesma tag) | `🟦 Meta - FindLocation` (mesma tag) | `🟦 Meta CAPI - FindLocation` |
| Clique Reservar (`reservar`) | `TR \| Click - Botao Reservar` | `🟧 GA4 - click_reservar` | `🟦 Meta - BotaoReservar` | `🟦 Meta CAPI - BotaoReservar` |
| Formulário enviado | `TR \| Form - Submit` | `🟧 GA4 - generate_lead_form` | `🟦 Meta - Lead Form` (track `Lead`) | `🟦 Meta CAPI - Lead (Form)` |

### Tags de configuração (base)

| Tag | Tipo | Trigger | Função |
|---|---|---|---|
| `🟧 GA4 - Config` | Google Tag (gtag) | All Pages | Inicializa GA4 com `transport_url` (Stape), linker cross-domain, `event_id` |
| `🟦 Meta - Config (Pixel Base)` | HTML | All Pages | `fbq('init')` do Pixel |

> **Google Ads:** mantemos as tags AW existentes no web (`GAds - Conversion Linker` e `Tag do Google AW-...`, decisão: inofensivas). Não criamos tags AW novas. A correspondência com o Ads é principalmente por **importação de eventos do GA4** (ver §3).

---

## 2. Correspondência de nomes por plataforma

| Evento (jornada) | GA4 (nomemáquina) | Meta Pixel (browser) | Meta CAPI (server) |
|---|---|---|---|
| Compra | `purchase` | `Purchase` | `Purchase` |
| Checkout passo 1 | `begin_checkout` | `InitiateCheckout` | `InitiateCheckout` |
| Pagamento passo 2 | `add_payment_info` | `AddPaymentInfo` | `AddPaymentInfo` |
| Adicionar carrinho | `add_to_cart` | `AddToCart` | `AddToCart` |
| Remover do carrinho | `remove_from_cart` | `RemoveFromCart` | `RemoveFromCart` |
| Busca | `search` | `Search` | `Search` |
| Resultados de busca | `view_search_results` | `ViewContent` | `ViewContent` |
| View page site | `page_view_site` | `PageViewSite` | `PageViewSite` |
| View page motor | `page_view_motor` | `PageViewMotor` | `PageViewMotor` |
| WhatsApp | `click_whatsapp` | `Contact` (custom) | `Contact` |
| Telefone | `click_phone` | `ContactPhone` (custom) | `ContactPhone` (custom) |
| Mapa | `click_mapa` | `FindLocation` | `FindLocation` |
| Botão Reservar | `click_reservar` | `BotaoReservar` | `BotaoReservar` |
| Formulário | `generate_lead_form` | `Lead` | `Lead` |

> ⚠️ Meta Pixel usa `trackCustom` = eventos custom (Contact, ContactPhone, BotaoReservar, FindLocation). `track` = eventos padrão da Meta (Purchase, Lead, Search, ViewContent, AddToCart, RemoveFromCart, InitiateCheckout, AddPaymentInfo).

---

## 3. Google Ads <-> GA4 (importação)

```
GA4 (eventos favoritados ✩)  --->  Google Ads (Metas -> Conversões -> Importar -> GA4)
        purchase (Principal)
        begin_checkout, add_payment_info, add_to_cart,
        search, click_whatsapp, click_phone, generate_lead_form (secundárias, opcional)
```

- **Favoritar no GA4 (✩)** é obrigatório: só eventos favoritados aparecem no import do Ads.
- Importação leva **24-48h** (usa dados processados do GA4, não Realtime).
- **Sem tags AW novas**: mantemos as existentes (`GAds - Conversion Linker` + `Tag do Google AW-...`); o vínculo principal de conversão é a importação do GA4 (evita duplicidade e conflito com tags de outras agências).

---

## 4. Correspondência de IDs (por cliente)

| Variável | Onde | Valor a preencher |
|---|---|---|
| `GA4 Measurement ID` | Web + Server | `G-XXXXX` |
| `Meta Pixel ID` | Web + Server | `1234567890` |
| `Facebook Access Token` | Server (só UI) | token secreto |
| `Hotel Name` | Web | nome do hotel |
| `Hotel Domain` | Web | `hotel.com.br` |
| `Booking Engine Domain` | Web | `book.omnibees.com` |
| `Stape URL` | Web | `https://XXXX.sae1.stape.io` |
| `Linker Domains` | Web | `["hotel.com.br","book.omnibees.com"]` |

---

*Regra de ouro: um evento de jornada = 1 trigger = 1 tag GA4 + 1 tag Meta no browser = 1 tag CAPI no server. Nunca duplicar fogo.*