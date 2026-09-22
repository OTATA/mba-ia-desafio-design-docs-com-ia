# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Autor** | Larissa (Tech Lead), com Bruno (Pedidos) e Diego (Plataforma) |
| **Status** | Pronto para implementação |
| **Data** | 2026-09-22 |
| **Documentos relacionados** | [PRD](PRD.md) · [RFC](RFC.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |
| **Fonte** | [`TRANSCRICAO.md`](../TRANSCRICAO.md) e o código da aplicação |

> Este documento detalha **como construir**. As decisões que o sustentam estão nos [ADRs](adrs/); a proposta em nível de arquitetura e as questões em aberto estão no [RFC](RFC.md).

---

## 1. Contexto e motivação técnica

A aplicação é um Order Management System em Node.js + TypeScript (ESM), Express 4, Prisma 5 sobre MySQL 8, com Zod para validação e Pino para log. O ciclo de vida do pedido é controlado por uma máquina de estados explícita em `src/modules/orders/order.status.ts`, e a mudança de status é executada em transação em `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), que hoje:

1. carrega o pedido com os itens;
2. rejeita transição igual (`ConflictError` / `INVALID_STATUS_TRANSITION`) ou inválida (`InvalidStatusTransitionError`);
3. debita estoque em `PENDING → PAID` e repõe em `PAID|PROCESSING → CANCELLED`;
4. atualiza `orders`;
5. insere em `order_status_history`;
6. relê o pedido com relações e retorna.

Não existe hoje nenhum ponto de extensão para efeitos externos: nem eventos, nem filas, nem hooks. A feature precisa criar esse ponto **sem** introduzir I/O de rede dentro da transação `[09:04] Bruno` e **sem** permitir que status mude sem evento registrado `[09:40] Bruno`.

O caminho escolhido é o padrão Outbox ([ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md)) consumido por um worker em processo separado ([ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md)).

---

## 2. Objetivos técnicos

1. Registrar o evento de mudança de status **atomicamente** com a própria mudança, usando a transação Prisma já existente.
2. Entregar o evento por HTTP POST ao endpoint do cliente em **até 10 segundos** no caminho feliz, com no máximo 2s de espera de polling `[09:02] Marcos`, `[09:10] Larissa`.
3. Tolerar indisponibilidade do cliente por até ~15 horas via retry com backoff, e produzir um estado terminal explícito (DLQ) quando esgotar `[09:17] Diego`.
4. Assinar todo envio com HMAC-SHA256 usando secret exclusiva do endpoint, com rotação sem downtime `[09:22] Sofia`.
5. Expor CRUD de configuração, histórico de entregas e replay administrativo de DLQ, seguindo o padrão de módulo do projeto.
6. Não introduzir dependência nova no `package.json` para o caminho principal (HMAC via `node:crypto`).

---

## 3. Escopo e exclusões

### 3.1 No escopo

- Models Prisma e migration para `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`.
- Módulo `src/modules/webhooks/` com controller, service, repository, routes e schemas.
- Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada por `OrderService.changeStatus`.
- Entry-point `src/worker.ts` + script `npm run worker`, com a lógica em `src/modules/webhooks/webhook.worker.ts`.
- Assinatura HMAC-SHA256, geração e rotação de secret com grace period.
- Retry com backoff, DLQ e endpoint administrativo de replay.
- Métricas, logs estruturados e correlação de request.

### 3.2 Fora do escopo desta fase

| Item | Origem |
| --- | --- |
| Webhooks inbound (cliente → plataforma) | `[09:02] Marcos`, `[09:03] Sofia` |
| Notificação por email quando o webhook do cliente falha | `[09:37] Larissa` — adiado para próxima fase |
| Dashboard/painel visual para o cliente | `[09:40] Larissa` — projeto separado do time de frontend |
| Rate limiting de saída por cliente | `[09:39] Diego` — observar e decidir depois |
| Arquivamento/expurgo de linhas entregues da outbox | `[09:08] Diego` — fora do escopo da feature |
| Múltiplos workers, particionamento por `order_id`, lock pessimista | `[09:13] Diego` — "problema do futuro" |
| Entrega exactly-once | `[09:25] Diego` — at-least-once é o contrato |
| Endurecimento de roles no CRUD de configuração | `[09:37] Sofia` — "mais pra frente" |

---

## 4. Modelo de dados

Quatro models novos em `prisma/schema.prisma`. Nenhuma tabela existente é alterada. Todas as PKs são UUID `Char(36)`, seguindo o padrão de todo o schema `[09:51] Larissa`.

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  FAILED
  DELIVERED
}

model WebhookEndpoint {
  id              String    @id @default(uuid()) @db.Char(36)
  customerId      String    @db.Char(36)
  url             String    @db.VarChar(500)   // https obrigatório (validado no Zod)
  secret          String    @db.VarChar(128)   // secret ativa
  previousSecret  String?   @db.VarChar(128)   // válida até previousSecretExpiresAt
  previousSecretExpiresAt DateTime?            // rotação: now + 24h
  eventStatuses   Json                         // lista de OrderStatus escutados
  active          Boolean   @default(true)
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  customer   Customer         @relation(fields: [customerId], references: [id])
  outbox     WebhookOutbox[]
  deliveries WebhookDelivery[]

  @@index([customerId])
  @@index([active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)  // = event_id / X-Event-Id
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)
  eventType     String              @db.VarChar(64)   // "order.status_changed"
  payload       Json                                   // snapshot renderizado na inserção
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(500)
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id])

  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@index([orderId])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id                String   @id @default(uuid()) @db.Char(36)
  webhookId         String   @db.Char(36)
  outboxId          String   @db.Char(36)
  attempt           Int
  requestPayload    Json
  responseStatus    Int?
  responseBody      String?  @db.Text          // truncado
  durationMs        Int
  success           Boolean
  errorCode         String?  @db.VarChar(64)   // WEBHOOK_*
  createdAt         DateTime @default(now())

  webhook WebhookEndpoint @relation(fields: [webhookId], references: [id])

  @@index([webhookId, createdAt])
  @@index([outboxId])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id           String   @id @default(uuid()) @db.Char(36)
  outboxId     String   @db.Char(36)
  webhookId    String   @db.Char(36)
  orderId      String   @db.Char(36)
  eventType    String   @db.VarChar(64)
  payload      Json
  failureReason String  @db.VarChar(500)
  attempts     Int
  replayedAt   DateTime?
  replayedById String?  @db.Char(36)
  createdAt    DateTime @default(now())

  @@index([webhookId])
  @@index([createdAt])
  @@map("webhook_dead_letter")
}
```

Notas de modelagem:

- O índice `@@index([status, nextAttemptAt])` é o que sustenta a query de polling. A reunião pediu índice em status e em `created_at` `[09:08] Diego`; `nextAttemptAt` é a coluna efetivamente filtrada por causa do backoff, e `createdAt` mantém o índice para ordenação e para as consultas por período.
- Uma linha de outbox por **par (evento, endpoint)**. O filtro de status já foi aplicado na inserção, então toda linha existente é trabalho real ([ADR-008](adrs/ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md)).
- `WebhookOutbox.id` **é** o `event_id` publicado em `X-Event-Id` ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)).
- `previousSecret` + `previousSecretExpiresAt` implementam o grace period de 24h `[09:21] Sofia`.

