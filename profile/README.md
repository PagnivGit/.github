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
  <a href="https://devs.pagniv.com">Documentação</a> ·
  <a href="https://portal.pagniv.com">Dashboard</a>
</p>

---

## Sobre

A **Pagniv** é uma plataforma de subadquirência Pix que permite empresas receberem pagamentos instantâneos com total controle. Infraestrutura BaaS para PSPs, Fintechs, plataformas SaaS e iGaming.

---

## Quick Start

### 1. Crie sua conta

Acesse [portal.pagniv.com](https://portal.pagniv.com) e cadastre sua empresa.

### 2. Obtenha sua API Key

Após aprovação, gere sua chave no dashboard em **Integrações > Chaves de API**.

```
sk_sandbox_abc123def456...   # Sandbox (testes)
sk_live_abc789def012...      # Produção (transações reais)
```

### 3. Crie sua primeira cobrança

```bash
curl -X POST https://api.pagniv.com/v1/charges \
  -H "X-API-Key: sk_sandbox_abc123def456" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 15000,
    "description": "Pedido #1234",
    "payerName": "João Silva",
    "payerDocument": "12345678900",
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
    "expiresAt": "2026-01-15T12:00:00Z",
    "createdAt": "2026-01-15T11:00:00Z"
  }
}
```

> Valores em **centavos**. R$ 150,00 = `15000`.

---

## Autenticação

Todas as chamadas à API utilizam o header `X-API-Key`:

```
X-API-Key: sk_sandbox_sua_chave_aqui
```

| Ambiente | Prefixo | Descrição |
|----------|---------|-----------|
| Sandbox | `sk_sandbox_` | Testes sem transações reais |
| Produção | `sk_live_` | Transações reais via Pix |

---

## Endpoints

Base URL: `https://api.pagniv.com/v1`

### Cobranças

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `POST` | `/v1/charges` | Criar cobrança Pix |
| `GET` | `/v1/charges` | Listar cobranças |
| `GET` | `/v1/charges/:id` | Detalhes de uma cobrança |
| `DELETE` | `/v1/charges/:id` | Cancelar cobrança pendente |
| `GET` | `/v1/charges/:id/checkout` | Dados públicos do checkout |

### Carteira

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `GET` | `/v1/balance` | Consultar saldo |
| `GET` | `/v1/transactions` | Listar transações |
| `GET` | `/v1/statement` | Extrato por período |

### Saques

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `POST` | `/v1/withdrawals` | Solicitar saque |
| `GET` | `/v1/withdrawals` | Listar saques |
| `GET` | `/v1/withdrawals/:id` | Detalhes do saque |

### Depósito

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `POST` | `/v1/deposit` | Gerar QR Code para depósito |

### Disputas

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `GET` | `/v1/disputes` | Listar disputas |
| `GET` | `/v1/disputes/:id` | Detalhes da disputa |
| `POST` | `/v1/disputes/:id/accept` | Aceitar disputa |
| `POST` | `/v1/disputes/:id/contest` | Contestar disputa |

> Disputas são abertas pelo admin. O merchant pode aceitar ou contestar com texto de resposta e evidências.

### Webhooks

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `POST` | `/v1/webhook-config` | Cadastrar webhook |
| `GET` | `/v1/webhook-config` | Listar webhooks |
| `PUT` | `/v1/webhook-config/:id` | Atualizar webhook |
| `DELETE` | `/v1/webhook-config/:id` | Remover webhook |
| `POST` | `/v1/webhook-config/:id/test` | Testar webhook |
| `GET` | `/v1/webhook-deliveries` | Histórico de entregas |

### Chaves de API

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `POST` | `/v1/api-keys` | Criar chave |
| `GET` | `/v1/api-keys` | Listar chaves |
| `DELETE` | `/v1/api-keys/:id` | Revogar chave |

### Organizações

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `POST` | `/v1/organizations` | Criar organização |
| `GET` | `/v1/organizations` | Listar organizações |
| `GET` | `/v1/organizations/:id` | Detalhes da organização |
| `PATCH` | `/v1/organizations/checkout-settings` | Configurações de checkout |
| `POST` | `/v1/organizations/invite` | Convidar membro |
| `GET` | `/v1/organizations/members` | Listar membros |
| `PATCH` | `/v1/organizations/members/:userId` | Alterar role do membro |
| `DELETE` | `/v1/organizations/members/:userId` | Remover membro |

### Documentos (KYC)

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| `POST` | `/v1/documents/upload` | Upload de documento (multipart) |
| `GET` | `/v1/documents` | Listar documentos |
| `GET` | `/v1/documents/:id/file` | Download do documento |
| `DELETE` | `/v1/documents/:id` | Remover documento |

---

## Criar Cobrança

```typescript
const response = await fetch('https://api.pagniv.com/v1/charges', {
  method: 'POST',
  headers: {
    'X-API-Key': 'sk_sandbox_sua_chave',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    amount: 15000,            // R$ 150,00 em centavos
    description: 'Pedido #1234',
    payerName: 'João Silva',
    payerDocument: '12345678900',
    payerEmail: 'joao@email.com',
    externalId: 'pedido-1234', // Idempotência
    expiresIn: 3600            // 1 hora
  })
})

const { data } = await response.json()
console.log(data.qrCode)       // Copia e cola
console.log(data.qrCodeBase64) // Imagem base64
```

### Parâmetros

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `amount` | `number` | Sim | Valor em centavos (min: 100 = R$ 1,00) |
| `description` | `string` | Não | Descrição da cobrança |
| `externalId` | `string` | Não | ID externo para idempotência |
| `payerName` | `string` | Não | Nome do pagador |
| `payerEmail` | `string` | Não | Email do pagador |
| `payerDocument` | `string` | Não | CPF/CNPJ do pagador |
| `expiresIn` | `number` | Não | Expiração em segundos (padrão: 3600) |

### Status da Cobrança

| Status | Descrição |
|--------|-----------|
| `PENDING` | Aguardando pagamento |
| `PAID` | Pagamento confirmado |
| `EXPIRED` | Tempo de expiração atingido |
| `CANCELLED` | Cancelada pelo merchant |
| `REFUNDED` | Reembolsada via disputa |
| `DISPUTED` | Em contestação |

---

## Webhooks

Receba notificações em tempo real quando eventos ocorrem.

### Eventos disponíveis

| Evento | Descrição |
|--------|-----------|
| `charge.paid` | Cobrança paga com sucesso |
| `charge.expired` | Cobrança expirada sem pagamento |
| `charge.refunded` | Cobrança reembolsada |
| `dispute.opened` | Nova disputa aberta |
| `dispute.resolved` | Disputa resolvida |

### Configurar

```bash
curl -X POST https://api.pagniv.com/v1/webhook-config \
  -H "X-API-Key: sk_sandbox_sua_chave" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://seusite.com/webhook/pagniv",
    "events": ["charge.paid", "charge.expired"]
  }'
```

> O campo `events` é opcional. Se omitido, o webhook recebe todos os eventos. O `secret` é auto-gerado se não informado.

### Payload recebido

```json
{
  "event": "charge.paid",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "txid": "pagniv_abc123def456",
    "amount": 15000,
    "netAmount": 14901,
    "status": "PAID",
    "paidAt": "2026-01-15T11:05:00Z",
    "externalId": "pedido-1234"
  },
  "timestamp": "2026-01-15T11:05:01Z"
}
```

### Verificar assinatura

Toda entrega inclui o header `X-Webhook-Signature` com assinatura HMAC-SHA256:

```typescript
import crypto from 'crypto'

function verifyWebhook(payload: string, signature: string, secret: string): boolean {
  const expected = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex')

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  )
}

// No seu endpoint
app.post('/webhook/pagniv', (req, res) => {
  const signature = req.headers['x-webhook-signature'] as string
  const isValid = verifyWebhook(JSON.stringify(req.body), signature, WEBHOOK_SECRET)

  if (!isValid) {
    return res.status(401).send('Invalid signature')
  }

  const { event, data } = req.body

  if (event === 'charge.paid') {
    // Processar pagamento confirmado
    console.log(`Cobrança ${data.id} paga: R$ ${(data.amount / 100).toFixed(2)}`)
  }

  res.status(200).send('OK')
})
```

### Gerenciar webhooks

```bash
# Atualizar webhook
curl -X PUT https://api.pagniv.com/v1/webhook-config/{id} \
  -H "X-API-Key: sk_sandbox_sua_chave" \
  -H "Content-Type: application/json" \
  -d '{ "url": "https://seusite.com/webhook/v2", "events": ["charge.paid"] }'

# Testar webhook
curl -X POST https://api.pagniv.com/v1/webhook-config/{id}/test \
  -H "X-API-Key: sk_sandbox_sua_chave"

# Remover webhook
curl -X DELETE https://api.pagniv.com/v1/webhook-config/{id} \
  -H "X-API-Key: sk_sandbox_sua_chave"

# Histórico de entregas
curl https://api.pagniv.com/v1/webhook-deliveries \
  -H "X-API-Key: sk_sandbox_sua_chave"
```

---

## Consultar Saldo

```bash
curl https://api.pagniv.com/v1/balance \
  -H "X-API-Key: sk_sandbox_sua_chave"
```

```json
{
  "success": true,
  "data": {
    "availableBalance": 125050,
    "pendingBalance": 30000,
    "reservedBalance": 15000,
    "blockedBalance": 0,
    "totalEarned": 1580000,
    "totalWithdrawn": 1420000,
    "totalFeesPaid": 34950,
    "breakdown": {
      "PIX": { "availableBalance": 100050, "pendingBalance": 20000 },
      "CARD": { "availableBalance": 25000, "pendingBalance": 10000 },
      "BOLETO": { "availableBalance": 0, "pendingBalance": 0 }
    }
  }
}
```

| Campo | Descrição |
|-------|-----------|
| `availableBalance` | Disponível para saque |
| `pendingBalance` | Em liquidação (D+N) |
| `reservedBalance` | Reservado para disputas abertas |
| `blockedBalance` | Bloqueado pelo admin |
| `breakdown` | Saldo por método de pagamento (PIX, CARD, BOLETO) |

---

## Solicitar Saque

```bash
curl -X POST https://api.pagniv.com/v1/withdrawals \
  -H "X-API-Key: sk_sandbox_sua_chave" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 50000,
    "pixKey": "empresa@email.com",
    "pixKeyType": "EMAIL"
  }'
```

### Parâmetros

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|:-----------:|-----------|
| `amount` | `number` | Sim | Valor em centavos (min: 100 = R$ 1,00) |
| `pixKey` | `string` | Sim | Chave Pix destino |
| `pixKeyType` | `string` | Sim | `CPF`, `CNPJ`, `EMAIL`, `PHONE` ou `EVP` |

### Resposta

```json
{
  "success": true,
  "data": {
    "id": "550e8400-...",
    "amount": 50000,
    "fee": 200,
    "netAmount": 49800,
    "status": "PENDING",
    "pixKey": "empresa@email.com",
    "pixKeyType": "EMAIL",
    "createdAt": "2026-01-15T14:00:00Z"
  }
}
```

### Status do Saque

| Status | Descrição |
|--------|-----------|
| `PENDING` | Solicitado, aguardando aprovação |
| `APPROVED` | Aprovado, será processado |
| `PROCESSING` | Em processamento no provedor Pix |
| `COMPLETED` | Pix enviado com sucesso |
| `FAILED` | Falha no envio — saldo devolvido |
| `REJECTED` | Rejeitado — saldo devolvido |

### Modo de cobrança da taxa

A taxa de saque pode ser configurada em dois modos (definido pelo admin por merchant):

| Modo | Comportamento | Exemplo (saque R$ 500, taxa R$ 2) |
|------|--------------|-----------------------------------|
| **Do saque** (padrão) | Taxa descontada do valor sacado | Recebe R$ 498, débito R$ 500 |
| **Do saldo** | Taxa debitada separadamente do saldo | Recebe R$ 500, débito R$ 502 |

> O modo é transparente para a API. O campo `netAmount` na resposta sempre reflete o valor que o merchant vai receber.

---

## Disputas

Disputas são abertas pelo admin quando um pagador contesta uma transação. O merchant pode aceitar ou contestar.

### Contestar disputa

```bash
curl -X POST https://api.pagniv.com/v1/disputes/{id}/contest \
  -H "X-API-Key: sk_sandbox_sua_chave" \
  -H "Content-Type: application/json" \
  -d '{
    "response": "O produto foi entregue conforme combinado.",
    "evidenceUrls": ["https://exemplo.com/comprovante.pdf"]
  }'
```

### Aceitar disputa

```bash
curl -X POST https://api.pagniv.com/v1/disputes/{id}/accept \
  -H "X-API-Key: sk_sandbox_sua_chave"
```

### Status da Disputa

| Status | Descrição |
|--------|-----------|
| `OPEN` | Aguardando resposta do merchant |
| `MERCHANT_RESPONDED` | Merchant contestou, aguardando análise |
| `UNDER_REVIEW` | Em análise pelo admin |
| `RESOLVED_MERCHANT` | Resolvida a favor do merchant |
| `RESOLVED_BUYER` | Resolvida a favor do comprador |
| `REFUNDED` | Valor reembolsado |

---

## Formato de resposta

Todas as respostas seguem o mesmo envelope:

**Sucesso:**

```json
{
  "success": true,
  "data": { ... }
}
```

**Erro:**

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Amount must be at least 100 (R$ 1.00)"
  }
}
```

### Códigos de erro

| Código | Descrição |
|--------|-----------|
| `VALIDATION_ERROR` | Validação de input falhou |
| `UNAUTHORIZED` | Token ou API Key inválida |
| `FORBIDDEN` | Permissão insuficiente |
| `NOT_FOUND` | Recurso não encontrado |
| `CONFLICT` | Recurso já existe |
| `RATE_LIMIT_EXCEEDED` | Muitas requisições |

### Paginação

Endpoints de listagem suportam:

| Parâmetro | Padrão | Descrição |
|-----------|--------|-----------|
| `page` | 1 | Página atual |
| `limit` | 20 | Itens por página (max: 100) |

Resposta inclui `meta`:

```json
{
  "meta": {
    "total": 150,
    "page": 1,
    "limit": 20,
    "pages": 8
  }
}
```

---

## Rate Limiting

Headers retornados em toda requisição:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 97
X-RateLimit-Reset: 1705312800
```

