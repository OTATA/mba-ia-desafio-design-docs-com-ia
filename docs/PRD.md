# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Product Manager** | Marcos |
| **Tech Lead** | Larissa |
| **Status** | Aprovado para implementação |
| **Data** | 2026-09-22 |
| **Prazo alvo** | Fim de novembro (compromisso com a Atlas Comercial) `[09:45] Marcos` |
| **Estimativa** | 3 sprints, incluindo a revisão de segurança `[09:46] Larissa` |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |
| **Fonte** | [`TRANSCRICAO.md`](../TRANSCRICAO.md) e o código da aplicação |

---

## 1. Resumo e contexto

O Order Management System hoje é um sistema fechado: clientes B2B integrados só descobrem que um pedido mudou de status se perguntarem. Esta feature inverte isso, entregando **notificações HTTP outbound** (webhooks) sempre que o status de um pedido de um cliente muda.

O cliente cadastra um ou mais endpoints `https`, escolhe quais status quer ouvir, e passa a receber uma requisição assinada a cada mudança relevante — em geral, em menos de 10 segundos. Se o endpoint dele estiver fora do ar, a plataforma reenvia com intervalos crescentes ao longo de cerca de 15 horas antes de desistir.

O escopo é estritamente **de saída**: a plataforma notifica, não recebe. Perguntado diretamente se os clientes também enviariam webhooks para nós, o PM foi objetivo — "eles querem receber, não mandar" `[09:02] Marcos`, `[09:03] Sofia`.

---

## 2. Problema e motivação

**O pedido veio de fora.** Três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** — fizeram pedido formal para serem notificados em tempo real quando o status dos pedidos deles muda `[09:00] Marcos`.

**O workaround atual é caro para eles.** Hoje esses clientes ficam batendo em `GET /orders` de tempos em tempos para verificar se algo mudou, o que torna a integração lenta e cara do lado deles `[09:00] Marcos`. O endpoint existe e funciona (`src/modules/orders/order.routes.ts:16`), mas polling é o pior jeito de descobrir uma mudança rara: o cliente paga por todas as consultas em que nada aconteceu.

**Há risco comercial concreto.** A Atlas sinalizou que, se a entrega não acontecer até o fim do trimestre, pode migrar para um concorrente `[09:00] Marcos`.

**A plataforma não tem por onde começar.** Não existe nenhum mecanismo de notificação externa, evento ou fila na aplicação. A mudança de status é um efeito puramente interno: atualiza o pedido, grava o histórico e ajusta estoque (`src/modules/orders/order.service.ts:126`). Nada sai do sistema.

---

## 3. Público-alvo e cenários de uso

### 3.1 Público

| Persona | Quem é | O que ganha |
| --- | --- | --- |
| **Cliente B2B integrado** | Atlas Comercial, MaxDistribuição, Nova Cargo `[09:00] Marcos` | Recebe a mudança de status em segundos, sem polling |
| **Time de desenvolvimento do cliente** | Quem integra do outro lado | Contrato estável, assinado, documentado no portal do desenvolvedor `[09:40] Marcos` |
| **Operação / suporte interno** | Quem responde "por que o cliente não recebeu?" | Histórico de entregas por endpoint e fila de falhas permanentes visível |
| **Administrador da plataforma** | Usuário com role `ADMIN` | Pode reprocessar manualmente eventos que falharam em definitivo `[09:35] Diego` |

### 3.2 Cenários de uso

**C1 — Notificação no caminho feliz.** O operador muda o pedido `ORD-000418` de `PROCESSING` para `SHIPPED`. Segundos depois, o sistema da Atlas recebe um POST assinado informando a mudança, e dispara internamente o aviso ao cliente final dela.

**C2 — Cliente escolhe o que quer ouvir.** A MaxDistribuição não se importa com `PENDING` nem com `PAID`; só quer saber quando vira `SHIPPED` e `DELIVERED` `[09:33] Marcos`. Cadastra o endpoint com esse filtro e recebe apenas esses dois eventos.

**C3 — Endpoint do cliente cai.** A Nova Cargo entra em manutenção planejada de duas horas — cenário já observado em produção `[09:16] Diego`. As entregas falham, são reagendadas com intervalos crescentes e chegam quando o endpoint volta, sem intervenção de ninguém.

