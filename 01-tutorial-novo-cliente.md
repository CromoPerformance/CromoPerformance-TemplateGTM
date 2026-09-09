# Tutorial: Configurar Tracking de Cliente Novo (Omnibees)

> Modelo padrão Cromo. Stack: GTM Web -> GTM Server (Stape.io) -> GA4 + Meta (Pixel + CAPI).
> Google Ads: **sem tags AW novas no GTM**. Mantemos as existentes (`GAds - Conversion Linker`, `Tag do Google AW-...`); conversões do Ads são importadas do GA4 (Etapa 8).

---

## Etapa 0: Coletar dados do cliente

| Dado | Onde pegar | Exemplo |
|---|---|---|
| Domínio do site | Cliente | `hotel.com.br` |
| Domínio do motor | Omnibees | `book.omnibees.com` (c=xxxx) |
| Measurement ID (GA4) | GA4 -> Admin -> Fluxo de dados | `G-XXXXXXXXXX` |
| Meta Pixel ID | Meta Events Manager -> Configurações | `123456789012345` |
| Access Token | Meta -> Events Manager -> API de Conversões -> Gerar | secreto (nunca commitar) |
| Nome do hotel | Cliente | `Hotel Exemplo` |

---

## Etapa 1: GA4 (se não existir)

1. Criar propriedade GA4 e fluxo de dados Web (se ainda não houver).
2. Copiar o **Measurement ID** (`G-XXXXX`).

---

## Etapa 2: Meta Pixel + Access Token

1. **Meta Events Manager -> Conectar fontes de dados -> Web.** Criar o pixel.
2. Copiar o **Pixel ID**.
3. Mesma tela -> **API de Conversões (CAPI)** -> **Gerar Access Token**. Copiar o token em lugar seguro (é secreto).

---

## Etapa 3: Stape.io