---

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox

Acontece dentro da transação de `OrderService.changeStatus`, imediatamente após a escrita em `order_status_history` e antes do reload final do pedido.

```
publishWebhookEvent(tx, order, fromStatus, toStatus)
  │
  ├─ 1. endpoints = tx.webhookEndpoint.findMany({
  │        where: { customerId: order.customerId, active: true } })
  │
  ├─ 2. interessados = endpoints.filter(e => e.eventStatuses.includes(toStatus))
  │        └─ se vazio → retorna sem inserir nada            [ADR-008]
  │
  ├─ 3. payload = renderEventPayload(order, fromStatus, toStatus)   [ADR-007]
  │        └─ snapshot: event_id, event_type, timestamp, order_id,
  │           order_number, from_status, to_status, customer_id, total_cents
  │
  ├─ 4. valida tamanho: Buffer.byteLength(JSON.stringify(payload)) <= 64 KiB
  │        └─ se exceder → throw WebhookPayloadTooLargeError
  │           (rollback da transação inteira)                 [09:23-09:24]
  │
  └─ 5. para cada endpoint interessado:
           tx.webhookOutbox.create({ status: PENDING, attempts: 0,
                                     nextAttemptAt: now, payload, ... })
```

Pontos críticos:

- **Recebe o `tx`, não abre transação própria.** A assinatura `publishWebhookEvent(tx, order, fromStatus, toStatus)` foi decidida na reunião justamente para evitar injetar um repository inteiro no `OrderService` `[09:41] Bruno`, `[09:41] Diego`. O tipo do primeiro parâmetro é `Prisma.TransactionClient`, o mesmo `TxClient` já aliasado em `src/modules/orders/order.service.ts:24`.
- **Nenhum I/O de rede.** Apenas leitura e escrita no mesmo banco, na mesma conexão da transação.
- **Qualquer erro aqui reverte a mudança de status.** É o comportamento desejado `[09:40] Bruno`.
- O `event_id` é o UUID gerado pelo Prisma como PK da linha de outbox. Quando o payload precisa conter o `event_id` (e precisa), o UUID é gerado na aplicação com `uuid` — dependência já presente no projeto e usada em `src/middlewares/request-logger.middleware.ts` — e passado explicitamente como `id`.

### 5.2 Processamento pelo worker

Entry-point `src/worker.ts`, espelhando `src/server.ts`: carrega `env`, cria o `PrismaClient` via `createPrismaClient()` de `src/config/database.ts`, instancia o processador e registra handlers de `SIGINT`/`SIGTERM` para shutdown gracioso.

```
loop a cada WEBHOOK_POLL_INTERVAL_MS (default 2000)      [09:09 Diego]
  │
  ├─ 1. batch = SELECT * FROM webhook_outbox
  │             WHERE status = 'PENDING' AND next_attempt_at <= NOW()
  │             ORDER BY created_at ASC
  │             LIMIT WEBHOOK_BATCH_SIZE (default 20)
  │
  ├─ 2. marca o batch como PROCESSING (UPDATE ... WHERE status='PENDING')
  │        └─ guarda contra reentrada se um ciclo demorar mais que o intervalo
  │
  └─ 3. para cada evento, sequencialmente (single-worker → ordering)  [09:12 Diego]
         │
         ├─ a. body = JSON.stringify(outbox.payload)   // snapshot, byte a byte
         ├─ b. timestamp = new Date().toISOString()
         ├─ c. signature = HMAC-SHA256(endpoint.secret, body) em hex
         ├─ d. POST endpoint.url
         │       headers: X-Event-Id, X-Signature, X-Timestamp,
         │                X-Webhook-Id, Content-Type: application/json
         │       timeout: 10s                                    [09:42 Diego]
         ├─ e. grava WebhookDelivery (attempt, status, body truncado, durationMs)
         └─ f. 2xx  → outbox.status = DELIVERED
                erro → ver 5.3
```

Regras do ciclo:

- **Ordem de `created_at`** garante ordering por `order_id` enquanto houver um único worker `[09:12] Diego`. Escalar quebra a garantia — limitação conhecida `[09:13] Larissa`.
- **Processamento sequencial dentro do batch.** Paralelizar dentro do worker reintroduziria o problema de ordering que [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) evita.
- **Sucesso = resposta HTTP 2xx.** Qualquer outro código, timeout ou erro de rede conta como falha.
- No boot, eventos deixados em `PROCESSING` por um crash anterior voltam para `PENDING` (o evento não se perde; no pior caso é reenviado, o que é coberto pelo contrato at-least-once de [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)).
- O HTTP client é o `fetch` nativo do Node 20 (`engines.node >= 20` no `package.json`), com `AbortSignal.timeout(10_000)`. Nenhuma dependência nova.

### 5.3 Retry

```
falha na tentativa N (N = attempts + 1)
  │
  ├─ registra WebhookDelivery { success: false, errorCode, responseStatus }
  │
  ├─ se N < 5:
  │     attempts    = N
  │     nextAttemptAt = now + BACKOFF[N]
  │     status      = PENDING
  │     lastError   = <motivo truncado em 500 chars>
  │
  └─ se N == 5:  → DLQ (5.4)
```