**C4 — Falha permanente e recuperação manual.** Um endpoint fica fora por mais de 15 horas. Os eventos esgotam as tentativas e vão para a fila de falhas permanentes. Quando o cliente avisa que voltou, um administrador dispara o reprocessamento `[09:18] Diego`.

**C5 — Rotação de credencial.** O cliente suspeita que a secret dele vazou em log da própria aplicação — situação que já aconteceu `[09:22] Diego`. Pede uma nova secret pela API, tem 24 horas para atualizar os sistemas dele, e não perde nenhuma notificação durante a troca `[09:21] Sofia`.

**C6 — Investigação de suporte.** O cliente diz que não recebeu um evento. O suporte consulta o histórico de entregas daquele endpoint e vê as tentativas, os códigos de resposta e os tempos `[09:34] Marcos`.

---

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| **O1** | Notificar mudanças de status em tempo quase real | Latência p95 entre o commit da mudança de status e a entrega bem-sucedida | **< 10 segundos** `[09:02] Marcos` |
| **O2** | Eliminar o polling dos clientes integrados | Volume de chamadas a `GET /orders` pelos 3 clientes B2B após a adoção | **Redução mensurável** frente à linha de base medida antes do lançamento `[09:00] Marcos` |
| **O3** | Entregar mesmo com indisponibilidade do cliente | Proporção de eventos entregues em até 5 tentativas, sem chegar à DLQ | **Acompanhar**; DLQ é exceção investigável, não rotina `[09:17] Diego` |
| **O4** | Atender o compromisso comercial com a Atlas | Feature em produção | **Fim de novembro** `[09:45] Marcos` |
| **O5** | Não degradar o fluxo de pedidos | Latência de `PATCH /orders/:id/status` antes × depois | Sem regressão perceptível `[09:04] Bruno` |
| **O6** | Adoção pelos clientes solicitantes | Clientes com ao menos um endpoint ativo | **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) `[09:00] Marcos` |

**O1 é a meta quantitativa dura da feature.** Ela vem de uma pergunta direta que o PM fez aos clientes: para eles, qualquer coisa abaixo de 10 segundos já é "tempo real"; o que não pode é ficar pendurado `[09:02] Marcos`. O desenho técnico coloca 2 segundos de polling dentro desse orçamento `[09:10] Larissa`.

> Nota de honestidade: as metas de O2 e O3 não têm número fechado porque nenhum número foi acordado na reunião. Registrá-los como "medir e acompanhar" é fiel à decisão tomada; inventar um alvo não seria.

---

## 5. Escopo

### 5.1 Incluso

- Cadastro, consulta, edição e remoção de endpoints de webhook por cliente.
- Filtro de status por endpoint: o cliente escolhe quais mudanças quer receber.
- Geração de secret na criação e rotação self-service com período de convivência de 24h.
- Entrega assinada por HMAC-SHA256, com transporte obrigatoriamente `https`.
- Reenvio automático com intervalos crescentes e fila de falhas permanentes.
- Consulta do histórico de entregas por endpoint.
- Reprocessamento manual de falhas permanentes, restrito a administradores e auditado.

### 5.2 Fora de escopo

| Item | Situação | Origem |
| --- | --- | --- |
| **Alerta por email ao cliente cujo webhook está falhando** | **Adiado para a próxima fase**, depois de medir o impacto | `[09:37] Marcos` (proposta), `[09:37] Larissa` (recusa), `[09:38] Marcos` |
| **Dashboard/painel visual para o cliente** | **Fora**. Nesta fase, só endpoints de API. Painel é projeto separado do time de frontend | `[09:39] Marcos` (proposta), `[09:40] Larissa` (recusa) |
| Webhooks inbound (cliente → plataforma) | Fora. O escopo é apenas outbound | `[09:02] Marcos`, `[09:03] Sofia` |
| Rate limiting de envio por cliente | Não agora. Observar e implementar se virar problema | `[09:38] Diego`, `[09:39] Larissa` |
| Arquivamento/expurgo de eventos já entregues | Fora do escopo desta feature | `[09:08] Diego` |
| Garantia de ordenação global entre pedidos | Fora. Ordenação apenas por pedido, e apenas enquanto houver um único processador | `[09:12] Diego`, `[09:13] Larissa` |
| Entrega exatamente uma vez | Fora. O contrato é at-least-once, com deduplicação pelo cliente | `[09:25] Diego` |
| Restrição de acesso por role no CRUD de configuração | Não nesta fase; qualquer role autenticada pode configurar | `[09:37] Sofia` |

