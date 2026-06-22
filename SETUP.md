# Binance Portfolio — Guia de Configuração

Você vai receber **2 arquivos**: `index.html` (o app) e este guia.  
Tudo que precisa fazer está aqui — SQL, código da função e passo a passo.  
Tempo estimado: **20–30 minutos**.

---

## O que você vai precisar

- Conta na Binance
- Conta no Supabase (grátis) — [supabase.com](https://supabase.com)
- Conta no GitHub (grátis) — [github.com](https://github.com)
- Conta no Netlify (grátis) — [netlify.com](https://netlify.com)

---

## Passo 1 — Criar projeto no Supabase

1. Acesse [supabase.com](https://supabase.com) → **Start your project**
2. Faça login com GitHub
3. Clique em **New Project** e preencha:
   - **Name:** `binance-portfolio`
   - **Database Password:** anote essa senha
   - **Region:** South America (São Paulo)
4. Clique em **Create new project** e aguarde ~2 minutos

---

## Passo 2 — Criar a tabela no banco de dados

1. No menu lateral, clique em **SQL Editor**
2. Clique em **New query**
3. Cole o bloco abaixo e clique em **Run** (▶):

```sql
-- Tabela principal de trades
CREATE TABLE IF NOT EXISTS binance_trades (
  id               BIGSERIAL PRIMARY KEY,
  user_id          UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  binance_trade_id TEXT,
  traded_at        TIMESTAMPTZ NOT NULL,
  pair             TEXT NOT NULL,
  base_asset       TEXT NOT NULL,
  quote_asset      TEXT NOT NULL,
  side             TEXT NOT NULL CHECK (side IN ('BUY', 'SELL')),
  price            NUMERIC NOT NULL,
  quantity         NUMERIC NOT NULL,
  total            NUMERIC NOT NULL,
  fee_amount       NUMERIC,
  fee_asset        TEXT,
  created_at       TIMESTAMPTZ DEFAULT NOW()
);

-- Índices para consultas rápidas
CREATE INDEX IF NOT EXISTS idx_bt_user
  ON binance_trades(user_id);
CREATE INDEX IF NOT EXISTS idx_bt_user_date
  ON binance_trades(user_id, traded_at DESC);

-- Unicidade para sincronização via API Binance (evita duplicatas)
CREATE UNIQUE INDEX IF NOT EXISTS uq_bt_api
  ON binance_trades(user_id, pair, binance_trade_id)
  WHERE binance_trade_id IS NOT NULL;

-- Unicidade para importação via CSV (evita duplicatas)
CREATE UNIQUE INDEX IF NOT EXISTS uq_bt_csv
  ON binance_trades(user_id, traded_at, pair, side, price, quantity);

-- Segurança: cada usuário acessa apenas os próprios trades
ALTER TABLE binance_trades ENABLE ROW LEVEL SECURITY;

CREATE POLICY "acesso proprio"
  ON binance_trades FOR ALL
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);
```

Deve aparecer **"Success. No rows returned"** — está correto.

---

## Passo 3 — Anotar as credenciais do Supabase

1. Clique em **Project Settings** (ícone de engrenagem no menu lateral)
2. Clique em **API**
3. Anote os dois valores:
   - **Project URL** — algo como `https://xyzxyz.supabase.co`
   - **anon / public** — string longa começando com `eyJ...`

Você vai usar esses valores no **Passo 6**.

---

## Passo 4 — Criar chave API na Binance (somente leitura)

> ⚠️ Esta chave **não permite** saques nem negociações. Serve apenas para ler o histórico.

1. Acesse [binance.com](https://binance.com) e faça login
2. Ícone do perfil → **Gerenciamento de API**
3. Clique em **Criar API** → **Chave de API gerada pelo sistema**
4. Dê um nome (ex: `portfolio-app`) e confirme com 2FA
5. Nas permissões marque **apenas**:
   - ✅ Leitura de informações
   - ❌ Negociação — NÃO marcar
   - ❌ Saques — NÃO marcar
6. Clique em **Salvar** e copie:
   - **API Key**
   - **Secret Key** — aparece **só uma vez**, copie agora

---

## Passo 5 — Configurar a Edge Function no Supabase

A Edge Function é o código que busca seus trades na Binance com segurança.  
Suas chaves Binance **nunca ficam expostas no browser**.

### 5.1 — Adicionar as chaves da Binance

1. No Supabase, vá em **Project Settings** → **Edge Functions**
2. Clique em **Edit secrets**
3. Adicione apenas estas **2 variáveis**:

| Nome | Valor |
|------|-------|
| `BINANCE_API_KEY` | Sua API Key da Binance |
| `BINANCE_SECRET_KEY` | Sua Secret Key da Binance |

> As credenciais do Supabase (`SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_ANON_KEY`) são injetadas automaticamente — não precisa adicionar.

4. Clique em **Save**

### 5.2 — Criar e publicar a função

1. No menu lateral, clique em **Edge Functions**
2. Clique em **Create a new function**
3. Nome da função: `sync-trades` (exatamente assim, sem maiúsculas)
4. Apague todo o código que aparecer e cole o código abaixo:

```typescript
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

// Lista de pares para buscar histórico de moedas já vendidas.
// Adicione aqui os pares das moedas que você já negociou e não tem mais em saldo.
// Exemplo: se você já vendeu toda a sua ETH, adicione 'ETHUSDT' nesta lista.
const PAIRS = [
  'BTCUSDT','BTCBRL',
  'SOLUSDT',
  'DOGEUSDT','DOGEBRL',
]

async function hmacSign(secret: string, message: string): Promise<string> {
  const enc = new TextEncoder()
  const key = await crypto.subtle.importKey(
    'raw', enc.encode(secret), { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']
  )
  const sig = await crypto.subtle.sign('HMAC', key, enc.encode(message))
  return Array.from(new Uint8Array(sig)).map(b => b.toString(16).padStart(2,'0')).join('')
}

function extractAssets(symbol: string): { base: string; quote: string } {
  const quotes = ['USDT','BUSD','BRL','BTC','BNB','ETH','USDC']
  for (const q of quotes) {
    if (symbol.endsWith(q)) return { base: symbol.slice(0, -q.length), quote: q }
  }
  return { base: symbol, quote: '' }
}

Deno.serve(async (req) => {
  if (req.method === 'OPTIONS') {
    return new Response(null, { headers: { 'Access-Control-Allow-Origin': '*', 'Access-Control-Allow-Headers': 'authorization, content-type' } })
  }

  const apiKey    = Deno.env.get('BINANCE_API_KEY')    ?? ''
  const apiSecret = Deno.env.get('BINANCE_SECRET_KEY') ?? ''
  const supaUrl   = Deno.env.get('SUPABASE_URL')       ?? ''
  const supaKey   = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY') ?? ''

  const authHeader = req.headers.get('Authorization')
  if (!authHeader) return new Response('Unauthorized', { status: 401 })

  const db = createClient(supaUrl, supaKey)
  const userDb = createClient(supaUrl, Deno.env.get('SUPABASE_ANON_KEY') ?? '', {
    global: { headers: { Authorization: authHeader } }
  })
  const { data: { user } } = await userDb.auth.getUser()
  if (!user) return new Response('Unauthorized', { status: 401 })

  // Saldo real da conta
  const balances: Record<string, number> = {}
  try {
    const ts = Date.now()
    const p = `timestamp=${ts}`
    const s = await hmacSign(apiSecret, p)
    const accRes = await fetch(`https://api.binance.com/api/v3/account?${p}&signature=${s}`, {
      headers: { 'X-MBX-APIKEY': apiKey }
    })
    if (accRes.ok) {
      const acc = await accRes.json()
      for (const b of (acc.balances || [])) {
        const free = parseFloat(b.free) + parseFloat(b.locked)
        if (free > 0) balances[b.asset] = free
      }
    }
  } catch {}

  // Ordens abertas
  let openOrders: Array<Record<string, unknown>> = []
  try {
    const ts = Date.now()
    const p = `timestamp=${ts}`
    const s = await hmacSign(apiSecret, p)
    const ooRes = await fetch(`https://api.binance.com/api/v3/openOrders?${p}&signature=${s}`, {
      headers: { 'X-MBX-APIKEY': apiKey }
    })
    if (ooRes.ok) {
      const raw = await ooRes.json()
      openOrders = (Array.isArray(raw) ? raw : []).map((o: any) => ({
        symbol:       o.symbol,
        side:         o.side,
        type:         o.type,
        price:        parseFloat(o.price),
        stop_price:   parseFloat(o.stopPrice),
        orig_qty:     parseFloat(o.origQty),
        executed_qty: parseFloat(o.executedQty),
        status:       o.status,
        time:         o.time,
      }))
    }
  } catch {}

  // Descoberta automática dos pares baseado no saldo atual
  const validSymbols = new Set<string>()
  try {
    const exRes = await fetch('https://api.binance.com/api/v3/exchangeInfo')
    if (exRes.ok) {
      const ex = await exRes.json()
      for (const s of (ex.symbols || [])) {
        if (s.status === 'TRADING') validSymbols.add(s.symbol)
      }
    }
  } catch {}

  const QUOTES = ['USDT','BRL','BTC','BUSD','USDC']
  const QUOTE_ONLY = new Set(['USDT','BRL'])
  const pairSet = new Set<string>(PAIRS)
  for (const asset of Object.keys(balances)) {
    if (QUOTE_ONLY.has(asset)) continue
    for (const q of QUOTES) {
      if (asset === q) continue
      const sym = asset + q
      if (validSymbols.size === 0 || validSymbols.has(sym)) pairSet.add(sym)
    }
  }

  const results: Record<string, number> = {}
  let totalInserted = 0

  for (const symbol of [...pairSet]) {
    try {
      const timestamp = Date.now()
      const params = `symbol=${symbol}&limit=1000&timestamp=${timestamp}`
      const sig = await hmacSign(apiSecret, params)
      const url = `https://api.binance.com/api/v3/myTrades?${params}&signature=${sig}`
      const res = await fetch(url, { headers: { 'X-MBX-APIKEY': apiKey } })
      if (!res.ok) { results[symbol] = 0; continue }
      const trades = await res.json()
      if (!Array.isArray(trades) || trades.length === 0) { results[symbol] = 0; continue }
      const { base, quote } = extractAssets(symbol)
      const rows = trades.map((t: any) => ({
        user_id:          user.id,
        binance_trade_id: String(t.id),
        traded_at:        new Date(t.time).toISOString(),
        pair:             symbol,
        base_asset:       base,
        quote_asset:      quote,
        side:             t.isBuyer ? 'BUY' : 'SELL',
        price:            parseFloat(t.price),
        quantity:         parseFloat(t.qty),
        total:            parseFloat(t.quoteQty),
        fee_amount:       parseFloat(t.commission),
        fee_asset:        t.commissionAsset,
      }))
      const { error } = await db.from('binance_trades').upsert(rows, {
        onConflict: 'user_id,pair,binance_trade_id',
        ignoreDuplicates: true,
      })
      results[symbol] = error ? 0 : trades.length
      totalInserted += results[symbol]
    } catch {
      results[symbol] = -1
    }
  }

  return new Response(JSON.stringify({ ok: true, totalInserted, results, balances, openOrders }), {
    headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' }
  })
})
```

5. Clique em **Deploy**

> **Sobre a lista `PAIRS`:** o app descobre automaticamente os pares das moedas que você tem em saldo. A lista `PAIRS` no topo do código serve só para moedas que você **já vendeu totalmente** e quer ver o histórico. Edite conforme suas moedas.

---

## Passo 6 — Editar e hospedar o app

### 6.1 — Criar repositório no GitHub

1. Acesse [github.com](https://github.com) → **New repository**
2. Nome: `binance-portfolio`
3. Visibilidade: **Private**
4. Clique em **Create repository**
5. Clique em **uploading an existing file**
6. Arraste o arquivo `index.html` e clique em **Commit changes**

### 6.2 — Editar as 2 linhas de configuração

1. No repositório, clique no arquivo `index.html`
2. Clique no ícone de lápis ✏️ para editar
3. Use `Ctrl+F` e busque por `SUPABASE_URL`
4. Substitua as **duas linhas** com seus valores do Passo 3:

```javascript
const SUPABASE_URL = 'https://SEU-PROJETO.supabase.co'
const SUPABASE_ANON_KEY = 'sua-anon-key-aqui'
```

5. Clique em **Commit changes** → **Commit changes**

### 6.3 — Hospedar no Netlify

1. Acesse [netlify.com](https://netlify.com) → login com GitHub
2. **Add new site** → **Import an existing project** → **GitHub**
3. Selecione o repositório `binance-portfolio`
4. Configurações de build:
   - **Build command:** deixe **em branco**
   - **Publish directory:** `.` (apenas um ponto)
5. Clique em **Deploy site**
6. Em ~1 minuto aparece o link, ex: `https://meu-portfolio.netlify.app`

