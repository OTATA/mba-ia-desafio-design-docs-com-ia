# ADR-002 — Worker em processo separado consumindo a outbox por polling de 2 segundos

- **Status:** Aceito
- **Data do registro:** 2026-09-22
- **Decisores:** Diego (Eng. Sênior / Plataforma), Larissa (Tech Lead), Bruno (Eng. Pleno / Pedidos), Marcos (PM)
- **Fonte:** `TRANSCRICAO.md` `[09:08]`–`[09:13]`, `[09:28]`, `[09:30]`

## Contexto

Com a outbox definida ([ADR-001](ADR-001-outbox-transacional-no-mysql.md)), restou definir **quem consome a tabela e com que latência**.

Duas restrições delimitam o espaço de solução:

- **Latência aceita pelo negócio:** os clientes B2B consideram "tempo real" qualquer coisa abaixo de 10 segundos; o que não pode acontecer é o evento ficar pendurado `[09:02] Marcos`.
- **Banco:** MySQL. Diferente do Postgres, não existe `LISTEN/NOTIFY` — não há como o banco acordar um processo externo `[09:09] Diego`.

Além disso, o projeto hoje tem um único entry-point, `src/server.ts`, que sobe a API Express (`src/app.ts`) e o `PrismaClient` singleton de `src/config/database.ts`.

## Decisão

O consumo da outbox é feito por um **worker rodando em processo Node separado da API**, em **loop de polling a cada 2 segundos** `[09:09] Diego`, `[09:10] Larissa`.

A cada ciclo o worker busca os eventos pendentes mais antigos em batch pequeno, processa e marca o resultado `[09:08]`, `[09:09] Diego`.

Detalhes estruturais decididos:

- **Processo separado, não thread/timer dentro da API.** Se a API reinicia, o worker não pode ir junto `[09:11] Diego`.
- **Novo entry-point `src/worker.ts`**, espelhando o papel de `src/server.ts`, exposto por um script `npm run worker` `[09:11] Larissa`.
- **A lógica de processamento mora dentro do módulo**, em `src/modules/webhooks/webhook.worker.ts`; o entry-point apenas faz o bootstrap `[09:28] Bruno`.
- **`PrismaClient` próprio.** Mesmo banco e mesma `DATABASE_URL`, mas instância nova, porque `PrismaClient` é por processo `[09:30] Bruno`, `[09:29] Diego`. Na prática o worker usa a factory já exportada em `src/config/database.ts` (`createPrismaClient()`), e não o singleton `prisma` importado pela API.
- **Single-worker por ora.** Com uma única instância processando em ordem de `created_at`, o cliente recebe os eventos de um pedido na ordem correta. Não há garantia de ordering global, e a garantia por `order_id` vale **enquanto for single-worker** `[09:12] Diego`, `[09:13] Larissa`.

**Latência resultante:** até 2 segundos de espera no pior caso antes da primeira tentativa, aceito explicitamente pelo time `[09:10] Larissa` e pelo PM `[09:10] Marcos`, com folga confortável contra o teto de 10s do negócio.

## Alternativas Consideradas

### A. Trigger de banco notificando o worker

Descartada. Trigger no MySQL executa SQL, mas não notifica processo externo. Para avisar o worker seria preciso improvisar um canal lateral — escrever em arquivo, bater em um endpoint — o que foi considerado gambiarra sem ganho real, já que polling de 2s já atende o requisito de latência com sobra `[09:09] Bruno` (proposta), `[09:09] Diego` (recusa).

**Trade-off recusado:** reatividade imediata em troca de um mecanismo de notificação frágil e não idiomático no MySQL.

### B. Worker dentro do mesmo processo da API (setInterval / job in-process)

Descartada. Amarraria o ciclo de vida do worker ao da API: um restart ou deploy da API interromperia o processamento de eventos `[09:11] Diego`.

**Trade-off recusado:** um único processo para operar em troca de acoplamento de ciclo de vida e de recursos entre API e entrega de webhooks.

### C. Múltiplos workers em paralelo desde o início

Descartada para esta fase. Escalar horizontalmente quebra a ordering por `order_id`; recuperá-la exigiria particionamento por `order_id` ou lock pessimista. Classificado como "problema do futuro" `[09:13] Diego`, `[09:13] Bruno`, e reforçado pelo PM: os clientes nunca pediram ordering global, só querem saber se cada pedido deles mudou `[09:14] Marcos`.

**Trade-off recusado:** throughput e tolerância a falha do consumidor em troca de complexidade de coordenação e perda de ordering.

## Consequências

### Positivas

- Isolamento de falhas: um travamento do worker não derruba a API e vice-versa.
- Deploy e restart independentes entre API e entrega de eventos.
- Implementação trivial de entender e depurar: um loop, um `SELECT ... WHERE status = 'PENDING' ORDER BY created_at`, um batch.
- Ordering por `order_id` sai de graça enquanto houver um único worker.

### Negativas

- **Latência mínima de 2 segundos** por design; não há caminho "instantâneo" `[09:10] Larissa`.
- **Polling gera carga constante** no MySQL mesmo sem eventos pendentes.
- **Ponto único de processamento:** com um único worker, se ele cair, nenhum evento é entregue até a recuperação. Os eventos não se perdem (ficam pendentes na outbox), mas a latência degrada silenciosamente.
- **A garantia de ordering é condicional** e precisa ser documentada como limitação conhecida `[09:13] Larissa` — escalar o número de workers a invalida.
- Mais um processo para operar, monitorar e incluir no pipeline de deploy.

## Decisões relacionadas

- [ADR-001 — Padrão Outbox transacional no MySQL](ADR-001-outbox-transacional-no-mysql.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
