# Da Reunião ao Documento — Processo de Produção

> Entrega do desafio **"Da Reunião ao Documento: Design Docs Gerados por IA"**.
> O enunciado original está preservado em [`desafio.md`](desafio.md).
> Os documentos produzidos estão em [`docs/`](docs/).

---

## 1. Sobre o desafio

O ponto de partida é um Order Management System real em Node.js + TypeScript, rodando em produção, e a transcrição literal de uma reunião de 55 minutos em que cinco pessoas — tech lead, PM, dois engenheiros e uma engenheira de segurança — fecham o desenho de uma feature nova: um sistema de webhooks que notifica clientes B2B quando o status de um pedido muda. A decisão técnica já estava tomada; o que não existia era registro nenhum além da gravação da call.

A tarefa foi transformar esse material em um pacote de documentação acionável — PRD, RFC, FDD, ADRs e um tracker de rastreabilidade — sem escrever uma linha de código de aplicação. O que torna o exercício difícil não é gerar texto: é **decidir o que entra**. A reunião mistura decisões fechadas, ideias descartadas na hora, itens adiados para uma próxima fase, dúvidas que ficaram em aberto e detalhes técnicos secundários, tudo na mesma conversa. Um documento que trate o "talvez próxima fase" do minuto `[09:37]` como requisito está errado, mesmo que bem escrito. A regra que organizou todo o trabalho foi: **nada entra em um documento sem origem identificável na transcrição ou no código** — e a verificação disso é o tracker.

---

## 2. Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code (Opus 5, 1M de contexto)** | Ferramenta principal de ponta a ponta. Leitura da transcrição e do código-fonte diretamente do repositório, extração e classificação do conteúdo da reunião, redação dos documentos e execução dos scripts de verificação. A janela de 1M permitiu manter a transcrição inteira e os arquivos relevantes do código no contexto ao mesmo tempo, sem trabalhar por trechos — o que importa muito num desafio cujo critério central é consistência entre documentos. |
| **Scripts Python/bash gerados na própria sessão** | Verificação mecânica do resultado: validação de todos os timestamps citados contra a transcrição, checagem de existência de cada caminho de arquivo mencionado e de cada link relativo, e contagem real das linhas do tracker. Ver §5. |

Uma escolha deliberada: **nenhuma ferramenta de chat sem acesso ao repositório foi usada**. Copiar trechos de código para um chat externo introduz exatamente o tipo de erro que o desafio penaliza — citar um arquivo que não existe, ou descrever um método com assinatura ligeiramente diferente da real.

---

## 3. Workflow adotado

Segui a ordem sugerida no enunciado (ADRs → RFC → FDD → PRD → Tracker → README), com uma etapa a mais no começo e outra no fim.

### Etapa 0 — Contextualização dupla

Antes de escrever qualquer coisa: leitura integral da transcrição **e** leitura do código real — não uma exploração superficial, mas os arquivos que a reunião nomeia. `order.service.ts`, `order.status.ts`, `app-error.ts`, `http-errors.ts`, `error.middleware.ts`, `auth.middleware.ts`, `logger/index.ts`, `schema.prisma`, `server.ts`, `app.ts`, `package.json`, `tests/orders.test.ts`.

Isso mudou o resultado de forma concreta. A reunião diz "a gente já tem um padrão de erros, classe `AppError`, códigos tipo `INSUFFICIENT_STOCK`" `[09:28]`. Só lendo `src/shared/errors/http-errors.ts` dá para ver que `NotFoundError` **fixa** o código `NOT_FOUND` no construtor — o que significa que `WebhookNotFoundError` não pode simplesmente herdar dela e emitir `WEBHOOK_NOT_FOUND`. Esse detalhe virou uma observação explícita no FDD (§11.2). Sem ler o código, o documento teria dito "é só herdar", e o desenvolvedor descobriria o problema na primeira hora de implementação.

### Etapa 1 — Mapa de extração antes de redigir

Antes do primeiro documento, produzi um mapa classificando cada trecho da reunião em quatro baldes: **decisão fechada**, **requisito**, **descartado/adiado** e **questão em aberto**. O balde "descartado/adiado" foi montado primeiro, de propósito: é a lista do que precisa **não** aparecer.

| Balde | Itens |
| --- | --- |
| Decisões fechadas | outbox no MySQL `[09:08]`, worker em polling 2s `[09:10]`, retry 5×/DLQ `[09:17-18]`, HMAC-SHA256 `[09:22]`, at-least-once `[09:26]`, reuso de padrões `[09:30]`, snapshot do payload `[09:52]`, filtro na inserção `[09:34]` |
| Descartado / adiado | email de alerta `[09:37]`, dashboard `[09:40]`, inbound `[09:02]`, arquivamento `[09:08]`, multi-worker `[09:13]`, exactly-once `[09:25]`, secret global `[09:21]`, trigger de banco `[09:09]`, Redis `[09:07]`, 3 tentativas `[09:16]` |
| Em aberto | rate limiting `[09:39]`, escala/ordering `[09:13]`, roles do CRUD `[09:37]`, retenção da outbox `[09:08]` |

