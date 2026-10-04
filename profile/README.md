<p align="center">
  <img src="https://pagniv.com/logo.svg" alt="Pagniv" height="60" />
</p>

<h1 align="center">Pagniv</h1>

<p align="center">
  <strong>Infraestrutura de pagamentos para desenvolvedores.</strong><br/>
  Pix, boleto e cartão numa API só. Integração simples, liquidação automática.
</p>

<p align="center">
  <a href="https://pagniv.com">Site</a> ·
  <a href="https://devs.pagniv.com">Documentação completa</a> ·
  <a href="https://portal.pagniv.com">Dashboard</a>
</p>

---

## O contrato, em 12 linhas

Tudo nesta API obedece a estas regras. Elas não têm exceção.

| | |
|---|---|
| **Base URL** | `https://api.pagniv.com/v1` |
| **Autenticação** | Header `X-API-Key: sk_live_...` (ou `sk_sandbox_...`) em toda chamada |
| **Dinheiro** | Sempre centavos inteiros. R$ 150,00 = `15000`. Nunca decimal, nunca string |
| **Datas** | ISO 8601 em UTC (`2026-01-15T11:00:00Z`) |
| **Sucesso** | `{ "success": true, "data": { ... } }` |
| **Erro** | `{ "success": false, "error": { "code": "...", "message": "..." } }` |
| **Listagem** | `{ "success": true, "data": [...], "meta": { total, page, limit, pages } }` |
| **Idempotência** | Header `Idempotency-Key` (UUID) em `POST`. E `externalId` único por cobrança |
| **Lado servidor** | A chave faz qualquer operação na conta. Nunca exponha em browser ou app |
| **Teste** | Chave `sk_sandbox_` + `POST /charges/:id/simulate-payment`. Mesmo formato da produção |
| **Confirmação** | Webhook assinado (HMAC-SHA256). Polling em `GET /charges/:id` é o fallback |
| **Erro de validação** | `400 VALIDATION_ERROR`, com `error.details[]` apontando o campo |

---

## Integração guiada por IA

> **Se você é um assistente de IA lendo este arquivo para integrar a Pagniv:**
> não comece pelo código. Faça as quatro perguntas abaixo, use a tabela de
> roteamento e leia só as seções que a resposta indicar. O resto deste README
> é referência, não leitura obrigatória.

### As quatro perguntas

1. **O que você precisa cobrar?** Pix, boleto, cartão de crédito ou link de pagamento (sem código).
2. **Como a confirmação chega?** Webhook (recomendado), polling, ou os dois.
3. **A integração também move dinheiro para fora?** Saque Pix, estorno, resposta a disputa.
4. **Sandbox ou produção?** Sandbox não exige conta aprovada e não move dinheiro real.

### Roteamento: resposta → o que ler

