# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **RFC** | 001 |
| **Título** | Sistema de Webhooks de Notificação de Pedidos |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-09-22 |
| **Revisores** | Diego (Eng. Sênior / Plataforma), Bruno (Eng. Pleno / Pedidos), Sofia (Eng. de Segurança), Marcos (Product Manager) |
| **Fonte** | Reunião técnica de webhooks — [`TRANSCRICAO.md`](../TRANSCRICAO.md) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

> Larissa se comprometeu, ao fim da reunião, a abrir o documento de design e marcar uma sessão de revisão com Bruno e Diego antes do início da implementação `[09:50] Larissa`. Este RFC é esse documento.

---

## 1. TL;DR

Propomos entregar notificações de mudança de status de pedido aos clientes B2B via **webhooks outbound**, usando o **padrão Outbox no MySQL que já temos**: a mudança de status e o registro do evento acontecem na mesma transação, e um **worker em processo separado**, em polling de 2 segundos, faz as entregas HTTP.

Falhas são tratadas com **backoff exponencial em 5 tentativas** (1m/5m/30m/2h/12h) e, esgotadas, o evento vai para uma **DLQ em tabela dedicada** com replay manual por endpoint administrativo. Cada requisição é assinada em **HMAC-SHA256** com **secret única por endpoint**, rotacionável com grace period de 24h. A garantia de entrega é **at-least-once**, com deduplicação do lado do cliente via `X-Event-Id`.

Nenhuma infraestrutura nova: mesmo MySQL, mesmo Prisma, mesmos padrões de módulo, erro e log do projeto. Estimativa de **três sprints**, incluindo a revisão de segurança `[09:46] Larissa`.

---

## 2. Contexto e problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente para serem notificados quando o status dos pedidos deles muda `[09:00] Marcos`. Hoje eles fazem polling em `GET /orders`, o que torna a integração lenta e cara do lado deles. A Atlas sinalizou possibilidade de migrar para um concorrente caso a entrega não aconteça até o fim do trimestre `[09:00] Marcos`.

A aplicação não tem hoje **nenhum** mecanismo de notificação externa, eventos ou filas. A mudança de status é uma operação inteiramente local: `OrderService.changeStatus` abre uma transação Prisma, valida a transição contra a máquina de estados (`src/modules/orders/order.status.ts`), movimenta estoque, atualiza `orders` e grava `order_status_history` (`src/modules/orders/order.service.ts:131`).

A restrição de latência é confortável: qualquer coisa **abaixo de 10 segundos** é considerada tempo real pelos clientes; o inaceitável é o evento ficar pendurado `[09:02] Marcos`. O escopo é **outbound apenas** — os clientes querem receber, não enviar `[09:02] Marcos`, `[09:03] Sofia`.

A restrição arquitetural que domina o desenho é outra: **não pode existir o caso de o status mudar e o evento não sair** `[09:40] Bruno`, e ao mesmo tempo **a indisponibilidade de um cliente não pode travar nem reverter a mudança de status** `[09:04] Bruno`. Essas duas exigências, juntas, eliminam tanto o disparo síncrono quanto a publicação pós-commit.

---

## 3. Proposta técnica

### 3.1 Visão geral

```
  PATCH /orders/:id/status
            │
            ▼
  ┌─────────────────────────────────────────────┐
  │  OrderService.changeStatus  (transação)     │
  │  ├─ valida transição (order.status.ts)      │
  │  ├─ movimenta estoque                       │
  │  ├─ UPDATE orders                           │
  │  ├─ INSERT order_status_history             │
  │  └─ publishWebhookEvent(tx, ...)            │
  │       └─ INSERT webhook_outbox (0..N linhas)│
  └─────────────────────────────────────────────┘
            │  commit atômico
            ▼
     [ webhook_outbox ]  ◄── índices: status, created_at
            │
            │  polling 2s, batch pequeno, ordem de created_at
            ▼
  ┌─────────────────────────────────────────────┐
  │  Worker  (processo separado, src/worker.ts) │
  │  ├─ assina payload (HMAC-SHA256)            │
  │  ├─ POST https://cliente/... (timeout 10s)  │
  │  ├─ registra tentativa em webhook_delivery  │
  │  └─ sucesso → DELIVERED                     │
  │     falha   → reagenda (1m/5m/30m/2h/12h)   │
  │     5ª falha→ webhook_dead_letter           │
  └─────────────────────────────────────────────┘
            │                          │
            ▼                          ▼
      cliente B2B            POST /admin/.../replay (ADMIN)
```