---

## 6. Requisitos funcionais

| # | Requisito | Origem |
| --- | --- | --- |
| **RF-01** | O cliente pode **cadastrar** um endpoint de webhook informando URL e a lista de status que deseja receber. O `customerId` é informado explicitamente, não inferido do token — o JWT atual representa o usuário operador, não o cliente | `[09:31] Marcos`, `[09:32] Bruno`, `[09:32] Larissa` |
| **RF-02** | A **secret é gerada pela plataforma** e devolvida ao cliente no momento da criação | `[09:31] Marcos` |
| **RF-03** | O cliente pode **listar** os endpoints de webhook de um cliente | `[09:33] Bruno` |
| **RF-04** | O cliente pode **editar** um endpoint já cadastrado | `[09:33] Bruno` |
| **RF-05** | O cliente pode **remover** um endpoint | `[09:33] Bruno` |
| **RF-06** | Cada endpoint declara **quais status quer ouvir**, e só recebe eventos desses status | `[09:33] Marcos`, `[09:34] Bruno` |
| **RF-07** | Um endpoint tem **estado ativo/inativo**; inativo não recebe eventos | `[09:21] Bruno` |
| **RF-08** | O cliente pode **rotacionar a secret** de um endpoint; a anterior continua válida por 24 horas | `[09:21] Sofia` |
| **RF-09** | Toda mudança de status de pedido **registra o evento de forma atômica** com a própria mudança. Não existe status alterado sem evento registrado | `[09:06] Diego`, `[09:40] Bruno` |
| **RF-10** | A entrega ao cliente é feita por **HTTP POST assinado**, com os headers de identificação do evento, assinatura, timestamp e identificação do endpoint | `[09:20] Sofia`, `[09:44] Diego`, `[09:44] Sofia` |
| **RF-11** | Entregas que falham são **reenviadas automaticamente** com intervalos crescentes, até 5 tentativas | `[09:15] Diego`, `[09:17] Diego` |
| **RF-12** | Esgotadas as tentativas, o evento é registrado como **falha permanente**, com payload, motivo e horário | `[09:18] Diego` |
| **RF-13** | O cliente pode **consultar o histórico de entregas** de um endpoint: sucesso ou falha, payload, resposta e tempo de resposta | `[09:34] Marcos` |
| **RF-14** | Um **administrador** pode reprocessar manualmente um evento em falha permanente, recolocando-o na fila | `[09:18] Diego`, `[09:35] Diego` |
| **RF-15** | O reprocessamento exige role **ADMIN** e é **registrado para auditoria**, incluindo quem o executou | `[09:36] Sofia`, `[09:36] Larissa` |
| **RF-16** | O payload entregue é um **retrato do pedido no momento da mudança**, com identificação do evento, tipo, horário, pedido, status de origem e destino, cliente e total — **sem os itens** | `[09:43] Diego`, `[09:52] Larissa` |

---

## 7. Requisitos não funcionais