---

## Passo 7 — Criar conta e sincronizar

1. Abra o link do Netlify no celular ou computador
2. Na tela de login, clique em **Criar conta**
3. Use seu e-mail e crie uma senha
4. **Verifique seu e-mail** — o Supabase envia um link de confirmação, clique nele antes de logar
5. Faça login com e-mail e senha
6. Clique em **Sincronizar** para importar seus trades da Binance
7. Aguarde 10–30 segundos — seus trades vão aparecer no gráfico

---

## Adicionar na tela inicial do celular

**iPhone (Safari):**  
Toque no ícone compartilhar → **Adicionar à Tela de Início** → **Adicionar**

**Android (Chrome):**  
Toque no menu ⋮ → **Adicionar à tela inicial** → **Adicionar**

O app abre em tela cheia, sem barra do navegador — igual a um app nativo.

---

## Dúvidas frequentes

**Minha chave Binance é segura?**  
Sim. A chave fica apenas nas variáveis de ambiente do Supabase, nunca no browser. Tem permissão somente leitura — impossível sacar ou negociar com ela.

**Não recebi o e-mail de confirmação do Supabase.**  
Verifique a caixa de spam. Se não chegar, acesse o Supabase → **Authentication** → **Users**, localize seu e-mail e clique em **Send confirmation email**.

**Como importar trades antigos de moedas já vendidas?**  
Na aba **Trades** do app há a opção de importar CSV exportado diretamente da Binance (Carteira → Histórico de negociações → Exportar CSV).

**Posso usar no celular e no computador ao mesmo tempo?**  
Sim. É um site — abre em qualquer dispositivo com o mesmo login.

**E se eu trocar de celular?**  
Basta abrir o link e fazer login. Todos os dados ficam no Supabase.
