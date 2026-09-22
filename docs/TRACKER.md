# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

Este documento é a referência cruzada do pacote de design docs. Cada requisito, decisão, restrição ou trade-off registrado em [PRD](PRD.md), [RFC](RFC.md), [FDD](FDD.md) e [ADRs](adrs/) tem aqui a sua origem — um trecho da reunião ([`TRANSCRICAO.md`](../TRANSCRICAO.md)) ou um ponto do código da aplicação.

**Como ler a coluna Localização:**

- `Fonte = TRANSCRICAO` → timestamp e falante, no formato `[hh:mm] Nome`.
- `Fonte = CODIGO` → caminho do arquivo no repositório.

**Regra de integridade adotada na produção dos documentos:** nenhum item entrou em um documento sem uma linha aqui. Quando não foi possível apontar origem, o item foi removido em vez de estimado. Onde o time registrou intenção sem fechar número (por exemplo, as metas O2 e O3 do PRD), isso está declarado no próprio documento em vez de virar um alvo inventado.

---

## 1. PRD — Problema, objetivos e escopo

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Pedido formal de três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) por notificação em tempo real | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-02 | docs/PRD.md | Problema | Clientes fazem polling em `GET /orders`, o que torna a integração lenta e cara para eles | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-03 | docs/PRD.md | Risco de negócio | Atlas pode migrar para o concorrente se não houver entrega no trimestre | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-04 | docs/PRD.md | Restrição | Escopo é outbound apenas: clientes querem receber, não enviar | TRANSCRICAO | `[09:02] Marcos` |
| PRD-CTX-05 | docs/PRD.md | Restrição | Confirmação de que outbound simplifica o desenho | TRANSCRICAO | `[09:03] Sofia` |
| PRD-CTX-06 | docs/PRD.md | Contexto (código) | Endpoint `GET /orders` usado hoje no polling dos clientes existe e está autenticado | CODIGO | `src/modules/orders/order.routes.ts` |
| PRD-CTX-07 | docs/PRD.md | Contexto (código) | Mudança de status é efeito puramente interno: atualiza pedido, grava histórico, ajusta estoque | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | O1 — latência p95 da notificação abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | O2 — reduzir o volume de polling dos clientes em `GET /orders` | TRANSCRICAO | `[09:00] Marcos` |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | O3 — entregar apesar de indisponibilidade do cliente; DLQ como exceção investigável | TRANSCRICAO | `[09:17] Diego` |
| PRD-OBJ-04 | docs/PRD.md | Objetivo/Prazo | O4 — feature em produção até fim de novembro | TRANSCRICAO | `[09:45] Marcos` |
| PRD-OBJ-05 | docs/PRD.md | Objetivo | O5 — não degradar a latência de `PATCH /orders/:id/status` | TRANSCRICAO | `[09:04] Bruno` |
| PRD-OBJ-06 | docs/PRD.md | Objetivo | O6 — três clientes solicitantes com endpoint ativo | TRANSCRICAO | `[09:00] Marcos` |
| PRD-OBJ-07 | docs/PRD.md | Restrição de estimativa | Três sprints, com a revisão de segurança incluída | TRANSCRICAO | `[09:46] Larissa` |
| PRD-UC-01 | docs/PRD.md | Cenário de uso | C2 — cliente quer ouvir só `SHIPPED` e `DELIVERED` | TRANSCRICAO | `[09:33] Marcos` |
| PRD-UC-02 | docs/PRD.md | Cenário de uso | C3 — cliente em manutenção planejada de duas horas | TRANSCRICAO | `[09:16] Diego` |
| PRD-UC-03 | docs/PRD.md | Cenário de uso | C4 — falha permanente recuperada por reprocessamento manual | TRANSCRICAO | `[09:18] Diego` |
| PRD-UC-04 | docs/PRD.md | Cenário de uso | C5 — rotação de secret após vazamento em log do cliente | TRANSCRICAO | `[09:22] Diego` |
| PRD-UC-05 | docs/PRD.md | Cenário de uso | C6 — suporte investiga não recebimento pelo histórico de entregas | TRANSCRICAO | `[09:34] Marcos` |
| PRD-OUT-01 | docs/PRD.md | Fora de escopo | Alerta por email ao cliente com webhook falhando — adiado para a próxima fase | TRANSCRICAO | `[09:37] Larissa` |
| PRD-OUT-02 | docs/PRD.md | Fora de escopo | Dashboard/painel visual — projeto separado do time de frontend | TRANSCRICAO | `[09:40] Larissa` |
| PRD-OUT-03 | docs/PRD.md | Fora de escopo | Webhooks inbound (cliente → plataforma) | TRANSCRICAO | `[09:02] Marcos` |
| PRD-OUT-04 | docs/PRD.md | Fora de escopo | Rate limiting de envio — observar e decidir depois | TRANSCRICAO | `[09:39] Diego` |
| PRD-OUT-05 | docs/PRD.md | Fora de escopo | Arquivamento de eventos entregues (~30 dias) | TRANSCRICAO | `[09:08] Diego` |
| PRD-OUT-06 | docs/PRD.md | Fora de escopo | Garantia de ordenação global entre pedidos | TRANSCRICAO | `[09:13] Diego` |
| PRD-OUT-07 | docs/PRD.md | Fora de escopo | Entrega exactly-once | TRANSCRICAO | `[09:25] Diego` |
| PRD-OUT-08 | docs/PRD.md | Fora de escopo | Restrição de role no CRUD de configuração — não nesta fase | TRANSCRICAO | `[09:37] Sofia` |