### Etapa 2 — ADRs primeiro

Oito ADRs em formato MADR. Os seis exigidos pelo enunciado, mais dois que a reunião fecha explicitamente e que mereciam registro próprio por terem alternativa real discutida: o snapshot do payload (ADR-007, alternativa levantada por Bruno em `[09:51]`) e o filtro de eventos na inserção (ADR-008, alternativa levantada por Diego em `[09:34]`).

Critério de qualidade aplicado: **toda seção "Alternativas Consideradas" precisa citar quem propôs, quem recusou e com que argumento**. Se a alternativa não tem um responsável e um motivo na transcrição, ela é genérica e sai.

### Etapa 3 — RFC

Escrito em cima dos ADRs, em nível de arquitetura. A disciplina aqui foi de **altura**: o RFC diz "HMAC-SHA256 com secret por endpoint e rotação de 24h" e para por aí; quem quiser o formato do header `X-Signature` vai ao FDD. Ganhou uma seção final de "Pedido aos revisores" com perguntas nominais, porque o documento é literalmente o que Larissa se comprometeu a abrir para revisão de Bruno e Diego em `[09:50]`.

### Etapa 4 — FDD

O documento mais longo, e o único que cita linhas de código. A seção obrigatória "Integração com o sistema existente" foi escrita com os arquivos abertos, referenciando 20+ caminhos reais e descrevendo, para cada um, o que muda e o que não muda.

### Etapa 5 — PRD

Por último entre os grandes, como o enunciado sugere. Como o problema, o público e o risco comercial vêm todos do PM nos primeiros dois minutos da call, o PRD ficou fortemente ancorado em `[09:00]`–`[09:02]`.

### Etapa 6 — Tracker

269 linhas mapeando cada item ao seu ponto de origem. Construído varrendo os documentos prontos, não em paralelo — assim ele funciona como **auditoria** do que foi escrito, e não como rascunho do que se pretendia escrever.

### Etapa 7 — Verificação mecânica

Ver §5. Esta etapa não estava no plano original e foi o que pegou o erro mais grave da produção.

---

## 4. Prompts customizados

### Prompt 1 — Filtragem dirigida da transcrição

Este foi o prompt de abertura, e é o mais importante do conjunto. Um pedido genérico ("resuma os requisitos desta reunião") devolve uma lista achatada em que "vamos mandar email pro cliente" aparece com o mesmo peso de "outbox no MySQL". A correção é **forçar a classificação e obrigar a citação**:

```
Leia TRANSCRICAO.md por inteiro. Não resuma ainda.

Classifique CADA trecho relevante em exatamente um dos quatro baldes:
  (A) DECISÃO FECHADA   — alguém propôs, o grupo concordou, ficou decidido
  (B) REQUISITO         — funcionalidade ou restrição pedida, sem debate
  (C) DESCARTADO/ADIADO — proposto e recusado, ou empurrado pra outra fase
  (D) EM ABERTO         — levantado, discutido, NÃO decidido

Para cada item devolva: | balde | [hh:mm] Falante | o que foi dito | quem
recusou/adiou e com que argumento (só para C e D) |

Regras:
- Comece pelo balde C. Quero primeiro a lista do que NÃO pode virar requisito.
- Um item só é (A) se houver uma fala de fechamento explícita. "Acho que sim"
  não é fechamento; "Decidido, X" é.
- Se a mesma coisa é dita duas vezes, registre o timestamp do FECHAMENTO,
  não o da primeira menção.
- Não infira nada que não esteja na transcrição. Se ficou ambíguo, mande pro
  balde D e diga por que é ambíguo.
```

A regra do "comece pelo balde C" é o coração do prompt: colocar o descarte em primeiro lugar muda o que o modelo trata como figura e o que trata como fundo. E a regra do timestamp de fechamento resolveu um problema real — o `[09:15]` de "eu sugiro 5" não é a mesma coisa que o `[09:17]` de "decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h".

### Prompt 2 — Blindagem contra alucinação no FDD

O FDD é onde a IA mais inventa: nomes de arquivo plausíveis, métodos que não existem, "conforme o padrão do projeto" sem nunca ter olhado o padrão. Este prompt impõe verificação antes da escrita:

```
Escreva a seção "Integração com o sistema existente" do FDD.

Antes de escrever CADA parágrafo:
1. Abra o arquivo que você vai citar. Não cite de memória.
2. Confirme que o símbolo que você vai mencionar (classe, método, export)
   existe de fato naquele arquivo, e anote a linha.
3. Se o que a transcrição descreve NÃO bate exatamente com o que está no
   código, escreva a divergência em vez de suavizá-la.

Formato por arquivo: caminho real → o que muda → o que NÃO muda → por que
(com [hh:mm] Falante quando a origem for a reunião).

Proibido:
- "provavelmente", "algo como", "um arquivo similar a"
- citar arquivo que você não abriu nesta sessão
- inventar assinatura de função: use a que está no código, ou a que a
  transcrição especifica literalmente
- descrever como novidade algo que o projeto já resolve

Cubra no mínimo: order.service.ts, as classes de erro, o middleware de erro,
o middleware de auth, o schema Prisma e o bootstrap do worker.
```

O item "escreva a divergência em vez de suavizá-la" produziu as duas observações mais úteis de todo o pacote: o problema do `NotFoundError` com código fixo, e o fato de que `createPrismaClient()` já existe em `src/config/database.ts` — ou seja, a decisão de `[09:30]` ("PrismaClient separado") já tem uma factory pronta no projeto, e o FDD pode apontar para ela em vez de descrever a criação de algo novo.

### Prompt 3 — Auto-auditoria do tracker

```
Você acabou de escrever docs/TRACKER.md. Não confie nele.

Escreva e RODE um script que:
  (a) extraia todo [hh:mm] Nome citado em docs/**/*.md e valide o par
      timestamp+falante contra TRANSCRICAO.md — reporte cada inválido;
  (b) extraia todo caminho de arquivo citado e verifique se existe no disco;
  (c) verifique se todo link relativo markdown resolve;
  (d) conte as linhas reais do tracker por Fonte.

Depois compare (d) com os números que você escreveu na seção de cobertura.
Se divergirem, o errado é o texto, não o script. Corrija o texto.
```

---

## 5. Iterações e ajustes

Quatro ciclos de geração, revisão crítica e correção. Os momentos concretos em que a IA errou ou entregou algo raso:

### Iteração 1 — Números inventados no resumo de cobertura do tracker

**O erro:** ao fechar o `TRACKER.md`, a seção "Resumo de cobertura" foi escrita com os números **estimados** durante a redação — "177 itens, 143 da transcrição (81%), 34 do código". Os números eram plausíveis, internamente consistentes e completamente falsos.

**Como apareceu:** o script do Prompt 3 contou as linhas reais: **269 itens, 221 da transcrição (82%), 48 do código**. A tabela tinha crescido durante a escrita e a estimativa nunca foi recalculada.

**Por que isso importa:** é exatamente a falha que o tracker existe para prevenir, cometida dentro do próprio tracker. E é o tipo de erro que passa em qualquer revisão por leitura — os números pareciam certos. Só contagem mecânica pega.

**Correção:** números substituídos pelos reais e a contagem virou etapa obrigatória do fechamento.

### Iteração 2 — Primeira versão dos ADRs com alternativas genéricas

**O erro:** a primeira passada produziu seções "Alternativas Consideradas" no estilo enciclopédia — "poderia-se usar Kafka", "outra opção seria gRPC". Tecnicamente corretas, absolutamente inúteis: nada disso foi discutido na reunião.

**Correção:** regra dura de que toda alternativa precisa de **proponente, recusante e argumento**, todos citáveis. Alternativa sem os três é cortada ou marcada explicitamente como não discutida. O ADR-004 é o exemplo: a alternativa "assinatura assimétrica" ficou no documento, mas declarando que **não foi colocada na mesa** e explicando qual seria o trade-off — em vez de fingir que o time a debateu.

### Iteração 3 — Confusão de altura entre RFC e FDD

**O erro:** a primeira versão do RFC trazia tabela de headers HTTP, exemplos de payload e a matriz de erros — ou seja, era um FDD menor. O enunciado é explícito sobre isso, e o conteúdo duplicado é sinal de que algo está no documento errado.

**Correção:** o detalhamento migrou inteiro para o FDD, e o RFC passou a apontar para lá. O critério aplicado na revisão foi: *o RFC pode citar uma decisão, mas não pode ser suficiente para implementá-la*. Em compensação, o RFC ganhou o que faltava e é genuinamente dele — a seção de questões em aberto com as quatro pendências reais da reunião, e o pedido nominal de revisão.

### Iteração 4 — Detalhes do código que a transcrição descreve por alto

