# ADR-003 — Retry com backoff exponencial de 5 tentativas e DLQ em tabela dedicada

- **Status:** Aceito
- **Data do registro:** 2026-09-22
- **Decisores:** Diego (Eng. Sênior / Plataforma), Larissa (Tech Lead), Bruno (Eng. Pleno / Pedidos), Marcos (PM)
- **Fonte:** `TRANSCRICAO.md` `[09:14]`–`[09:19]`, `[09:42]`

## Contexto

A entrega de webhooks depende de um sistema que não controlamos. O cliente pode estar fora do ar, lento ou respondendo erro. A pergunta colocada foi direta: "se o cliente tá offline, o que a gente faz?" `[09:14] Larissa`.

O histórico operacional deu o parâmetro concreto: já houve cliente com indisponibilidade de **duas horas** em manutenção planejada `[09:16] Diego`. Qualquer política de retry precisa cobrir esse tipo de janela sem intervenção manual.

Também ficou definido que uma resposta que não chega em **10 segundos** é tratada como falha e entra no fluxo de retry `[09:42] Diego`, `[09:42] Sofia`.

## Decisão

**Retry com backoff exponencial, 5 tentativas no total, e falha permanente registrada em Dead Letter Queue persistida em tabela separada.**

### Política de retry

| Tentativa | Espera desde a falha anterior | Tempo acumulado desde a 1ª falha |
| --------- | ----------------------------- | -------------------------------- |
| 1 (retry) | 1 minuto                      | ~1 min                           |
| 2         | 5 minutos                     | ~6 min                           |
| 3         | 30 minutos                    | ~36 min                          |
| 4         | 2 horas                       | ~2h36                            |
| 5         | 12 horas                      | ~14h36                           |

Progressão 1m / 5m / 30m / 2h / 12h, totalizando quase 15 horas entre a primeira falha e a última tentativa `[09:17] Diego`, `[09:17] Larissa`. O PM validou a janela: um cliente fora por 15 horas já tem um problema sério do lado dele `[09:17] Marcos`.

**Timeout de cada tentativa:** 10 segundos `[09:42] Diego`.

### Dead Letter Queue

Esgotadas as 5 tentativas, o evento é movido para uma **tabela `webhook_dead_letter` separada**, contendo o payload, o motivo da falha e o timestamp `[09:18] Diego`.

### Reprocessamento

O replay é **manual, via endpoint administrativo** `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente `[09:18] Diego`, `[09:35] Diego`. O endpoint exige role `ADMIN` e registra em log quem executou o replay — ver [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) `[09:36] Sofia`, `[09:36] Larissa`.

## Alternativas Consideradas

### A. 3 tentativas, mais agressivo

Proposta por Bruno `[09:16]`. Descartada: com uma progressão agressiva, três tentativas cobririam cerca de 30 minutos e matariam o evento de um cliente que teve indisponibilidade matinal — cenário já observado em produção com janela de duas horas `[09:16] Diego`.

**Trade-off recusado:** fila mais enxuta e falha detectada mais cedo em troca de perda de eventos de clientes com indisponibilidade normal e recuperável.

### B. Retry indefinido com backoff

Descartada. É uma posição defensável, mas cria eventos pendurados para sempre quando o cliente simplesmente sumiu, sem nunca produzir um sinal claro de falha permanente `[09:15] Diego`.

**Trade-off recusado:** entrega eventual garantida em troca de crescimento ilimitado da fila e ausência de um estado terminal observável.

### C. Marcar `failed` na própria outbox em vez de tabela separada

Alternativa levantada por Larissa `[09:17]`. Descartada: manter os fracassos permanentes na mesma tabela polui a leitura da outbox principal — que o worker varre a cada 2 segundos — e mistura fila de trabalho com evidência de investigação. A tabela dedicada deixa a outbox limpa e serve de base para debug e reprocessamento `[09:18] Diego`, `[09:18] Bruno`.

**Trade-off recusado:** um modelo de dados a menos em troca de leitura mais suja da fila ativa.

### D. Reprocessamento automático da DLQ

Não adotada. Quem reprocessa é uma pessoa com role `ADMIN`, por endpoint explícito `[09:18] Diego`. Mexer em fila de entrega de notificação foi classificado como operação sensível, não rotina de operador `[09:36] Sofia`.

## Consequências

### Positivas

- Cobre indisponibilidades reais de clientes (horas, não minutos) sem intervenção humana.
- Existe um estado terminal explícito: ou entregou, ou está na DLQ. Nada fica ambíguo.
- A DLQ funciona como evidência para debug e como fila de reprocessamento controlado `[09:18] Diego`.
- A outbox ativa permanece pequena e rápida de varrer.

### Negativas

- **Até ~15 horas entre a primeira falha e o descarte.** Um evento pode ser entregue muito depois de ter deixado de ser útil para o cliente.
- Um evento em retry longo ocupa espaço na outbox por horas, e a ordering por `order_id` fica comprometida na prática: eventos posteriores do mesmo pedido podem ser entregues antes de um evento antigo em backoff. Limitação conhecida, decorrente do modelo single-worker de [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md).
- **Toda recuperação de DLQ é manual.** Se um cliente ficar fora por mais de 15 horas, alguém precisa notar e agir — e não há notificação automática, porque o alerta por email foi explicitamente adiado para a próxima fase `[09:37] Larissa`.
- Duas tabelas e duas máquinas de estado para manter em sincronia (outbox e dead letter).

## Decisões relacionadas

- [ADR-002 — Worker em processo separado com polling](ADR-002-worker-em-processo-separado-com-polling.md)
- [ADR-005 — Entrega at-least-once com X-Event-Id](ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