## 2. PRD — Requisitos funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar endpoint de webhook com URL e lista de status | TRANSCRICAO | `[09:31] Marcos` |
| PRD-FR-01b | docs/PRD.md | Restrição | `customerId` informado explicitamente, não extraído do JWT | TRANSCRICAO | `[09:32] Larissa` |
| PRD-FR-01c | docs/PRD.md | Restrição | JWT atual representa o usuário operador, não o cliente | TRANSCRICAO | `[09:32] Bruno` |
| PRD-FR-01d | docs/PRD.md | Contexto (código) | Payload do JWT carrega `sub`, `email` e `role`, sem noção de cliente | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | `[09:31] Marcos` |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Listar endpoints de webhook de um cliente | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Editar endpoint (PATCH) | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Remover endpoint (DELETE) | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Filtro de status por endpoint | TRANSCRICAO | `[09:33] Marcos` |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Endpoint tem estado ativo/inativo; registro guarda url, secret, customer_id e estado | TRANSCRICAO | `[09:21] Bruno` |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Rotação de secret com validade paralela de 24h para a anterior | TRANSCRICAO | `[09:21] Sofia` |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Registro atômico do evento junto com a mudança de status | TRANSCRICAO | `[09:06] Diego` |
| PRD-FR-09b | docs/PRD.md | Restrição | Não pode existir status alterado sem evento registrado | TRANSCRICAO | `[09:40] Bruno` |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Entrega por HTTP POST assinado com headers de identificação | TRANSCRICAO | `[09:44] Diego` |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Reenvio automático com intervalos crescentes, até 5 tentativas | TRANSCRICAO | `[09:15] Diego` |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Falha permanente registrada com payload, motivo e horário | TRANSCRICAO | `[09:18] Diego` |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Histórico de entregas por endpoint (sucesso/falha, payload, response, tempo) | TRANSCRICAO | `[09:34] Marcos` |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Reprocessamento manual de item da DLQ por endpoint admin | TRANSCRICAO | `[09:18] Diego` |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Reprocessamento exige role ADMIN e é auditado | TRANSCRICAO | `[09:36] Sofia` |
| PRD-FR-15b | docs/PRD.md | Decisão | Reuso do `requireRole` existente para a exigência de ADMIN | TRANSCRICAO | `[09:36] Larissa` |
| PRD-FR-16 | docs/PRD.md | Requisito Funcional | Payload é retrato do pedido no momento da mudança, sem os itens | TRANSCRICAO | `[09:43] Diego` |

