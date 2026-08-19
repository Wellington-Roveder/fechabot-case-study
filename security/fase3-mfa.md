# Fase 3 — MFA (TOTP)

Última fase do roadmap — a mais visível para o usuário final,
implementada por último porque depende das duas anteriores: MFA sobre
um modelo de sessão sem revogação real (Fase 2) ou vulnerável a XSS
(Fase 1) teria valor de segurança bem menor.

## P8 — MFA via TOTP

**Severidade original:** Média (defesa em profundidade) | **CWE:** CWE-308

- **Decisão de escopo:** opt-in, não obrigatório para toda conta —
  decisão de produto, priorizando conversão de cadastro sobre fricção
  de segurança nesta fase específica do produto. Fica registrado como
  algo revisitável no futuro, não como decisão permanente.
- **Ativação:** o usuário recebe um secret e QR code para configurar no
  aplicativo autenticador de sua escolha; só depois de confirmar um
  código válido é que o secret é persistido (criptografado) e 10
  códigos de backup de uso único são gerados e exibidos — uma única
  vez, no momento da ativação.
- **Login em duas etapas:** quando MFA está ativo, a senha correta não
  emite sessão completa de imediato — emite um token intermediário de
  vida curta (5 minutos), sem os privilégios de uma sessão real. Só
  depois do código TOTP (ou um código de backup) ser validado é que a
  sessão completa é emitida.
- **Desativação:** segue o mesmo padrão de step-up da Fase 2 — exige
  reentrada de senha, não só um clique em um botão de configurações.
- **Frontend:** a tela de MFA foi implementada como rota isolada,
  decisão consciente para não mexer no arquivo de configurações
  existente (grande e sensível o suficiente para não valer o risco de
  uma mudança não relacionada ali).
- **Validação:** fluxo completo testado — ativação, login com código
  TOTP, login com código de backup (confirmando que o uso único é
  respeitado — o mesmo código não funciona duas vezes), e desativação.

## Por que token intermediário, não sessão parcial

A escolha de emitir um token de curta duração e sem privilégios de
sessão — em vez de, por exemplo, uma sessão "fraca" que já permite
algum acesso — evita a categoria inteira de bugs onde alguém encontra
uma rota que "esqueceram" de proteger contra sessões pré-MFA. Um token
sem claims de sessão simplesmente não abre superfície nenhuma além do
próprio endpoint de verificação.
