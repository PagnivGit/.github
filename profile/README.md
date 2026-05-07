<p align="center">
  <img src="https://pagniv.com/logo.svg" alt="Pagniv" height="60" />
</p>

<h1 align="center">Pagniv</h1>

<p align="center">
  <strong>Infraestrutura Pix para desenvolvedores.</strong><br/>
  API simples, QR Code em segundos, liquidação automática.
</p>

<p align="center">
  <a href="https://pagniv.com">Site</a> ·
  <a href="https://devs.pagniv.com">Documentação completa</a> ·
  <a href="https://portal.pagniv.com">Dashboard</a>
</p>

---

> Este README cobre o **mínimo necessário** para integrar a API.
> Para detalhes, exemplos completos e referência de cada endpoint, consulte [devs.pagniv.com](https://devs.pagniv.com).

---

## Quick Start

### 1. Crie sua conta

Acesse [portal.pagniv.com](https://portal.pagniv.com) e cadastre sua empresa.

### 2. Obtenha sua API Key

No dashboard, vá em **Integrações → Chaves de API** e gere uma chave:

```
sk_sandbox_abc123...   # Sandbox (testes - sem transação real)
sk_live_abc789...      # Produção (transações reais)
```

### 3. Crie sua primeira cobrança

```bash
curl -X POST https://api.pagniv.com/v1/charges \
  -H "X-API-Key: sk_sandbox_abc123..." \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 15000,
    "description": "Pedido #1234",
    "externalId": "pedido-1234",
    "expiresIn": 3600
  }'
```

**Resposta:**

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "txid": "pagniv_abc123def456",
    "amount": 15000,
    "feeAmount": 99,
    "netAmount": 14901,
    "status": "PENDING",
    "qrCode": "00020126580014br.gov.bcb.pix...",
    "qrCodeBase64": "data:image/png;base64,...",
    "expiresAt": "2026-05-04T12:00:00Z",
    "createdAt": "2026-05-04T11:00:00Z"
  }
}
```

> Valores em **centavos**. R$ 150,00 = `15000`. Mínimo: `100` (R$ 1,00).

---

## Autenticação

Envie o header `X-API-Key` em toda requisição. Base URL: `https://api.pagniv.com/v1`.

> ⚠️ **Esta API é server-to-server.** Nunca exponha sua API Key em frontend (browser, mobile app, JS público). Quem tem a chave faz qualquer operação na sua conta. Chamadas devem partir do seu backend.

| Ambiente | Prefixo | Comportamento |
|---|---|---|
| Sandbox | `sk_sandbox_` | Testes - sem transações reais |
| Produção | `sk_live_` | Transações reais via Pix |

---

## Idempotência

Para evitar duplicidade em retries de rede ou timeout, envie o header `Idempotency-Key` (UUID v4 recomendado):

```bash
curl -X POST https://api.pagniv.com/v1/charges \
  -H "X-API-Key: sk_live_..." \
  -H "Idempotency-Key: 9b1d3e7c-5f8a-4d2e-9b1d-3e7c5f8a4d2e" \
  -H "Content-Type: application/json" \
  -d '{ "amount": 15000 }'
```

A mesma chave reusada em até 24h retorna a resposta original sem reprocessar. Recomendado em `POST /charges` e `POST /withdrawals`.

> `Idempotency-Key` (header) protege contra **retry de rede**.
> `externalId` (body) é o **identificador do seu pedido** no seu sistema.
> Pra integrações robustas, use os dois.

---

## Cobranças

Base path: `/v1/charges`

| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/v1/charges` | Criar cobrança Pix |
| `GET` | `/v1/charges` | Listar (com filtros e paginação) |
| `GET` | `/v1/charges/:id` | Detalhar (use como fallback do webhook) |
| `GET` | `/v1/charges/:id/checkout` | Dados públicos do checkout (sem API Key) |
| `DELETE` | `/v1/charges/:id` | Cancelar cobrança PENDING |
| `POST` | `/v1/charges/:id/refund` | Estornar cobrança paga |
| `POST` | `/v1/charges/:id/simulate-payment` | Simular pagamento (apenas sandbox) |

### Criar

```typescript
const res = await fetch('https://api.pagniv.com/v1/charges', {
  method: 'POST',
  headers: {
    'X-API-Key': process.env.PAGNIV_API_KEY,
    'Idempotency-Key': crypto.randomUUID(),
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    amount: 15000,            // R$ 150,00 em centavos
    description: 'Pedido #1234',
    externalId: 'pedido-1234',
    expiresIn: 3600,          // 1 hora
    payerName: 'João Silva',
    payerEmail: 'joao@email.com',
    payerDocument: '12345678900',
  }),
})