## 3. PRD — Requisitos não funcionais, dependências e riscos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de notificação abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Verificação da fila a cada 2 segundos | TRANSCRICAO | `[09:09] Diego` |
| PRD-NFR-02b | docs/PRD.md | Trade-off | Latência mínima de 2 segundos aceita explicitamente | TRANSCRICAO | `[09:10] Larissa` |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos por tentativa de entrega | TRANSCRICAO | `[09:42] Diego` |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | 5 tentativas com intervalos 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Diego` |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Payload limitado a 64KB, com erro em vez de truncagem | TRANSCRICAO | `[09:24] Diego` |
| PRD-NFR-05b | docs/PRD.md | Decisão | Preferência por erro em vez de truncar payload grande | TRANSCRICAO | `[09:23] Sofia` |
| PRD-NFR-05c | docs/PRD.md | Classificação | Limite de 64KB tratado como RNF, não como decisão arquitetural | TRANSCRICAO | `[09:24] Larissa` |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | `https` obrigatório; `http` recusado na validação | TRANSCRICAO | `[09:23] Sofia` |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Assinatura HMAC-SHA256 sobre o corpo | TRANSCRICAO | `[09:20] Sofia` |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Uma secret por endpoint; nunca secret global | TRANSCRICAO | `[09:21] Sofia` |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once com dedup pelo cliente | TRANSCRICAO | `[09:24] Diego` |
| PRD-NFR-10 | docs/PRD.md | Restrição | Ordenação por pedido apenas, e só com processador único | TRANSCRICAO | `[09:12] Diego` |
| PRD-NFR-10b | docs/PRD.md | Restrição | Limitação de ordenação deve ser documentada como conhecida | TRANSCRICAO | `[09:13] Larissa` |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Processamento em processo separado da API | TRANSCRICAO | `[09:11] Diego` |
| PRD-NFR-12 | docs/PRD.md | Restrição | Nenhuma chamada HTTP dentro da transação de mudança de status | TRANSCRICAO | `[09:04] Bruno` |
| PRD-NFR-13 | docs/PRD.md | Requisito Não Funcional | Sem infraestrutura nova; reuso dos padrões do projeto | TRANSCRICAO | `[09:30] Larissa` |
| PRD-DEP-01 | docs/PRD.md | Dependência | Máquina de estados de pedidos define o vocabulário de eventos | CODIGO | `src/modules/orders/order.status.ts` |
| PRD-DEP-02 | docs/PRD.md | Dependência | Transação de `changeStatus` é o gatilho da feature | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-DEP-03 | docs/PRD.md | Dependência | `requireRole` existente atende a exigência de ADMIN | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-DEP-04 | docs/PRD.md | Dependência | MySQL já em produção, sem infraestrutura adicional | CODIGO | `docker-compose.yml` |
| PRD-DEP-05 | docs/PRD.md | Dependência organizacional | Revisão de segurança de no mínimo 2 dias úteis antes do deploy | TRANSCRICAO | `[09:46] Sofia` |
| PRD-DEP-06 | docs/PRD.md | Dependência organizacional | Documentação de integração no portal do desenvolvedor | TRANSCRICAO | `[09:40] Marcos` |
| PRD-DEP-07 | docs/PRD.md | Dependência organizacional | Documentar em destaque a possibilidade de evento duplicado | TRANSCRICAO | `[09:26] Marcos` |
| PRD-DEP-08 | docs/PRD.md | Dependência organizacional | Sessão de revisão do design com Bruno e Diego antes de codar | TRANSCRICAO | `[09:50] Larissa` |
| PRD-DEP-09 | docs/PRD.md | Dependência organizacional | PM confirma prazo e integração com os três clientes | TRANSCRICAO | `[09:47] Marcos` |
| PRD-RISK-01 | docs/PRD.md | Risco | R1 — perda da Atlas por atraso na entrega | TRANSCRICAO | `[09:00] Marcos` |
| PRD-RISK-02 | docs/PRD.md | Risco | R2 — vazamento de secret; já houve caso real com cliente | TRANSCRICAO | `[09:22] Diego` |
| PRD-RISK-03 | docs/PRD.md | Risco | R3 — falha do módulo derruba mudanças de status por estar na transação | TRANSCRICAO | `[09:40] Bruno` |
| PRD-RISK-04 | docs/PRD.md | Risco | R4 — processador único indisponível interrompe entregas | TRANSCRICAO | `[09:12] Diego` |
| PRD-RISK-05 | docs/PRD.md | Risco | R5 — cliente sem dedup processa evento duplicado | TRANSCRICAO | `[09:25] Sofia` |
| PRD-RISK-06 | docs/PRD.md | Risco | R6 — crescimento indefinido das tabelas sem política de expurgo | TRANSCRICAO | `[09:08] Diego` |
| PRD-RISK-07 | docs/PRD.md | Risco | R7 — rajada de 50 eventos em um minuto sobrecarrega o cliente | TRANSCRICAO | `[09:38] Diego` |
| PRD-TEST-01 | docs/PRD.md | Estratégia de teste | Regressão: suíte de pedidos deve passar sem alteração | CODIGO | `tests/orders.test.ts` |
| PRD-TEST-02 | docs/PRD.md | Estratégia de teste | Revisão manual de segurança sobre HMAC e geração de secret | TRANSCRICAO | `[09:46] Sofia` |
| PRD-TEST-03 | docs/PRD.md | Estratégia de teste | Validação de ordenação com três mudanças em sequência | TRANSCRICAO | `[09:12] Larissa` |

## 4. RFC — Proposta, alternativas e questões em aberto

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-CTX-01 | docs/RFC.md | Contexto | Pergunta de abertura: disparo síncrono ou fila/outbox | TRANSCRICAO | `[09:03] Larissa` |
| RFC-CTX-02 | docs/RFC.md | Contexto (código) | `changeStatus` valida transição, movimenta estoque, grava histórico em transação | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-PROP-01 | docs/RFC.md | Proposta | Padrão outbox no MySQL, na mesma transação da mudança de status | TRANSCRICAO | `[09:06] Diego` |
| RFC-PROP-02 | docs/RFC.md | Proposta | Worker separado em polling de 2s, com `src/worker.ts` e `npm run worker` | TRANSCRICAO | `[09:11] Larissa` |
| RFC-PROP-03 | docs/RFC.md | Proposta | Retry com backoff e DLQ em tabela dedicada | TRANSCRICAO | `[09:18] Diego` |
| RFC-PROP-04 | docs/RFC.md | Proposta | HMAC-SHA256 com secret por endpoint e rotação com grace de 24h | TRANSCRICAO | `[09:22] Sofia` |
| RFC-PROP-05 | docs/RFC.md | Proposta | At-least-once com `X-Event-Id` para dedup no cliente | TRANSCRICAO | `[09:26] Larissa` |
| RFC-PROP-06 | docs/RFC.md | Proposta | Módulo `src/modules/webhooks` seguindo a estrutura padrão | TRANSCRICAO | `[09:27] Bruno` |
| RFC-PROP-07 | docs/RFC.md | Proposta | Filtro de eventos aplicado na inserção da outbox | TRANSCRICAO | `[09:34] Bruno` |
| RFC-PROP-08 | docs/RFC.md | Proposta | Payload como snapshot renderizado na inserção | TRANSCRICAO | `[09:52] Larissa` |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo síncrono no service de pedidos — travaria a transação e criaria dilema de rollback | TRANSCRICAO | `[09:04] Bruno` |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / fila externa — overengineering para um time pequeno | TRANSCRICAO | `[09:07] Diego` |
| RFC-ALT-02b | docs/RFC.md | Alternativa | Proposta original de Redis Streams como opção | TRANSCRICAO | `[09:07] Larissa` |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger de banco — MySQL não tem `LISTEN/NOTIFY` | TRANSCRICAO | `[09:09] Diego` |
| RFC-ALT-03b | docs/RFC.md | Alternativa | Proposta original de trigger para maior reatividade | TRANSCRICAO | `[09:09] Bruno` |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | 3 tentativas — janela curta demais frente a indisponibilidade de 2h | TRANSCRICAO | `[09:16] Diego` |
| RFC-ALT-04b | docs/RFC.md | Alternativa | Proposta original de 3 tentativas, mais agressiva | TRANSCRICAO | `[09:16] Bruno` |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Retry indefinido — evento pendurado para sempre | TRANSCRICAO | `[09:15] Diego` |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | DLQ como flag `failed` na própria outbox — sujaria a leitura da fila | TRANSCRICAO | `[09:18] Diego` |
| RFC-ALT-06b | docs/RFC.md | Alternativa | Pergunta que abriu a comparação entre tabela separada e flag | TRANSCRICAO | `[09:17] Larissa` |
| RFC-ALT-07 | docs/RFC.md | Alternativa descartada | Secret global — "se vaza uma, vaza tudo" | TRANSCRICAO | `[09:21] Sofia` |
| RFC-ALT-08 | docs/RFC.md | Alternativa descartada | Exactly-once — exige coordenação dos dois lados | TRANSCRICAO | `[09:25] Diego` |
| RFC-ALT-09 | docs/RFC.md | Alternativa descartada | Renderizar payload no envio — evento refletiria estado errado | TRANSCRICAO | `[09:52] Larissa` |
| RFC-ALT-09b | docs/RFC.md | Alternativa | Pergunta que abriu a comparação snapshot × renderização tardia | TRANSCRICAO | `[09:51] Bruno` |
| RFC-ALT-10 | docs/RFC.md | Alternativa descartada | Filtrar eventos na entrega — gravaria linhas destinadas ao descarte | TRANSCRICAO | `[09:34] Bruno` |
| RFC-ALT-10b | docs/RFC.md | Alternativa | Pergunta que abriu a comparação inserção × entrega | TRANSCRICAO | `[09:34] Diego` |
| RFC-ALT-11 | docs/RFC.md | Alternativa descartada | Múltiplos workers desde o início — quebraria a ordenação | TRANSCRICAO | `[09:13] Diego` |
| RFC-OPEN-01 | docs/RFC.md | Questão em aberto | Q1 — rate limiting de envio: observar e decidir depois | TRANSCRICAO | `[09:39] Diego` |
| RFC-OPEN-01b | docs/RFC.md | Questão em aberto | Q1 — registrado como "observar e decidir depois" | TRANSCRICAO | `[09:39] Larissa` |
| RFC-OPEN-02 | docs/RFC.md | Questão em aberto | Q2 — escala multi-worker e ordenação: "problema do futuro" | TRANSCRICAO | `[09:13] Diego` |
| RFC-OPEN-03 | docs/RFC.md | Questão em aberto | Q3 — endurecimento de roles no CRUD: "mais pra frente" | TRANSCRICAO | `[09:37] Sofia` |
| RFC-OPEN-04 | docs/RFC.md | Questão em aberto | Q4 — retenção/arquivamento da outbox fora do escopo | TRANSCRICAO | `[09:08] Diego` |
| RFC-IMP-01 | docs/RFC.md | Impacto | Transação de `changeStatus` ganha leitura e inserções | TRANSCRICAO | `[09:40] Bruno` |
| RFC-IMP-02 | docs/RFC.md | Impacto (código) | Middleware de erro não precisa de alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| RFC-IMP-03 | docs/RFC.md | Impacto | Processo novo a operar: `npm run worker` | TRANSCRICAO | `[09:11] Larissa` |
| RFC-RISK-01 | docs/RFC.md | Risco | Cronograma pressionado pela ameaça comercial da Atlas | TRANSCRICAO | `[09:00] Marcos` |
| RFC-RISK-02 | docs/RFC.md | Risco | Dependência de cronograma: 2 dias úteis de revisão de segurança | TRANSCRICAO | `[09:46] Sofia` |
| RFC-REV-01 | docs/RFC.md | Metadado | Revisores do RFC são os participantes da reunião | TRANSCRICAO | `[09:50] Larissa` |

## 5. FDD — Fluxos, contratos e erros

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-CTX-01 | docs/FDD.md | Contexto técnico | Sequência atual de `changeStatus`: valida, movimenta estoque, atualiza, grava histórico | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-CTX-02 | docs/FDD.md | Contexto técnico | Transições válidas e regras de débito/reposição de estoque | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-MODEL-01 | docs/FDD.md | Decisão de modelagem | PK em UUID, seguindo o padrão do projeto | TRANSCRICAO | `[09:51] Larissa` |
| FDD-MODEL-02 | docs/FDD.md | Decisão de modelagem | Padrão `@default(uuid()) @db.Char(36)` em todos os models existentes | CODIGO | `prisma/schema.prisma` |
| FDD-MODEL-03 | docs/FDD.md | Requisito Não Funcional | Índices em status e `created_at` para a varredura do worker | TRANSCRICAO | `[09:08] Diego` |
| FDD-MODEL-04 | docs/FDD.md | Decisão de modelagem | Registro do endpoint guarda url, secret, customer_id e estado ativo | TRANSCRICAO | `[09:21] Bruno` |
| FDD-MODEL-05 | docs/FDD.md | Decisão de modelagem | DLQ em tabela separada com payload, motivo e timestamp | TRANSCRICAO | `[09:18] Diego` |
| FDD-MODEL-06 | docs/FDD.md | Decisão de modelagem | Secret anterior com expiração, para o grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Inserção do evento na outbox dentro da transação de mudança de status | TRANSCRICAO | `[09:06] Diego` |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Assinatura `publishWebhookEvent(tx, order, fromStatus, toStatus)` | TRANSCRICAO | `[09:41] Bruno` |
| FDD-FLOW-03 | docs/FDD.md | Decisão | Função recebe o `tx`, sem injetar repository no `OrderService` | TRANSCRICAO | `[09:41] Diego` |
| FDD-FLOW-04 | docs/FDD.md | Contexto (código) | Tipo `TxClient = Prisma.TransactionClient` já declarado no serviço de pedidos | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-FLOW-05 | docs/FDD.md | Fluxo | Sem endpoint interessado, nenhuma linha é inserida | TRANSCRICAO | `[09:34] Bruno` |
| FDD-FLOW-06 | docs/FDD.md | Restrição | Falha na inserção do evento reverte a mudança de status inteira | TRANSCRICAO | `[09:40] Bruno` |
| FDD-FLOW-07 | docs/FDD.md | Restrição | Publicação fora da transação perderia toda a garantia | TRANSCRICAO | `[09:41] Diego` |
| FDD-FLOW-08 | docs/FDD.md | Fluxo | Worker lê pendentes mais antigos em batch pequeno, processa e marca | TRANSCRICAO | `[09:08] Diego` |
| FDD-FLOW-09 | docs/FDD.md | Fluxo | Loop de polling a cada 2 segundos | TRANSCRICAO | `[09:09] Diego` |
| FDD-FLOW-10 | docs/FDD.md | Restrição | Processamento sequencial em ordem de `created_at` preserva a ordenação por pedido | TRANSCRICAO | `[09:12] Diego` |
| FDD-FLOW-11 | docs/FDD.md | Fluxo | Retry com backoff 1m/5m/30m/2h/12h, total de 5 tentativas | TRANSCRICAO | `[09:17] Diego` |
| FDD-FLOW-12 | docs/FDD.md | Fluxo | Esgotadas as tentativas, evento vai para a DLQ | TRANSCRICAO | `[09:15] Diego` |
| FDD-FLOW-13 | docs/FDD.md | Fluxo | Replay recoloca o evento na outbox como pendente | TRANSCRICAO | `[09:18] Diego` |
| FDD-FLOW-14 | docs/FDD.md | Requisito Funcional | Replay logado com quem executou, para auditoria | TRANSCRICAO | `[09:36] Sofia` |
| FDD-FLOW-15 | docs/FDD.md | Contexto (código) | Bootstrap do worker espelha `server.ts`: shutdown em SIGINT/SIGTERM e `$disconnect` | CODIGO | `src/server.ts` |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | `POST /webhooks` — cadastro, com secret devolvida na criação | TRANSCRICAO | `[09:31] Marcos` |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | `GET /webhooks` — listagem por cliente | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | `PATCH /webhooks/:id` — edição | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | `DELETE /webhooks/:id` — remoção | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | Rotação de secret com grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | `GET /webhooks/:id/deliveries` — últimos 100 envios com resultado e tempo | TRANSCRICAO | `[09:34] Marcos` |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | `POST /admin/webhooks/dead-letter/:id/replay` | TRANSCRICAO | `[09:35] Diego` |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Payload outbound: event_id, event_type, timestamp, order_id, order_number, from/to_status, customer_id, total_cents | TRANSCRICAO | `[09:43] Diego` |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Itens do pedido fora do payload; detalhe via `GET /orders/:id` | TRANSCRICAO | `[09:43] Diego` |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `Content-Type` | TRANSCRICAO | `[09:44] Diego` |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | Header `X-Webhook-Id` com o id do cadastro | TRANSCRICAO | `[09:44] Sofia` |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato (código) | Envelope paginado `{ data, pagination }` reutilizado nas listagens | CODIGO | `src/shared/http/response.ts` |
| FDD-CONTRATO-13 | docs/FDD.md | Contrato (código) | Teto de `pageSize` em 100, alinhado ao schema de listagem de pedidos | CODIGO | `src/modules/orders/order.schemas.ts` |
| FDD-CONTRATO-14 | docs/FDD.md | Contrato (código) | Prefixo `/api/v1` herdado da montagem do router | CODIGO | `src/app.ts` |
| FDD-CONTRATO-15 | docs/FDD.md | Contrato (código) | `DELETE` responde 204 sem corpo, como no controller de pedidos | CODIGO | `src/modules/orders/order.controller.ts` |
| FDD-ERR-01 | docs/FDD.md | Restrição | Prefixo `WEBHOOK_` em todos os códigos de erro do módulo | TRANSCRICAO | `[09:29] Larissa` |
| FDD-ERR-02 | docs/FDD.md | Decisão | Códigos citados na reunião: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` | TRANSCRICAO | `[09:28] Bruno` |
| FDD-ERR-03 | docs/FDD.md | Decisão (código) | Classes de erro seguem `InsufficientStockError` / `InvalidStatusTransitionError` | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-ERR-04 | docs/FDD.md | Decisão (código) | Contrato `statusCode` / `errorCode` / `details` da classe base | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-ERR-05 | docs/FDD.md | Erro | `WEBHOOK_INVALID_URL` — URL precisa ser `https` | TRANSCRICAO | `[09:23] Sofia` |
| FDD-ERR-06 | docs/FDD.md | Erro | `WEBHOOK_PAYLOAD_TOO_LARGE` — teto de 64KB | TRANSCRICAO | `[09:24] Diego` |
| FDD-ERR-07 | docs/FDD.md | Erro | `WEBHOOK_DELIVERY_TIMEOUT` — resposta acima de 10s tratada como falha | TRANSCRICAO | `[09:42] Diego` |
| FDD-ERR-08 | docs/FDD.md | Erro | `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` — 5ª falha leva à DLQ | TRANSCRICAO | `[09:15] Diego` |
| FDD-ERR-09 | docs/FDD.md | Erro | `WEBHOOK_ENDPOINT_INACTIVE` — endpoint tem estado ativo | TRANSCRICAO | `[09:21] Bruno` |
| FDD-ERR-10 | docs/FDD.md | Decisão (código) | `403 FORBIDDEN` no replay vem do `requireRole` existente | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-ERR-11 | docs/FDD.md | Decisão (código) | `VALIDATION_ERROR` produzido pelo middleware de validação Zod | CODIGO | `src/middlewares/validate.middleware.ts` |