| # | Requisito | Valor | Origem |
| --- | --- | --- | --- |
| **RNF-01** | Latência de notificação | Abaixo de 10 segundos no caminho feliz | `[09:02] Marcos` |
| **RNF-02** | Intervalo de verificação da fila | 2 segundos; latência mínima aceita de 2s | `[09:09] Diego`, `[09:10] Larissa` |
| **RNF-03** | Timeout de cada tentativa de entrega | 10 segundos | `[09:42] Diego` |
| **RNF-04** | Política de reenvio | 5 tentativas, com intervalos de 1min, 5min, 30min, 2h e 12h | `[09:17] Diego` |
| **RNF-05** | Tamanho máximo do payload | 64 KB; acima disso, erro — nunca truncar | `[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa` |
| **RNF-06** | Transporte | `https` obrigatório; `http` recusado na validação | `[09:23] Sofia` |
| **RNF-07** | Assinatura | HMAC-SHA256 sobre o corpo da requisição | `[09:20] Sofia` |
| **RNF-08** | Isolamento de credenciais | Uma secret por endpoint; nunca uma secret global | `[09:21] Sofia` |
| **RNF-09** | Garantia de entrega | At-least-once; o cliente deduplica pelo identificador do evento | `[09:24] Diego`, `[09:25] Diego` |
| **RNF-10** | Ordenação | Por pedido, e apenas enquanto houver um único processador. Sem garantia global | `[09:12] Diego`, `[09:13] Larissa` |
| **RNF-11** | Isolamento operacional | O processamento roda em processo separado da API; reinício da API não afeta as entregas | `[09:11] Diego` |
| **RNF-12** | Impacto no fluxo de pedidos | Nenhuma chamada HTTP dentro da transação de mudança de status | `[09:04] Bruno` |
| **RNF-13** | Consistência de infraestrutura | Sem infraestrutura nova: mesmo banco, mesma stack, padrões existentes do projeto | `[09:07] Diego`, `[09:30] Larissa` |

---

## 8. Decisões e trade-offs principais

Cada decisão está registrada em detalhe em seu ADR. Resumo do que foi decidido e do que se abriu mão:

| Decisão | O que ganhamos | O que abrimos mão | ADR |
| --- | --- | --- | --- |
| Registrar o evento na mesma transação da mudança de status, em vez de disparar na hora | Impossibilidade de status mudar sem evento; nenhuma dependência de terceiro no fluxo crítico | Entrega deixa de ser instantânea e passa a ser eventual | [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) |
| Usar o MySQL existente em vez de uma fila dedicada | Zero infraestrutura nova, adequado a um time pequeno | Teto de throughput de uma tabela; sem primitivas maduras de fila | [ADR-001](adrs/ADR-001-outbox-transacional-no-mysql.md) |
| Processador separado, verificando a fila a cada 2s | Isolamento de falhas; simplicidade | 2 segundos de latência mínima por design; carga constante no banco | [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) |
| Um único processador nesta fase | Ordenação por pedido sai de graça | Ponto único de processamento; escalar exige rever a garantia | [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) |
| 5 tentativas ao longo de ~15h, em vez de 3 ou de infinitas | Cobre indisponibilidade real de cliente sem intervenção | Evento pode ser entregue muito depois de ser útil | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| Secret por endpoint, rotacionável com 24h de convivência | Vazamento fica contido; rotação sem downtime para o cliente | Duas credenciais válidas ao mesmo tempo durante a janela | [ADR-004](adrs/ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once em vez de exatamente-uma-vez | Simplicidade e alinhamento com o padrão de mercado (Stripe, GitHub) | A deduplicação vira responsabilidade do cliente `[09:25] Sofia` | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Reaproveitar os padrões do projeto | Curva de aprendizado zero; nenhuma dependência nova | O módulo herda as limitações do que já existe | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |
| Gravar o payload no momento da mudança | Evento historicamente correto e estável entre tentativas | Cliente pode receber informação defasada após retry longo | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao-do-evento.md) |
| Filtrar o interesse do cliente na entrada da fila | Fila só contém trabalho real | Mudar o filtro não recupera eventos passados | [ADR-008](adrs/ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md) |

---

## 9. Dependências

### 9.1 Técnicas

- **Máquina de estados de pedidos** (`src/modules/orders/order.status.ts`) — define o vocabulário de eventos. Só existe webhook porque existe transição controlada.
- **Transação de mudança de status** (`src/modules/orders/order.service.ts:126`) — é o gatilho da feature e o único ponto do código existente alterado no caminho crítico `[09:40] Bruno`.
- **Autenticação e roles existentes** (`src/middlewares/auth.middleware.ts`) — a exigência de role `ADMIN` no reprocessamento reaproveita o `requireRole` já implementado `[09:36] Larissa`.
- **MySQL e Prisma já em produção** — sem infraestrutura nova `[09:07] Diego`.
- **Nenhuma dependência npm nova** para o caminho principal (ver [FDD](FDD.md) §10).