const { data } = await res.json()
console.log(data.qrCode)        // Pix copia e cola
console.log(data.qrCodeBase64)  // Imagem base64
```

### Parâmetros

| Campo | Tipo | Obrigatório | Descrição |
|---|---|:-:|---|
| `amount` | `number` | Sim | Valor em centavos (mín: 100) |
| `description` | `string` | Não | Descrição |
| `externalId` | `string` | Não | ID do pedido no seu sistema |
| `expiresIn` | `number` | Não | Expiração em segundos (padrão: 3600) |
| `payerName` | `string` | Não | Nome do pagador |
| `payerEmail` | `string` | Não | Email do pagador |
| `payerDocument` | `string` | Não | CPF/CNPJ do pagador |

### Status

| Status | Descrição |
|---|---|
| `PENDING` | Aguardando pagamento |
| `PAID` | Pagamento confirmado |
| `EXPIRED` | Tempo de expiração atingido |
| `CANCELLED` | Cancelada via API |
| `REFUNDED` | Estornada |
| `DISPUTED` | Em contestação |

### Estornar

```bash
curl -X POST https://api.pagniv.com/v1/charges/{id}/refund \
  -H "X-API-Key: sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": 5000, "reason": "Cliente solicitou cancelamento" }'
```

`amount` é opcional (default = líquido total). Estorna para a origem via Pix e debita o saldo disponível.

---

## Webhooks

Receba notificações HTTP quando eventos ocorrem.

### Eventos

| Evento | Descrição |
|---|---|
| `charge.paid` | Cobrança paga |
| `charge.expired` | Expirada sem pagamento |
| `charge.refunded` | Estornada |
| `dispute.opened` | Nova disputa |
| `dispute.resolved` | Disputa resolvida |
| `withdrawal.completed` | Saque processado pelo provedor |
| `withdrawal.failed` | Saque falhou no provedor (saldo devolvido) |
| `withdrawal.rejected` | Saque rejeitado pelo admin (saldo devolvido) |

### Configurar

```bash
curl -X POST https://api.pagniv.com/v1/webhook-config \
  -H "X-API-Key: sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://seusite.com/webhooks/pagniv",
    "events": ["charge.paid", "charge.refunded"]
  }'
```

A resposta inclui um `secret` - guarde para verificar a assinatura.

### Payloads recebidos

Todo evento usa o mesmo envelope (`event`, `data`, `timestamp`). O conteúdo de `data` varia conforme o evento.

**`charge.paid`**

```json
{
  "event": "charge.paid",
  "data": {
    "id": "550e8400-...",
    "txid": "pagniv_abc123def456",
    "amount": 15000,
    "netAmount": 14901,
    "feeAmount": 99,
    "status": "PAID",
    "paidAt": "2026-05-04T11:05:00Z",
    "externalId": "pedido-1234"
  },
  "timestamp": "2026-05-04T11:05:01Z"
}
```

**`charge.refunded`**

```json
{
  "event": "charge.refunded",
  "data": {
    "id": "550e8400-...",
    "txid": "pagniv_abc123def456",
    "amount": 15000,
    "netAmount": 14901,
    "feeAmount": 99,
    "status": "REFUNDED",
    "paidAt": "2026-05-04T11:05:00Z",
    "refundedAt": "2026-05-05T09:32:10Z",
    "refundedAmount": 14901,
    "refundReason": "Solicitação do cliente",
    "externalId": "pedido-1234"
  },
  "timestamp": "2026-05-05T09:32:11Z"
}
```

**`charge.expired`**

```json
{
  "event": "charge.expired",
  "data": {
    "id": "550e8400-...",
    "txid": "pagniv_abc123def456",
    "amount": 15000,
    "netAmount": 14901,
    "feeAmount": 99,
    "status": "EXPIRED",
    "paidAt": null,
    "externalId": "pedido-1234"
  },
  "timestamp": "2026-05-04T12:00:01Z"
}
```

**`dispute.opened`**

```json
{
  "event": "dispute.opened",
  "data": {
    "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "chargeId": "550e8400-...",
    "amount": 14901,
    "reason": "FRAUD",
    "description": "Não reconheço esta transação",
    "status": "OPEN",
    "deadline": "2026-05-11T11:05:00Z",
    "createdAt": "2026-05-04T14:22:00Z"
  },
  "timestamp": "2026-05-04T14:22:01Z"
}
```

**`dispute.resolved`**

```json
{
  "event": "dispute.resolved",
  "data": {
    "id": "7c9e6679-...",
    "chargeId": "550e8400-...",
    "amount": 14901,
    "reason": "FRAUD",
    "description": "Não reconheço esta transação",
    "status": "RESOLVED_MERCHANT",
    "deadline": "2026-05-11T11:05:00Z",
    "createdAt": "2026-05-04T14:22:00Z",
    "resolvedAt": "2026-05-07T10:11:00Z",
    "resolution": "Evidência aceita; merchant venceu."
  },
  "timestamp": "2026-05-07T10:11:01Z"
}
```

**`withdrawal.completed`**

```json
{
  "event": "withdrawal.completed",
  "data": {
    "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "amount": 50000,
    "fee": 200,
    "netAmount": 49800,
    "pixKey": "12345678900",
    "pixKeyType": "CPF",
    "status": "COMPLETED",
    "provider": "starpago",
    "providerTxId": "E60746948202605041130ABC123",
    "createdAt": "2026-05-04T11:30:00Z"
  },
  "timestamp": "2026-05-04T11:30:15Z"
}
```

**`withdrawal.failed`**

```json
{
  "event": "withdrawal.failed",
  "data": {
    "id": "f47ac10b-...",
    "amount": 50000,
    "fee": 200,
    "netAmount": 49800,
    "pixKey": "12345678900",
    "pixKeyType": "CPF",
    "status": "FAILED",
    "provider": "starpago",
    "providerTxId": null,
    "createdAt": "2026-05-04T11:30:00Z"
  },
  "timestamp": "2026-05-04T11:30:20Z"
}
```

**`withdrawal.rejected`**

```json
{
  "event": "withdrawal.rejected",
  "data": {
    "id": "f47ac10b-...",
    "amount": 50000,
    "fee": 200,
    "netAmount": 49800,
    "pixKey": "12345678900",
    "pixKeyType": "CPF",
    "status": "REJECTED",
    "provider": null,
    "providerTxId": null,
    "createdAt": "2026-05-04T11:30:00Z",
    "rejectedAt": "2026-05-04T13:00:00Z",
    "rejectedReason": "Chave Pix divergente do cadastro"
  },
  "timestamp": "2026-05-04T13:00:01Z"
}
```

### Verificar assinatura HMAC-SHA256

Use o **rawBody** (string original do request), não `JSON.stringify(req.body)`. Diferença de espaçamento ou ordem de chaves invalida a assinatura.

```typescript
import crypto from 'crypto'
import express from 'express'