### 3.2 Os cinco pilares da proposta

**1. Outbox transacional no MySQL.** A linha de evento é inserida na mesma transação que altera o status. Se o commit aconteceu, o evento existe; se houve rollback, o evento sumiu junto `[09:06] Diego`. Se a inserção do evento falhar, a mudança de status inteira é revertida `[09:40] Bruno`. → [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md)

**2. Worker separado em polling de 2s.** Processo Node próprio (`src/worker.ts` + `npm run worker`), com `PrismaClient` próprio apontando para o mesmo banco. Processo separado porque o worker não pode morrer junto com um restart da API `[09:11] Diego`. Polling porque o MySQL não tem `LISTEN/NOTIFY` e 2 segundos atendem com sobra o teto de 10s `[09:09] Diego`. Instância única por ora, o que dá ordering por `order_id` de graça `[09:12] Diego`. → [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md)

**3. Retry com backoff e DLQ.** Cinco tentativas em 1m/5m/30m/2h/12h, timeout de 10s por tentativa, e DLQ em tabela dedicada com payload, motivo e timestamp `[09:17] Diego`, `[09:18] Diego`, `[09:42] Diego`. Reprocessamento manual via `POST /admin/webhooks/dead-letter/:id/replay`, restrito a role `ADMIN` e auditado `[09:36] Sofia`. → [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)

**4. Segurança por HMAC-SHA256 com secret por endpoint.** Assinatura do corpo enviada em `X-Signature`; secret única por endpoint, gerada pela plataforma e devolvida na criação; rotação self-service com as duas secrets válidas por 24h `[09:20]`–`[09:22] Sofia`. URL obrigatoriamente `https`, validada no schema Zod `[09:23] Sofia`. → [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)

**5. At-least-once com `X-Event-Id`.** UUID gerado na entrada do evento na outbox, constante entre todas as tentativas. O cliente deduplica do lado dele — padrão de mercado adotado por Stripe e GitHub `[09:25] Diego`. → [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)

### 3.3 Superfície de API

Um módulo `src/modules/webhooks/` com a estrutura padrão do projeto expõe o CRUD de configuração (criar, listar, editar, remover), a rotação de secret, o histórico de entregas (`GET /webhooks/:id/deliveries` `[09:34] Marcos`) e o replay administrativo de DLQ. O CRUD exige apenas autenticação; o replay exige role `ADMIN` `[09:36] Larissa`, `[09:37] Sofia`.

Cada endpoint declara a lista de status que quer ouvir, e o filtro é aplicado **na inserção** da outbox — se ninguém escuta aquele status, nenhuma linha é criada `[09:34] Bruno`. → [ADR-008](adrs/ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md)

O payload é um **snapshot** do estado do pedido no momento da mudança, enxuto (sem itens) `[09:43] Diego`, `[09:52] Larissa`. → [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao-do-evento.md)

Os contratos completos — endpoints, payloads de exemplo, headers, status codes e matriz de erros `WEBHOOK_*` — estão no [FDD](FDD.md), seção "Contratos públicos".

### 3.4 Reuso do que já existe

Nada de stack nova. `AppError` e suas subclasses para erros, com prefixo `WEBHOOK_` nos códigos; `errorMiddleware` sem alteração; Pino para log; Zod + `validate()` para entrada; `authenticate` e `requireRole` para acesso; `paginated()` para listagens; Prisma para dados `[09:27]`–`[09:30]`. → [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)

---

## 4. Alternativas consideradas

### 4.1 Disparo síncrono dentro de `changeStatus`

**Proposta:** chamar o webhook do cliente diretamente no service de pedidos, no momento da mudança de status.