### 9.2 Organizacionais

- **Revisão de segurança da Sofia antes do deploy.** Pediu no mínimo **dois dias úteis** para revisar HMAC e geração de secret com calma `[09:46] Sofia`; o prazo de três sprints já a inclui `[09:47] Larissa`.
- **Documentação no portal do desenvolvedor.** O PM assumiu documentar como integrar via API `[09:40] Marcos` e, em destaque, o fato de que o mesmo evento pode chegar mais de uma vez `[09:26] Marcos`. Sem isso, o contrato at-least-once vira incidente do cliente.
- **Sessão de revisão do design** com Bruno e Diego antes do início da implementação `[09:50] Larissa`.
- **Comunicação aos três clientes** sobre prazo e forma de integração `[09:47] Marcos`, `[09:49] Marcos`.

---

## 10. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| **R1** | **Perda do cliente Atlas por atraso.** A Atlas indicou que pode migrar para o concorrente se a entrega não sair no trimestre `[09:00] Marcos` | Média | **Alto** — perda de receita e efeito de referência sobre os outros dois clientes | Escopo cortado ao essencial (sem email, sem dashboard, sem rate limiting); estimativa de 3 sprints acordada com a revisão de segurança já dentro `[09:46]`; PM confirma prazo com o cliente `[09:47] Marcos` |
| **R2** | **Vazamento de secret.** Já houve caso de cliente expondo a secret em log da própria aplicação `[09:22] Diego` | Média | **Alto** — terceiro consegue forjar eventos válidos para aquele endpoint | Secret por endpoint contém o raio a um único cadastro `[09:21] Sofia`; rotação self-service com 24h de convivência `[09:21] Sofia`; revisão de segurança dedicada antes do deploy `[09:46] Sofia` |
| **R3** | **Falha do módulo de webhooks derruba mudanças de status**, já que o registro do evento está dentro da transação crítica `[09:40] Bruno` | Média | **Alto** — impacto direto na operação de pedidos, que hoje funciona | Nenhuma chamada de rede dentro da transação `[09:04] Bruno`; superfície mínima (uma leitura e as inserções); cobertura por testes de integração no fluxo de `changeStatus` |
| **R4** | **Processador único fora do ar**: as entregas param sem nenhum sintoma visível na API | Média | Alto | Os eventos ficam na fila e são entregues na recuperação `[09:06] Diego`; alerta sobre a idade do evento pendente mais antigo (ver [FDD](FDD.md) §9.1) |
| **R5** | **Cliente sem deduplicação processa o mesmo evento duas vezes**, já que o contrato é at-least-once `[09:25] Sofia` | Média | Médio — efeito colateral no sistema do cliente, fora do nosso alcance | Identificador estável do evento em todas as tentativas `[09:25] Diego`; documentação em destaque no portal `[09:26] Marcos` |
| **R6** | **Crescimento indefinido das tabelas de eventos**, sem política de expurgo nesta fase `[09:08] Diego` | Alta | Médio — degradação progressiva de performance do banco | Filtro na entrada reduz o volume `[09:34] Bruno`; monitoramento de tamanho; política de retenção tratada como questão aberta (Q4 do [RFC](RFC.md)) |
| **R7** | **Rajada de eventos sobrecarrega o endpoint do cliente** — 50 mudanças em um minuto viram 50 chamadas `[09:38] Diego` | Média | Médio | Decisão consciente de observar antes de agir `[09:39] Diego`; monitorar tentativas por endpoint; rate limiting entra se virar problema real |

---

## 11. Critérios de aceitação

Do ponto de vista de produto, a feature está pronta quando:

