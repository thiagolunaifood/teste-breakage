# incentivos-experiments

LPs de discovery do iFood Benefícios para experimentos de **incentivo com breakage** (valores com prazo limitado pra uso no iFood).

Construídas seguindo o playbook em `../playbook_contrucao_lps_app.md`. Hospedagem na Vercel, abertas dentro da WebView do app via banner ou Braze.

---

## Estrutura

```
incentivos-experiments/
├── breakage-5.html        ┐
├── breakage-10.html       │  LPs principais — 1 por valor de incentivo
├── breakage-15.html       │  Idênticas exceto: title, h1, copy "R$ N", hero,
├── breakage-20.html       │  e constante INCENTIVE_VALUE no topo do <script>
├── breakage-25.html       ┘
├── sucesso.html           # Página pós-CTA (compartilhada). Lê ?valor= da URL.
├── reward-card-5.svg      ┐
├── reward-card-10.svg     │  Hero exportado do Figma — 1 por variante
├── reward-card-15.svg     │
├── reward-card-20.svg     │
├── reward-card-25.svg     ┘
├── icon-check.svg                  # Check verde da página de sucesso
├── success-icon-groceries.svg      # Card "Mercados com desconto"
├── success-icon-frete-gratis.svg   # Card "Restaurantes com frete grátis"
├── success-icon-coin.svg           # Card "Ver meu extrato"
├── fonts/                          # Tipo iFood Titulos + Textos (WOFF2)
├── vercel.json                     # cleanUrls + rewrites + X-Frame-Options ALLOWALL
└── README.md
```

### Variantes do teste

| Valor   | URL pública                                              | Hero                  |
|---------|----------------------------------------------------------|-----------------------|
| R$ 5    | `https://<projeto>.vercel.app/breakage-5`                | `reward-card-5.svg`   |
| R$ 10   | `https://<projeto>.vercel.app/breakage-10`               | `reward-card-10.svg`  |
| R$ 15   | `https://<projeto>.vercel.app/breakage-15`               | `reward-card-15.svg`  |
| R$ 20   | `https://<projeto>.vercel.app/breakage-20`               | `reward-card-20.svg`  |
| R$ 25   | `https://<projeto>.vercel.app/breakage-25`               | `reward-card-25.svg`  |

URL pra colocar no banner do backoffice (substitua `{N}` pelo valor):

```
rawRoute?path=/webView?withAppBar=false&url=https://<projeto>.vercel.app/breakage-{N}
```

URL pra usar em push notification ou in-app message do Braze (setting "Deeplink into application"):

```
ifoodbenefits://inapp/webView?withAppBar=false&url=https://<projeto>.vercel.app/breakage-{N}
```

Observações sobre o deeplink:
- O prefixo `/inapp/` é obrigatório — sem ele o app cai na home
- `withAppBar=false` deixa a LP usar a tela inteira (sem barra nativa em cima)
- Mesmo formato funciona pra push e in-app
- Pra segmentar a origem no Amplitude, adicione `?source=push` ou `?source=inapp` no fim da `url` interna (vai parar em `event_properties.referrer`)

---

## Fluxo do experimento

1. Usuário toca o banner / push / in-app na home do app
2. WebView abre `/breakage-{N}` → apresenta o presente de R$ N com prazo de 4 dias
3. Usuário toca **Quero meu presente** → tracking `lp_cta_click` + navega pra `/sucesso?valor={N}`
4. `/sucesso` confirma a liberação (check verde grande, título dinâmico "Presente de R$ N liberado!") e oferece 3 atalhos: mercado com desconto, restaurantes com frete grátis, ver extrato
5. Cada atalho dispara um deeplink configurado em `DEEPLINKS` (sucesso.html). Os três estão configurados.
6. Link **Usar presente depois** volta pra home do iFood Benefícios via `RedirectToDeepLinkInternal` → `ifoodbenefits://inapp/home`

---

## Como adicionar / atualizar uma variante

As 5 LPs são clones uma da outra. Pra gerar uma nova variante (ex.: R$ 30) ou regenerar todas após uma mudança de copy:

```bash
# Editar uma das variantes como "fonte da verdade", por exemplo breakage-10.html.
# Depois, regenerar as outras a partir dela:

cd incentivos-experiments
SRC=breakage-10.html
SRC_VALUE=10

for v in 5 15 20 25; do
  sed -e "s|R\$ $SRC_VALUE|R\$ $v|g" \
      -e "s|/reward-card-$SRC_VALUE\.svg|/reward-card-$v.svg|g" \
      -e "s|const INCENTIVE_VALUE = $SRC_VALUE;|const INCENTIVE_VALUE = $v;|" \
      $SRC > breakage-$v.html
done
```

Pontos que mudam por variante (ao gerar manualmente):
1. `<title>` e `<meta name="description">` — `R$ N`
2. `<h1>` da seção principal — `R$ N`
3. `alt` do hero — `R$ N`
4. `src` do hero — `/reward-card-N.svg`
5. Info card 1 — texto "valor de R$ N"
6. FAQ 1 — "seu presente de R$ N"
7. **Constante `INCENTIVE_VALUE` no `<script>`** — propaga pro Amplitude

A copy do CTA e dos demais elementos é genérica (não menciona valor).

---

## Tipografia

Usa **Tipo iFood** (Fabio Haag Type, proprietária — uso exclusivo iFood):

- `TipoiFood Titulos` (Bold 700, ExtraBold 800) — h1, h2, CTA, números
- `TipoiFood Textos` (Regular 400, Medium 500, Bold 700) — body, FAQ, info cards