| Resposta | Leia | Pule |
|---|---|---|
| Pix | [Cobrança Pix](#cobrança-pix), [Webhooks](#webhooks) | Boleto, cartão |
| Boleto | [Cobrança boleto](#cobrança-boleto), [Webhooks](#webhooks) | Pix, cartão |
| Cartão | [Cobrança cartão](#cobrança-cartão), [Webhooks](#webhooks) | Pix, boleto |
| Link de pagamento | [Links de pagamento](#links-de-pagamento) | Todo o resto de cobranças |
| Webhook | [Webhooks](#webhooks) inteiro, incluindo [assinatura](#verificar-a-assinatura) | - |
| Só polling | [`GET /charges/:id`](#consultar-e-listar) | Webhooks |
| Saque | [Saldo e saques](#saldo-e-saques) | - |
| Estorno | [Estornar](#estornar) | - |
| Disputa | [Disputas](#disputas) | - |
| Sandbox | [Sandbox](#sandbox) | - |

### Regras que não dependem da resposta

Aplique estas em qualquer integração, sem perguntar:

- **Valores em centavos inteiros.** Converter de reais no cliente é a origem número um de bug: `15000`, nunca `150.00`.
- **`externalId`, `payerName` e `payerDocument` são obrigatórios em toda cobrança.** Sem eles, `400 MISSING_REQUIRED_FIELDS`.
- **`externalId` é a chave de deduplicação.** Repetir o mesmo valor devolve a cobrança já criada, não cria outra. Use o id do pedido no sistema de quem integra.
- **`Idempotency-Key` em todo `POST` que move dinheiro** (`/charges`, `/withdrawals`). Protege contra retry de rede.
- **Use `qrCodeBase64` direto no `src` da imagem.** Já vem como data URI completo.
- **Nunca confie no corpo do webhook sem verificar a assinatura**, e verifique sobre o **corpo cru**.
- **Trate `429` com backoff exponencial.** `X-RateLimit-Reset` vem em **segundos restantes**, não em timestamp: espere esse tanto, não converta para data.
- **Guarde o `id` da cobrança.** É por ele que se consulta, cancela e estorna.

### Erros de integração que este README existe para evitar

| Sintoma | Causa | Correção |
|---|---|---|
| Cobrança com valor 100x errado | Valor enviado em reais | Enviar centavos inteiros |
| `<img>` quebrada no checkout | Concatenar `data:image/png;base64,` no `qrCodeBase64` | Usar o campo como veio |
| Assinatura do webhook nunca confere | HMAC sobre `JSON.stringify(req.body)` | HMAC sobre o corpo cru (`express.raw`) |
| Pedido pago duas vezes | Webhook reentregue e tratado sem checar estado | Tratar `charge.paid` como idempotente, pelo `id` |
| `400 MISSING_REQUIRED_FIELDS` | Falta `externalId`, `payerName` ou `payerDocument` | Enviar os três |
| `403 CARD_NOT_ENABLED` / `BOLETO_NOT_ENABLED` | Conta sem o método liberado | Falar com o suporte. Em sandbox já funciona |
| `422 AMOUNT_ABOVE_MAX` | Acima do limite por transação da conta | Ver [Limites](#limites-por-conta) |

---

## Começando

### 1. Conta e chave

Crie a conta em [portal.pagniv.com](https://portal.pagniv.com) e gere a chave em **Integrações → Chaves de API**:

```
sk_sandbox_...   testes, sem transação real, não exige conta aprovada
sk_live_...      produção
```

### 2. Primeira cobrança

```bash
curl -X POST https://api.pagniv.com/v1/charges \
  -H "X-API-Key: sk_sandbox_..." \
  -H "Idempotency-Key: 9b1d3e7c-5f8a-4d2e-9b1d-3e7c5f8a4d2e" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 15000,
    "externalId": "pedido-1234",
    "payerName": "João Silva",
    "payerDocument": "12345678909",
    "description": "Pedido #1234",
    "expiresIn": 3600
  }'
```

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "txid": "pagniv_abc123def456",
    "paymentMethod": "PIX",
    "amount": 15000,
    "feeAmount": 674,
    "netAmount": 14326,
    "affiliateAmount": 0,
    "qrCode": "00020126580014br.gov.bcb.pix...",
    "qrCodeBase64": "data:image/png;base64,iVBORw0KGgo...",
    "pixKey": "pagniv@pagniv.com",
    "barcode": null,
    "digitableLine": null,
    "boletoUrl": null,
    "dueDate": null,
    "status": "PENDING",
    "expiresAt": "2026-01-15T11:30:00Z",
    "createdAt": "2026-01-15T10:30:00Z"
  }
}
```

> `feeAmount` e `netAmount` saem da tabela de taxas **da sua conta**, que é
> acordada comercialmente. Os valores do exemplo (3,50% + R$ 1,49) são só
> ilustrativos: nunca calcule a taxa do seu lado, leia o que a resposta devolve.

### 3. Simule o pagamento

```bash
curl -X POST https://api.pagniv.com/v1/charges/550e8400-e29b-41d4-a716-446655440000/simulate-payment \
  -H "X-API-Key: sk_sandbox_..."
```

A cobrança vira `PAID` e o webhook `charge.paid` é disparado. Sem custo, sem dinheiro real.

---

## Mapa de endpoints

Toda rota abaixo aceita `X-API-Key`. O escopo na tabela é exigido quando a
chave tem escopos marcados; chave sem escopo gravado alcança tudo.

### Cobranças

| Método | Rota | Escopo | O que faz |
|---|---|---|---|
| `POST` | `/charges` | `charges:write` | Cria cobrança Pix, boleto ou cartão |
| `GET` | `/charges` | `charges:read` | Lista com filtro e paginação |
| `GET` | `/charges/:id` | `charges:read` | Detalha uma cobrança |
| `GET` | `/charges/card-config` | `charges:read` | Chave e endpoint de tokenização de cartão |
| `DELETE` | `/charges/:id` | `charges:write` | Cancela cobrança `PENDING` |
| `POST` | `/charges/:id/refund` | `charges:write` | Estorna cobrança paga |
| `POST` | `/charges/:id/simulate-payment` | `charges:write` | Marca como paga (somente sandbox) |
| `GET` | `/charges/:id/checkout` | público | Dados do checkout, sem chave |

### Links de pagamento

| Método | Rota | Escopo | O que faz |
|---|---|---|---|
| `POST` | `/payment-links` | `charges:write` | Cria o link |
| `GET` | `/payment-links` | `charges:read` | Lista os links da conta |
| `GET` | `/payment-links/:id` | `charges:read` | Detalha um link |
| `PATCH` | `/payment-links/:id` | `charges:write` | Edita (inclusive `isActive`) |
| `DELETE` | `/payment-links/:id` | `charges:write` | Remove |

### Carteira e saques

| Método | Rota | Escopo | O que faz |
|---|---|---|---|
| `GET` | `/balance` | `wallet:read` | Saldos |
| `GET` | `/transactions` | `wallet:read` | Lançamentos da carteira |
| `GET` | `/statement` | `wallet:read` | Extrato por período |
| `GET` | `/withdrawals` | `wallet:read` | Lista saques |
| `GET` | `/withdrawals/:id` | `wallet:read` | Detalha um saque |
| `POST` | `/withdrawals` | `withdrawals:write` | Solicita saque |
| `POST` | `/withdrawals/decode-qr` | `withdrawals:write` | Lê um Pix copia e cola antes de pagar |

> **Destinos de saque salvos se cadastram no dashboard**, não pela API: as rotas
> `/withdrawal-destinations` exigem sessão do painel. A chave de API continua
> podendo **usar** um destino já salvo, passando o `destinationId` em
> `POST /withdrawals`.

### Webhooks

| Método | Rota | Escopo | O que faz |
|---|---|---|---|
| `POST` | `/webhook-config` | `charges:write` | Cria a configuração e devolve o `secret` |
| `GET` | `/webhook-config` | `charges:read` | Lista as configurações |
| `PUT` | `/webhook-config/:id` | `charges:write` | Edita URL, eventos ou segredo |
| `DELETE` | `/webhook-config/:id` | `charges:write` | Remove |
| `POST` | `/webhook-config/:id/test` | `charges:write` | Dispara um evento de teste |
| `GET` | `/webhook-deliveries` | `charges:read` | Histórico de entregas e tentativas |

### Disputas e relatórios

| Método | Rota | Escopo | O que faz |
|---|---|---|---|
| `GET` | `/disputes` | `charges:read` | Lista (filtro `?status=`) |
| `GET` | `/disputes/stats` | `charges:read` | Em aberto, valor em risco, vencendo, taxa de ganho |
| `GET` | `/disputes/:id` | `charges:read` | Detalha |
| `POST` | `/disputes/:id/accept` | `charges:write` | Aceita e reembolsa o pagador |
| `POST` | `/disputes/:id/contest` | `charges:write` | Contesta com texto e evidências |
| `GET` | `/reports/transactions` | `charges:read` | Transações do período (JSON ou CSV) |
| `GET` | `/reports/summary` | `charges:read` | Resumo do período |
| `GET` | `/reports/daily` | `charges:read` | Volume diário |
| `GET` | `/reports/payment-methods` | `charges:read` | Distribuição por método |

---

## Autenticação

```
X-API-Key: sk_live_...
```

> **Esta API é server-to-server.** Quem tem a chave faz qualquer operação na
> conta. As chamadas partem do seu backend, nunca de browser, app mobile ou
> JavaScript público.

| Ambiente | Prefixo | Comportamento |
|---|---|---|
| Sandbox | `sk_sandbox_` | Sem transação real. Funciona antes da conta ser aprovada |
| Produção | `sk_live_` | Transações reais |

### Escopos

Ao criar a chave você escolhe o que ela pode fazer. O preset "Somente cobranças" gera uma chave **sem** permissão de saque, o que reduz o estrago se ela vazar.

| Escopo | Libera |
|---|---|
| `charges:write` | Criar, cancelar e estornar cobranças. Também: criar e editar links de pagamento, configurar webhooks, aceitar e contestar disputas |
| `charges:read` | Consultar cobranças, links, webhooks, entregas, disputas e relatórios |
| `wallet:read` | Saldo, extrato e leitura de saques |
| `withdrawals:write` | Solicitar saques Pix |

Chamada sem o escopo necessário devolve `403 INSUFFICIENT_SCOPE`. Chave sem nenhum escopo marcado tem acesso total.

Link de pagamento, webhook e disputa entram em `charges:*` porque são o mesmo
dinheiro por outra porta: um link é uma forma de cobrar, trocar a URL do
webhook desvia o aviso de pagamento, e aceitar uma disputa devolve o valor ao
pagador.

### Restrição por IP e validade

- **IPs autorizados:** informe IPs ou faixas CIDR (IPv4 ou IPv6). Chamada de outro IP recebe `403 API_KEY_IP_NOT_ALLOWED`. Vazio significa qualquer IP.
- **Validade:** com data de expiração definida, depois dela a chave devolve `401 API_KEY_EXPIRED` e você gera outra.

---

## Idempotência

Duas proteções diferentes, que se complementam:

| Mecanismo | Onde | Protege contra |
|---|---|---|
| `Idempotency-Key` | Header, UUID v4 | Retry de rede e timeout. A mesma chave em até 24h devolve a resposta original sem reprocessar |
| `externalId` | Corpo da cobrança | Pedido duplicado. O mesmo `externalId` devolve a cobrança já existente |

```bash
curl -X POST https://api.pagniv.com/v1/charges \
  -H "X-API-Key: sk_live_..." \
  -H "Idempotency-Key: 9b1d3e7c-5f8a-4d2e-9b1d-3e7c5f8a4d2e" \
  -H "Content-Type: application/json" \
  -d '{ "amount": 15000, "externalId": "pedido-1234", "payerName": "João Silva", "payerDocument": "12345678909" }'
```

Use os dois em qualquer integração de produção.

---

## Cobranças

### Campos

Três campos são obrigatórios em toda cobrança, por compliance PLD-FT e por deduplicação: `externalId`, `payerName` e `payerDocument`. Sem eles, `400 MISSING_REQUIRED_FIELDS`.

| Campo | Tipo | Obrigatório | Regra |
|---|---|:-:|---|
| `amount` | `integer` | Sim | Centavos. Mínimo `100`, máximo `99999999` |
| `externalId` | `string` | Sim | Até 255 caracteres. Único por conta, deduplica |
| `payerName` | `string` | Sim | Até 255 caracteres. Pontuação é removida: `João Silva-Jr.` é gravado como `João SilvaJr` |
| `payerDocument` | `string` | Sim | CPF (11 dígitos) ou CNPJ (14), com dígito verificador válido |
| `paymentMethod` | `string` | Não | `PIX` (padrão), `BOLETO` ou `CARD` |
| `description` | `string` | Não | Até 255 caracteres |
| `payerEmail` | `string` | Não | E-mail válido |
| `expiresIn` | `integer` | Não | Expiração do Pix em segundos, de `300` a `86400`. Padrão `3600` |
| `dueDate` | `string` | Não | Vencimento do boleto (ISO 8601). Sem ele, 3 dias |
| `payerAddress` | `object` | Não | Endereço no boleto: `zipCode`, `line1`, `city`, `state` |
| `cardToken` | `string` | Condicional | Obrigatório quando `paymentMethod` é `CARD` |
| `installments` | `integer` | Não | Parcelas de `1` a `18` no cartão. Padrão `1`, sem juros |
| `statementDescriptor` | `string` | Não | Texto na fatura do portador. A API aceita até 22 caracteres, mas só os **13 primeiros** chegam à fatura, e só letras, números e espaço. Vazio vira `PAGNIV` |
| `payerPhone` | `string` | Não | Telefone do portador. Melhora a aprovação no antifraude do cartão |

### Status

| Status | Significa |
|---|---|
| `PENDING` | Aguardando pagamento |
| `PAID` | Pagamento confirmado |
| `EXPIRED` | Passou de `expiresAt` sem pagamento |
| `CANCELLED` | Cancelada via API |
| `REFUNDED` | Estornada |
| `DISPUTED` | Em contestação |

### Cobrança Pix

```typescript
const res = await fetch('https://api.pagniv.com/v1/charges', {
  method: 'POST',
  headers: {
    'X-API-Key': process.env.PAGNIV_API_KEY,
    'Idempotency-Key': crypto.randomUUID(),
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    amount: 15000,
    externalId: 'pedido-1234',
    payerName: 'João Silva',
    payerDocument: '12345678909',
    description: 'Pedido #1234',
    expiresIn: 3600,
  }),
})

const { data } = await res.json()
data.qrCode        // copia e cola, para o botão de copiar
data.qrCodeBase64  // data URI completo, use direto em <img src={...}>
```

> **`qrCodeBase64` é sempre um data URI completo** (`data:image/png;base64,...`).
> Use no `src` sem concatenar prefixo. Quando a adquirente não devolve imagem, a
> Pagniv desenha a partir do copia e cola, e o sandbox responde no mesmo formato
> da produção. Em boleto e cartão, `qrCode` e `qrCodeBase64` vêm `null`.

Para mostrar o pagamento sem construir tela, mande o cliente para
`https://pix.pagniv.com/{id}`: é o checkout hospedado da própria cobrança.

### Cobrança boleto

Boleto registrado. Exige `paymentMethod: "BOLETO"` e o CPF ou CNPJ em `payerDocument`. O endereço (`payerAddress`) é opcional: sem ele a Pagniv usa um endereço padrão, já que boleto registrado exige endereço.

> **Boleto depende de habilitação da conta.** Sem liberação, a criação devolve
> `403 BOLETO_NOT_ENABLED`. Em sandbox funciona desde o primeiro dia.

```typescript
const res = await fetch('https://api.pagniv.com/v1/charges', {
  method: 'POST',
  headers: { 'X-API-Key': process.env.PAGNIV_API_KEY, 'Content-Type': 'application/json' },
  body: JSON.stringify({
    paymentMethod: 'BOLETO',
    amount: 15000,
    externalId: 'pedido-1234',
    payerName: 'João Silva',
    payerDocument: '12345678909',
    dueDate: '2026-01-23',
    payerAddress: {
      zipCode: '01310930',
      line1: 'Avenida Paulista, 1106',
      city: 'São Paulo',
      state: 'SP',
    },
  }),
})

const { data } = await res.json()
data.digitableLine  // linha digitável
data.barcode        // código de barras
data.boletoUrl      // PDF
data.dueDate        // vencimento
```

Quando o boleto compensa, o webhook `charge.paid` é disparado igual ao Pix. A liquidação segue o `settlementDays` da conta (D+1 por padrão).

### Cobrança cartão

Pagamento **síncrono**: a resposta já vem `PAID` ou o erro `402 CARD_DECLINED`. O número do cartão nunca passa pela API da Pagniv, porque a tokenização acontece no navegador do comprador.

> **Cartão depende de habilitação da conta.** Sem liberação, `403 CARD_NOT_ENABLED`.
> O campo `cardEnabled` em `GET /charges/card-config` diz o estado da sua conta.

**Passo 1, no navegador: tokenizar.** Busque a config e use `tokenizeUrl` e `publicKey` como vieram, sem fixar no código. Isso mantém a integração imune a troca de infraestrutura de processamento.

```typescript
const cfg = await fetch('https://api.pagniv.com/v1/charges/card-config', {
  headers: { 'X-API-Key': process.env.PAGNIV_API_KEY },
}).then(r => r.json())

// cfg.data = { publicKey, tokenizeUrl, maxInstallments, cardEnabled }

const tok = await fetch(`${cfg.data.tokenizeUrl}?appId=${cfg.data.publicKey}`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    type: 'card',
    card: { number: '4000000000000010', holder_name: 'João Silva', exp_month: 12, exp_year: 30, cvv: '123' },
  }),
}).then(r => r.json())

const cardToken = tok.id
```

**Passo 2, no seu backend: criar a cobrança.**

```typescript
const res = await fetch('https://api.pagniv.com/v1/charges', {
  method: 'POST',
  headers: { 'X-API-Key': process.env.PAGNIV_API_KEY, 'Content-Type': 'application/json' },
  body: JSON.stringify({
    paymentMethod: 'CARD',
    cardToken,
    amount: 15000,
    externalId: 'pedido-1234',
    payerName: 'João Silva',
    payerDocument: '12345678909',
    installments: 3,
  }),
})

const { data } = await res.json()
data.status          // PAID
data.cardBrand       // visa
data.cardLastDigits  // 0010
data.installments    // 3
```

Aprovado: a cobrança entra como `PAID`, o saldo é creditado seguindo o `settlementDays` e o webhook `charge.paid` é disparado. Recusado: `402 CARD_DECLINED` e nenhuma cobrança é criada. O parcelamento é sem juros para o comprador, e o custo das parcelas sai da margem do merchant.

### Consultar e listar

```bash
curl "https://api.pagniv.com/v1/charges?status=PAID&startDate=2026-01-01T00:00:00Z&limit=50" \
  -H "X-API-Key: sk_live_..."
```

| Filtro | Valores |
|---|---|
| `status` | `PENDING`, `PAID`, `EXPIRED`, `CANCELLED`, `REFUNDED`, `DISPUTED` |
| `paymentMethod` | `PIX`, `CARD`, `BOLETO` |
| `startDate`, `endDate` | ISO 8601, sobre a data de criação |
| `search` | txid, `externalId`, nome, documento, e-mail ou descrição |
| `minAmount`, `maxAmount` | Centavos |
| `format` | `csv` devolve CSV no lugar de JSON |
| `page`, `limit` | `limit` até 100. Padrão 20 |

A cobrança detalhada traz `partialRefund`: `true` quando aquela cobrança aceita estorno de valor parcial, `false` quando só aceita o valor cheio. Use para decidir se a sua tela oferece campo de valor no estorno.

### Estornar

```bash
curl -X POST https://api.pagniv.com/v1/charges/{id}/refund \
  -H "X-API-Key: sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{ "amount": 5000, "reason": "Cliente solicitou cancelamento" }'
```

`amount` é opcional e, omitido, estorna o líquido total. O valor volta ao pagador via Pix e sai do saldo disponível. Com um estorno já em andamento para a mesma cobrança, a resposta é `400 REFUND_IN_PROGRESS`.

### Limites por conta

Além do mínimo de R$ 1,00 e do teto de R$ 999.999,99 da API, cada conta tem limites próprios, visíveis no dashboard:

| Erro | Quando |
|---|---|
| `422 AMOUNT_BELOW_MIN` | Abaixo do mínimo por cobrança da conta |
| `422 AMOUNT_ABOVE_MAX` | Acima do teto por cobrança da conta |
| `422 DAILY_LIMIT_EXCEEDED` | A cobrança estouraria o limite diário |
| `422 MONTHLY_LIMIT_EXCEEDED` | A cobrança estouraria o limite mensal |
| `422 WITHDRAWAL_AMOUNT_ABOVE_MAX` | Saque acima do teto por saque |

Todos são `422` e a mensagem traz o valor do limite e o uso atual. Nenhum deles se resolve com retry: ou o valor muda, ou o limite da conta muda.

---

## Links de pagamento

Para cobrar sem construir checkout. Você cria o link uma vez e o pagador escolhe o método na página hospedada, que gera a cobrança sozinha.

```bash
curl -X POST https://api.pagniv.com/v1/payment-links \
  -H "X-API-Key: sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Mensalidade do plano Pro",
    "amountMode": "FIXED",
    "amount": 9900,
    "paymentMethods": ["PIX", "CARD"],
    "expiresIn": 3600
  }'
```

```json
{
  "success": true,
  "data": {
    "id": "8f2b1c44-0a7e-4b10-9d33-1c0f9a6e4b21",
    "slug": "k3Qm9xT2bW4r",
    "title": "Mensalidade do plano Pro",
    "amountMode": "FIXED",
    "amount": 9900,
    "paymentMethods": ["PIX", "CARD"],
    "isActive": true,
    "createdAt": "2026-01-15T10:30:00Z"
  }
}
```

O link público é `https://pix.pagniv.com/l/{slug}`.

| Campo | Tipo | Obrigatório | Regra |
|---|---|:-:|---|
| `title` | `string` | Sim | 1 a 120 caracteres |
| `amountMode` | `string` | Não | `FIXED` (padrão), `FREE` ou `RANGE` |
| `amount` | `integer` | Condicional | Centavos. Usado quando `amountMode` é `FIXED` |
| `minAmount`, `maxAmount` | `integer` | Condicional | Centavos. Usados em `RANGE` |
| `paymentMethods` | `string[]` | Não | `PIX`, `BOLETO`, `CARD`. Padrão `["PIX"]` |
| `description` | `string` | Não | Até 500 caracteres |
| `expiresIn` | `integer` | Não | Expiração das cobranças geradas, de `60` a `86400` segundos |
| `whiteLabel` | `boolean` | Não | Esconde o branding Pagniv na página |
| `hideMerchantHeader` | `boolean` | Não | Esconde o cabeçalho com nome e avatar do merchant |
| `customColor` | `string` | Não | Hex `#RRGGBB` usado como base do gradiente da página |

As cobranças nascidas do link aparecem normalmente em `GET /charges` e disparam os mesmos webhooks. Para desativar um link sem apagar o histórico, use `PATCH` com `{ "isActive": false }`.

---

## Webhooks

A Pagniv faz `POST` no seu endpoint quando um evento acontece. É a fonte primária de confirmação. Polling em `GET /charges/:id` fica como fallback para quando o evento não chegar em tempo razoável.

### Eventos

| Evento | Quando dispara |
|---|---|
| `charge.paid` | Cobrança paga (Pix, boleto compensado ou cartão aprovado) |
| `charge.expired` | Passou de `expiresAt` sem pagamento |
| `charge.refunded` | Cobrança estornada |
| `dispute.opened` | Disputa aberta sobre uma cobrança paga |
| `dispute.resolved` | Disputa encerrada |
| `withdrawal.completed` | Saque pago pelo provedor |
| `withdrawal.failed` | Saque falhou no provedor, saldo devolvido |
| `withdrawal.rejected` | Saque recusado pelo time Pagniv, saldo devolvido |

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

A resposta inclui o `secret`: guarde, é com ele que você verifica a assinatura. Lista de `events` vazia significa todos os eventos. Até 5 configurações por conta (`400 WEBHOOK_LIMIT_REACHED` acima disso). A URL precisa ser `https`.

Antes de subir para produção, `POST /webhook-config/:id/test` dispara um evento de teste contra a sua URL e devolve o status e o corpo da resposta, o que resolve metade dos problemas de integração sem esperar uma transação real.

### Como a entrega funciona

| | |
|---|---|
| Método | `POST` com `Content-Type: application/json` |
| Headers | `X-Webhook-Signature: sha256=<hex>`, `X-Webhook-Id: <uuid>`, `User-Agent: Plataforma-Webhook/1.0` |
| Timeout | 10 segundos |
| Sucesso | Qualquer `2xx` |
| Retry | 5 tentativas, backoff exponencial de 2s, 4s, 8s, 16s e 32s |
| Histórico | `GET /webhook-deliveries` mostra tentativas, status e resposta |

Responda `2xx` assim que receber e processe depois. Processamento demorado dentro do handler estoura o timeout e gera reentrega.

**O mesmo evento pode chegar mais de uma vez.** Trate o handler como idempotente, pelo `data.id` da cobrança.

### Envelope

Todo evento tem a mesma forma. Só o conteúdo de `data` muda.

```json
{
  "event": "charge.paid",
  "data": { },
  "timestamp": "2026-01-15T11:05:01Z"
}
```

<details>
<summary><strong>charge.paid</strong>, <strong>charge.expired</strong>, <strong>charge.refunded</strong></summary>

```json
{
  "event": "charge.paid",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "txid": "pagniv_abc123def456",
    "amount": 15000,
    "netAmount": 14326,
    "feeAmount": 674,
    "status": "PAID",
    "paidAt": "2026-01-15T11:05:00Z",
    "externalId": "pedido-1234"
  },
  "timestamp": "2026-01-15T11:05:01Z"
}
```

Em `charge.expired`, `status` é `EXPIRED` e `paidAt` é `null`.

Em `charge.refunded`, `status` é `REFUNDED` e estes dois campos entram dentro de `data`:

```json
{
  "refundedAmount": 14326,
  "refundReason": "Solicitação do cliente"
}
```
</details>

<details>
<summary><strong>dispute.opened</strong> e <strong>dispute.resolved</strong></summary>

```json
{
  "event": "dispute.opened",
  "data": {
    "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "chargeId": "550e8400-e29b-41d4-a716-446655440000",
    "amount": 14326,
    "reason": "FRAUD",
    "description": "Não reconheço esta transação",
    "status": "OPEN",
    "deadline": "2026-01-22T11:05:00Z",
    "createdAt": "2026-01-15T14:22:00Z"
  },
  "timestamp": "2026-01-15T14:22:01Z"
}
```

Em `dispute.resolved`, `status` vira `RESOLVED_MERCHANT`, `RESOLVED_BUYER` ou `REFUNDED`, e estes campos entram dentro de `data`:

```json
{
  "resolvedAt": "2026-01-18T10:11:00Z",
  "resolution": "Evidência aceita, merchant venceu."
}
```
</details>

<details>
<summary><strong>withdrawal.completed</strong>, <strong>withdrawal.failed</strong>, <strong>withdrawal.rejected</strong></summary>

```json
{
  "event": "withdrawal.completed",
  "data": {
    "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "amount": 50000,
    "fee": 200,
    "netAmount": 49800,
    "pixKey": "12345678909",
    "pixKeyType": "CPF",
    "status": "COMPLETED",
    "providerTxId": "E60746948202601151130ABC123",
    "createdAt": "2026-01-15T11:30:00Z"
  },
  "timestamp": "2026-01-15T11:30:15Z"
}
```

Em `withdrawal.failed`, `status` é `FAILED` e `providerTxId` é `null`.

Em `withdrawal.rejected`, `status` é `REJECTED` e estes campos entram dentro de `data`:

```json
{
  "rejectedAt": "2026-01-15T13:00:00Z",
  "rejectedReason": "Chave Pix divergente do cadastro"
}
```

Nos dois casos de falha, o saldo já voltou para a carteira.
</details>

### Verificar a assinatura

A assinatura é o HMAC-SHA256 do **corpo cru** da requisição, com o `secret` da configuração. Reserializar o JSON muda espaçamento e ordem de chaves, e a assinatura não confere mais.

```typescript
import crypto from 'crypto'
import express from 'express'

const app = express()

// O corpo cru é obrigatório para o HMAC. Sem express.raw, nada confere.
app.use('/webhooks/pagniv', express.raw({ type: 'application/json' }))

function assinaturaConfere(corpoCru: Buffer, header: string, secret: string): boolean {
  const recebida = Buffer.from(header.replace('sha256=', ''), 'hex')
  const esperada = crypto.createHmac('sha256', secret).update(corpoCru).digest()
  // Tamanhos diferentes fazem timingSafeEqual lançar, então confira antes.
  if (recebida.length !== esperada.length) return false
  return crypto.timingSafeEqual(recebida, esperada)
}

app.post('/webhooks/pagniv', async (req, res) => {
  const header = req.headers['x-webhook-signature'] as string | undefined
  if (!header || !assinaturaConfere(req.body, header, process.env.PAGNIV_WEBHOOK_SECRET!)) {
    return res.status(401).send('assinatura inválida')
  }

  // Responda primeiro, processe depois: o timeout da entrega é de 10s.
  res.status(200).send('ok')

  const { event, data } = JSON.parse(req.body.toString())
  switch (event) {
    case 'charge.paid':          await marcarPedidoPago(data.externalId, data.id); break
    case 'charge.expired':       await liberarEstoque(data.externalId); break
    case 'charge.refunded':      await registrarEstorno(data.id, data.refundedAmount); break
    case 'dispute.opened':       await avisarRisco(data); break
    case 'dispute.resolved':     await fecharDisputa(data.id, data.status); break
    case 'withdrawal.completed': await baixarSaque(data.id); break
    case 'withdrawal.failed':
    case 'withdrawal.rejected':  await saldoDevolvido(data.id); break
  }
})
```

`marcarPedidoPago` precisa ser idempotente: a mesma entrega pode chegar de novo.

---

## Saldo e saques

### Consultar saldo

```bash
curl https://api.pagniv.com/v1/balance -H "X-API-Key: sk_live_..."
```

```json
{
  "success": true,
  "data": {
    "availableBalance": 125050,
    "withdrawableBalance": 110050,
    "disputeDebt": 0,
    "pendingBalance": 30000,
    "reservedBalance": 15000,
    "blockedBalance": 0,
    "totalEarned": 980000,
    "totalWithdrawn": 820000,
    "totalFeesPaid": 34950,
    "updatedAt": "2026-01-15T11:05:00Z"
  }
}
```

| Campo | O que é |
|---|---|
| `availableBalance` | Saldo disponível, bruto |
| `withdrawableBalance` | **O que dá para sacar de fato**: disponível menos o retido por disputa |
| `disputeDebt` | Dívida de disputa já encerrada e ainda não recuperada |
| `pendingBalance` | Em liquidação (D+N) |
| `reservedBalance` | Reservado para disputas em aberto |
| `blockedBalance` | Bloqueado |
| `totalEarned`, `totalWithdrawn`, `totalFeesPaid` | Acumulados da conta |

Para decidir quanto sacar, use `withdrawableBalance`, não `availableBalance`.

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

| Campo | Tipo | Obrigatório | Regra |
|---|---|:-:|---|
| `amount` | `integer` | Sim | Centavos |
| `destinationId` | `string` | Não | Destino salvo. Dispensa `pixKey` e `pixKeyType` |
| `pixKey` | `string` | Condicional | Obrigatório sem `destinationId` |
| `pixKeyType` | `string` | Condicional | `CPF`, `CNPJ`, `EMAIL`, `PHONE` ou `EVP` |
| `source` | `string` | Não | `MANUAL` (padrão) ou `QR_CODE` |

A resposta traz `method`: `PIX` quando o saque é processado automaticamente, `TED` quando o destino é uma conta bancária salva.

**Saque TED** nasce `PENDING` e é pago manualmente pelo time Pagniv, porque não há execução automática de TED. A resposta inclui o retrato dos dados bancários usados (`bankCode`, `bankName`, `agency`, `accountNumber`, `accountDigit`, `accountType`, `holderName`, `holderDocument`).

**Estados:**

```
PIX:  PENDING → APPROVED → PROCESSING → COMPLETED
TED:  PENDING → COMPLETED
```

`FAILED` e `REJECTED` encerram o saque e devolvem o saldo. Nos dois casos sai o webhook correspondente.

### Pagar um Pix copia e cola

Decodifica um BR Code e devolve a chave, o tipo e o valor, para você confirmar antes de pagar com `POST /withdrawals` e `source: "QR_CODE"`.

```bash
curl -X POST https://api.pagniv.com/v1/withdrawals/decode-qr \
  -H "X-API-Key: sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{ "brCode": "00020126330014br.gov.bcb.pix0111..." }'
```

```json
{
  "success": true,
  "data": {
    "pixKey": "12345678909",
    "pixKeyType": "CPF",
    "amount": 1000,
    "merchantName": "Fulano de Tal",
    "merchantCity": "BRASILIA",
    "txid": "***"
  }
}
```

`amount` vem em centavos, ou `null` quando o QR não traz valor fixo. QR estático traz a chave no próprio payload; QR dinâmico traz uma URL, que o backend resolve (só `https`, sem redirect, host que não resolva para IP privado).

> O pagamento sai como Pix por chave, sem repassar o `txid`. O valor chega
> certo, mas a baixa automática da cobrança dinâmica no PSP de quem recebe não
> é garantida.

---

## Disputas

A disputa abre quando o pagador contesta uma cobrança paga junto ao banco (devolução, MED ou chargeback) e o provedor reporta. O valor é reservado na carteira e sai o webhook `dispute.opened`.

| Método | Rota | O que faz |
|---|---|---|
| `GET` | `/disputes` | Lista, com filtro `?status=` |
| `GET` | `/disputes/stats` | Em aberto, valor em risco, vencendo, taxa de ganho |
| `GET` | `/disputes/:id` | Detalha |
| `POST` | `/disputes/:id/accept` | Aceita e reembolsa o pagador |
| `POST` | `/disputes/:id/contest` | Contesta com texto e evidências |

```bash
curl -X POST https://api.pagniv.com/v1/disputes/{id}/contest \
  -H "X-API-Key: sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "response": "O produto foi entregue conforme combinado.",
    "evidences": ["https://exemplo.com/comprovante.pdf"]
  }'
```

```
OPEN → MERCHANT_RESPONDED → UNDER_REVIEW → RESOLVED_MERCHANT | RESOLVED_BUYER | REFUNDED
```

Cada disputa tem `deadline`. Contestação depois dele devolve `400 DEADLINE_PASSED`, e a disputa segue para resolução sem a sua defesa. Automatize: ao receber `dispute.opened`, dispare a contestação com as evidências que você já tem.

---

## Sandbox

Sandbox não exige conta aprovada e não move dinheiro real. Fora isso, o formato das respostas é idêntico ao da produção, de propósito: sandbox que responde diferente não ensaia nada.

1. Use uma chave `sk_sandbox_`.
2. Crie a cobrança normalmente. O QR é de teste.
3. Configure o webhook para validar seu handler de ponta a ponta.
4. Simule o pagamento:

```bash
curl -X POST https://api.pagniv.com/v1/charges/{id}/simulate-payment \
  -H "X-API-Key: sk_sandbox_..."
```

A cobrança vira `PAID` e o `charge.paid` é entregue. `simulate-payment` em cobrança de produção devolve `400 NOT_SANDBOX_CHARGE`.

---

## Erros

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Dados inválidos",
    "details": [{ "field": "amount", "message": "Valor deve ser positivo" }]
  }
}
```

`details` só aparece em erro de validação, e aponta o campo exato.

### O que fazer com cada classe

| Faixa | Significa | Ação |
|---|---|---|
| `400`, `422` | O pedido está errado ou esbarra num limite | Corrigir e reenviar. Retry sem mudança repete o erro |
| `401` | Chave inválida, ausente, revogada ou expirada | Conferir a chave |
| `402` | Cartão recusado | Pedir outro meio de pagamento ao comprador |
| `403` | Chave, IP, escopo ou conta sem permissão | Ajustar a chave ou falar com o suporte |
| `404` | Recurso não existe, não é da sua conta, ou a URL está errada | Conferir o `id` e o caminho |
| `409` | Estado atual não permite a operação | Reler o recurso antes de repetir |
| `429` | Limite de requisições | Backoff exponencial. `X-RateLimit-Reset` traz os segundos que faltam |
| `5xx` | Falha do nosso lado | Retry com backoff. Cobrança: use `Idempotency-Key` e `externalId` |

### Códigos

| HTTP | Código | Quando |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Campo inválido. Ver `details` |
| 400 | `MISSING_REQUIRED_FIELDS` | Falta `externalId`, `payerName` ou `payerDocument` |
| 400 | `AMOUNT_TOO_LOW` | Cobrança abaixo de R$ 1,00 |
| 400 | `INSUFFICIENT_BALANCE` | Saque ou estorno maior que o saldo disponível |
| 400 | `INVALID_PIX_KEY` | Chave Pix não bate com o tipo informado |
| 400 | `CHARGE_NOT_PENDING` | Cancelamento de cobrança que não está pendente |
| 400 | `CHARGE_NOT_PAID` | Estorno de cobrança que não foi paga |
| 400 | `ALREADY_REFUNDED` | Cobrança já estornada |
| 400 | `REFUND_IN_PROGRESS` | Já existe um estorno em andamento |
| 400 | `REFUND_NOT_SUPPORTED` | Estorno parcial numa cobrança que só aceita o valor cheio |
| 400 | `AMOUNT_EXCEEDS_NET` | Estorno maior que o líquido da cobrança |
| 400 | `NOT_SANDBOX_CHARGE` | `simulate-payment` em cobrança de produção |
| 400 | `BOLETO_DOCUMENT_REQUIRED` | Boleto sem CPF ou CNPJ do pagador |
| 400 | `CARD_TOKEN_REQUIRED` | Cartão sem `cardToken` |
| 400 | `CARD_DOCUMENT_REQUIRED` | Cartão sem CPF ou CNPJ do pagador |
| 400 | `DEADLINE_PASSED` | Contestação depois do prazo da disputa |
| 400 | `BANK_ACCOUNT_REQUIRED` | A conta exige conta bancária cadastrada antes de sacar |
| 400 | `DESTINATION_REQUIRED` | Saque sem `destinationId` nem `pixKey` |
| 400 | `WEBHOOK_REQUIRES_HTTPS` | URL de webhook não é `https` |
| 400 | `WEBHOOK_LIMIT_REACHED` | Mais de 5 configurações de webhook |
| 401 | `UNAUTHORIZED` | Chave inválida, ausente ou revogada |
| 401 | `API_KEY_EXPIRED` | Chave passou da validade |
| 402 | `CARD_DECLINED` | Emissor recusou o cartão |
| 403 | `INSUFFICIENT_SCOPE` | Chave sem o escopo da operação |
| 403 | `API_KEY_IP_NOT_ALLOWED` | IP de origem fora da allowlist da chave |
| 403 | `CARD_NOT_ENABLED` | Cartão não liberado para a conta |
| 403 | `BOLETO_NOT_ENABLED` | Boleto não liberado para a conta |
| 403 | `ORG_BLOCKED` | Conta bloqueada |
| 403 | `ORG_NOT_ACTIVE` | Conta ainda não aprovada. Em sandbox a operação funciona |
| 403 | `RESTRICTED_PARTY` | Documento do pagador em lista restritiva |
| 404 | `CHARGE_NOT_FOUND` | Cobrança inexistente ou de outra conta |
| 404 | `NOT_FOUND` | Recurso inexistente ou de outra conta |
| 404 | `ROUTE_NOT_FOUND` | A URL não existe. Confira método e caminho |
| 422 | `AMOUNT_BELOW_MIN` | Abaixo do mínimo por cobrança da conta |
| 422 | `AMOUNT_ABOVE_MAX` | Acima do teto por cobrança da conta |
| 422 | `DAILY_LIMIT_EXCEEDED` | Estouraria o limite diário |
| 422 | `MONTHLY_LIMIT_EXCEEDED` | Estouraria o limite mensal |
| 422 | `WITHDRAWAL_AMOUNT_ABOVE_MAX` | Saque acima do teto por saque |
| 429 | `RATE_LIMITED` | Limite de requisições excedido |

---

## Paginação e rate limit

### Paginação

`?page=1&limit=20`, com `limit` até 100.

```json
{
  "success": true,
  "data": [],
  "meta": { "total": 150, "page": 1, "limit": 20, "pages": 8 }
}
```

### Rate limit

| Escopo | Teto |
|---|---|
| Geral, por chave de API | 300 por minuto |
| `POST /charges` | 60 por minuto, por conta |
| `POST /withdrawals` | 60 por minuto, por conta |
| `POST /withdrawals/decode-qr` | 120 por minuto, por conta |
| Sem autenticação, por IP | 120 por minuto |

Headers em toda resposta:

```
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 297
X-RateLimit-Reset: 42
```

| Header | O que é |
|---|---|
| `X-RateLimit-Limit` | Teto **daquela rota**, não o geral |
| `X-RateLimit-Remaining` | Quantas chamadas sobram na janela |
| `X-RateLimit-Reset` | **Segundos até a janela reabrir**, não um timestamp. Espere esse tanto antes de tentar de novo |

O teto por conta vale para a conta inteira, não por chave: duas chaves da mesma conta dividem as 60 cobranças por minuto.

---

## Checklist de produção

- [ ] A chave vive em variável de ambiente, nunca no código nem no frontend
- [ ] Escopo da chave limitado ao que a integração usa
- [ ] IPs do seu servidor na allowlist da chave
- [ ] `Idempotency-Key` em `POST /charges` e `POST /withdrawals`
- [ ] `externalId` preenchido com o id do pedido no seu sistema
- [ ] Webhook configurado, com assinatura verificada sobre o corpo cru
- [ ] Handler do webhook idempotente e respondendo `2xx` antes de processar
- [ ] Polling em `GET /charges/:id` como fallback do webhook
- [ ] `429` e `5xx` tratados com backoff exponencial
- [ ] Fluxo inteiro testado em sandbox, com `simulate-payment`

---

## Recursos

| | |
|---|---|
| **Cobranças Pix** | QR dinâmico, copia e cola, confirmação instantânea por webhook |
| **Boleto registrado** | Linha digitável, código de barras e PDF, compensação por webhook |
| **Cartão de crédito** | Tokenização no navegador (PCI fora do seu servidor), aprovação síncrona, até 18x sem juros |
| **Links de pagamento** | Cobrança sem código, página hospedada, white-label opcional |
| **Webhooks** | HMAC-SHA256, 5 tentativas com backoff, histórico de entregas |
| **Estorno** | Total ou parcial, via API |
| **Saques** | Pix e TED, com destinos salvos e liquidação automática |
| **Disputas** | Contestação com evidências, prazo e estatísticas |
| **Relatórios** | Transações, resumo, volume diário e distribuição por método, em JSON ou CSV |
| **Sandbox** | Ambiente completo, com simulação de pagamento |

---

## Links

| | |
|---|---|
| Site | [pagniv.com](https://pagniv.com) |
| Documentação completa | [devs.pagniv.com](https://devs.pagniv.com) |
| Dashboard | [portal.pagniv.com](https://portal.pagniv.com) |
| GitHub | [github.com/PagnivGit](https://github.com/PagnivGit) |

Este README cobre a integração inteira. [devs.pagniv.com](https://devs.pagniv.com) traz a referência navegável, com exemplos por linguagem e a opção de abrir qualquer página direto num assistente de IA.

---

<p align="center">
  Feito pela equipe <strong>Pagniv</strong><br/>
  <a href="https://pagniv.com">pagniv.com</a>
</p>
