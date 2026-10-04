# Validação cadastral (KYC etapa 2)

Motor de risco proprietário que roda depois da coleta documental e antes de o
seller entrar em processamento. Valida consistência documental, situação
cadastral do CNPJ e titularidade bancária, e produz um parecer estruturado e
auditável.

Este documento é interno. A API pública (`/v1`) está no
[README da raiz](../../../README.md); nada daqui é exposto a merchant.

## Arquivos

| Arquivo | O que faz |
|---|---|
| `validacao-cadastral.ts` | Validações determinísticas, sem provedor externo: `validarCPF`, `validarCNPJ`, `normalizarDocumento`, `normalizarNome`, `matchNome` (similaridade de token com limiar configurável) e as máscaras de CPF e conta para log |
| `cnpj-lookup.ts` | Situação cadastral do CNPJ via BrasilAPI (reflete a Receita), com timeout, retry com backoff e cache por CNPJ. Erro de rede vira resultado indisponível, nunca derruba o fluxo |
| `avaliacao.ts` | Lógica pura: roda as checagens e aplica a matriz de risco |
| `kyc-validation.service.ts` | Orquestrador `executar(orgId, { dryRun, triggeredBy })` |
| `adapters.ts` | Pontos de integração para provedores ainda não contratados |

## Matriz de risco

| Resultado | Quando |
|---|---|
| `BLOQUEAR` | Qualquer reprovação |
| `REVISAR` | Divergência não crítica, ou indisponibilidade de um check crítico |
| `OBSERVAR` | O resto |

## Execução

Assíncrona, pela fila BullMQ `kyc-validation` (worker em `src/worker.ts`). Cada
rodada é persistida em `kyc_validations`: o parecer, cada check com status e
motivo, e o responsável. A auditoria é gravada **antes** de qualquer mudança de
status.

O dry-run é controlado pela flag `KYC_DRY_RUN`, **ligada por padrão**: calcula e
registra o parecer sem alterar o `kycStatus`.

## Endpoints (admin)

`authMiddleware` + `adminMiddleware`.

| Método | Rota | O que faz |
|---|---|---|
| `POST` | `/v1/admin/kyc/validations/:orgId` | Dispara a validação (assíncrona, 202). Body opcional `{ dryRun }` |
| `GET` | `/v1/admin/kyc/validations/:orgId` | Histórico de rodadas do seller (trilha de auditoria) |

## Como o modelo genérico foi mapeado na stack

O modelo de dados do motor difere do genérico de mercado:

- `seller` corresponde a `Organization` (razão social em `legalName`).
- O responsável (CPF e nome) vem do `User` com papel `OWNER`.
- Os dados bancários vêm de `WithdrawalDestination`.
- Não há CNAE nem endereço declarados no cadastro, então a checagem de CNAE é
  informativa: registra o que a Receita retorna, sem bloquear.

## Não implementado (os adapters retornam `nao_configurado`)

- **Situação cadastral de CPF na Receita** (`validarSituacaoCPF`): plugar um
  provedor no adapter.
- **Biometria, OCR do documento, prova de vida e face match**
  (`verificarBiometria`): plugar um provedor no adapter.
- **Titularidade bancária** via DICT (chave Pix) ou pelo retorno do provedor de
  pagamento na primeira transferência: o ponto de integração já está comentado
  em `avaliacao.ts`.