const app = express()
// Captura o body cru pra HMAC
app.use('/webhooks/pagniv', express.raw({ type: 'application/json' }))

function verifyWebhook(rawBody: Buffer, signature: string, secret: string): boolean {
  const received = signature.replace('sha256=', '')
  const expected = crypto.createHmac('sha256', secret).update(rawBody).digest('hex')
  return crypto.timingSafeEqual(Buffer.from(received), Buffer.from(expected))
}

app.post('/webhooks/pagniv', (req, res) => {
  const signature = req.headers['x-webhook-signature'] as string
  const isValid = verifyWebhook(req.body, signature, process.env.WEBHOOK_SECRET!)
  if (!isValid) return res.status(401).send('Invalid signature')

  const payload = JSON.parse(req.body.toString())
  switch (payload.event) {
    case 'charge.paid':           /* marca pedido como pago */ break
    case 'charge.refunded':       /* trata estorno */ break
    case 'charge.expired':        /* libera estoque, etc */ break
    case 'dispute.opened':        /* avisa time de risco */ break
    case 'dispute.resolved':      /* atualiza status interno */ break
    case 'withdrawal.completed':  /* baixa o saque internamente */ break
    case 'withdrawal.failed':     /* notifica que saldo voltou pro merchant */ break
    case 'withdrawal.rejected':   /* idem, com rejectedReason no payload */ break
  }

  res.status(200).send('OK')
})
```

> Use webhook como fonte primária. Se sua aplicação não receber o evento em tempo razoável, faça polling em `GET /v1/charges/:id` como fallback.

---

## Sandbox: como testar

1. Use uma chave `sk_sandbox_*`
2. Crie uma cobrança normalmente (`POST /charges`) - recebe QR Code de teste
3. Configure seu webhook (opcional, para validar handler end-to-end)
4. Simule o pagamento:

```bash
curl -X POST https://api.pagniv.com/v1/charges/{id}/simulate-payment \
  -H "X-API-Key: sk_sandbox_..."
```

A cobrança vira `PAID` e o webhook `charge.paid` é disparado para o endpoint configurado. **Sem custo, sem dinheiro real.**

---

## Saldo e Saques

### Consultar saldo

```bash
curl https://api.pagniv.com/v1/balance \
  -H "X-API-Key: sk_live_..."
```

```json
{
  "success": true,
  "data": {
    "availableBalance": 125050,
    "pendingBalance": 30000,
    "reservedBalance": 15000,
    "blockedBalance": 0
  }
}
```

| Campo | Descrição |
|---|---|
| `availableBalance` | Disponível para saque |
| `pendingBalance` | Em liquidação (D+N) |
| `reservedBalance` | Reservado para disputas em aberto |
| `blockedBalance` | Bloqueado |

### Solicitar saque

```bash
curl -X POST https://api.pagniv.com/v1/withdrawals \
  -H "X-API-Key: sk_live_..." \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 50000,
    "pixKey": "empresa@email.com",
    "pixKeyType": "EMAIL"
  }'