Os 5 WOFF2 vivem em `/fonts/`. Não republicar em outro repo.

---

## Tracking (Amplitude)

API key (`AMPLITUDE_API_KEY = '70c4aa9143716ab2bb6a43ebbaf81bf3'`) preenchida em todos os HTMLs. Se for esvaziada, `trackAmplitude()` vira no-op silencioso (não quebra a página).

**Eventos disparados:**

| Página              | Event type                | Properties extras                       |
|---------------------|---------------------------|-----------------------------------------|
| breakage-{N}        | `lp_page_view`            | `available_channels`                    |
| breakage-{N}        | `lp_faq_open`             | `question`                              |
| breakage-{N}        | `lp_cta_click`            | `cta_type=positive`, `cta_label`        |
| sucesso             | `sucesso_page_view`       | —                                       |
| sucesso             | `sucesso_option_click`    | `option` (mercado/frete_gratis/extrato), `channel`, `target`, `dispatched` |
| sucesso             | `sucesso_later_click`     | — (link "Usar presente depois")         |

**Properties padrão (em TODOS os eventos):** `teste=breakage`, `variant=breakage`, **`incentive_value` (5/10/15/20/25)**, `session_uuid`, `person_id`, `person_id_source`, `platform`, `page_url`, `referrer`.

Pra segmentar resultados do A/B/C/D/E no Amplitude, filtre por `event_property: incentive_value = N`. Pra ver o funil agregado de todas as variantes, segmente só por `event_property: teste = breakage`.

---

## Identidade do usuário

Cascata: querystring `?person_id=` → JWT do cookie `aAccessToken` → `null`. Sempre tem `session_uuid` em `localStorage` como fallback estável por dispositivo.

Hoje, sem cooperação do backoffice/mobile, o `session_uuid` é o identificador confiável. Correlação com `personId` real fica post-hoc via time-window join com o evento `banner_click` (ver seção 9.4 do playbook).

Importante: cookies setados em `.ifood.com.br` (incluindo `aAccessToken`) **não chegam** em `*.vercel.app` por causa de same-origin. Pra que `person_id` venha via JWT, a LP precisaria ser servida de subdomínio `.ifood.com.br`.

---

## QA / debug

Modal de debug gateado por `?debug=1`:

```
https://<projeto>.vercel.app/breakage-10?debug=1
https://<projeto>.vercel.app/sucesso?valor=10&debug=1
https://<projeto>.vercel.app/breakage-10?person_id=teste123&debug=1
```

Mostra: person_id (fonte), session_uuid, JWT claims, cookies, canais Flutter detectados, plataforma, query params, URL completa, variant, **incentive_value**, status da Amplitude key.

---

## Deploy

Pushes pra `main` no GitHub disparam deploy automático na Vercel.

```bash
cd incentivos-experiments
git add -A
git commit -m "<mensagem>"
git push
```

Vercel reconhece como projeto estático (Framework Preset: **Other**, build command vazio, output directory vazio).

---

## Checklist pré-go-live

- [x] `AMPLITUDE_API_KEY` preenchida em todos os HTMLs
- [x] 5 variantes geradas (breakage-5/10/15/20/25.html)
- [x] 5 heros copiados (reward-card-N.svg)
- [x] `sucesso.html` propaga `incentive_value` em todos os eventos via `?valor=`
- [x] Todos os deeplinks de `DEEPLINKS` (sucesso.html) configurados: `mercado`, `frete_gratis`, `extrato`
- [x] "Usar presente depois" usa `RedirectToDeepLinkInternal` → `ifoodbenefits://inapp/home` (não desloga no Android)
- [ ] Debug modal não atrapalha (acessar sem `?debug=1` confirma que está escondido)
- [ ] Testado no app Android (`?debug=1` pra ver canais Flutter)
- [ ] Testado no app iOS (`?debug=1`)
- [ ] 5 URLs configuradas no backoffice/Braze (`rawRoute?path=/webView?withAppBar=false&url=https://<projeto>.vercel.app/breakage-{N}`)
- [ ] Time de produto avisado do go-live, splits e sample size combinados

---

## Decisões registradas

- **Stack:** HTML standalone vanilla — sem build, peso mínimo, TTI rápido na WebView
- **Multi-variante:** 5 HTMLs separados em vez de 1 paramétrico — diff trivial entre variantes, deploy idempotente, debug por URL direto.
- **Property numérica:** `incentive_value` é número (não string `'5'`/`'10'`) — permite média, soma e regressão direta no Amplitude.
- **`sucesso.html` compartilhada:** o valor chega via querystring (`?valor=`), validado contra `ALLOWED_VALUES`. Página é única em vez de 5 cópias.
- **Hero:** Reward Card SVG (do Figma) preservado como está, sem adaptar pro hero padrão do playbook. 1 por variante.
- **Cores:** gradiente `#8f0340 → #69022d` e primário `#a41d50` do protótipo.
- **Ícones info cards:** SVGs inline (círculo magenta `#B5004C` + glifo branco) — sem dependência de URL externa.
- **CTA:** só primário ("Quero meu presente") + navegação simples pra `/sucesso?valor=N`. Sem CTA secundário, sem pesquisa positiva/negativa.
- **Seções:** mantém só o que existia no protótipo (3 info cards + 3 FAQ). Sem social proof, bullets de valor, depoimentos, disclaimer.
- **Navegação pra home:** `RedirectToDeepLinkInternal` com `ifoodbenefits://inapp/home` (NÃO usar `RedirectToRoute '/'` — desloga no Android).
