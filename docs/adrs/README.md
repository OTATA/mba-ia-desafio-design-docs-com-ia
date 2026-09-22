# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto.
Cada decisão arquitetural relevante é registrada em um arquivo individual, no formato
`ADR-NNN-titulo-em-kebab-case.md`, seguindo o padrão MADR: Status, Contexto, Decisão,
Alternativas Consideradas e Consequências.

## Índice

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-outbox-transacional-no-mysql.md) | Padrão Outbox transacional no MySQL existente | Aceito |
| [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) | Worker em processo separado com polling de 2s | Aceito |
| [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela dedicada | Aceito |
| [ADR-004](ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret única por endpoint e rotação com grace de 24h | Aceito |
| [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com deduplicação por `X-Event-Id` | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso máximo dos padrões e da infraestrutura já existentes | Aceito |
| [ADR-007](ADR-007-snapshot-do-payload-na-insercao-do-evento.md) | Payload gravado como snapshot na inserção na outbox | Aceito |
| [ADR-008](ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md) | Filtro de eventos aplicado na inserção, não na entrega | Aceito |

Todos os ADRs acima derivam da reunião técnica registrada em [`TRANSCRICAO.md`](../../TRANSCRICAO.md).
A rastreabilidade item a item está em [`docs/TRACKER.md`](../TRACKER.md).
