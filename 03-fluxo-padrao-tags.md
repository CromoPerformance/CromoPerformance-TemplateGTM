# Fluxo Padrão: o que dispara em cada plataforma

> Browser = GTM Web (site + motor). Server = GTM Server (Stape). Cada evento mostra: o que o browser envia e o que o server reenvia.

---

## Arquitetura (visão geral)

```
[Site do Hotel]                    [Motor Omnibees (book.omnibees.com)]
   GTM Web (mesmo container)          GTM Web (mesmo container)
   ├─ GA4 - Config (transport_url ─┐   ├─ dataLayer (ecommerce events)
   │   -> Stape)                   │   │
   ├─ Meta - Config (Pixel init)   │   │
   └─ Tags de evento GA4 + Meta    │   │
                                  v   v
                        [GTM Server (Stape)]
                         ├─ GA4 Server (reenvia tudo p/ GA4)
                         └─ Meta CAPI (reenvia p/ Meta Events Manager)
```

- **Browser** → dispara tags GA4 + Meta Pixel + manda os dados para o Stape (`transport_url`).
- **Server** → recebe os eventos vindos do Stape e reenvia para GA4 e Meta (CAPI). O `event_id` gerado no browser viaja junto e **não é recriado** no server (dedup).

---

## Tags de base (disparam em toda página, site e motor)

### Browser
| Tag | O que faz | Parâmetros |
|---|---|---|
| `🟧 GA4 - Config` | inicializa GA4 | `tagId={{GA4 Measurement ID}}`, `transport_url={{Stape URL}}`, `send_page_view=true`, `allow_linker=true`, `linker_domains={{Linker Domains}}`, `decorate_forms=true`, `event_id={{Event ID}}` |
| `🟦 Meta - Config (Pixel Base)` | inicializa Meta Pixel | `fbq('init', {{Meta Pixel ID}})` |

### Server
| Tag | O que faz | Parâmetros |
|---|---|---|
| `🟧 GA4 Server` | reenvia tudo que chega | `measurementId={{GA4 Measurement ID}}`, `dataSource=request`, `defaultParametersToInclude=all` |

> Dispara no trigger **All Pages** (browser) e no client GA4 (server). O `GA4 - Config` envia `event_id` para dedup; as demais tags de evento do browser herdam esse mesmo `event_id` via `{{Event ID}}`.

---

## Eventos do Site (página do hotel)

| Evento | Browser GA4 | Browser Meta | Server CAPI | Parâmetros enviados |
|---|---|---|---|---|
| Page view | `page_view_site` | `PageViewSite` | `PageViewSite` | `event_id` |
| Clique WhatsApp | `click_whatsapp` | trackCustom `Contact` | `Contact` | `content_name=WhatsApp Contact`, `content_category={{Hotel Name}}`, `event_id` |
| Clique Telefone | `click_phone` | trackCustom `ContactPhone` | `ContactPhone` | `content_name={{Hotel Name}}`, `event_id` |
| Clique Mapa | `click_mapa` | `FindLocation` | `FindLocation` | `event_id` |
| Clique Reservar | `click_reservar` | track `BotaoReservar` | `BotaoReservar` | `content_name={{Hotel Name}}`, `content_category=Hotel Search`, `event_id` |
| Formulário enviado | `generate_lead_form` | track `Lead` | `Lead` | `content_name={{Hotel Name}}`, `event_id` |

---

## Eventos do Motor Omnibees (e-commerce)

