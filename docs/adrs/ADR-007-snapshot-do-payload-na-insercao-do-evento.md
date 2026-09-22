# ADR-007 — Payload gravado como snapshot no momento da inserção na outbox

- **Status:** Aceito
- **Data do registro:** 2026-09-22
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior / Plataforma), Bruno (Eng. Pleno / Pedidos)
- **Fonte:** `TRANSCRICAO.md` `[09:43]`, `[09:51]`–`[09:52]`

## Contexto

A linha da outbox pode guardar o evento de duas formas: apenas a referência (`order_id`, `from_status`, `to_status`), deixando o worker montar o payload no momento do envio, ou o payload já renderizado.

A diferença só aparece quando há atraso entre a inserção e a entrega — e, com retry de até ~15 horas ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)), esse atraso é um cenário normal, não excepcional. Nesse intervalo o pedido pode mudar de status de novo, ser cancelado ou ter o total alterado.

A pergunta foi colocada ao final da reunião: "o evento da outbox guarda o payload renderizado já, ou guarda só order_id e renderiza na hora do envio?" `[09:51] Bruno`.

## Decisão

**O payload é renderizado e gravado como snapshot no momento da inserção na outbox** `[09:52] Larissa`, `[09:52] Diego`, `[09:52] Bruno`.

O evento entregue reflete o estado do pedido **no instante em que o status mudou**, não no instante do envio. Renderizar na hora do envio produziria casos esquisitos, em que o cliente recebe um evento "mudou para PAID" com dados que já não correspondem àquele momento `[09:52] Larissa`.

O conteúdo do snapshot foi definido na reunião `[09:43] Diego`:

- `event_id` (UUID do evento — ver [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md))
- `event_type`, no formato `order.status_changed`
- `timestamp` em ISO 8601
- `order_id`, `order_number`
- `from_status`, `to_status`
- `customer_id`
- campos básicos do pedido, como `total_cents`

**Os itens do pedido não vão no payload**, para não inflá-lo. Cliente que precisar do detalhe faz um `GET /orders/:id` depois `[09:43] Diego`, `[09:44] Bruno`. O endpoint existe e retorna o pedido com itens, histórico e cliente (`src/modules/orders/order.controller.ts`, `OrderRepository.findByIdWithRelations`).

O payload enxuto também sustenta o limite de 64KB por envio, acima do qual o evento é recusado com erro em vez de truncado `[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa`.

## Alternativas Consideradas

### A. Guardar só `order_id` e renderizar na hora do envio

Alternativa real, levantada por Bruno `[09:51]`. Descartada: com retry de horas, o payload renderizado tardiamente descreveria um estado diferente daquele que gerou o evento, criando incoerência entre `to_status` e os demais campos `[09:52] Larissa`.

**Trade-off recusado:** linhas de outbox menores e dados sempre "frescos" em troca de eventos historicamente incorretos.

### B. Snapshot completo, incluindo os itens do pedido

Descartada. Inflaria cada linha da outbox e cada requisição de saída sem necessidade — os clientes pediram para saber *que o status mudou*, e o detalhe já está disponível por `GET /orders/:id` `[09:43] Diego`.

**Trade-off recusado:** payload autossuficiente em troca de crescimento da tabela, do tráfego de saída e do risco de esbarrar no teto de 64KB.

## Consequências

### Positivas

- O evento é imutável e historicamente correto: descreve o que aconteceu, quando aconteceu.
- Todas as tentativas de retry e o replay manual de DLQ enviam exatamente o mesmo corpo — o que é pré-requisito para a assinatura HMAC ser estável entre tentativas ([ADR-004](ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)) e para a deduplicação por `event_id` fazer sentido ([ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)).
- O worker não precisa consultar `orders` a cada envio: lê a linha da outbox e dispara. Menos queries por evento.
- O payload persistido é evidência direta para debug e para o histórico de entregas exposto em `GET /webhooks/:id/deliveries` `[09:34] Marcos`.

### Negativas

- **O cliente pode receber informação defasada.** Um evento entregue após 12 horas de retry mostra o estado de 12 horas atrás; o pedido já pode ter avançado. O contrato precisa deixar claro que o payload é um registro histórico, não o estado atual.
- Duplicação de dados: o mesmo conteúdo existe em `orders` e, serializado, na linha da outbox. Isso aumenta o tamanho da tabela — agravado pela ausência de política de arquivamento nesta fase `[09:08] Diego`.
- Mudar o formato do payload no futuro não reescreve eventos já enfileirados: eventos antigos continuarão no formato antigo até serem entregues ou descartados.

## Decisões relacionadas

- [ADR-001 — Padrão Outbox transacional no MySQL](ADR-001-outbox-transacional-no-mysql.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-005 — Entrega at-least-once com X-Event-Id](ADR-005-entrega-at-least-once-com-x-event-id.md)