**O erro:** a primeira versão do FDD repetia a transcrição sem conferir. Dizia "o worker abre um PrismaClient novo" e "as classes de erro herdam do padrão existente" — frases que soam certas e que um desenvolvedor não consegue executar.

**Correção, com o código aberto:**

| Ponto | O que a transcrição diz | O que o código mostra | O que o FDD passou a dizer |
| --- | --- | --- | --- |
| Client do worker | "instância nova, PrismaClient é por processo" `[09:30]` | `createPrismaClient()` já é exportada em `src/config/database.ts`, ao lado do singleton `prisma` | Use a factory existente; o singleton é o que `server.ts` consome |
| Erro `WEBHOOK_NOT_FOUND` | "códigos tipo `WEBHOOK_NOT_FOUND`" `[09:28]` | `NotFoundError` fixa `'NOT_FOUND'` no construtor | Estender `AppError` direto; herdar de `NotFoundError` exigiria alterar classe compartilhada |
| Nomes de campo | payload em snake_case `[09:43]` | API existente usa camelCase (`customerId`, `totalCents`) | Convenção explícita: gestão em camelCase, payload outbound em snake_case |
| Índices da outbox | "índice em status e `created_at`" `[09:08]` | — | Índice composto `(status, nextAttemptAt)`, porque o backoff faz o filtro real ser por `nextAttemptAt`; `createdAt` mantém índice próprio para a ordenação |

### Verificação final

Todos os checks passaram no estado entregue:

```
citações [hh:mm] Nome verificadas contra TRANSCRICAO.md .... 655 | inválidas: 0
caminhos de código citados ................................ 28 | inexistentes: 2
                                     (src/worker.ts e webhook.worker.ts, ambos
                                      artefatos a criar, sempre em tempo futuro)
links relativos markdown .................................. 115 | quebrados: 0
linhas do tracker ......... 269 | TRANSCRICAO 221 (82%) | CODIGO 48 (18%)
arquivos de código distintos no tracker ................... 24
```

---

## 6. Como navegar a entrega

```
.
├── README.md                    ← você está aqui (processo de produção)
├── desafio.md                     enunciado original
├── TRANSCRICAO.md                 fonte primária (não alterada)
└── docs/
    ├── PRD.md                     por que e o quê        (produto)
    ├── RFC.md                     como propomos resolver (arquitetura)
    ├── FDD.md                     como construir         (implementação)
    ├── TRACKER.md                 de onde veio cada coisa
    └── adrs/
        ├── README.md              índice dos ADRs
        ├── ADR-001-outbox-transacional-no-mysql.md
        ├── ADR-002-worker-em-processo-separado-com-polling.md
        ├── ADR-003-retry-com-backoff-exponencial-e-dlq.md
        ├── ADR-004-autenticacao-hmac-sha256-com-secret-por-endpoint.md
        ├── ADR-005-entrega-at-least-once-com-x-event-id.md
        ├── ADR-006-reuso-dos-padroes-existentes-do-projeto.md
        ├── ADR-007-snapshot-do-payload-na-insercao-do-evento.md
        └── ADR-008-filtragem-de-eventos-na-insercao-da-outbox.md
```

### Ordem sugerida de leitura

**Para entender a feature** (~15 min): [`docs/PRD.md`](docs/PRD.md) → [`docs/RFC.md`](docs/RFC.md).
Problema, escopo e por que a Atlas está impaciente; depois a proposta técnica e o que ficou em aberto.

**Para implementar** (~40 min): [`docs/RFC.md`](docs/RFC.md) → [`docs/adrs/`](docs/adrs/) → [`docs/FDD.md`](docs/FDD.md).
O RFC dá o mapa, os ADRs explicam por que cada decisão é o que é, e o FDD tem o modelo de dados, os fluxos, os contratos e a seção §11 com a integração arquivo por arquivo.

**Para revisar como revisor da reunião:** [`docs/RFC.md`](docs/RFC.md), direto na seção 8 ("Pedido aos revisores"), que lista as cinco perguntas nominais.

**Para auditar a rastreabilidade:** [`docs/TRACKER.md`](docs/TRACKER.md) ao lado de [`TRANSCRICAO.md`](TRANSCRICAO.md). Cada linha aponta para um `[hh:mm] Nome` ou um caminho de arquivo.

### Nota sobre o código

Nenhum arquivo em `src/`, `prisma/`, `tests/` ou de configuração foi modificado. A entrega é puramente documental; o código serviu de contexto e de verificação — os 26 arquivos de código citados nos documentos existem no repositório, e as duas exceções (`src/worker.ts` e `src/modules/webhooks/webhook.worker.ts`) são artefatos que a própria reunião propõe criar (`[09:11] Larissa`, `[09:28] Bruno`), sempre descritos em tempo futuro.