| Evento | Browser GA4 | Browser Meta | Server CAPI | Parâmetros enviados |
|---|---|---|---|---|
| Page view | `page_view_motor` | `PageViewMotor` | `PageViewMotor` | `event_id` |
| Busca (ecommerceSearch) | `search` | track `Search` | `Search` | `content_name=Busca de Quartos - {{Hotel Name}}`, `content_category=Hotel Search`, `search_string={{DLV - Search String}}`, `event_id` |
| Resultados (mesmo push) | `view_search_results` | track `ViewContent` | `ViewContent` | `content_name={{DLV - Room Name}}`, `content_ids=[{{DLV - Room ID}}]`, `content_type=product`, `event_id` |
| Add to cart | `add_to_cart` | track `AddToCart` | `AddToCart` | `value={{DLV - Final Price}}`, `currency={{DLV - Currency}}`, `content_name={{DLV - Room Name}}`, `content_ids=[{{DLV - Room ID}}]`, `content_type=product`, `event_id` |
| Remove from cart | `remove_from_cart` | track `RemoveFromCart` | `RemoveFromCart` | idem AddToCart |
| Checkout passo 1 | `begin_checkout` | track `InitiateCheckout` | `InitiateCheckout` | idem + `num_items=1` |
| Checkout passo 2 | `add_payment_info` | track `AddPaymentInfo` | `AddPaymentInfo` | idem |
| Purchase | `purchase` | track `Purchase` | `Purchase` | idem + `transaction_id={{DLV - Transaction ID}}`, `num_items=1` |

> **`{{Event ID}}`** (variável JS que gera `evt_<timestamp>_<random>`) é incluída em **todas** as tags de evento (GA4 e Meta browser, e via Automap no server). É a chave do dedup entre browser e server.

### O que NÃO dispara no motor (não criar)
`view_item`, `view_cart`, `select_item`, `view_promotion`, `select_promotion`, formulário GAds.

---

## Dedup (browser ↔ server)

| Plataforma | Mecanismo | O que impede duplicata |
|---|---|---|
| GA4 | `event_id` no config + `message_id` | GA4 descarta o evento duplicado (browser vs server) |
| Meta | `eventID` no pixel + CAPI usa o **mesmo** via Automap | Meta conta 1× (browser + server com mesmo ID) |

> **Nunca recriar `event_id` no server**: quebra o dedup e gera contagem duplicada. O server lê a chave `event_id` vinda do request (variável `Event ID` tipo `ed`/Automap).

---

## Google Ads

- **Sem tags AW novas.** Permanecem no web: `GAds - Conversion Linker` (decora `gclid`; coexiste com o linker do GA4 sem conflito) e `Tag do Google AW-...` (fica inerte enquanto não preencher Conversion ID/Label real). Nada no server.
- Conversões vêm por **importação do GA4**: eventos favoritados (✩) no GA4 → importar em Google Ads → 24-48h → `purchase` como Principal.
- O `GA4 - Config` cuida do **linker GA4** (cross-domain entre site e motor); o `Ads Conversion Linker` segue no container por decisão do cliente.

---

## Resumo visual por evento

```
Site PageView ──► GA4(page_view_site) + Meta(PageViewSite) ──► Server(PageViewSite)
WhatsApp -------► GA4(click_whatsapp) + Meta(Contact) ------► Server(Contact)
Phone ----------► GA4(click_phone) + Meta(ContactPhone) -----► Server(ContactPhone)
Mapa -----------► GA4(click_mapa) + Meta(FindLocation) -----► Server(FindLocation)
Reservar -------► GA4(click_reservar) + Meta(BotaoReservar) -► Server(BotaoReservar)
Form -----------► GA4(generate_lead_form) + Meta(Lead) ------► Server(Lead)
─────────────────────────────────────────────────────────────────────
Motor PageView ► GA4(page_view_motor) + Meta(PageViewMotor) ► Server(PageViewMotor)
ecommerceSearch► GA4(search+view_search_results) + Meta(Search+ViewContent) ► Server
add_to_cart ---► GA4 + Meta(AddToCart) ──► Server(AddToCart)
remove_from_cart ► GA4 + Meta(RemoveFromCart) ► Server(RemoveFromCart)
begin_checkout ► GA4 + Meta(InitiateCheckout) ► Server(InitiateCheckout)
add_payment_info ► GA4 + Meta(AddPaymentInfo) ► Server(AddPaymentInfo)
purchase ------► GA4 + Meta(Purchase) ────► Server(Purchase)
```

> As 7 tags GA4 de e-commerce do motor (`search`, `view_search_results`, `add_to_cart`, `remove_from_cart`, `begin_checkout`, `add_payment_info`, `purchase`) enviam `sendEcommerceData=true`: o GTM lê `value`/`currency`/`items` do dataLayer automaticamente. Tags de pageview e de interação do site enviam `false` (sem ecommerce). `page_view_motor` deve seguir `false` (é pageview).