```

| Campo | Tipo | Obrigatório | Descrição |
|---|---|:-:|---|
| `amount` | `number` | Sim | Valor em centavos |
| `pixKey` | `string` | Sim | Chave Pix destino |
| `pixKeyType` | `string` | Sim | `CPF`, `CNPJ`, `EMAIL`, `PHONE`, `EVP` |

**Status:** `PENDING` → `APPROVED` → `PROCESSING` → `COMPLETED` (ou `FAILED`/`REJECTED` em caso de erro - saldo é devolvido).

---

## Disputas

| Método | Endpoint | Descrição |
|---|---|---|
| `GET` | `/v1/disputes` | Listar |
| `GET` | `/v1/disputes/:id` | Detalhar |
| `POST` | `/v1/disputes/:id/accept` | Aceitar (reembolsa pagador) |
| `POST` | `/v1/disputes/:id/contest` | Contestar com texto + evidências |

```bash
curl -X POST https://api.pagniv.com/v1/disputes/{id}/contest \
  -H "X-API-Key: sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "response": "O produto foi entregue conforme combinado.",
    "evidenceUrls": ["https://exemplo.com/comprovante.pdf"]
  }'
```

**Status:** `OPEN` → `MERCHANT_RESPONDED` → `UNDER_REVIEW` → `RESOLVED_MERCHANT` ou `RESOLVED_BUYER` (ou `REFUNDED`).

---

## Formato de resposta

```json
// Sucesso
{ "success": true, "data": { ... } }

// Erro
{ "success": false, "error": { "code": "VALIDATION_ERROR", "message": "..." } }
```

### Códigos de erro comuns

| HTTP | Código | Quando |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Campo obrigatório faltando ou inválido |
| 400 | `INSUFFICIENT_BALANCE` | Saque ou estorno > saldo disponível |
| 400 | `INVALID_PIX_KEY` | Chave Pix em formato inválido |
| 400 | `NOT_SANDBOX_CHARGE` | `simulate-payment` em cobrança de produção |
| 401 | `UNAUTHORIZED` | API Key inválida, ausente ou revogada |
| 403 | `FORBIDDEN` | Sem permissão para o recurso |
| 403 | `ORG_BLOCKED` | Organização bloqueada |
| 404 | `NOT_FOUND` | Recurso não existe ou não é seu |
| 409 | `CHARGE_NOT_PENDING` | Operação inválida pro status atual da cobrança |
| 429 | `RATE_LIMITED` | Limite de requisições excedido |

### Paginação

Endpoints de listagem aceitam `?page=1&limit=20` (máx. 100). Resposta inclui `meta`:

```json
{
  "data": [...],
  "meta": { "total": 150, "page": 1, "limit": 20, "pages": 8 }
}
```

---

## Rate Limiting

| Endpoint | Limite |
|---|---|
| Geral | 100/minuto por API Key |
| `POST /charges` | 60/minuto |
| `POST /withdrawals` | 5/hora |

Headers retornados em toda requisição:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 97
X-RateLimit-Reset: 1714824000
```

---

## Recursos

- **Cobranças Pix** - QR Code dinâmico, copia e cola, confirmação instantânea via webhook
- **Webhooks** - HMAC-SHA256, retries automáticos
- **Estorno** - total ou parcial via API
- **Saques** - Pix para qualquer chave, liquidação automática
- **Disputas** - fluxo de contestação com evidências
- **Sandbox** - ambiente completo de testes com simulação de pagamento

---

## Descadastro de comunicações de marketing

Toda comunicação de marketing enviada pela Pagniv inclui um link de descadastro:

```
GET /v1/unsubscribe?token=<jwt>
```

Endpoint público que aceita um token JWT assinado, marca o destinatário (merchant ou lead) como descadastrado e retorna uma página HTML de confirmação. Tokens expiram em 90 dias mas persistem o estado de opt-out indefinidamente. Em compliance com a LGPD, o link sempre será incluído nos emails marketing.

---

## Links

| | |
|---|---|
| Site | [pagniv.com](https://pagniv.com) |
| Documentação completa | [devs.pagniv.com](https://devs.pagniv.com) |
| Dashboard | [portal.pagniv.com](https://portal.pagniv.com) |
| GitHub | [github.com/PagnivGit](https://github.com/PagnivGit) |

---

<p align="center">
  Feito pela equipe <strong>Pagniv</strong><br/>
  <a href="https://pagniv.com">pagniv.com</a>
</p>