**Por que foi descartada:** a transação de mudança de status já é pesada — atualiza `orders`, insere em `order_status_history` e decrementa `stockQuantity`. Acrescentar um HTTP call no meio faz com que qualquer cliente lento trave a mudança de status de outros pedidos `[09:04] Bruno`. Pior, cria um dilema insolúvel: se o cliente está fora do ar, dar rollback na mudança de status é inaceitável `[09:04] Bruno`.

**Trade-off que motivou o descarte:** ganharíamos a implementação mais simples possível, ao custo de acoplar a disponibilidade do fluxo crítico de pedidos à disponibilidade de sistemas de terceiros.

### 4.2 Fila/stream externo (Redis Streams ou equivalente)

**Proposta:** publicar os eventos em um Redis Stream e consumir de lá, em vez de usar uma tabela `[09:07] Larissa`.

**Por que foi descartada:** exigiria subir e operar infraestrutura nova. Com um time pequeno, subir um Redis Cluster para esse volume foi classificado como overengineering; a outbox no MySQL existente resolve `[09:07] Diego`.

**Trade-off que motivou o descarte:** abriríamos mão de throughput e de primitivas de fila maduras em troca de zero custo operacional adicional e de atomicidade nativa com a transação de pedidos — que, com fila externa, voltaria a ser um problema de *dual write*.

### 4.3 Trigger de banco em vez de polling

**Proposta:** usar trigger no MySQL para notificar o worker de forma reativa, evitando o polling `[09:09] Bruno`.

**Por que foi descartada:** o MySQL não tem listener nativo equivalente ao `LISTEN/NOTIFY` do Postgres. Trigger executa SQL, mas não avisa processo externo; qualquer notificação exigiria improviso (escrever em arquivo, bater em endpoint). E o polling de 2s já atende o requisito de latência com folga `[09:09] Diego`.

**Trade-off que motivou o descarte:** reatividade imediata em troca de um mecanismo frágil e não idiomático, para ganhar 2 segundos dentro de um orçamento de 10.

### 4.4 Outras alternativas rejeitadas (resumo)

| Alternativa | Motivo do descarte | Fonte |
| --- | --- | --- |
| 3 tentativas de retry em vez de 5 | Janela curta demais; já houve cliente com 2h de indisponibilidade planejada | `[09:16] Diego` |
| Retry indefinido com backoff | Evento pendurado para sempre se o cliente sumir | `[09:15] Diego` |
| DLQ como flag `failed` na própria outbox | Suja a leitura da outbox principal; tabela separada serve melhor a debug e replay | `[09:18] Diego` |
| Secret global da plataforma | "Se vaza uma, vaza tudo" | `[09:21] Sofia` |
| Exactly-once | Exige coordenação dos dois lados; at-least-once resolve 99% dos casos | `[09:25] Diego` |
| Renderizar o payload na hora do envio | Com retry de horas, o evento descreveria um estado que não é o da mudança | `[09:52] Larissa` |
| Filtrar eventos na entrega em vez da inserção | Gravaria linhas destinadas ao descarte | `[09:34] Bruno` |

---

## 5. Questões em aberto

### Q1 — Rate limiting de envio por cliente

Se um cliente tem 50 pedidos mudando de status em um minuto, hoje ele recebe 50 chamadas seguidas `[09:38] Diego`. A posição registrada é **não implementar agora**: observar em produção e implementar se virar problema `[09:39] Diego`, `[09:39] Larissa`. Fica como ponto a decidir com base em dados reais de volume.

### Q2 — Escala para múltiplos workers e garantia de ordering

O desenho atual depende de **um único worker** para preservar a ordem de entrega por `order_id`. Escalar horizontalmente quebra essa garantia. Os caminhos levantados foram particionar por `order_id` ou usar lock pessimista, mas a discussão foi explicitamente adiada: "problema do futuro, não agora" `[09:13] Diego`. A limitação deve ser documentada como conhecida `[09:13] Larissa`. Não há gatilho definido para reabrir a decisão.

### Q3 — Endurecimento das permissões do CRUD de configuração

O CRUD de webhooks aceita, nesta fase, qualquer role autenticada. Perguntado diretamente, o time de segurança respondeu "por enquanto sim, mais pra frente a gente pode endurecer" `[09:37] Sofia`. Não ficou definido o critério nem o prazo desse endurecimento.

### Q4 — Política de retenção e arquivamento da outbox

