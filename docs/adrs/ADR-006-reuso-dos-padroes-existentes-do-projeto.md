# ADR-006 — Reuso máximo dos padrões e da infraestrutura já existentes no projeto

- **Status:** Aceito
- **Data do registro:** 2026-09-22
- **Decisores:** Bruno (Eng. Pleno / Pedidos), Larissa (Tech Lead), Diego (Eng. Sênior / Plataforma), Sofia (Eng. de Segurança)
- **Fonte:** `TRANSCRICAO.md` `[09:27]`–`[09:30]`, `[09:36]`, e o código da aplicação

> Este é o ADR que ancora o módulo de webhooks no código existente. Todos os caminhos de arquivo citados abaixo existem no repositório.

## Contexto

O módulo de webhooks é o primeiro componente novo de peso desde que a aplicação foi para produção. Há uma tentação natural de trazer bibliotecas e convenções próprias junto com ele — cliente HTTP novo, logger próprio, formato de erro próprio, estrutura de pastas própria.

A codebase, porém, já tem um padrão claro e uniforme, verificável em qualquer módulo existente `[09:27] Bruno`:

- **Estrutura por domínio** em `src/modules/<dominio>/` com `controller`, `service`, `repository`, `routes` e `schemas` — ver `src/modules/orders/` e `src/modules/customers/`.
- **Erros** com `AppError` como base (`src/shared/errors/app-error.ts`), subclasses HTTP em `src/shared/errors/http-errors.ts` e erros de domínio específicos como `InsufficientStockError` e `InvalidStatusTransitionError`, todos carregando um `errorCode` em SCREAMING_SNAKE_CASE (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`) `[09:28] Bruno`.
- **Tratamento centralizado** em `src/middlewares/error.middleware.ts`, que já sabe serializar `AppError`, `ZodError` e `Prisma.PrismaClientKnownRequestError` no envelope `{ error: { code, message, details } }`.
- **Logging** com Pino, instanciado uma vez em `src/shared/logger/index.ts`, com `redact` de campos sensíveis `[09:29] Bruno`.
- **Validação** com Zod, aplicada por `validate()` em `src/middlewares/validate.middleware.ts`.
- **Autenticação e autorização** com `authenticate` e `requireRole` em `src/middlewares/auth.middleware.ts`.

## Decisão

**O módulo de webhooks reaproveita ao máximo o que já existe. Nada de infraestrutura nova** `[09:30] Larissa`.

Concretamente:

| Aspecto | O que será reutilizado | Onde está hoje |
| --- | --- | --- |
| Estrutura do módulo | `controller` / `service` / `repository` / `routes` / `schemas` em `src/modules/webhooks/` | padrão de `src/modules/orders/` |
| Erros | Herdar de `AppError` e das subclasses HTTP existentes | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` |
| Códigos de erro | Prefixo `WEBHOOK_` em todos os códigos do módulo: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` etc. | mesma convenção de `INSUFFICIENT_STOCK` / `INVALID_STATUS_TRANSITION` |
| Serialização de erro | Nenhuma mudança necessária: o middleware já captura qualquer `AppError` | `src/middlewares/error.middleware.ts` |
| Logs | Logger Pino existente, sem dependência nova | `src/shared/logger/index.ts` |
| Validação de entrada | Schemas Zod + middleware `validate()` | `src/middlewares/validate.middleware.ts` |
| Autenticação | `authenticate` no CRUD; `requireRole('ADMIN')` no replay de DLQ | `src/middlewares/auth.middleware.ts` |
| Paginação | Helper `paginated()` nas listagens | `src/shared/http/response.ts` |
| Acesso a dados | Prisma com os mesmos models/índices do schema atual | `prisma/schema.prisma`, `src/config/database.ts` |
| Entry-point do worker | `src/worker.ts` espelhando o bootstrap de `src/server.ts` | `src/server.ts` |

Duas consequências específicas foram fechadas na reunião:

- **Prefixo `WEBHOOK_` para tudo do módulo** `[09:29] Larissa`.
- **`PrismaClient` separado no worker.** Mesmo banco e mesma `DATABASE_URL`, instância nova, porque `PrismaClient` é por processo Node `[09:29] Diego`, `[09:30] Bruno`. A factory `createPrismaClient()` de `src/config/database.ts` já existe e é exatamente o que o worker deve usar, em vez do singleton `prisma` exportado no mesmo arquivo e consumido por `src/server.ts`.
- **`requireRole` reaproveitado** no endpoint de replay de DLQ, exigindo role `ADMIN` `[09:36] Larissa`, com log de quem executou a operação, para auditoria `[09:36] Sofia`.

## Alternativas Consideradas

### A. Módulo autônomo com stack própria

Tratar webhooks como subsistema independente, com seu próprio formato de erro, seu próprio logger e sua própria estrutura de pastas — justificável pelo fato de o worker rodar em outro processo.

Descartada. O worker roda em outro processo, mas continua no mesmo repositório, no mesmo deploy e lido pelas mesmas pessoas. Duas convenções de erro e dois formatos de log no mesmo repo é custo permanente de manutenção sem ganho.

**Trade-off recusado:** liberdade de desenho no módulo novo em troca de fragmentação das convenções e do ferramental de observabilidade.

### B. Compartilhar a mesma instância de `PrismaClient` entre API e worker

Descartada por impossibilidade prática: `PrismaClient` é por processo, e o worker é outro processo `[09:30] Bruno`. O compartilhamento é de banco e de configuração, não de objeto.

### C. Introduzir mecanismo próprio de tratamento de erro para o módulo

Descartada. O middleware centralizado já trata `AppError`, Zod e Prisma, e vai capturar os erros de webhook sem nenhuma alteração `[09:29] Bruno`.

## Consequências

### Positivas

- Um desenvolvedor que conhece `src/modules/orders/` consegue navegar `src/modules/webhooks/` sem contexto adicional.
- Nenhuma dependência nova no `package.json` para o caminho principal da feature — HMAC-SHA256 sai do módulo `crypto` nativo do Node.
- Os erros do módulo aparecem no mesmo envelope JSON do resto da API, sem trabalho extra no middleware.
- O prefixo `WEBHOOK_` torna trivial filtrar, agrupar e alertar sobre erros do módulo em cima dos logs Pino já estruturados.

### Negativas

- **O módulo herda as limitações do que já existe.** O logger, por exemplo, tem uma lista fixa de `redactPaths` em `src/shared/logger/index.ts` que não cobre a secret de webhook; ela precisa ser estendida, e esquecer disso vira vazamento em log.
- Reutilizar o padrão de módulo de domínio para algo que é metade CRUD e metade processamento assíncrono força um encaixe imperfeito: o `webhook.worker.ts` não é `controller`, `service` nem `repository`, e mora no módulo por convenção, não por simetria `[09:28] Bruno`.
- Ficar preso ao MySQL/Prisma existente significa que a fila é uma tabela, com as limitações de throughput que isso traz — aceito em [ADR-001](ADR-001-outbox-transacional-no-mysql.md).
- Qualquer mudança futura nos contratos compartilhados (`AppError`, middleware de erro, `validate`) passa a ter o módulo de webhooks como consumidor adicional a considerar.

## Decisões relacionadas

- [ADR-001 — Padrão Outbox transacional no MySQL](ADR-001-outbox-transacional-no-mysql.md)
- [ADR-002 — Worker em processo separado com polling](ADR-002-worker-em-processo-separado-com-polling.md)
- [ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint](ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)