## 6. FDD — Resiliência, observabilidade, integração e aceite

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout de 10s por tentativa | TRANSCRICAO | `[09:42] Diego` |
| FDD-RES-02 | docs/FDD.md | Resiliência | 5 tentativas com backoff exponencial | TRANSCRICAO | `[09:15] Diego` |
| FDD-RES-03 | docs/FDD.md | Resiliência | Estado terminal explícito na DLQ | TRANSCRICAO | `[09:18] Diego` |
| FDD-RES-04 | docs/FDD.md | Resiliência | Recuperação manual via endpoint admin | TRANSCRICAO | `[09:18] Diego` |
| FDD-RES-05 | docs/FDD.md | Resiliência | Idempotência via `X-Event-Id` estável entre tentativas | TRANSCRICAO | `[09:25] Diego` |
| FDD-RES-06 | docs/FDD.md | Resiliência | Isolamento de processo entre API e worker | TRANSCRICAO | `[09:11] Diego` |
| FDD-RES-07 | docs/FDD.md | Trade-off | Sem canal de fallback: email adiado para a próxima fase | TRANSCRICAO | `[09:37] Larissa` |
| FDD-RES-08 | docs/FDD.md | Trade-off | Sem rate limiting: 50 mudanças viram 50 chamadas | TRANSCRICAO | `[09:38] Diego` |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logger Pino reutilizado, sem dependência nova | TRANSCRICAO | `[09:29] Bruno` |
| FDD-OBS-02 | docs/FDD.md | Observabilidade (código) | `redactPaths` precisa ser estendido para cobrir a secret | CODIGO | `src/shared/logger/index.ts` |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Métrica de latência ponta a ponta valida o requisito de <10s | TRANSCRICAO | `[09:02] Marcos` |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Alerta sobre idade do evento pendente mitiga o worker único | TRANSCRICAO | `[09:12] Diego` |
| FDD-OBS-05 | docs/FDD.md | Observabilidade | Log de auditoria do replay com o usuário que executou | TRANSCRICAO | `[09:36] Sofia` |
| FDD-OBS-06 | docs/FDD.md | Observabilidade (código) | Correlação por `X-Request-Id` gerado no middleware de log de request | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| FDD-INT-01 | docs/FDD.md | Integração | `order.service.ts` — chamada de `publishWebhookEvent` dentro da transação | TRANSCRICAO | `[09:40] Bruno` |
| FDD-INT-02 | docs/FDD.md | Integração (código) | `order.service.ts` — ponto exato entre histórico e reload do pedido | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-03 | docs/FDD.md | Integração | Classes de erro do módulo herdam do padrão existente | TRANSCRICAO | `[09:28] Bruno` |
| FDD-INT-04 | docs/FDD.md | Integração (código) | Reexportação das novas classes no índice de erros | CODIGO | `src/shared/errors/index.ts` |
| FDD-INT-05 | docs/FDD.md | Integração | Middleware de erro central já trata AppError, Zod e Prisma | TRANSCRICAO | `[09:29] Bruno` |
| FDD-INT-06 | docs/FDD.md | Integração (código) | Middleware de erro serializa qualquer `AppError` sem alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-07 | docs/FDD.md | Integração | `requireRole('ADMIN')` aplicado apenas no replay | TRANSCRICAO | `[09:36] Larissa` |
| FDD-INT-08 | docs/FDD.md | Integração | CRUD aceita qualquer role autenticada nesta fase | TRANSCRICAO | `[09:37] Sofia` |
| FDD-INT-09 | docs/FDD.md | Integração (código) | `authenticate` aplicado no topo do router, como no módulo de pedidos | CODIGO | `src/modules/orders/order.routes.ts` |
| FDD-INT-10 | docs/FDD.md | Integração (código) | Novos models e enum seguindo as convenções do schema | CODIGO | `prisma/schema.prisma` |
| FDD-INT-11 | docs/FDD.md | Integração | Worker usa PrismaClient próprio, mesmo banco | TRANSCRICAO | `[09:30] Bruno` |
| FDD-INT-12 | docs/FDD.md | Integração (código) | Factory `createPrismaClient()` disponível para o processo do worker | CODIGO | `src/config/database.ts` |
| FDD-INT-13 | docs/FDD.md | Integração | Entry-point `src/worker.ts` e script `npm run worker` | TRANSCRICAO | `[09:11] Larissa` |
| FDD-INT-14 | docs/FDD.md | Integração | Lógica de processamento dentro do módulo (`webhook.worker.ts`) | TRANSCRICAO | `[09:28] Bruno` |
| FDD-INT-15 | docs/FDD.md | Integração (código) | Scripts novos seguem o padrão de `dev` / `start` | CODIGO | `package.json` |
| FDD-INT-16 | docs/FDD.md | Integração | Módulo `src/modules/webhooks` com a estrutura padrão do projeto | TRANSCRICAO | `[09:27] Bruno` |
| FDD-INT-17 | docs/FDD.md | Integração (código) | Registro dos controllers e do router no bootstrap da aplicação | CODIGO | `src/routes/index.ts` |
| FDD-INT-18 | docs/FDD.md | Integração (código) | Novas variáveis de ambiente validadas pelo schema Zod de env | CODIGO | `src/config/env.ts` |
| FDD-INT-19 | docs/FDD.md | Integração (código) | Vocabulário de `eventStatuses` vem do enum de status de pedido | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-ACC-01 | docs/FDD.md | Critério de aceite | Sem endpoint interessado, nenhuma linha de outbox é criada | TRANSCRICAO | `[09:34] Bruno` |
| FDD-ACC-02 | docs/FDD.md | Critério de aceite | Falha na inserção reverte pedido, histórico e estoque | TRANSCRICAO | `[09:40] Bruno` |
| FDD-ACC-03 | docs/FDD.md | Critério de aceite | Snapshot não muda quando o pedido muda depois | TRANSCRICAO | `[09:52] Larissa` |
| FDD-ACC-04 | docs/FDD.md | Critério de aceite | Worker sobe por `npm run worker` como processo independente | TRANSCRICAO | `[09:11] Larissa` |
| FDD-ACC-05 | docs/FDD.md | Critério de aceite | Três mudanças em sequência chegam na ordem de `created_at` | TRANSCRICAO | `[09:12] Diego` |
| FDD-ACC-06 | docs/FDD.md | Critério de aceite | `OPERATOR` recebe 403 no endpoint de replay | TRANSCRICAO | `[09:36] Sofia` |
| FDD-ACC-07 | docs/FDD.md | Critério de aceite | Endpoints diferentes têm secrets diferentes | TRANSCRICAO | `[09:21] Sofia` |
| FDD-ACC-08 | docs/FDD.md | Critério de aceite | Cadastro com `http://` é recusado | TRANSCRICAO | `[09:23] Sofia` |
| FDD-ACC-09 | docs/FDD.md | Critério de aceite | Secret nunca aparece em listagem nem em log | TRANSCRICAO | `[09:22] Diego` |
| FDD-ACC-10 | docs/FDD.md | Critério de aceite (código) | Suíte de pedidos existente deve passar sem alteração | CODIGO | `tests/orders.test.ts` |