A intenção declarada é arquivar linhas entregues "depois de 30 dias ou assim", mas isso foi colocado **fora do escopo desta feature** `[09:08] Diego`. Sem uma política definida, a tabela cresce indefinidamente e o acompanhamento fica por conta da operação.

---

## 6. Impacto e riscos

### 6.1 Impacto no sistema existente

| Área | Impacto |
| --- | --- |
| `src/modules/orders/order.service.ts` | A transação de `changeStatus` ganha uma leitura (endpoints do customer) e 0..N escritas (linhas de outbox). É a única alteração em código de produção existente no caminho crítico. |
| `prisma/schema.prisma` | Quatro novos models e uma migration. Nenhuma alteração em tabela existente. |
| `src/app.ts`, `src/routes/index.ts` | Registro do novo módulo, seguindo o padrão dos demais. |
| `src/shared/errors/*` | Novas classes de erro `WEBHOOK_*`. O middleware de erro não muda `[09:29] Bruno`. |
| Deploy / operação | Um processo novo (`npm run worker`) a subir, monitorar e reiniciar. |

### 6.2 Riscos principais

| Risco | Impacto | Mitigação |
| --- | --- | --- |
| Falha no módulo de webhooks derruba a mudança de status (a inserção da outbox está na transação crítica) | Alto | Superfície mínima na transação: apenas leitura de endpoints + insert; nenhum I/O de rede. Cobertura por testes de integração no fluxo de `changeStatus` |
| Worker único fora do ar: entregas param sem erro visível na API | Alto | Eventos permanecem na outbox e são entregues na recuperação; métrica de idade do evento pendente mais antigo e alerta operacional (ver Observabilidade no FDD) |
| Vazamento de secret pelo cliente — já aconteceu antes `[09:22] Diego` | Alto | Secret por endpoint limita o raio; rotação self-service com grace de 24h `[09:21] Sofia` |
| Crescimento indefinido da tabela de outbox, sem política de arquivamento `[09:08]` | Médio | Filtro na inserção reduz volume ([ADR-008](adrs/ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md)); monitoramento de tamanho; política a definir (Q4) |
| Cliente sem deduplicação processa evento duplicado | Médio | Documentação destacada no portal do desenvolvedor `[09:26] Marcos`; `X-Event-Id` em todas as tentativas |
| Prazo: Atlas ameaça migrar se não houver entrega no trimestre `[09:00] Marcos` | Médio | Estimativa de 3 sprints acordada, com a revisão de segurança de 2 dias já incluída `[09:46]` |

### 6.3 Dependências de cronograma

Sofia pediu **pelo menos dois dias úteis** para revisar o código de segurança — especificamente HMAC e geração de secret — **antes do deploy** `[09:46] Sofia`. A revisão está dentro da estimativa de três sprints `[09:47] Larissa`.

---

## 7. Decisões relacionadas

| ADR | Decisão |
| --- | --- |
| [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) | Padrão Outbox transacional no MySQL existente |
| [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) | Worker em processo separado com polling de 2s |
| [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ dedicada |
| [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação com grace de 24h |
| [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com deduplicação por `X-Event-Id` |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso máximo dos padrões existentes do projeto |
| [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao-do-evento.md) | Payload como snapshot na inserção |
| [ADR-008](adrs/ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md) | Filtro de eventos na inserção da outbox |

---

## 8. Pedido aos revisores

Pontos em que o feedback é mais útil:

1. **Superfície da transação crítica** — a leitura de endpoints dentro de `changeStatus` é aceitável, ou vale pré-carregar/cachear a configuração de webhooks por customer? (Bruno, Diego)
2. **Q1 — rate limiting** — a decisão de "observar e decidir depois" é confortável para a fase 1, ou queremos ao menos um teto defensivo? (Diego)
3. **Q3 — permissões do CRUD** — manter qualquer role autenticada nesta fase é aceitável do ponto de vista de segurança? (Sofia)
4. **Grace period de 24h** — a coexistência de duas secrets válidas está adequadamente modelada? (Sofia)
5. **Escopo de fase 1** — o corte (sem email de alerta, sem dashboard, sem rate limiting) sustenta o compromisso com a Atlas? (Marcos)
