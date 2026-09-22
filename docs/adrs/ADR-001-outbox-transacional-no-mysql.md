# ADR-001 — Padrão Outbox transacional no MySQL existente

- **Status:** Aceito
- **Data do registro:** 2026-09-22
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior / Plataforma), Bruno (Eng. Pleno / Pedidos)
- **Fonte:** Reunião técnica de webhooks — `TRANSCRICAO.md` `[09:03]`–`[09:08]`, `[09:40]`–`[09:41]`, `[09:51]`

## Contexto

Três clientes B2B pediram notificação em tempo real de mudança de status de pedido `[09:00] Marcos`. A aplicação hoje não tem nenhum mecanismo de notificação externa: a mudança de status acontece inteiramente dentro de `OrderService.changeStatus`, em uma única transação Prisma (`src/modules/orders/order.service.ts:131`) que atualiza `orders`, insere em `order_status_history` e movimenta `stockQuantity` dos produtos.

A pergunta de abertura foi: disparar o HTTP de notificação de forma síncrona dentro dessa transação, ou registrar o evento e entregar de forma assíncrona? `[09:03] Larissa`

Dois problemas tornam o caminho síncrono inviável:

1. A transação já é pesada (três escritas em tabelas diferentes). Um HTTP call para um cliente lento dentro dela trava a mudança de status de outros pedidos `[09:04] Bruno`.
2. Se o cliente estiver fora do ar, não existe ação correta: dar rollback na mudança de status por causa de uma notificação é inaceitável `[09:04] Bruno`.

Ao mesmo tempo, a garantia que o time quer é forte: *não pode existir o caso de o status mudar e o evento não sair* `[09:40] Bruno`.

## Decisão

Adotamos o **padrão Outbox persistido no próprio MySQL da aplicação**.

Na mesma transação SQL que atualiza `orders` e `order_status_history`, o serviço também insere uma linha na tabela `webhook_outbox` com o evento a ser entregue `[09:06] Diego`. Um worker separado lê essa tabela e executa as chamadas HTTP (ver [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)).

Consequências diretas do acoplamento transacional:

- Se a transação de mudança de status commitou, o evento está registrado.
- Se a transação deu rollback, o evento desaparece junto. Não há inconsistência possível `[09:06] Diego`.
- Se a inserção na outbox falhar, a mudança de status inteira sofre rollback `[09:40] Bruno`.

Decisões de modelagem que acompanham esta:

- **Chave primária UUID**, seguindo o padrão de todo o schema atual, onde todos os modelos usam `@id @default(uuid()) @db.Char(36)` `[09:51] Larissa` / `prisma/schema.prisma`.
- **Índices em `status` e `created_at`**, para o worker varrer apenas os pendentes mais antigos em batches pequenos `[09:08] Diego`.
- A integração com o `OrderService` é feita por uma **função que recebe o `tx` client** da transação em andamento — `publishWebhookEvent(tx, order, fromStatus, toStatus)` — e não pela injeção de um repository inteiro no serviço de pedidos `[09:41] Bruno`, `[09:41] Diego`.

## Alternativas Consideradas

### A. Disparo síncrono do HTTP dentro de `changeStatus`

Descartada. Acopla a latência (e a disponibilidade) de um sistema de terceiro à transação crítica de pedidos, e cria o dilema insolúvel do rollback quando o destino está indisponível `[09:04] Bruno`, `[09:06] Diego`.

**Trade-off recusado:** simplicidade de implementação em troca de risco operacional direto no fluxo de pedidos.

### B. Fila/stream externo (Redis Streams ou similar)

Descartada. Resolveria o desacoplamento, mas exige subir e operar infraestrutura nova. O time é pequeno e subir um Redis Cluster para esse volume é overengineering; o MySQL que já está em produção resolve `[09:07] Larissa`, `[09:07] Diego`.

**Trade-off recusado:** melhor escalabilidade e throughput em troca de custo operacional e de infraestrutura desproporcional ao problema.

### C. Publicar o evento fora da transação, logo após o commit

Descartada implicitamente pelo critério de garantia: "se ficar fora da transação, perde a garantia toda" `[09:41] Diego`. Uma falha do processo entre o commit e a publicação produziria um status alterado sem evento correspondente.

## Consequências

### Positivas

- Atomicidade real entre mudança de status e registro do evento, sem *dual write* e sem necessidade de compensação.
- Zero infraestrutura nova: mesmo MySQL, mesmo Prisma, mesma stack de deploy.
- A outbox funciona como registro auditável do que foi produzido, independente do que foi entregue.
- O evento entregue reflete o estado do pedido no momento da mudança, porque o payload é gravado como snapshot na inserção (ver [ADR-007](ADR-007-snapshot-do-payload-na-insercao-do-evento.md)).

### Negativas

- A transação de `changeStatus` ganha mais uma escrita, aumentando levemente o tempo de lock.
- A tabela `webhook_outbox` cresce indefinidamente dentro do escopo desta feature. O arquivamento de linhas entregues (mencionado como "30 dias ou assim") foi explicitamente deixado **fora do escopo** `[09:08] Diego`, o que transfere para a operação o acompanhamento do crescimento da tabela.
- A entrega passa a ser eventualmente consistente: existe uma janela entre o commit e a entrega efetiva, limitada pelo intervalo de polling (ver [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)).
- Uma falha na inserção do evento derruba a mudança de status. É o comportamento desejado `[09:40] Bruno`, mas significa que um bug no módulo de webhooks pode impactar o fluxo de pedidos.

## Decisões relacionadas

- [ADR-002 — Worker em processo separado com polling](ADR-002-worker-em-processo-separado-com-polling.md)
- [ADR-007 — Snapshot do payload na inserção do evento](ADR-007-snapshot-do-payload-na-insercao-do-evento.md)
- [ADR-008 — Filtragem de eventos na inserção da outbox](ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md)