Limites específicos por endpoint:

| Endpoint | Limite |
|----------|--------|
| `POST /v1/charges` | 60/minuto |
| `POST /v1/withdrawals` | 5/hora |
| `POST /v1/auth/login` | 15/15min |
| `POST /v1/auth/register` | 3/hora |

---

## Recursos

| Recurso | Descrição |
|---------|-----------|
| **Cobranças Pix** | QR Codes dinâmicos com confirmação instantânea |
| **Dashboard** | Cobranças, carteira, saques e disputas em tempo real |
| **Webhooks** | Assinatura HMAC, test delivery, histórico completo |
| **Multi-tenant** | Múltiplas organizações com roles (Owner, Admin, Finance, Viewer) |
| **Sandbox** | Ambiente completo para testes |
| **Liquidação automática** | Settlement configurável (D+0 a D+30) |
| **Disputas** | Contestação com evidências e resolução admin |
| **Split de pagamento** | Comissão automática para afiliados |
| **Depósito via Pix** | QR Code para adicionar saldo à carteira |
| **Taxa de saque flexível** | Desconto do saque ou do saldo, por merchant |
| **KYC** | Upload e validação de documentos |
| **Audit log** | Rastreamento de todas as ações |
| **Sub-contas** | Hierarquia de merchants com conta pai |

---

## Segmentos

PSPs · Fintechs · iGaming · SaaS · Marketplace · Infoprodutos · Serviços Financeiros

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