`BACKOFF = [1min, 5min, 30min, 2h, 12h]`, ou seja, ~14h36 acumulados entre a primeira falha e a última tentativa `[09:17] Diego`, `[09:17] Larissa`. As 5 tentativas incluem a primeira: uma entrega falha 4 vezes antes de esgotar o backoff e ir para a DLQ na quinta.

Os valores vivem em constante do módulo (`WEBHOOK_BACKOFF_SCHEDULE_MS`), não em env var: são decisão de produto registrada em [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md), não configuração de ambiente.

### 5.4 DLQ

```
esgotadas as 5 tentativas
  │
  ├─ transação:
  │    ├─ INSERT webhook_dead_letter { outboxId, webhookId, orderId,
  │    │                               eventType, payload, failureReason,
  │    │                               attempts: 5 }
  │    └─ UPDATE webhook_outbox SET status = 'FAILED'
  │
  └─ log nível error, code = WEBHOOK_MAX_ATTEMPTS_EXCEEDED
```

A linha da outbox fica como `FAILED` (rastro do que aconteceu), mas sai do conjunto varrido pelo polling, que filtra por `PENDING`. A DLQ é a fila de trabalho humano `[09:18] Diego`.

**Replay:** `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (role `ADMIN`) cria uma **nova** linha de outbox com o mesmo `payload` e `status = PENDING`, `attempts = 0`, e marca `replayedAt`/`replayedById` na linha da DLQ `[09:18] Diego`, `[09:35] Diego`. O `event_id` no corpo permanece o original, de modo que o cliente que já processou o evento consegue deduplicar `[09:25] Diego`. O replay é logado com o id do usuário que o executou, para auditoria `[09:36] Sofia`.

---

## 6. Contratos públicos

**Prefixo:** todas as rotas ficam sob `/api/v1`, montado em `src/app.ts:67`.
**Autenticação:** `Authorization: Bearer <jwt>` em todas as rotas, via `authenticate` (`src/middlewares/auth.middleware.ts`). O replay de DLQ exige adicionalmente `requireRole('ADMIN')` `[09:36] Larissa`.
**Convenção de nomes:** as rotas de gestão usam **camelCase** no corpo, como todo o resto da API (`src/modules/orders/order.schemas.ts`). O **payload outbound** para o cliente usa **snake_case**, conforme o formato definido na reunião `[09:43] Diego`.
**Envelope de erro:** `{ "error": { "code", "message", "details"? } }`, produzido por `src/middlewares/error.middleware.ts` sem nenhuma alteração `[09:29] Bruno`.

### 6.1 `POST /api/v1/webhooks` — cadastrar endpoint

Cria o cadastro e devolve a secret. **Esta é a única resposta em que a secret aparece** `[09:31] Marcos`.

Request:

```json
{
  "customerId": "6f1b1f2a-8d3c-4c44-9a71-0f2d5a1c9b10",
  "url": "https://hooks.atlascomercial.com.br/oms/orders",
  "eventStatuses": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:

```json
{
  "id": "0d5e7c61-2a4b-4d8e-9f31-7c1a2b3d4e5f",
  "customerId": "6f1b1f2a-8d3c-4c44-9a71-0f2d5a1c9b10",
  "url": "https://hooks.atlascomercial.com.br/oms/orders",
  "eventStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_9f2c1d8a4b6e0f3c7d5a2b9e1c4f8a6d",
  "createdAt": "2026-09-22T13:40:11.482Z"
}
```

| Código | Quando |
| --- | --- |
| `201` | Criado |
| `400` | `VALIDATION_ERROR` (Zod) ou `WEBHOOK_INVALID_URL` (URL não `https`) |
| `401` | Sem token válido |
| `404` | `NOT_FOUND` — `customerId` inexistente |
| `422` | `WEBHOOK_INVALID_EVENT_FILTER` — status fora do enum `OrderStatus` |

Semântica: `eventStatuses` é a lista de status que este endpoint quer ouvir; ela é consultada na inserção da outbox `[09:33] Marcos`, `[09:34] Bruno`. `customerId` vem no corpo, **não do JWT** — o JWT atual representa o usuário operador, não o cliente `[09:32] Bruno`, `[09:32] Larissa`.

### 6.2 `GET /api/v1/webhooks?customerId=...` — listar endpoints de um cliente

Response `200 OK` (formato de `paginated()`, `src/shared/http/response.ts`):

```json
{
  "data": [
    {
      "id": "0d5e7c61-2a4b-4d8e-9f31-7c1a2b3d4e5f",
      "customerId": "6f1b1f2a-8d3c-4c44-9a71-0f2d5a1c9b10",
      "url": "https://hooks.atlascomercial.com.br/oms/orders",
      "eventStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-09-22T13:40:11.482Z",
      "updatedAt": "2026-09-22T13:40:11.482Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

A secret **nunca** é retornada aqui. Códigos: `200`, `400` (query inválida), `401`.

### 6.3 `PATCH /api/v1/webhooks/:id` — editar endpoint

Request (todos os campos opcionais):

```json
{
  "url": "https://hooks.atlascomercial.com.br/v2/oms/orders",
  "eventStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false
}
```

Response `200 OK`: o endpoint atualizado, no mesmo formato de 6.2 (sem secret).

Códigos: `200` · `400` `VALIDATION_ERROR` / `WEBHOOK_INVALID_URL` · `401` · `404` `WEBHOOK_NOT_FOUND` · `422` `WEBHOOK_INVALID_EVENT_FILTER`.

### 6.4 `DELETE /api/v1/webhooks/:id` — remover endpoint

Response `204 No Content`, sem corpo — mesma semântica de `DELETE /orders/:id` (`src/modules/orders/order.controller.ts:48`).

Códigos: `204` · `401` · `404` `WEBHOOK_NOT_FOUND`.

Eventos já enfileirados para um endpoint removido são descartados pelo worker com `WEBHOOK_NOT_FOUND` registrado em `webhook_deliveries`.

### 6.5 `POST /api/v1/webhooks/:id/secret/rotate` — rotacionar secret

Response `200 OK`:

```json
{
  "id": "0d5e7c61-2a4b-4d8e-9f31-7c1a2b3d4e5f",
  "secret": "whsec_3b7d9e1f5a2c8046be1d7f3a9c5e2048",
  "previousSecretExpiresAt": "2026-09-23T13:52:30.114Z"
}
```

Semântica: a nova secret passa a assinar imediatamente; a anterior continua **aceitável do lado do cliente** até `previousSecretExpiresAt` (24h), dando tempo de migrar os sistemas dele `[09:21] Sofia`. Após esse instante a antiga é apagada do registro.

Códigos: `200` · `401` · `404` `WEBHOOK_NOT_FOUND` · `409` `WEBHOOK_ROTATION_IN_PROGRESS` (já existe uma secret anterior dentro do grace period).

### 6.6 `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

Últimas entregas do endpoint, com resultado, payload, resposta e tempo `[09:34] Marcos`. Default `pageSize=100`, teto de 100, alinhado ao `max(100)` já usado em `listOrdersQuerySchema` (`src/modules/orders/order.schemas.ts:25`).

Response `200 OK`:

```json
{
  "data": [
    {
      "id": "b81c0f44-6d2e-4a9b-8c17-5e3f0a2d9b64",
      "outboxId": "e2a7c913-4f65-4a1d-b0c8-9d3e7f1a2b54",
      "attempt": 2,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 412,
      "errorCode": null,
      "requestPayload": {
        "event_id": "e2a7c913-4f65-4a1d-b0c8-9d3e7f1a2b54",
        "event_type": "order.status_changed",
        "timestamp": "2026-09-22T13:58:02.110Z",
        "order_id": "9c4d2e10-77ab-4f39-91de-2b6c8a1f4057",
        "order_number": "ORD-000418",
        "from_status": "PROCESSING",
        "to_status": "SHIPPED",
        "customer_id": "6f1b1f2a-8d3c-4c44-9a71-0f2d5a1c9b10",
        "total_cents": 164500
      },
      "createdAt": "2026-09-22T13:59:03.520Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

Códigos: `200` · `401` · `404` `WEBHOOK_NOT_FOUND`.

### 6.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — reprocessar DLQ

Exige role `ADMIN` `[09:36] Sofia`, `[09:36] Larissa`. Recoloca o evento na outbox como pendente `[09:18] Diego`.

Response `202 Accepted`:

```json
{
  "deadLetterId": "44f0b6c1-9a77-4d52-8b30-1e6c5a2f7d98",
  "outboxId": "7a1e3c58-0b62-4fd9-9c14-8e2d6b3f05a7",
  "status": "PENDING",
  "replayedAt": "2026-09-22T14:10:44.907Z",
  "replayedById": "2f9d4b70-5c81-4a33-bd06-9e1a7c4f2b58"
}
```

Códigos: `202` · `401` `UNAUTHORIZED` · `403` `FORBIDDEN` (role insuficiente, produzido por `requireRole`) · `404` `WEBHOOK_DEAD_LETTER_NOT_FOUND` · `409` `WEBHOOK_ALREADY_REPLAYED`.

### 6.8 Contrato outbound — o que o cliente recebe

`POST <url cadastrada>` — `Content-Type: application/json`, timeout de 10s do nosso lado.

Headers `[09:44] Diego`, `[09:44] Sofia`:

| Header | Conteúdo |
| --- | --- |
| `X-Event-Id` | UUID do evento, estável entre todas as tentativas e no replay `[09:25] Diego` |
| `X-Signature` | `sha256=<hex>` — HMAC-SHA256 do corpo exato, com a secret do endpoint |
| `X-Timestamp` | Instante do envio, ISO 8601, para o cliente detectar replay attack se quiser |
| `X-Webhook-Id` | Id do cadastro de webhook, para clientes com múltiplos endpoints |
| `Content-Type` | `application/json` |

Corpo `[09:43] Diego`:

```json
{
  "event_id": "e2a7c913-4f65-4a1d-b0c8-9d3e7f1a2b54",
  "event_type": "order.status_changed",
  "timestamp": "2026-09-22T13:58:02.110Z",
  "order_id": "9c4d2e10-77ab-4f39-91de-2b6c8a1f4057",
  "order_number": "ORD-000418",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "6f1b1f2a-8d3c-4c44-9a71-0f2d5a1c9b10",
  "total_cents": 164500
}
```

Semântica esperada do cliente:

- Responder **2xx** para confirmar recebimento. Qualquer outra resposta, ou ausência de resposta em 10s, dispara retry.
- **Deduplicar por `event_id`** — a entrega é at-least-once `[09:24] Diego`.
- O corpo é um **snapshot histórico** do pedido no momento da mudança, não o estado atual. Os itens do pedido **não** vêm no payload; quem precisar do detalhe faz `GET /api/v1/orders/:id` `[09:43] Diego`.
- Verificação da assinatura: `HMAC_SHA256(secret, corpo_bruto)` comparado em tempo constante com `X-Signature`. Durante as 24h de grace period de uma rotação, o cliente deve aceitar assinatura válida com a secret nova **ou** com a anterior `[09:21] Sofia`.

---

## 7. Matriz de erros

Todos os códigos do módulo usam prefixo `WEBHOOK_` `[09:29] Larissa`. As classes herdam de `AppError` e das subclasses já existentes em `src/shared/errors/http-errors.ts`, e são exportadas por `src/shared/errors/index.ts` — o `errorMiddleware` as serializa sem alteração `[09:29] Bruno`.

| Código | HTTP | Classe sugerida (base) | Quando ocorre | Ação |
| --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `WebhookNotFoundError` (`NotFoundError`) | Endpoint inexistente em `GET`/`PATCH`/`DELETE`/`rotate`/`deliveries` | Cliente corrige o id |
| `WEBHOOK_INVALID_URL` | 400 | `WebhookInvalidUrlError` (`ValidationError`) | URL não é `https` ou é malformada `[09:23] Sofia` | Cliente corrige a URL |
| `WEBHOOK_SECRET_REQUIRED` | 422 | `WebhookSecretRequiredError` (`UnprocessableEntityError`) | Envio não pode ser assinado: endpoint sem secret ativa | Rotacionar secret |
| `WEBHOOK_INVALID_EVENT_FILTER` | 422 | `WebhookInvalidEventFilterError` (`UnprocessableEntityError`) | `eventStatuses` contém valor fora do enum `OrderStatus` | Cliente corrige a lista |
| `WEBHOOK_ROTATION_IN_PROGRESS` | 409 | `WebhookRotationInProgressError` (`ConflictError`) | Rotação pedida com grace period de 24h ainda ativo | Aguardar expiração |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `WebhookPayloadTooLargeError` (`UnprocessableEntityError`) | Payload renderizado excede 64 KiB `[09:23] Sofia`, `[09:24] Diego` | Investigar: nenhum evento normal chega perto |
| `WEBHOOK_ENDPOINT_INACTIVE` | 409 | `WebhookEndpointInactiveError` (`ConflictError`) | Operação de entrega/rotate sobre endpoint com `active = false` `[09:21] Bruno` | Reativar por `PATCH` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `WebhookDeadLetterNotFoundError` (`NotFoundError`) | Replay de um id inexistente na DLQ | Conferir o id |
| `WEBHOOK_ALREADY_REPLAYED` | 409 | `WebhookAlreadyReplayedError` (`ConflictError`) | Replay de item da DLQ já reprocessado | Nenhuma |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (interno) | — | Cliente não respondeu em 10s `[09:42] Diego` | Retry automático |
| `WEBHOOK_DELIVERY_FAILED` | — (interno) | — | Resposta não-2xx ou erro de rede/TLS | Retry automático |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | — (interno) | — | 5ª tentativa falhou `[09:15] Diego` | Move para DLQ |

Os três últimos não são erros HTTP de API: são códigos registrados em `webhook_deliveries.errorCode`, em `webhook_outbox.lastError` e nos logs do worker. Aparecem para o cliente apenas no histórico de entregas (6.6).

Erros **não** específicos do módulo continuam usando as classes existentes: `UNAUTHORIZED` e `FORBIDDEN` vêm de `authenticate` / `requireRole`, e `VALIDATION_ERROR` vem do `validate()` com Zod.

---

## 8. Estratégias de resiliência

| Mecanismo | Configuração | Origem |
| --- | --- | --- |
| Timeout por tentativa | 10 s (`AbortSignal.timeout`) | `[09:42] Diego` |
| Tentativas | 5, incluindo a primeira | `[09:15]`, `[09:17]` |
| Backoff | 1m → 5m → 30m → 2h → 12h (~14h36 acumuladas) | `[09:17] Diego` |
| Estado terminal | `webhook_dead_letter` + outbox `FAILED` | `[09:18] Diego` |
| Recuperação da DLQ | Manual, endpoint `ADMIN`, auditado | `[09:18]`, `[09:36]` |
| Atomicidade | Evento inserido na mesma transação da mudança de status | `[09:06]`, `[09:40]` |
| Idempotência | `X-Event-Id` estável; dedup no cliente | `[09:25] Diego` |
| Limite de payload | 64 KiB; erro, sem truncagem | `[09:23]`, `[09:24]` |
| Transporte | `https` obrigatório | `[09:23] Sofia` |
| Isolamento de processo | Worker fora da API | `[09:11] Diego` |
| Shutdown gracioso | `SIGINT`/`SIGTERM`: termina o evento em voo, devolve o restante do batch a `PENDING`, `prisma.$disconnect()` | padrão de `src/server.ts:13-21` |
| Recuperação de crash | No boot, `PROCESSING` → `PENDING` | decorrência de `[09:24] Diego` (at-least-once) |

**Fallback:** não há canal alternativo de entrega nesta fase. O email de alerta ao cliente foi explicitamente adiado `[09:37] Larissa`, e o dashboard está fora de escopo `[09:40] Larissa`. O único caminho de recuperação após a DLQ é o replay manual.

**Não implementado por decisão:** rate limiting de saída. Se um cliente tiver 50 mudanças de status em um minuto, receberá 50 chamadas. A posição é observar e implementar se virar problema `[09:38] Diego`, `[09:39] Larissa` — ver Q1 no [RFC](RFC.md).

---

## 9. Observabilidade

Nenhuma ferramenta nova: tudo em cima do Pino já configurado em `src/shared/logger/index.ts` `[09:29] Bruno`.

### 9.1 Métricas

Emitidas como contadores/gauges derivados de log estruturado (`metric` no payload do log), consumíveis pela stack de observabilidade existente:

| Métrica | Tipo | Dimensões | Para que serve |
| --- | --- | --- | --- |
| `webhook_events_published_total` | contador | `event_type`, `to_status` | Volume produzido na outbox |
| `webhook_delivery_attempts_total` | contador | `webhook_id`, `attempt`, `success` | Taxa de sucesso e perfil de retry |
| `webhook_delivery_duration_ms` | histograma | `webhook_id` | Latência do endpoint do cliente; base para revisitar o timeout de 10s |
| `webhook_end_to_end_latency_ms` | histograma | `to_status` | `delivered_at − created_at`: é a métrica que prova o requisito de <10s `[09:02] Marcos` |
| `webhook_outbox_pending` | gauge | — | Tamanho da fila; cresce se o worker estiver parado |
| `webhook_outbox_oldest_pending_age_s` | gauge | — | **Alerta principal.** Sobe quando o worker único cai — o risco de ponto único de [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) |
| `webhook_dead_letter_total` | contador | `webhook_id` | Clientes com falha permanente; substitui, por ora, o alerta por email adiado `[09:37]` |
| `webhook_poll_cycle_duration_ms` | histograma | — | Detecta ciclo mais lento que o intervalo de 2s |

### 9.2 Logs

Eventos estruturados Pino, no padrão de `snake_case` já usado no projeto (`server_started`, `http_request`, `shutdown_initiated`):

| Evento | Nível | Campos |
| --- | --- | --- |
| `webhook_event_published` | info | `event_id`, `webhook_id`, `order_id`, `to_status` |
| `webhook_delivery_attempt` | info | `event_id`, `webhook_id`, `attempt`, `response_status`, `duration_ms`, `success` |
| `webhook_delivery_failed` | warn | `event_id`, `webhook_id`, `attempt`, `error_code`, `next_attempt_at` |
| `webhook_dead_lettered` | error | `event_id`, `webhook_id`, `order_id`, `attempts`, `failure_reason` |
| `webhook_dead_letter_replayed` | warn | `dead_letter_id`, `new_outbox_id`, `user_id` — **exigência de auditoria** `[09:36] Sofia` |
| `webhook_secret_rotated` | warn | `webhook_id`, `user_id`, `previous_secret_expires_at` |
| `webhook_worker_started` / `webhook_worker_stopped` | info | `poll_interval_ms`, `batch_size` |

**Redação obrigatória.** `src/shared/logger/index.ts` tem hoje uma lista fixa de `redactPaths` (`*.password`, `*.passwordHash`, `*.token`, `*.accessToken`, headers de auth). Ela **precisa ser estendida** com `*.secret`, `*.previousSecret` e `*.signature`. Sem isso, a secret gerada em `POST /webhooks` pode vazar em log de erro — risco levantado na reunião a partir de um caso real de cliente `[09:22] Diego`.

O logger do worker usa a mesma factory `createLogger()`, com `base.service` ajustado para distinguir o processo da API nos índices de log.

### 9.3 Tracing e correlação

O projeto já gera um id de correlação por requisição: `requestLogger` (`src/middlewares/request-logger.middleware.ts`) aceita `x-request-id` do cliente ou gera um UUID, grava em `req.id` e devolve em `X-Request-Id`.

A feature estende essa cadeia até a entrega:

1. `publishWebhookEvent` recebe o `requestId` da mudança de status e o grava na linha de outbox (campo `correlationId`, `Char(36)`).
2. O worker inclui `correlation_id` em todos os logs de tentativa daquele evento.
3. A requisição de saída propaga `X-Request-Id` para o cliente.

Com isso, o caminho `PATCH /orders/:id/status` → linha de outbox → tentativas de entrega → DLQ é rastreável por um único identificador, atravessando dois processos. Não há APM distribuído no projeto hoje, e esta feature não introduz um; a correlação por id é o mecanismo de tracing disponível e suficiente para o diagnóstico descrito.

---

## 10. Dependências e compatibilidade

### 10.1 Dependências

| Dependência | Situação |
| --- | --- |
| MySQL 8 (`docker-compose.yml`) | Já existe. Recebe 4 tabelas novas |
| Prisma 5.22 (`@prisma/client`) | Já existe. Nova migration |
| Zod 3.23 | Já existe. Novos schemas |
| Pino 9.5 | Já existe. `redactPaths` estendido |
| `uuid` 11.0 | Já existe, usado em `src/middlewares/request-logger.middleware.ts` |
| `node:crypto` | Nativo. HMAC-SHA256 |
| `fetch` + `AbortSignal.timeout` | Nativo no Node ≥ 20 (`engines` do `package.json`) |
| **Novas dependências npm** | **Nenhuma** |

Novas variáveis de ambiente, a acrescentar ao `envSchema` de `src/config/env.ts` e ao `.env.example`:

| Variável | Default | Uso |
| --- | --- | --- |
| `WEBHOOK_POLL_INTERVAL_MS` | `2000` | Intervalo do polling `[09:09] Diego` |
| `WEBHOOK_BATCH_SIZE` | `20` | Tamanho do batch `[09:08] Diego` |
| `WEBHOOK_HTTP_TIMEOUT_MS` | `10000` | Timeout por tentativa `[09:42] Diego` |
| `WEBHOOK_MAX_PAYLOAD_BYTES` | `65536` | Teto de 64 KiB `[09:24] Diego` |

O worker reutiliza `DATABASE_URL`, `LOG_LEVEL` e `NODE_ENV` já validados. Note que `env.ts` roda `process.exit(1)` se a validação falhar — o worker herda esse comportamento, o que é adequado para um processo supervisionado.

### 10.2 Compatibilidade

- **Retrocompatível.** Nenhum contrato existente muda. `PATCH /orders/:id/status` mantém request e response idênticos; o efeito colateral é invisível para quem chama.
- **Sem webhooks cadastrados, nada muda.** O filtro na inserção ([ADR-008](adrs/ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md)) faz com que uma base sem endpoints não insira nenhuma linha. A única diferença mensurável é uma query a mais na transação.
- **A migration é aditiva:** só `CREATE TABLE`. Não altera `orders`, `order_items`, `order_status_history`, `customers`, `products` nem `users`.
- **Deploy:** a API pode subir antes do worker. Eventos acumulam como `PENDING` e são drenados quando o worker entrar no ar.
- **Rollback:** desativar o worker interrompe as entregas sem afetar a API. Reverter a alteração em `OrderService` interrompe a produção de eventos. As tabelas podem permanecer vazias sem impacto.
- **Testes existentes:** `tests/orders.test.ts` exercita `changeStatus` em todos os caminhos (transição válida, inválida, estoque insuficiente, cancelamento). Como nenhum cenário tem webhook cadastrado, todos devem continuar passando sem alteração — isso é, na prática, o teste de regressão da retrocompatibilidade.

---

## 11. Integração com o sistema existente

> Seção obrigatória. Todos os caminhos abaixo existem no repositório.

### 11.1 `src/modules/orders/order.service.ts` — o ponto crítico

É a **única alteração em código de produção existente no caminho crítico** `[09:40] Bruno`.

Em `changeStatus` (linhas 126-179), dentro do `this.prisma.$transaction(async (tx) => { ... })`, entre a criação do `orderStatusHistory` (linha 159) e o reload do pedido (linha 169), entra:

```ts
await publishWebhookEvent(tx, order, from, to);
```

Detalhes:

- A função recebe o **`tx` da transação em andamento**, cujo tipo é `Prisma.TransactionClient` — exatamente o alias `TxClient` já declarado na linha 24 do arquivo. Foi a forma escolhida na reunião para evitar injetar um repository inteiro no `OrderService` `[09:41] Bruno`, `[09:41] Diego`.
- O objeto `order` carregado na linha 132 já traz `customerId`, `orderNumber`, `totalCents` e `status` — tudo o que o snapshot precisa `[09:43] Diego`. Não é necessária consulta adicional ao pedido.
- Posição importa: depois das validações de transição (linhas 140-149) e da movimentação de estoque (151-156), de modo que nenhum evento seja produzido para uma transição que será rejeitada.
- Uma exceção lançada pela função propaga e reverte a transação inteira — inclusive estoque e histórico. É o comportamento exigido `[09:40] Bruno`.
- `OrderService.create` (linha 50) também escreve histórico, com `fromStatus: null` e `toStatus: PENDING`. A reunião tratou exclusivamente de *mudança* de status `[09:40] Bruno`; a criação do pedido **não** publica evento nesta fase.

### 11.2 `src/shared/errors/http-errors.ts`, `app-error.ts` e `index.ts` — erros

As classes de webhook seguem o padrão de `InsufficientStockError` e `InvalidStatusTransitionError`: herdam de uma subclasse HTTP, fixam o `errorCode` e carregam `details` estruturados.

```ts
export class WebhookNotFoundError extends NotFoundError { /* WEBHOOK_NOT_FOUND */ }

export class WebhookPayloadTooLargeError extends UnprocessableEntityError {
  constructor(sizeBytes: number, maxBytes: number) {
    super('Webhook payload exceeds the maximum allowed size',
          'WEBHOOK_PAYLOAD_TOO_LARGE', { sizeBytes, maxBytes });
  }
}
```

Todas são reexportadas em `src/shared/errors/index.ts`, junto com as existentes. O contrato `AppError` (`statusCode`, `errorCode`, `details`) de `src/shared/errors/app-error.ts` não muda.

> Observação: `NotFoundError` hoje tem construtor `(resource = 'Resource')` e fixa o código `NOT_FOUND`. Para produzir `WEBHOOK_NOT_FOUND`, `WebhookNotFoundError` deve estender `AppError` diretamente (404 + código próprio) ou `NotFoundError` precisa aceitar um código opcional. A segunda opção altera uma classe compartilhada; **a primeira é a recomendada**, por não tocar em código existente.

### 11.3 `src/middlewares/error.middleware.ts` — tratamento centralizado

**Nenhuma alteração.** O middleware já trata qualquer `AppError` (linha 15), serializando `code`, `message` e `details`, e já cobre `ZodError` e `Prisma.PrismaClientKnownRequestError`. Os erros do módulo de webhooks são capturados de graça `[09:29] Bruno`.

### 11.4 `src/middlewares/auth.middleware.ts` — autenticação e autorização

`authenticate` protege todas as rotas do módulo, aplicado com `router.use(authenticate)` no topo do router, como em `src/modules/orders/order.routes.ts:14`.

`requireRole('ADMIN')` é aplicado **apenas** na rota de replay de DLQ `[09:36] Larissa`. O `ForbiddenError` resultante já produz `403 FORBIDDEN` pelo middleware de erro. O CRUD de configuração aceita qualquer role autenticada nesta fase `[09:37] Sofia`.

O `req.user.id` fornecido pelo middleware é o valor gravado em `replayedById` e no log `webhook_dead_letter_replayed`, atendendo à exigência de auditoria `[09:36] Sofia`.

### 11.5 `prisma/schema.prisma` — modelo de dados

Recebe o enum `WebhookOutboxStatus` e os quatro models da seção 4, seguindo as convenções do arquivo: `@id @default(uuid()) @db.Char(36)`, `@@map` para nome de tabela em snake_case, `createdAt`/`updatedAt` padrão e `@@index` explícito. `WebhookEndpoint.customerId` referencia `Customer`, que ganha a relação inversa `webhooks WebhookEndpoint[]` — única linha adicionada a um model existente. `OrderStatus`, já declarado no arquivo, é o vocabulário de `eventStatuses`.

### 11.6 `src/config/database.ts` e `src/server.ts` — o processo do worker

`src/worker.ts` é criado espelhando `src/server.ts`: função `bootstrap()`, log de início, handlers de `SIGINT`/`SIGTERM` com `prisma.$disconnect()`, e `catch` final com `logger.fatal` + `process.exit(1)`.

A diferença é a origem do client: `src/server.ts` importa o singleton `prisma`; o worker chama **`createPrismaClient()`**, a factory já exportada em `src/config/database.ts`, obtendo instância própria. Mesmo banco, mesma `DATABASE_URL`, processo diferente `[09:29] Diego`, `[09:30] Bruno`.

Acompanha o script em `package.json`, no padrão dos existentes:

```json
"worker": "tsx watch --env-file=.env src/worker.ts",
"worker:start": "node --env-file=.env dist/worker.js"
```

### 11.7 `src/app.ts` e `src/routes/index.ts` — composição

`buildControllers` (`src/app.ts:26`) instancia `WebhookRepository` → `WebhookService` → `WebhookController`, na mesma sequência dos demais módulos, e adiciona `webhooks` ao tipo `Controllers` (`src/routes/index.ts:13`). `buildApiRouter` registra `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` e `router.use('/admin/webhooks', buildAdminWebhookRouter(controllers.webhooks))`, herdando o prefixo `/api/v1` de `src/app.ts:67`.

### 11.8 `src/middlewares/validate.middleware.ts` e `src/shared/http/response.ts` — entrada e saída

Os schemas de `webhook.schemas.ts` são aplicados com `validate({ body, params, query })`, como em `src/modules/orders/order.routes.ts:16-24`. A validação de `https` usa Zod (`z.string().url().startsWith('https://')`), conforme decidido `[09:23] Sofia`. As listagens usam `paginated()` de `src/shared/http/response.ts`, produzindo o mesmo envelope `{ data, pagination }` do resto da API.

### 11.9 `src/modules/orders/order.status.ts` — vocabulário de eventos

Os valores aceitos em `eventStatuses` são os do enum `OrderStatus`. O arquivo também informa quais transições são possíveis (`transitions`), o que serve de base para os testes: não faz sentido cadastrar um filtro esperando um evento para uma transição que a máquina de estados nunca produz.

### 11.10 `src/shared/logger/index.ts` — log

Reuso direto de `logger` e `createLogger()`. A única alteração é a extensão de `redactPaths` com `*.secret`, `*.previousSecret` e `*.signature` (ver 9.2).

---

## 12. Critérios de aceite técnicos

### Outbox e transação

- [ ] Mudança de status com 1 endpoint ativo que escuta o `toStatus` cria exatamente 1 linha em `webhook_outbox`, no mesmo commit.
- [ ] Mudança de status com 2 endpoints interessados cria 2 linhas, uma por endpoint.
- [ ] Mudança de status **sem** endpoint interessado não cria nenhuma linha `[09:34] Bruno`.
- [ ] Falha forçada na inserção da outbox reverte `orders`, `order_status_history` **e** o estoque `[09:40] Bruno`.
- [ ] Transição rejeitada (`INVALID_STATUS_TRANSITION` ou `INSUFFICIENT_STOCK`) não produz linha de outbox.
- [ ] Payload gravado é snapshot: alterar o pedido depois não altera o payload enfileirado `[09:52] Larissa`.
- [ ] Payload acima de 64 KiB resulta em `WEBHOOK_PAYLOAD_TOO_LARGE` e rollback `[09:24] Larissa`.

### Worker e entrega

- [ ] Worker sobe como processo independente por `npm run worker` `[09:11] Larissa`.
- [ ] Derrubar a API não interrompe o worker, e vice-versa `[09:11] Diego`.
- [ ] Evento pendente é entregue em até 2s + tempo de resposta do cliente, com latência ponta a ponta abaixo de 10s no caminho feliz `[09:02] Marcos`.
- [ ] Três mudanças de status do mesmo pedido em sequência rápida chegam ao cliente na ordem `created_at` `[09:12] Diego`.
- [ ] Resposta 2xx marca o evento como `DELIVERED` e registra a tentativa em `webhook_deliveries`.
- [ ] Cliente que não responde em 10s produz `WEBHOOK_DELIVERY_TIMEOUT` e reagendamento `[09:42] Diego`.
- [ ] Crash do worker no meio de um batch não perde eventos: no boot, `PROCESSING` volta a `PENDING`.

### Retry e DLQ

- [ ] Falhas sucessivas agendam a próxima tentativa em 1m, 5m, 30m, 2h e 12h, nessa ordem `[09:17] Diego`.
- [ ] A 5ª falha insere em `webhook_dead_letter` com payload, motivo e timestamp, e marca a outbox como `FAILED` `[09:18] Diego`.
- [ ] Evento em DLQ não é mais varrido pelo polling.
- [ ] `POST /admin/webhooks/dead-letter/:id/replay` com role `ADMIN` recria a linha de outbox como `PENDING` `[09:18] Diego`.
- [ ] O mesmo endpoint com role `OPERATOR` retorna `403 FORBIDDEN` `[09:36] Sofia`.
- [ ] O replay é logado com o id do usuário que o executou `[09:36] Sofia`.

### Segurança

- [ ] `X-Signature` confere com `HMAC_SHA256(secret_do_endpoint, corpo_bruto)` `[09:20] Sofia`.
- [ ] Endpoints diferentes, do mesmo cliente ou de clientes distintos, têm secrets diferentes `[09:21] Sofia`.
- [ ] Cadastro com URL `http://` é recusado com `WEBHOOK_INVALID_URL` `[09:23] Sofia`.
- [ ] Após rotação, envios usam a nova secret, e a anterior consta com expiração em 24h `[09:21] Sofia`.
- [ ] Decorrido o grace period, a secret anterior é removida do registro.
- [ ] A secret aparece **apenas** na resposta de criação e na de rotação; nunca em `GET`, nunca em log `[09:22] Diego`.

### Contrato e integração

- [ ] Todas as tentativas do mesmo evento enviam o mesmo `X-Event-Id` `[09:25] Diego`.
- [ ] Os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type` estão presentes em todo envio `[09:44]`.
- [ ] O payload não contém os itens do pedido `[09:43] Diego`.
- [ ] Todos os códigos de erro do módulo começam com `WEBHOOK_` `[09:29] Larissa`.
- [ ] `tests/orders.test.ts` passa sem alteração.
- [ ] `npm run lint` e `npm run build` passam.

---

## 13. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| R1 | Erro no módulo de webhooks derruba mudanças de status, por estar dentro da transação crítica | Média | Alto | Superfície mínima na transação (1 read + N inserts, sem rede); testes de integração cobrindo os caminhos de `changeStatus`; rollback do `publishWebhookEvent` reversível por feature flag de ambiente |
| R2 | Worker único indisponível: entregas param sem sintoma na API | Média | Alto | Alerta sobre `webhook_outbox_oldest_pending_age_s`; eventos permanecem na outbox e drenam na recuperação; supervisão de processo no deploy |
| R3 | Secret vazada em log da plataforma | Baixa | Alto | Extensão obrigatória de `redactPaths` em `src/shared/logger/index.ts`; secret ausente de todas as respostas exceto criação e rotação; revisão de segurança dedicada `[09:46] Sofia` |
| R4 | Crescimento indefinido de `webhook_outbox` e `webhook_deliveries` sem política de expurgo `[09:08]` | Alta | Médio | Filtro na inserção reduz o volume; monitoramento de tamanho de tabela; retenção fica como questão aberta Q4 do [RFC](RFC.md) |
| R5 | Transação de `changeStatus` fica mais lenta com a leitura extra de endpoints | Média | Médio | Índice `@@index([customerId])` em `webhook_endpoints`; a consulta é por chave indexada e retorna poucas linhas; medir `PATCH /orders/:id/status` antes e depois |
| R6 | Cliente sem deduplicação processa evento repetido | Média | Médio | `X-Event-Id` em todas as tentativas; documentação destacada no portal `[09:26] Marcos` |
| R7 | Evento em backoff longo é entregue fora de ordem em relação a eventos posteriores do mesmo pedido | Média | Baixo | Limitação conhecida e documentada; `timestamp` e `from_status`/`to_status` no payload permitem ao cliente reconstruir a ordem |
| R8 | Rajada de eventos para um único cliente (50 em um minuto) sobrecarrega o endpoint dele `[09:38]` | Média | Médio | Monitorar `webhook_delivery_attempts_total` por `webhook_id`; rate limiting só se virar problema real `[09:39] Diego` |
