# ADR-008 — Filtro de eventos aplicado na inserção da outbox, não na entrega

- **Status:** Aceito
- **Data do registro:** 2026-09-22
- **Decisores:** Bruno (Eng. Pleno / Pedidos), Diego (Eng. Sênior / Plataforma), Marcos (PM)
- **Fonte:** `TRANSCRICAO.md` `[09:33]`–`[09:34]`

## Contexto

Cada endpoint de webhook cadastrado declara **quais status ele quer ouvir** — por exemplo, "só quero saber quando vira `SHIPPED` e `DELIVERED`" `[09:33] Marcos`. Os valores possíveis são os da máquina de estados existente: `PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED` (`prisma/schema.prisma`, enum `OrderStatus`; transições em `src/modules/orders/order.status.ts`).

Definido o filtro, sobrou a pergunta de **onde aplicá-lo**: na escrita, decidindo se a linha entra na outbox, ou na leitura, decidindo se o worker envia `[09:34] Diego`.

## Decisão

**O filtro é aplicado na inserção.** Se nenhum webhook ativo do customer se interessa por aquele status, o evento **não é inserido** na outbox `[09:34] Bruno`, `[09:34] Diego`.

A avaliação acontece dentro da mesma transação de `changeStatus`, na função `publishWebhookEvent(tx, order, fromStatus, toStatus)` descrita em [ADR-001](ADR-001-outbox-transacional-no-mysql.md): a função consulta os endpoints do `customerId` do pedido, seleciona os que escutam o `toStatus`, e insere uma linha de outbox por endpoint interessado. Nenhum endpoint interessado significa nenhuma linha inserida.

Motivação registrada: economiza linha na tabela `[09:34] Bruno`.

## Alternativas Consideradas

### A. Filtrar no momento do envio

Alternativa explicitamente colocada na mesa por Diego `[09:34]`: inserir sempre um evento por mudança de status e deixar o worker decidir para quais endpoints enviar.

Descartada. Gravaria linhas que já nascem destinadas ao descarte, inflando uma tabela que o worker varre a cada 2 segundos e que, nesta fase, não tem política de arquivamento `[09:08] Diego`.

**Trade-off recusado:** um registro completo de todas as mudanças de status na outbox (útil para auditoria e para reconfiguração retroativa de filtros) em troca de crescimento desnecessário da fila ativa.

### B. Sem filtro — todo endpoint recebe todos os status

Não foi proposto. Contraria o requisito funcional do PM `[09:33] Marcos` e geraria tráfego de saída que o cliente não pediu, agravando a preocupação de volume que motivou a discussão de rate limiting `[09:38] Diego`.

## Consequências

### Positivas

- A outbox só contém trabalho que realmente será executado; a varredura do worker fica mais barata.
- O volume de linhas escala com o interesse declarado pelos clientes, não com o volume bruto de mudanças de status.
- A decisão de "para quem enviar" acontece uma única vez, na escrita, e não a cada tentativa de entrega — o que combina com o snapshot por endpoint de [ADR-007](ADR-007-snapshot-do-payload-na-insercao-do-evento.md).

### Negativas

- **O filtro é avaliado no instante da mudança de status e nunca reavaliado.** Se um cliente adicionar `PAID` à lista de status depois do fato, os eventos de `PAID` anteriores não existem na outbox e não podem ser reenviados. Não há backfill.
- **A transação de `changeStatus` ganha uma leitura extra** dos endpoints de webhook do customer, além da escrita da linha. Isso alonga a transação crítica de pedidos descrita em `src/modules/orders/order.service.ts:131`.
- Um endpoint inativo ou mal configurado no momento da mudança perde o evento silenciosamente — a ausência de linha na outbox não deixa rastro de que algo foi filtrado.
- A outbox deixa de servir como log completo de mudanças de status. Para essa finalidade, a fonte de verdade continua sendo `order_status_history` (`prisma/schema.prisma`, model `OrderStatusHistory`), que registra toda transição independentemente de webhooks.

## Decisões relacionadas

- [ADR-001 — Padrão Outbox transacional no MySQL](ADR-001-outbox-transacional-no-mysql.md)
- [ADR-007 — Snapshot do payload na inserção do evento](ADR-007-snapshot-do-payload-na-insercao-do-evento.md)