1. Criar conta em [stape.io](https://stape.io) (login Google).
2. Plano **Free**.
3. Criar container:
   - Name: `CROMO - <CLIENTE>`
   - Region: `sa-east-1` (ou a mais próxima do cliente).
4. Copiar a **Container URL** (ex.: `https://XXXX.sae1.stape.io`). É o `{{Stape URL}}`.

---

## Etapa 4: GTM Server

1. [tagmanager.google.com](https://tagmanager.google.com) -> criar container:
   - Account: a conta da Cromo do cliente
   - Name: `[SERVER] <CLIENTE>`
   - Platform: **Server**
2. **Administrador -> Importar Container** -> arquivo `server-container-import.json` -> opção **Merge**.
3. Preencher variáveis (Workspace -> Variáveis):
   - `GA4 Measurement ID`: do Etapa 1
   - `Facebook Pixel ID`: do Etapa 2
   - `Facebook Access Token`: colar o token do Etapa 2 **direto na UI** (não salvar em arquivo)
4. **Publicar** o container.
5. Copiar a URL do server (Admin -> Server-side tagging).

> Já vem pronto do import: Client GA4 com FPID, tag `GA4 Server`, 14 tags `Meta CAPI`, triggers, templates.

---

## Etapa 5: Conectar Stape -> GTM Server

1. No Stape: colar a URL do GTM Server para receber as requests.
2. Confirmar que o container aparece como `active`.

---

## Etapa 6: GTM Web

1. No GTM (conta do cliente): criar container `[WEB] <CLIENTE>` (ou reutilizar o existente).
2. **Importar Container** (Merge) o arquivo `web-container-import.json`.
3. Preencher variáveis constantes:

| Variável | Valor |
|---|---|
| `GA4 Measurement ID` | `G-XXXXX` |
| `Hotel Name` | nome do hotel |
| `Meta Pixel ID` | pixel do Etapa 2 |
| `Hotel Domain` | `hotel.com.br` |
| `Booking Engine Domain` | `book.omnibees.com` |
| `Stape URL` | `https://XXXX.sae1.stape.io` |
| `Linker Domains` | `["hotel.com.br","book.omnibees.com"]` (JSON string) |

4. Conferir a tag `GA4 - Config`: `transport_url = {{Stape URL}}`, `linker_domains = {{Linker Domains}}`, `event_id = {{Event ID}}`.
5. Ajustar triggers por-site (se precisar):
   - WhatsApp: `{{Click URL}}` contém `whatsapp` (cobre `api.whatsapp.com` e `wa.me`)
   - Mapa: `maps` / `waze`
   - Botão Reservar: `reservar` (ou texto do botão, se diferente)
6. **Google Ads:** manter as tags existentes `GAds - Conversion Linker` e `Tag do Google AW-...` (decisão: não quebram nada, ficam). Configurar o Conversion ID/Label reais na `Tag do Google AW-...` se houver campanha ativa. Não criar tags AW novas.
7. **Publicar.**

---

## Etapa 7: Conectar GTM Web -> Server

1. GTM Web -> **Administrador -> Configurações do container** -> ativar **Server-side tagging**.
2. Colar a **Server Container URL** do Stape.
3. **Proxy type:** `Stape`.
4. Salvar.

---

## Etapa 8: Instalar GTM no site e no motor

1. **Site (WordPress):** inserir o snippet `<head>` e `<body>` do GTM Web no tema (ou via plugin, ex. GTM4WP).
2. **Motor Omnibees:** [myhotel2.omnibees.com](https://myhotel2.omnibees.com) -> **BeeDirect -> Configurações -> Estatísticas -> Scripts de Rastreamento**:
   - Ativar "Ativar outros scripts de rastreamento"
   - **Head Script:** snippet `<head>` do GTM Web
   - **Body Script:** snippet `<body>` (noscript)

> O mesmo container roda no site e no motor: consistência de tags e dedup.

---

## Etapa 9: Validar

1. **GTM Preview:** abrir site e motor em preview -> tags GA4/Meta devem disparar nas ações reais.
2. **GA4 Tempo real / DebugView:** eventos `page_view_site`, `page_view_motor`, `search`, `add_to_cart`, `begin_checkout`, `purchase`, `click_whatsapp`, `click_phone`, `generate_lead_form`.
3. **Meta Test Events:** ver "Navegador" e "Servidor" como **Processado** com o mesmo `event_id` (dedup).
4. **Stape:** requests chegando ao server.
5. Confirmar que **nenhum** evento aparece duplicado (GA4 e Meta).

---

## Etapa 10: Google Ads (importar conversões do GA4)

1. **No GA4:** Relatórios -> Engajamento -> Eventos -> **favoritar (✩)** os eventos a importar:
   `purchase`, `begin_checkout`, `add_payment_info`, `add_to_cart`, `search`, `click_whatsapp`, `click_phone`, `generate_lead_form`.
   > Só eventos favoritados aparecem na lista de importação do Ads.
2. **No Google Ads:** Metas -> Conversões -> **Novo -> Importar -> Google Analytics 4** -> selecionar os eventos favoritados.
3. Aguardar **24-48h** (o Ads importa dados processados do GA4, não tempo real).
4. Marcar `purchase` como **Principal** (otimização de lances); demais como secundárias (opcional).

**Não criar tags AW no GTM nem no server.** Vínculo é feito direto em Metas -> Conversões.

---

## Resumo do fluxo de dados

```
Site + Motor (GTM Web)
   |  eventos no dataLayer + {{Event ID}}
   v
Stape (transport_url)  ->  GTM Server
   |  GA4 Server  (dataSource=request)  ->  GA4  ->  Google Ads (import ✩)
   +  Meta CAPI  (autoMap + event_id)   ->  Meta Events Manager
```

*Referência: arquivos `GTM-WBXQ2657_workspace5.json` (web) e `GTM-PWT4FPS2_workspace5.json` (server) na pasta teste.*