1. Um cliente consegue cadastrar um endpoint `https`, escolher os status de interesse e receber a secret na resposta `[09:31] Marcos`.
2. Mudar o status de um pedido desse cliente resulta em uma requisição assinada no endpoint dele em **menos de 10 segundos** `[09:02] Marcos`.
3. O cliente recebe **apenas** os status que declarou querer ouvir `[09:33] Marcos`.
4. Cadastro com URL `http` é recusado com erro de validação `[09:23] Sofia`.
5. Com o endpoint do cliente fora do ar, as entregas são reenviadas automaticamente e chegam quando ele volta, dentro da janela de ~15 horas `[09:17] Diego`.
6. Esgotadas as tentativas, o evento aparece na fila de falhas permanentes com payload e motivo `[09:18] Diego`.
7. Um usuário `ADMIN` consegue reprocessar um evento em falha permanente; um usuário sem essa role recebe erro de permissão `[09:36] Sofia`.
8. O reprocessamento fica registrado com a identificação de quem o executou `[09:36] Sofia`.
9. O cliente consegue consultar o histórico de entregas do endpoint, com resultado, payload, resposta e tempo `[09:34] Marcos`.
10. O cliente consegue rotacionar a secret e continuar recebendo eventos durante as 24 horas de convivência `[09:21] Sofia`.
11. Um pedido com três mudanças de status em sequência gera três notificações na ordem em que ocorreram `[09:12] Diego`.
12. Nenhum cliente sem webhook cadastrado é afetado: o comportamento de `PATCH /orders/:id/status` permanece idêntico.
13. A revisão de segurança da Sofia foi concluída antes do deploy `[09:46] Sofia`.

Os critérios técnicos correspondentes — incluindo comportamento transacional, formato de headers, códigos de erro e recuperação de crash — estão no [FDD](FDD.md) §12.

---

## 12. Estratégia de testes e validação

### 12.1 Níveis de teste

| Nível | O que cobre | Base existente |
| --- | --- | --- |
| **Unitário** | Cálculo do backoff, geração e verificação da assinatura, avaliação do filtro de status, montagem do payload, validação de tamanho | Vitest, já configurado (`vitest.config.ts`) |
| **Integração (API)** | CRUD de endpoints, rotação de secret, histórico de entregas, permissão no reprocessamento | Padrão de `tests/orders.test.ts` com Supertest e `tests/helpers/factories.ts` |
| **Integração (transação)** | Registro do evento junto com a mudança de status, rollback conjunto, ausência de registro quando ninguém escuta | Mesmos cenários já cobertos em `tests/orders.test.ts` para `changeStatus` |
| **Ponta a ponta** | Mudança de status → fila → entrega ao endpoint → confirmação, com servidor HTTP de teste simulando o cliente | Novo, previsto na estimativa `[09:46] Larissa` |
| **Resiliência** | Endpoint que responde erro, que demora mais que o timeout e que volta no meio da sequência de reenvios; esgotamento até a falha permanente e reprocessamento | Novo |
| **Regressão** | `tests/orders.test.ts` deve passar **sem alteração** — é a prova de que clientes sem webhook não são afetados | Existente |
| **Segurança** | Revisão manual da Sofia sobre HMAC e geração de secret, **mínimo 2 dias úteis, antes do deploy** | `[09:46] Sofia` |

### 12.2 Validação em produção

- **Piloto com um cliente.** Habilitar primeiro para um dos três solicitantes e acompanhar latência e taxa de sucesso antes de liberar para os demais.
- **Medir O1 com dado real:** distribuição p95 do tempo entre a mudança de status e a entrega confirmada, contra a meta de 10 segundos.
- **Medir O2:** comparar o volume de chamadas dos clientes a `GET /orders` antes e depois da adoção — a linha de base precisa ser capturada **antes** do lançamento `[09:00] Marcos`.
- **Acompanhar O3:** volume de eventos que chegam à fila de falhas permanentes; qualquer ocorrência é investigada individualmente, não tratada como rotina.
- **Validar O5:** comparar a latência de `PATCH /orders/:id/status` antes e depois, para confirmar que a escrita adicional na transação não degradou o fluxo de pedidos `[09:04] Bruno`.

### 12.3 Validação com os clientes

O PM confirma o prazo e a forma de integração diretamente com os três clientes `[09:47] Marcos`, `[09:49] Marcos`, e publica no portal do desenvolvedor a documentação de integração `[09:40] Marcos`, com destaque para a possibilidade de recebimento duplicado `[09:26] Marcos`.
