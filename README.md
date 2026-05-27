# incentivos-experiments

LPs de discovery do iFood Benefícios para experimentos de **incentivo com breakage** (valores com prazo limitado pra uso no iFood).

Construídas seguindo o playbook em `../playbook_contrucao_lps_app.md`. Hospedagem na Vercel, abertas dentro da WebView do app via banner.

---

## Estrutura

```
incentivos-experiments/
├── breakage.html        # LP principal — apresenta a oferta (R$ 35, 4 dias)
├── sucesso.html         # Página pós-CTA — confirma liberação + 3 atalhos
├── reward-card.svg      # Hero do breakage (Figma export)
├── icon-check.svg       # Check verde da página de sucesso
├── success-icon-*.svg   # Ícones dos cards da página de sucesso
├── fonts/               # Tipo iFood Titulos + Textos (WOFF2)
├── vercel.json          # cleanUrls + rewrites + X-Frame-Options ALLOWALL
└── README.md
```

URLs públicas após deploy:

- `https://<projeto>.vercel.app/breakage`
- `https://<projeto>.vercel.app/sucesso`

URL pra colocar no banner do backoffice:

```
rawRoute?path=/webView?withAppBar=false&url=https://<projeto>.vercel.app/breakage
```

URL pra usar em push notification ou in-app message do Braze (setting "Deeplink into application"):

```
ifoodbenefits://inapp/webView?withAppBar=false&url=https://<projeto>.vercel.app/breakage
```

Observações sobre o deeplink:
- O prefixo `/inapp/` é obrigatório — sem ele o app cai na home
- `withAppBar=false` deixa a LP usar a tela inteira (sem barra nativa em cima)
- Mesmo formato funciona pra push e in-app
- Pra segmentar a origem no Amplitude, adicione `?source=push` ou `?source=inapp` no fim da `url` interna (vai parar em `event_properties.referrer` / podemos parsear como source)

---

## Fluxo do experimento

1. Usuário toca o banner na home do app
2. WebView abre `/breakage` → apresenta o presente de R$ 35 com prazo de 4 dias
3. Usuário toca **Quero meu presente** → tracking `lp_cta_click` + navega pra `/sucesso`
4. `/sucesso` confirma a liberação (check verde grande) e oferece 3 atalhos: mercado com desconto, restaurantes com frete grátis, ver extrato
5. Cada atalho dispara um deeplink configurado em `DEEPLINKS` (sucesso.html). `mercado` e `frete_gratis` já têm deeplinks `ifood://list/...`. `extrato` ainda está vazio — preencher quando soubermos a rota real.
6. Link **Usar presente depois** volta pra home do iFood Benefícios (canal `RedirectToRoute` com fallback `OnClose`).

---

## Tipografia

Usa **Tipo iFood** (Fabio Haag Type, proprietária — uso exclusivo iFood):

- `TipoiFood Titulos` (Bold 700, ExtraBold 800) — h1, h2, CTA, números
- `TipoiFood Textos` (Regular 400, Medium 500, Bold 700) — body, FAQ, info cards

Os 5 WOFF2 vivem em `/fonts/`. Não republicar em outro repo.

---

## Tracking (Amplitude)

A API key fica vazia em produção até alguém preencher `AMPLITUDE_API_KEY` no topo do `<script>` de cada HTML. Quando vazia, `trackAmplitude()` é no-op silencioso.

**Eventos disparados:**

| Página       | Event type                | Properties extras                       |
|--------------|---------------------------|-----------------------------------------|
| breakage     | `lp_page_view`            | `available_channels`                    |
| breakage     | `lp_faq_open`             | `question`                              |
| breakage     | `lp_cta_click`            | `cta_type=positive`, `cta_label`        |
| sucesso      | `sucesso_page_view`       | —                                       |
| sucesso      | `sucesso_option_click`    | `option` (mercado/frete_gratis/extrato), `channel`, `target`, `dispatched` |
| sucesso      | `sucesso_later_click`     | — (link "Usar presente depois")         |

Todos carregam: `variant=breakage`, `session_uuid`, `person_id` (se houver), `person_id_source`, `platform`, `page_url`, `referrer`.

---

## Identidade do usuário

Cascata: querystring `?person_id=` → JWT do cookie `aAccessToken` → `null`. Sempre tem `session_uuid` em `localStorage` como fallback estável por dispositivo.

Hoje, sem cooperação do backoffice/mobile, o `session_uuid` é o identificador confiável. Correlação com `personId` real fica post-hoc via time-window join com o evento `banner_click` (ver seção 9.4 do playbook).

---

## QA / debug

Modal de debug gateado por `?debug=1`:

```
https://<projeto>.vercel.app/breakage?debug=1
https://<projeto>.vercel.app/breakage?person_id=teste123&debug=1
```

Mostra: person_id (fonte), session_uuid, JWT claims, cookies, canais Flutter detectados, endereço, query params, URL completa, plataforma, variant, status da Amplitude key.

---

## Deploy

```bash
cd incentivos-experiments

# Init git (primeira vez)
git init
git config user.email "<id>+<usuario>@users.noreply.github.com"
git config user.name "<usuario>"
git add .
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/<usuario>/incentivos-experiments.git
git push -u origin main
```

Em vercel.com/dashboard → **Add New → Project** → importa o repo. Framework Preset: **Other**, build command vazio, output directory vazio. Deploy.

Pushes futuros pra `main` disparam deploy automático.

---

## Checklist pré-go-live

- [ ] `AMPLITUDE_API_KEY` preenchida em `breakage.html` e `sucesso.html`
- [ ] Target em `DEEPLINKS.extrato` (sucesso.html) preenchido com a rota real do extrato. `mercado` e `frete_gratis` já estão configurados.
- [ ] `HOME_ROUTE.target` em `sucesso.html` confirmado com mobile (rota nomeada da home do iFood Benefícios)
- [ ] Debug modal não atrapalha (acessar sem `?debug=1` confirma que está escondido)
- [ ] Testado no app Android (`?debug=1` pra ver canais Flutter)
- [ ] Testado no app iOS (`?debug=1`)
- [ ] URL configurada no backoffice no formato `rawRoute?path=/webView?withAppBar=false&url=...`
- [ ] Time de produto avisado do go-live, sample size combinado

---

## Decisões registradas

- **Stack:** HTML standalone vanilla — sem build, peso mínimo, TTI rápido na WebView
- **Hero:** Reward Card SVG (do Figma) preservado como está, sem adaptar pro hero padrão do playbook
- **Cores:** gradiente `#8f0340 → #69022d` e primário `#a41d50` do protótipo (não foi puxado pro `#A91046` do playbook)
- **Ícones info cards:** SVGs inline (círculo magenta `#B5004C` + glifo branco) — sem dependência de URL externa, design vindo do Figma
- **CTA:** só primário ("Quero meu presente") + navegação simples pra `/ativado`. Sem CTA secundário, sem pesquisa positiva/negativa.
- **Seções:** mantém só o que existia no protótipo (3 info cards + 3 FAQ). Sem social proof, bullets de valor, depoimentos, disclaimer.
