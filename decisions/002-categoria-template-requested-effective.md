# ADR 002 — Categoria de template: `requested_category` separada de `effective_category`

**Status:** Aceito e implementado (Sprint 10.5)

---

## Contexto

Um template de mensagem WhatsApp aprovado pela Meta tem uma categoria
(`UTILITY`, `MARKETING`, etc.) que afeta custo e regras de envio. O
cliente escolhe essa categoria ao criar o template. O problema: a Meta
pode **reclassificar a categoria de um template já aprovado**, de forma
automática, via webhook — sem que o cliente peça ou seja avisado antes
do fato acontecer.

Isso levanta uma pergunta que não é só técnica, é de experiência do
usuário: **quando isso acontece, o que o sistema faz com a categoria
que o cliente originalmente escolheu?**

Se o sistema simplesmente sobrescrevesse a categoria com o valor vindo
da Meta, duas coisas se perderiam:

1. **A intenção original do cliente.** Não haveria mais registro do que
   ele pediu — só do que está valendo agora. Isso inviabiliza qualquer
   fluxo de "isso mudou em relação ao que você configurou".
2. **O consentimento informado.** Uma mudança de categoria pode ter
   implicação de custo e de regras de uso para o cliente. Ele precisa
   ser avisado e concordar em continuar usando o template nessas novas
   condições — não é algo que o sistema deveria decidir silenciosamente
   por ele.

## Decisão

Separar em dois campos com fontes de verdade distintas:

- **`requested_category`** — a categoria que o cliente escolheu.
  **Nunca é tocada pelo webhook da Meta.** É a "intenção declarada" e
  permanece estável até o cliente mudar ativamente.
- **`effective_category`** — a categoria que está realmente valendo na
  Meta neste momento. Só muda através de um evento real recebido via
  webhook (`template_category_update`).

Junto com isso:

- **`template_category_history`** — tabela que registra toda mudança
  real de `effective_category`, com guarda de idempotência (comparação
  contra o valor atual no banco via `SELECT ... FOR UPDATE`, já que
  webhooks da Meta não garantem ordem de entrega).
- **Notificação persistente ao cliente** — quando `effective_category`
  diverge de `requested_category`, o cliente vê um modal (não um banner
  que desaparece) com duas opções explícitas: **"Continuar usando"** ou
  **"Criar nova versão"**. A escolha fica registrada
  (`category_change_acknowledged`, `category_changed_at`).

O ponto central da decisão não é o modelo de dados em si — é que ele
**existe para viabilizar o consentimento**. Sem os dois campos
separados, não haveria como o sistema saber "o cliente já foi avisado
dessa mudança específica e concordou" versus "isso mudou e ninguém
percebeu ainda".

## Alternativas consideradas

### Um único campo, sobrescrito pelo webhook

A alternativa mais simples: `category` como campo único, atualizado
diretamente quando a Meta notifica uma mudança.

**Por que foi descartada:** essa abordagem resolveria o problema técnico
("qual categoria está valendo agora") mas quebraria a experiência que
o produto precisa entregar. Sem um campo que preserve o que o cliente
pediu originalmente, não há como construir o fluxo de "isso mudou,
você concorda em continuar?" — o sistema perderia a capacidade de
diferenciar entre "o cliente sabe e aceitou" e "isso simplesmente
aconteceu por baixo dos panos". A rastreabilidade não era um requisito
técnico abstrato — era o que viabilizava avisar e obter concordância
do cliente, que é o requisito real por trás da decisão.

## Consequências

**Positivas:**
- O cliente nunca é surpreendido silenciosamente por uma mudança de
  categoria — ele explicitamente reconhece a mudança antes de continuar.
- Auditoria completa: é possível responder "o que o cliente pediu" e
  "o que estava valendo" para qualquer ponto no tempo, via
  `template_category_history`.
- A idempotência via `SELECT ... FOR UPDATE` protege contra
  duplicidade em caso de reentrega de webhook pela Meta — cenário real
  de webhooks, não hipotético.

**Trade-offs assumidos:**
- Mais uma tabela e mais um fluxo de UI (modal de reconhecimento) para
  manter — a simplicidade de um campo único foi trocada por
  corretude de UX.
- O modal de reconhecimento é um ponto de fricção deliberado: o cliente
  precisa interagir antes de seguir usando o template normalmente. Essa
  fricção é intencional — é o mecanismo de consentimento — mas exige
  cuidado de copy para não parecer um bloqueio arbitrário.

## Validação

Testado com 41 casos automatizados contra Postgres real, cobrindo
contrato HTTP, segurança do webhook (assinatura inválida), idempotência
sob reentrega, e isolamento entre tenants. Validado também ponta a
ponta com um evento real de reclassificação vindo da Meta, não só em
ambiente simulado.
