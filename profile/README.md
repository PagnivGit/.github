<p align="center">
  <img src="BANNER_URL" alt="Pagniv — Infraestrutura Pix para negócios que crescem" width="100%" />
</p>

<h1 align="center">Pagniv</h1>

<p align="center">
  <strong>O motor Pix por trás dos negócios que crescem.</strong><br/>
  API confiável, dashboard completo e liquidação automática.
</p>

<p align="center">
  <a href="https://pagniv.com">Site</a> •
  <a href="https://devs.pagniv.com">Documentação</a> •
  <a href="https://portal.pagniv.com">Dashboard</a>
</p>

---

## Sobre

A **Pagniv** é uma plataforma de subadquirência Pix que permite empresas receberem pagamentos instantâneos com total controle. Conecte sua aplicação à nossa infraestrutura em minutos e escale sem preocupações.

- **Integração rápida** — API RESTful. Do zero à primeira cobrança em menos de 30 minutos.
- **Dashboard completo** — Acompanhe cobranças, transações, saques e disputas com métricas em tempo real.
- **Segurança de nível financeiro** — Autenticação por chaves de API, ambientes separados e rate limiting.
- **Webhooks com garantia de entrega** — Notificações em tempo real com retry automático e assinatura HMAC.

---

## Como funciona

| Etapa | Descrição |
|-------|-----------|
| **1. Crie sua conta** | Cadastre-se com os dados da sua empresa e escolha o plano de tarifa adequado ao seu volume. |
| **2. Compliance e validação** | Analisamos o perfil da sua empresa para garantir conformidade regulatória e segurança. |
| **3. Integre e valide** | Conecte sua aplicação, gere cobranças Pix reais e valide o fluxo antes de ativar em produção. |
| **4. Ative e escale** | Com tudo aprovado, sua infraestrutura está no ar. Receba, reconcilie e movimente com total controle. |

---

## Integre em poucas linhas

```typescript
import { Pagniv } from '@pagniv/sdk'

const charge = await pagniv.charges.create({
  amount: 150.00,
  description: 'Pedido #1234',
  payer: {
    name: 'João Silva',
    document: '123.456.789-00'
  }
})

// QR Code pronto em segundos
console.log(charge.qrCode)
```

> **QR Code gerado em menos de 3 segundos. 99.9% de uptime garantido.**

---

## Recursos

| Recurso | Descrição |
|---------|-----------|
| **Cobranças Pix** | Gere QR Codes dinâmicos, acompanhe status e receba confirmação instantânea. |
| **Dashboard** | Visão completa de cobranças, carteira, saques e disputas em tempo real. |
| **Webhooks** | Notificações com retry automático (até 5 tentativas), assinatura HMAC e painel de rastreamento. |
| **Multi-tenant** | Gerencie múltiplas organizações com roles granulares (Owner, Admin, Finance, Viewer). |
| **Sandbox** | Ambiente completo para testes antes de ir para produção. |
| **API Keys** | Chaves separadas por ambiente (sandbox/production) com controle total. |
| **Liquidação automática** | Settlement configurável — defina o prazo e receba automaticamente. |
| **Saques** | Solicite saques direto pelo dashboard com aprovação automática ou manual. |

---

## Segmentos atendidos

E-commerce • iGaming • SaaS/Software • Marketplace • Infoprodutos/EAD • Serviços Financeiros • Saúde • Varejo

---

## Links úteis

| Recurso | URL |
|---------|-----|
| Site institucional | [pagniv.com](https://pagniv.com) |
| Documentação da API | [devs.pagniv.com](https://devs.pagniv.com) |
| Dashboard | [portal.pagniv.com](https://portal.pagniv.com) |

---

<p align="center">
  Feito com dedicação pela equipe <strong>Pagniv</strong><br/>
  <a href="https://pagniv.com">pagniv.com</a>
</p>
