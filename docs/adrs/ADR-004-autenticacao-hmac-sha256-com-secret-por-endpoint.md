# ADR-004 — Autenticação das entregas com HMAC-SHA256 e secret única por endpoint

- **Status:** Aceito
- **Data do registro:** 2026-09-22
- **Decisores:** Sofia (Eng. de Segurança), Diego (Eng. Sênior / Plataforma), Bruno (Eng. Pleno / Pedidos), Larissa (Tech Lead)
- **Fonte:** `TRANSCRICAO.md` `[09:19]`–`[09:24]`, `[09:44]`

## Contexto

A feature expõe dados de pedidos para um endpoint HTTP fora da nossa infraestrutura. O cliente que recebe a requisição precisa de duas garantias `[09:19] Sofia`:

1. **Autenticidade** — a requisição veio realmente da nossa plataforma, e não de alguém se passando por ela.
2. **Integridade** — o payload não foi adulterado no caminho.

O cenário também tem um histórico relevante: já houve cliente que vazou a própria secret em log de aplicação `[09:22] Diego`. Qualquer desenho precisa tratar o vazamento como evento esperado, não excepcional.

## Decisão

**Assinatura HMAC-SHA256 sobre o corpo da requisição, com secret única por endpoint de webhook e suporte a rotação com grace period de 24 horas** `[09:22] Sofia`.

### Assinatura

- Algoritmo: **HMAC-SHA256**, escolhido por ser o padrão de mercado e ter biblioteca disponível em qualquer stack séria do lado do cliente `[09:20] Sofia`.
- Entrada da assinatura: o **corpo do request** `[09:22] Sofia`.
- A assinatura viaja no header **`X-Signature`** `[09:20] Sofia`.
- Acompanham o envio os headers `X-Timestamp`, com o timestamp do envio, para que o cliente possa detectar replay attack se quiser, e `X-Webhook-Id`, com o id do cadastro de webhook, para que clientes com vários endpoints saibam qual cadastro originou aquele envio `[09:44] Diego`, `[09:44] Sofia`.

### Gestão da secret

- **Uma secret por endpoint de webhook**, não uma secret global da plataforma. Se uma vaza, vaza só aquele endpoint `[09:21] Sofia`.
- A secret é **gerada pela plataforma** e devolvida ao cliente **na criação** do webhook `[09:31] Marcos`.
- A secret é **rotacionável por endpoint de API**: o cliente pede uma nova secret quando quiser `[09:21] Sofia`.
- **Grace period de 24 horas:** ao rotacionar, a secret antiga continua válida em paralelo por 24h, dando ao cliente tempo de migrar os sistemas dele. Depois disso, a antiga morre `[09:21] Sofia`.
- O modelo de dados do endpoint guarda `url`, `secret`, `customer_id` e estado ativo `[09:21] Bruno`.

### Transporte

URL de webhook **tem que ser `https`**. Cadastro com `http` é recusado com erro de validação. Isso não é uma decisão arquitetural própria, e sim uma validação no schema Zod do módulo `[09:23] Sofia` — ver [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) e a matriz de erros no FDD (`WEBHOOK_INVALID_URL`).

## Alternativas Consideradas

### A. Secret global da plataforma

Descartada explicitamente. Uma secret compartilhada entre todos os clientes transforma um único vazamento em comprometimento de toda a base: "se vaza uma, vaza tudo" `[09:21] Sofia`.

**Trade-off recusado:** gestão de credenciais trivial (uma chave, um lugar) em troca de raio de explosão máximo em caso de vazamento.

### B. Rotação imediata, sem grace period

Descartada. Invalidar a secret antiga no instante da rotação quebra a integração do cliente entre o momento em que ele pede a nova chave e o momento em que ele termina de aplicá-la nos sistemas dele. As 24h existem justamente para cobrir essa janela `[09:21] Sofia`.

**Trade-off recusado:** revogação instantânea em troca de downtime garantido na integração do cliente a cada rotação.

### C. Algoritmos alternativos de assinatura (ex.: par de chaves assimétrico)

Não chegou a ser colocado na mesa como proposta concorrente. Quando Bruno perguntou "HMAC com qual algoritmo?", a resposta fechou em SHA-256 pelo critério de padrão de mercado e disponibilidade de biblioteca do lado do cliente `[09:20] Bruno`, `[09:20] Sofia`. Assinatura assimétrica dispensaria o compartilhamento de segredo, mas aumentaria a barreira de integração para os clientes B2B, que é exatamente o custo que o projeto quer evitar.

**Trade-off recusado:** dispensar o segredo compartilhado em troca de integração mais difícil para o cliente.

## Consequências

### Positivas

- O cliente consegue validar origem e integridade com poucas linhas de código, usando biblioteca padrão.
- Vazamento de secret fica contido a um único endpoint de um único cliente `[09:21] Sofia`.
- A rotação é uma operação self-service do cliente, sem ticket e sem janela de indisponibilidade `[09:21] Sofia`.
- `X-Timestamp` dá ao cliente o material necessário para implementar proteção contra replay, se ele quiser `[09:44] Diego`.

### Negativas

- **O segredo é compartilhado.** Ambos os lados guardam a mesma chave, e o vazamento no lado do cliente — que já aconteceu `[09:22] Diego` — está fora do nosso controle.
- Durante as 24h de grace period **duas secrets são válidas simultaneamente** para o mesmo endpoint, o que amplia a janela de exposição de uma secret potencialmente já comprometida e exige modelagem de mais de uma chave ativa por endpoint.
- A secret precisa ser armazenada de forma recuperável (não é hash) para assinar cada envio, o que a torna um dado sensível em repouso no banco. O logger do projeto já redige campos sensíveis (`src/shared/logger/index.ts`), e a lista de `redactPaths` precisa ser estendida para cobrir a secret de webhook.
- Não há proteção contra replay imposta por nós: detectar reenvio é responsabilidade opcional do cliente `[09:44] Diego`.
- Sofia pediu **pelo menos dois dias úteis de revisão de segurança** sobre HMAC e geração de secret antes do deploy `[09:46] Sofia`. Isso é uma dependência de cronograma, não opcional.

## Decisões relacionadas

- [ADR-005 — Entrega at-least-once com X-Event-Id](ADR-005-entrega-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](ADR-006-reuso-dos-padroes-existentes-do-projeto.md)
