# ADR-005 — Garantia de entrega at-least-once com deduplicação por `X-Event-Id`

- **Status:** Aceito
- **Data do registro:** 2026-09-22
- **Decisores:** Diego (Eng. Sênior / Plataforma), Larissa (Tech Lead), Sofia (Eng. de Segurança), Marcos (PM)
- **Fonte:** `TRANSCRICAO.md` `[09:24]`–`[09:26]`, `[09:44]`, `[09:51]`

## Contexto

O desenho adotado — outbox ([ADR-001](ADR-001-outbox-transacional-no-mysql.md)) com worker em polling ([ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)) e retry ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)) — torna a duplicata inevitável em certos cenários. O caso clássico: o cliente processa a requisição e responde, mas a resposta se perde ou estoura o timeout de 10 segundos; do nosso lado a tentativa é marcada como falha e reagendada, e o cliente recebe o mesmo evento de novo.

A escolha, portanto, não é entre "com duplicata" e "sem duplicata", mas entre assumir a duplicata explicitamente e tentar construir exactly-once.

## Decisão

**A plataforma garante entrega at-least-once. O cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado para isso** `[09:24] Diego`.

Para viabilizar a deduplicação do lado do cliente:

- Todo evento recebe um **UUID gerado no momento em que entra na outbox**, único por evento `[09:25] Diego`. O uso de UUID segue o padrão do schema, onde todos os modelos usam `@default(uuid())` `[09:51] Larissa` / `prisma/schema.prisma`.
- Esse identificador viaja em **todas as tentativas de entrega** do mesmo evento, no header **`X-Event-Id`** `[09:25] Diego`, `[09:44] Diego`, e também dentro do corpo do payload, no campo `event_id` `[09:43] Diego`.
- O cliente deduplica pelo `event_id` do lado dele `[09:25] Diego`.

A responsabilidade da deduplicação é do cliente. Isso é reconhecido explicitamente `[09:25] Sofia` e aceito por ser o padrão de mercado — Stripe e GitHub operam do mesmo jeito `[09:25] Diego`. O PM assumiu documentar isso em destaque no portal do desenvolvedor `[09:26] Marcos`.

## Alternativas Consideradas

### A. Exactly-once

Descartada. Garantir entrega exatamente uma vez exigiria coordenação entre os dois lados — confirmação transacional, protocolo de acknowledgement, estado compartilhado de entregas. A complexidade não se justifica: at-least-once com `event_id` resolve 99% dos casos `[09:25] Diego`.

**Trade-off recusado:** eliminar a duplicata em troca de um protocolo de coordenação distribuída que o cliente também teria que implementar.

### B. Deduplicação do lado da plataforma

Não adotada. Poderíamos manter registro de entregas confirmadas e suprimir reenvios, mas isso não resolve o caso que origina a duplicata — a resposta perdida, em que nós não sabemos se o cliente processou. Sem confirmação confiável, a supressão viraria perda de evento, que é o modo de falha mais grave dos dois.

**Trade-off recusado:** menos duplicatas visíveis em troca de risco de descartar eventos que nunca chegaram.

### C. At-most-once (uma tentativa, sem retry)

Nunca foi proposto, e é incompatível com a decisão de retry de [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md). Eliminaria duplicatas ao custo de perder eventos em qualquer indisponibilidade momentânea do cliente — exatamente o problema que a feature existe para resolver.

## Consequências

### Positivas

- O worker pode reprocessar livremente (retry, restart no meio de um batch, replay manual de DLQ) sem risco de perda de evento.
- Modelo familiar para clientes B2B: é o mesmo contrato de Stripe e GitHub `[09:25] Diego`.
- O `event_id` serve simultaneamente como chave de deduplicação para o cliente e como identificador de correlação nos nossos logs e no histórico de entregas.
- Habilita o replay manual da DLQ sem cuidados adicionais: o evento reprocessado carrega o mesmo `event_id`, e o cliente que já o processou simplesmente descarta.

### Negativas

- **Transfere trabalho para o cliente** `[09:25] Sofia`. Cliente que não implementar deduplicação vai processar o mesmo evento duas vezes, e o efeito disso (pedido duplicado no ERP dele, por exemplo) acontece fora do nosso alcance.
- Cria uma dependência de documentação: o contrato só funciona se estiver claramente publicado no portal do desenvolvedor `[09:26] Marcos`. Documentação ruim aqui vira incidente do cliente.
- O suporte precisa estar preparado para a pergunta "por que recebi duas vezes?", que passa a ser comportamento esperado e não bug.

## Decisões relacionadas

- [ADR-003 — Retry com backoff exponencial e DLQ](ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint](ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-007 — Snapshot do payload na inserção do evento](ADR-007-snapshot-do-payload-na-insercao-do-evento.md)
