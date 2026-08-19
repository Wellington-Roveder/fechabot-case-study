# Fase 2 — Revogação de sessão e credenciais de terceiro

Com a base de sessão da Fase 1 no lugar, esta fase endereça um problema
diferente: uma sessão válida (cookie correto, assinatura correta,
ainda não expirada) pode mesmo assim precisar ser invalidada antes do
tempo — por exemplo, se o usuário suspeitar de acesso indevido à
própria conta.

## P4 — Allowlist de sessão via Redis

**Severidade original:** Média | **CWE:** CWE-613

- **Decisão de escopo:** entre uma blacklist simples (negar só tokens
  explicitamente revogados) e uma allowlist (negar por padrão, exigir
  presença ativa), optou-se pela segunda — mais robusta, escolhida
  deliberadamente mesmo custando mais esforço de implementação.
- **Por que Redis e não Postgres:** guardar as sessões ativas
  diretamente no banco relacional também foi considerado. A escolha
  por Redis não veio de uma vantagem técnica abstrata — veio de reuso
  de infraestrutura já existente: o Redis já rodava no projeto como
  broker do Celery, então usá-lo para a allowlist evitou introduzir
  mais uma peça de infraestrutura nova só para este propósito.
- **Implementação:** toda sessão ativa é registrada no Redis no
  momento do login, com TTL alinhado à expiração do token. A
  verificação de autenticação passa a checar essa allowlist além da
  assinatura e expiração do token em si. Logout remove o registro —
  revogação real, não só client-side. Um endpoint adicional permite
  revogar todas as sessões de um usuário de uma vez.
- **Validação:** testado com duas sessões simultâneas — logout em uma
  invalida a outra imediatamente, confirmado tanto pela interface
  quanto reutilizando o cookie de sessão diretamente via requisição
  manual (não só assumindo que o navegador "esqueceu" o cookie).

Mais contexto sobre essa decisão específica, incluindo a alternativa
descartada (JWT curto + refresh token), no
[ADR 001 — Modelo de sessão](../decisions/001-modelo-sessao-cookie-csrf-redis.md).

## P5 — Step-up authentication para credencial WhatsApp

**Severidade original:** Média | **CWE:** CWE-620

- **Decisão de escopo:** aplicado apenas na criação/rotação da
  credencial (não na remoção) — decisão consciente, já que remover uma
  credencial não introduz nada malicioso, é reversível e perceptível
  pelo próprio dono da conta.
- **Implementação:** a senha atual do usuário é exigida e revalidada
  antes de aceitar uma nova credencial de WhatsApp Business — mesmo com
  uma sessão já autenticada e válida. O limite de tentativas para essa
  rota foi alinhado ao mesmo padrão usado contra força-bruta no login.
- **Validação:** três cenários testados — sem senha informada (bloqueado
  antes mesmo de chegar ao servidor), senha incorreta (rejeitado com
  mensagem específica), senha correta (aceito).

## P6 — Limite de tamanho no histórico de conversa da IA

**Severidade original:** Baixa-Média | **CWE:** CWE-770

Uma mudança pequena, mas que fecha uma superfície real: o histórico de
conversa enviado ao endpoint de chat com a IA passou a ter um limite
rígido de tamanho no backend — impedindo que uma chamada direta à API
(fora do fluxo normal do frontend) enviasse um payload arbitrariamente
grande e gerasse custo desproporcional de uso do modelo de IA.

## Por que essas três juntas

P4 e P5 resolvem a mesma pergunta de fundo — "uma sessão ou ação
sensível pode ser revogada/exigir prova extra mesmo estando
tecnicamente válida?" — só que em pontos diferentes: P4 para a sessão
inteira, P5 para uma ação específica de alto impacto (trocar a
credencial que autoriza o envio de mensagens em nome do negócio do
cliente). P6 é menor em escopo, mas foi agrupado nesta fase por ser
outra forma da mesma categoria: limitar o que uma sessão autenticada
pode fazer, mesmo sendo legítima.
