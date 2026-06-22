# Portfolio Binance — Documentação do Projeto

Aplicativo web para acompanhar seus trades de criptomoedas da Binance: saldo,
custo médio, lucro/prejuízo (P&L) e um gráfico de preço histórico com seus
pontos de compra e venda marcados.

> App no ar: **https://binance-portfolio-tau.vercel.app**
> Repositório: **goulartcus/binance-portfolio**

---

## 1. Visão geral

- **Frontend:** um único arquivo `index.html` (HTML + CSS + JavaScript puro, sem framework).
- **Backend:** [Supabase](https://supabase.com) — banco de dados (PostgreSQL), autenticação e uma *Edge Function*.
- **Hospedagem do site:** [Vercel](https://vercel.com) (deploy automático a cada `git push`).
- **Dados de mercado:** API pública da Binance (preços, histórico, WebSocket de preço ao vivo).
- **Dados da sua conta:** API privada da Binance (saldo e trades), acessada **somente** pelo servidor (Edge Function), nunca pelo navegador.

```
Navegador (index.html)
   │
   ├── Supabase Auth ............ login por e-mail/senha
   ├── Supabase DB (binance_trades) .. histórico de trades (com RLS por usuário)
   ├── Supabase Edge Function (sync-trades) .. puxa saldo + trades da Binance
   └── Binance API pública ...... preços e gráfico (klines + WebSocket)
```

---

## 2. Estrutura de arquivos

```
portfolio/
├── index.html                         ← o app inteiro (UI + lógica)
├── logo-btc.png                       ← logo do header (B branco, fundo transparente)
├── favicon.png                        ← ícone da aba do navegador (B branco em fundo escuro)
├── vercel.json                        ← configuração do Vercel (cabeçalho CSP)
├── portfolio_schema.sql               ← script SQL do banco (rodar no Supabase)
├── README.md                          ← este arquivo
└── supabase/
    └── functions/
        └── sync-trades/
            └── index.ts               ← Edge Function (Deno/TypeScript)
```

---

## 3. Como funciona o frontend (`index.html`)

### Login
- Tela de login usa `db.auth.signInWithPassword`.
- `onAuthStateChange` detecta sessão ativa e chama `startApp()`.
- Há um *fallback* com `getSession()` caso o evento não dispare.

### Carregamento dos dados (`startApp`)
1. `fetchUsdtBrl()` — pega a cotação USDT/BRL atual.
2. `loadTrades()` — lê a tabela `binance_trades` do usuário logado.
3. `loadBtcUsdHistory()` — baixa o histórico diário do BTC/USD (usado para converter compras antigas em BRL pelo preço da época).
4. `startBinanceWS()` — conecta no WebSocket da Binance para o preço do BTC ao vivo.
5. `buildCoinSelector()` + `initCharts()` — monta o seletor de moeda e o gráfico.
6. `autoSync()` — em segundo plano, chama a Edge Function para atualizar saldo e trades.

### Abas
- **Dashboard:** gráfico de preço + marcadores de trades, e cards de resumo da carteira.
- **Ativos:** um card por moeda (quantidade, custo médio, P&L realizado).
- **Trades:** tabela com todo o histórico, filtrável por par.
- **Ordens abertas:** ordens que ainda estão ativas na Binance (par, lado, tipo, preço, gatilho, quantidade). Atualizadas a cada sincronização.
- **Importar CSV:** importa o CSV exportado da Binance (ver seção 6).

### Cálculo do custo médio (importante)
O custo médio do BTC usa o **método da média de todas as compras**:

```
custo médio = (soma do valor de TODAS as compras) / (quantidade total comprada)
```

- Vendas **não** alteram o custo médio.
- Compras em BRL são convertidas para USD pelo preço do BTC/USD **da época** da compra.
- P&L realizado = valor recebido nas vendas − (custo médio × quantidade vendida).
- O saldo exibido usa o **saldo real da Binance** quando disponível; senão, calcula (comprado − vendido).

### Gráfico
- Mostra o preço histórico em **candles (velas verde/vermelho)** + seus trades marcados com
  círculos estilo Binance: **B** (compra, verde) e **S** (venda, vermelho).
- Três modos: **Diário** (histórico completo), **4H** (abre com ~1,5 mês de zoom) e
  **1H** (abre com ~1 semana de zoom); dá para arrastar para ver mais.
- Linhas da escala em valores redondos (para BTC, ~de 10k em 10k) e tooltip que
  some ao tirar o dedo da tela (no celular).
- As velas são desenhadas por um **plugin próprio em canvas** (sem dependência externa), e a
  **escala de preço (Y) se ajusta automaticamente** ao trecho de tempo visível — igual TradingView.
- Seletor de moeda em formato de **caixa de seleção (dropdown)**, com botão **Reset** do zoom ao lado.
- Zoom com scroll, arrastar para mover (plugin `chartjs-plugin-zoom`).
- **Resumo de ordens abaixo do gráfico:** chips com lado + preço das ordens abertas
  da moeda selecionada (sem quantidade). Para ordens stop, usa o preço de gatilho.
  Atualiza junto quando você troca a moeda no dropdown.

---

## 4. Edge Function `sync-trades` (Supabase)

Roda no servidor (Deno). É a única parte que acessa suas **chaves privadas** da Binance.
O navegador só chama essa função com o token de login; as chaves nunca ficam expostas.

### O que ela faz
1. Valida o usuário pelo token de autenticação.
2. **Saldo real:** chama `/api/v3/account` e guarda o saldo de **todas** as moedas com saldo > 0.
3. **Descoberta automática de pares** (implementado em jun/2026):
   - Baixa a lista de símbolos válidos da Binance (`/api/v3/exchangeInfo`).
   - Para cada moeda que você tem em saldo, monta os pares candidatos
     (`{MOEDA}USDT`, `{MOEDA}BRL`, `{MOEDA}BTC`, `{MOEDA}BUSD`, `{MOEDA}USDC`)
     e mantém só os que **existem de fato** na Binance.
   - Soma isso à lista BASE (`PAIRS`), que serve de rede de segurança para moedas
     que você já vendeu e não aparecem mais no saldo.
4. **Ordens abertas:** chama `/api/v3/openOrders` (uma vez, sem `symbol`) e traz
   todas as ordens ainda ativas na conta.
5. Para cada par, chama `/api/v3/myTrades` e grava os trades em `binance_trades`
   (com `upsert` ignorando duplicatas).
6. Retorna `{ ok, totalInserted, results, balances, openOrders }`.

> **Consequência prática:** qualquer moeda nova que você comprar é reconhecida
> automaticamente — aparece no gráfico, em Ativos e no resumo, **com histórico**,
> sem precisar editar código.

### Variáveis de ambiente (configuradas no painel do Supabase)
| Variável | Para que serve |
|---|---|
| `BINANCE_API_KEY` | Chave da API da Binance (somente leitura) |
| `BINANCE_SECRET_KEY` | Segredo da API da Binance |
| `SUPABASE_URL` | URL do projeto Supabase |
| `SUPABASE_SERVICE_ROLE_KEY` | Chave de serviço (grava no banco ignorando RLS) |
| `SUPABASE_ANON_KEY` | Chave pública (valida o usuário) |

---

## 5. Banco de dados (`portfolio_schema.sql`)

### Tabela `binance_trades`
| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | BIGSERIAL | chave primária |
| `user_id` | UUID | dono do trade (FK para `auth.users`) |
| `traded_at` | TIMESTAMPTZ | data/hora do trade |
| `pair` | TEXT | par (ex: `BTCUSDT`) |
| `base_asset` | TEXT | moeda comprada/vendida (ex: `BTC`) |
| `quote_asset` | TEXT | moeda de cotação (ex: `USDT`) |
| `side` | TEXT | `BUY` ou `SELL` |
| `price` | NUMERIC | preço unitário |
| `quantity` | NUMERIC | quantidade |
| `total` | NUMERIC | valor total |
| `fee_amount` | NUMERIC | taxa paga |
| `fee_asset` | TEXT | moeda da taxa |

- **Anti-duplicata:** índice `UNIQUE (user_id, traded_at, pair, side, price, quantity)`.
- **Segurança (RLS):** a política `own_trades` garante que cada usuário só vê e altera os próprios trades.
- Existe também a view `portfolio_positions` (posição agregada por ativo), disponível para consultas.

---

## 6. Como adicionar dados

Há dois caminhos para os trades entrarem no banco:

1. **Sincronizar (automático):** botão **⟳ Sincronizar** (ou a sincronização em
   segundo plano ao abrir o app). Com a descoberta automática de pares, cobre
   qualquer moeda que você tenha em saldo.
2. **Importar CSV (manual):** Binance → Carteira → Histórico de Trades → Exportar
   → CSV. Depois use a aba **Importar CSV**. Útil para trazer histórico antigo de
   moedas que você já vendeu (e que não aparecem mais no saldo).

A parte visual é **sempre automática**: qualquer moeda presente no banco aparece
sozinha no seletor do gráfico, na aba Ativos e no resumo — sem mexer em código.

---

## 7. Deploy / Publicação

### Frontend (site) — via Vercel
O Vercel republica **automaticamente** a cada `git push`. Fluxo padrão:

```powershell
cd "<pasta do projeto>\portfolio"
git add .
git commit -m "descrição da mudança"
git push
```

Depois, no navegador, **Ctrl + Shift + R** para recarregar sem cache.

### Edge Function — via painel do Supabase
A função roda no Supabase, **não** no Vercel. Não há CLI configurada aqui, então
o deploy é pelo painel web:

1. Acesse o painel do Supabase → seu projeto → **Edge Functions** → `sync-trades`.
2. Cole o conteúdo de `supabase/functions/sync-trades/index.ts`.
3. Clique em **Deploy**.

> O `git push` **não** atualiza a Edge Function — ela precisa desse deploy manual
> separado sempre que o `index.ts` mudar.

---

## 8. Segurança

- A `SUPABASE_ANON_KEY` fica no `index.html` — isso é **normal e seguro**, pois o
  RLS protege os dados (cada usuário só acessa o que é seu).
- As chaves da **Binance** e a `SERVICE_ROLE_KEY` ficam **apenas** nas variáveis de
  ambiente da Edge Function (servidor). Nunca aparecem no navegador.
- Recomendado: criar a chave da Binance **somente com permissão de leitura**
  (sem saque, sem trade).

---

## 9. Histórico de mudanças (jun/2026)

- Logo BTC adicionado ao header; fundo tornado 100% transparente (B branco limpo).
- Favicon criado (B branco em fundo escuro) e ligado no `<head>`.
- Botões de intervalo do gráfico (1H/4H/1D/1S) **removidos**; gráfico fixo no diário.
- Corrigido bug em que os botões de intervalo não funcionavam no celular
  (uso de `event.target`, substituído por passagem do elemento).
- Seletor de moedas trocado de grade de botões para **dropdown** (mais limpo no celular),
  com **Reset** ao lado e largura reduzida.
- Corrigida a legibilidade dos dropdowns no tema escuro (fundo escuro + texto claro).
- Cards de resumo da carteira passam a **abrir ocultos** por padrão.
- **Edge Function:** descoberta automática de pares — qualquer moeda em saldo é
  sincronizada com histórico, sem lista fixa.
- Nova aba **Ordens abertas** + leitura das ordens ativas via `/api/v3/openOrders`.
- Resumo de ordens abertas (lado + preço) logo **abaixo do gráfico**, filtrado pela
  moeda selecionada.
- Gráfico trocado de linha para **candles (velas)**; botões **Diário** e **1H**
  (1H abre com zoom de ~1 semana).
- Marcadores de trade estilo Binance: círculo **B** (compra) / **S** (venda).
- Ajustes de layout para **celular**: logo "Portfolio/Binance" empilhado, botão
  Sincronizar só com ícone, aba "Importar CSV" recolhida em ícone, abas roláveis,
  e chips de ordens só com bolinha + valor colorido.
- **Gráfico maximizado no celular:** altura solta (ocupa ~62% da tela),
  título removido, controles (moeda/Diário/1H/Reset) numa linha só, e escala de
  preço **abreviada** ($60k, $100k) e estreita, sem valores negativos.

---

## 10. Solução de problemas

| Sintoma | Causa provável / solução |
|---|---|
| Mudança no site não aparece | Cache do navegador → **Ctrl + Shift + R** ou janela anônima |
| Imagem/logo não atualiza | Vercel pode levar ~30s para republicar após o `push` |
| Moeda nova não aparece com histórico | Confirme que a Edge Function foi **re-deployada** no Supabase |
| "Sincronizar" dá erro | Verifique as variáveis de ambiente da Binance no Supabase |
| Saldo aparece mas sem custo médio | Moeda recebida sem trade (ex: airdrop), ou histórico só via CSV |