## 7. ADRs

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Padrão outbox no MySQL, evento inserido na transação da mudança de status | TRANSCRICAO | `[09:06] Diego` |
| ADR-001-A | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | Fechamento formal: "outbox em MySQL" | TRANSCRICAO | `[09:08] Larissa` |
| ADR-001-B | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Alternativa | Síncrono descartado por peso da transação e impossibilidade de rollback | TRANSCRICAO | `[09:04] Bruno` |
| ADR-001-C | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Alternativa | Redis Streams descartado como overengineering | TRANSCRICAO | `[09:07] Diego` |
| ADR-001-D | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Consequência | Arquivamento de linhas entregues fora do escopo | TRANSCRICAO | `[09:08] Diego` |
| ADR-001-E | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Decisão | PK em UUID, seguindo o padrão do projeto | TRANSCRICAO | `[09:51] Larissa` |
| ADR-001-F | docs/adrs/ADR-001-outbox-transacional-no-mysql.md | Contexto (código) | Transação atual de `changeStatus` com três escritas | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-002 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Polling em loop de 2 segundos | TRANSCRICAO | `[09:09] Diego` |
| ADR-002-A | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker obrigatoriamente em processo separado da API | TRANSCRICAO | `[09:11] Diego` |
| ADR-002-B | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Alternativa | Trigger de banco descartado: MySQL não notifica processo externo | TRANSCRICAO | `[09:09] Diego` |
| ADR-002-C | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Trade-off | Latência mínima de 2s aceita | TRANSCRICAO | `[09:10] Larissa` |
| ADR-002-D | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Consequência | Ordenação apenas por pedido e só enquanto single-worker | TRANSCRICAO | `[09:12] Diego` |
| ADR-002-E | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Restrição | Clientes nunca pediram ordenação global | TRANSCRICAO | `[09:14] Marcos` |
| ADR-002-F | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Contexto (código) | `src/server.ts` como modelo de bootstrap do novo entry-point | CODIGO | `src/server.ts` |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Backoff exponencial com 5 tentativas: 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Diego` |
| ADR-003-A | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | DLQ em tabela dedicada com payload, motivo e timestamp | TRANSCRICAO | `[09:18] Diego` |
| ADR-003-B | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa | 3 tentativas descartadas por janela curta | TRANSCRICAO | `[09:16] Diego` |
| ADR-003-C | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa | Retry indefinido descartado por evento pendurado | TRANSCRICAO | `[09:15] Diego` |
| ADR-003-D | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Trade-off | Janela de ~15h validada pelo PM | TRANSCRICAO | `[09:17] Marcos` |
| ADR-003-E | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Replay manual por endpoint admin | TRANSCRICAO | `[09:18] Diego` |
| ADR-003-F | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Restrição | Timeout de 10s classifica a tentativa como falha | TRANSCRICAO | `[09:42] Diego` |
| ADR-004 | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, assinatura em `X-Signature` | TRANSCRICAO | `[09:20] Sofia` |
| ADR-004-A | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | Secret única por endpoint | TRANSCRICAO | `[09:21] Sofia` |
| ADR-004-B | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Decisão | Rotação com grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| ADR-004-C | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Contexto | Caso real de cliente que vazou secret em log | TRANSCRICAO | `[09:22] Diego` |
| ADR-004-D | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Restrição | `https` obrigatório, validado no schema Zod | TRANSCRICAO | `[09:23] Sofia` |
| ADR-004-E | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Requisito Não Funcional | Motivação: autenticidade e integridade do payload | TRANSCRICAO | `[09:19] Sofia` |
| ADR-004-F | docs/adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md | Consequência (código) | Lista de `redactPaths` do logger precisa cobrir a secret | CODIGO | `src/shared/logger/index.ts` |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | Garantia at-least-once assumida explicitamente | TRANSCRICAO | `[09:24] Diego` |
| ADR-005-A | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | UUID por evento enviado em `X-Event-Id` para dedup no cliente | TRANSCRICAO | `[09:25] Diego` |
| ADR-005-B | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Alternativa | Exactly-once descartado por exigir coordenação dos dois lados | TRANSCRICAO | `[09:25] Diego` |
| ADR-005-C | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de dedup transferida ao cliente | TRANSCRICAO | `[09:25] Sofia` |
| ADR-005-D | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Mitigação | Documentação em destaque no portal do desenvolvedor | TRANSCRICAO | `[09:26] Marcos` |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso máximo: AppError, Pino, middleware de erro, módulos, Zod, códigos de erro | TRANSCRICAO | `[09:30] Larissa` |
| ADR-006-A | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Estrutura de módulo por domínio replicada em `src/modules/webhooks` | TRANSCRICAO | `[09:27] Bruno` |
| ADR-006-B | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Prefixo `WEBHOOK_` em todos os códigos de erro | TRANSCRICAO | `[09:29] Larissa` |
| ADR-006-C | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | PrismaClient separado por processo | TRANSCRICAO | `[09:30] Bruno` |
| ADR-006-D | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | `requireRole` reaproveitado para exigir ADMIN no replay | TRANSCRICAO | `[09:36] Larissa` |
| ADR-006-E | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Contexto (código) | Padrão controller/service/repository/routes/schemas por domínio | CODIGO | `src/modules/orders/order.routes.ts` |
| ADR-006-F | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Contexto (código) | Classe base de erro com `statusCode`, `errorCode` e `details` | CODIGO | `src/shared/errors/app-error.ts` |
| ADR-006-G | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Contexto (código) | Erros de domínio com código em SCREAMING_SNAKE_CASE | CODIGO | `src/shared/errors/http-errors.ts` |
| ADR-006-H | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Contexto (código) | Middleware central já cobre AppError, Zod e Prisma | CODIGO | `src/middlewares/error.middleware.ts` |
| ADR-006-I | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Contexto (código) | `authenticate` e `requireRole` disponíveis para o novo módulo | CODIGO | `src/middlewares/auth.middleware.ts` |
| ADR-006-J | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Contexto (código) | Composição de dependências por módulo no bootstrap | CODIGO | `src/app.ts` |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao-do-evento.md | Decisão | Payload renderizado e gravado na inserção (snapshot) | TRANSCRICAO | `[09:52] Larissa` |
| ADR-007-A | docs/adrs/ADR-007-snapshot-do-payload-na-insercao-do-evento.md | Alternativa | Renderizar no envio, guardando só `order_id` | TRANSCRICAO | `[09:51] Bruno` |
| ADR-007-B | docs/adrs/ADR-007-snapshot-do-payload-na-insercao-do-evento.md | Confirmação | Concordância com snapshot na inserção | TRANSCRICAO | `[09:52] Diego` |
| ADR-007-C | docs/adrs/ADR-007-snapshot-do-payload-na-insercao-do-evento.md | Decisão | Conteúdo do payload, sem os itens do pedido | TRANSCRICAO | `[09:43] Diego` |
| ADR-007-D | docs/adrs/ADR-007-snapshot-do-payload-na-insercao-do-evento.md | Trade-off | Payload enxuto como objetivo explícito | TRANSCRICAO | `[09:44] Bruno` |
| ADR-007-E | docs/adrs/ADR-007-snapshot-do-payload-na-insercao-do-evento.md | Contexto (código) | `GET /orders/:id` retorna o pedido com itens, histórico e cliente | CODIGO | `src/modules/orders/order.repository.ts` |
| ADR-008 | docs/adrs/ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md | Decisão | Filtro aplicado na inserção; sem interessado, nada é inserido | TRANSCRICAO | `[09:34] Bruno` |
| ADR-008-A | docs/adrs/ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md | Alternativa | Filtrar na hora do envio, levantado como opção | TRANSCRICAO | `[09:34] Diego` |
| ADR-008-B | docs/adrs/ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md | Requisito Funcional | Cliente escolhe os status que quer ouvir | TRANSCRICAO | `[09:33] Marcos` |
| ADR-008-C | docs/adrs/ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md | Contexto (código) | Valores possíveis vêm do enum `OrderStatus` | CODIGO | `prisma/schema.prisma` |
| ADR-008-D | docs/adrs/ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md | Consequência (código) | Histórico completo de transições continua em `order_status_history` | CODIGO | `prisma/schema.prisma` |

---

## 8. Resumo de cobertura

| Métrica | Valor |
| --- | --- |
| Total de itens rastreados | 269 |
| Itens com `Fonte = TRANSCRICAO` | 221 (82%) |
| Itens com `Fonte = CODIGO` | 48 (18%) |
| Arquivos de código distintos referenciados | 24 |
| Documentos cobertos | PRD, RFC, FDD e os 8 ADRs |

**Arquivos de código referenciados:** `src/modules/orders/order.service.ts`, `order.status.ts`, `order.routes.ts`, `order.controller.ts`, `order.repository.ts`, `order.schemas.ts`, `src/shared/errors/app-error.ts`, `http-errors.ts`, `index.ts`, `src/shared/logger/index.ts`, `src/shared/http/response.ts`, `src/middlewares/auth.middleware.ts`, `error.middleware.ts`, `validate.middleware.ts`, `request-logger.middleware.ts`, `src/config/database.ts`, `src/config/env.ts`, `src/app.ts`, `src/routes/index.ts`, `src/server.ts`, `prisma/schema.prisma`, `package.json`, `tests/orders.test.ts`, `docker-compose.yml`.

**Arquivos citados que ainda não existem:** `src/worker.ts` e `src/modules/webhooks/*` são artefatos **a serem criados** pela feature, propostos na própria reunião (`[09:11] Larissa`, `[09:27] Bruno`, `[09:28] Bruno`). Aparecem nos documentos sempre em tempo futuro e nunca como `Fonte = CODIGO` neste tracker — todas as 48 linhas de código acima apontam para arquivos existentes no repositório.

**Itens deliberadamente sem linha aqui:** detalhes de implementação que são consequência direta de uma decisão já rastreada e não trazem informação nova (por exemplo, o nome interno de uma constante ou a escolha de `fetch` nativo em vez de uma biblioteca HTTP, que decorre de ADR-006 e da versão de Node declarada em `package.json`).
