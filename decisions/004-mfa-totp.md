# ADR 004 — MFA via TOTP, não SMS ou e-mail

**Status:** Aceito e implementado (Sprint 10.6, Fase 3)

---

## Contexto

Com o modelo de sessão já reconstruído (cookie httpOnly + CSRF +
allowlist Redis — [ADR 001](./001-modelo-sessao-cookie-csrf-redis.md)),
a próxima camada de defesa em profundidade fazia sentido ser MFA: mesmo
que uma senha vaze, uma segunda prova de identidade barra o acesso.

A pergunta de arquitetura não era só "implementar MFA", mas **qual
canal usar** — e isso tem implicações reais de custo, dependência
externa e complexidade de infraestrutura, não só de segurança.

## Decisão

TOTP (Time-based One-Time Password, o padrão usado por apps como
Google Authenticator e Authy), opt-in por usuário:

- Ativação gera um secret e QR code; só é persistido (criptografado)
  depois que o usuário confirma um código válido gerado a partir dele.
- 10 códigos de backup de uso único são gerados no momento da ativação,
  exibidos uma única vez.
- Login com MFA ativo passa por duas etapas: senha correta emite um
  token intermediário de vida curta (5 minutos, sem privilégios de
  sessão), e só o código TOTP (ou um código de backup) correto emite a
  sessão completa.

Mais detalhes de implementação e validação em
[`security/fase3-mfa.md`](../security/fase3-mfa.md).

## Alternativas consideradas

### SMS e e-mail

Ambos foram cogitados antes do TOTP. A investigação incluiu pesquisar
como sistemas de autenticação OAuth costumam configurar múltiplos
canais de MFA em conjunto.

**Por que foram descartados, por agora:** SMS e e-mail como canal de
MFA introduzem uma dependência de infraestrutura de envio (gateway de
SMS, ou um serviço de e-mail transacional dedicado) que o projeto não
tinha motivo para adicionar só para isso. TOTP, em comparação,
**fica inteiramente dentro do próprio sistema** — não depende de
nenhum serviço de terceiro no caminho crítico do login, exigiu poucas
migrações de banco, e tem bibliotecas maduras (`pyotp`, `qrcode`) que
resolvem a geração e validação do código sem reinventar a criptografia
por trás do padrão.

Essa decisão teve também um componente deliberado de aprendizado: parte
do valor de implementar TOTP especificamente foi entender como o
padrão funciona por baixo — a base do algoritmo, o papel do secret
compartilhado, a janela de tolerância de tempo — não só integrar uma
biblioteca sem entender o que ela faz.

## Consequências

**Positivas:**
- Zero dependência de infraestrutura externa para o segundo fator —
  nada que possa falhar ou atrasar por causa de um gateway de SMS fora
  do ar.
- Sem custo por verificação (SMS tem custo por envio; TOTP não).
- Implementação ficou contida — poucas migrações, sem novo serviço no
  docker-compose.

**Trade-offs assumidos:**
- **Fricção de setup maior para o usuário final.** TOTP exige instalar
  um app autenticador; SMS não exige nada além de um celular já em
  mãos. Para um público de gestores de PME que talvez não usem esse
  tipo de app no dia a dia, isso é uma barreira de adoção real —
  parcialmente mitigado por ser opt-in, não obrigatório.
- **Perda do dispositivo autenticador é um cenário de recuperação mais
  manual** do que "perder o celular que recebe SMS" — daí a
  importância dos códigos de backup gerados na ativação.
- **SMS/e-mail como canal adicional continua em aberto** como
  possibilidade futura, não descartado permanentemente — a decisão foi
  sobre o que fazia sentido implementar primeiro, com o orçamento de
  tempo e complexidade que o projeto tinha nesta fase, não uma rejeição
  definitiva de outros canais.
