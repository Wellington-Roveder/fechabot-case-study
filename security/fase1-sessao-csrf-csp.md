# Fase 1 — Modelo de sessão: cookie httpOnly, CSRF, CSP

Primeira fase do roadmap de segurança da Sprint 10.6 — endereça a base
do modelo de sessão antes de qualquer coisa mais avançada (revogação,
MFA) fazer sentido.

## P1 — Migração de token para cookie httpOnly

**Severidade original:** Alta | **CWE:** CWE-522, CWE-921

- **Antes:** token de sessão salvo em `localStorage` — legível por
  qualquer script executando na página, incluindo um eventual XSS de
  terceiro.
- **Depois:** o login responde via `Set-Cookie` com o token marcado
  `HttpOnly`, `Secure`, `SameSite=Lax`. O frontend passa a enviar
  credenciais automaticamente via cookie, sem manusear token algum em
  JavaScript.
- **Validação:** confirmado via DevTools (flag HttpOnly presente) e via
  teste funcional completo — login → navegação → logout → bloqueio de
  acesso pós-logout.

Mais contexto sobre essa decisão específica no
[ADR 001 — Modelo de sessão](../decisions/001-modelo-sessao-cookie-csrf-redis.md).

## P2 — CSRF via double-submit cookie

**Severidade original:** Informativo → Alta condicional | **CWE:** CWE-352

- **Implementação:** um segundo cookie, não-`httpOnly` (portanto legível
  por JavaScript), é comparado contra um header enviado pelo frontend em
  toda mutação. Um cookie de sessão sozinho não autentica mais uma
  escrita sem esse par presente e batendo.
- **Exceções documentadas:** rotas de login e verificação de MFA (a
  sessão ainda não existe nesses pontos) e o endpoint de webhook
  (autenticado via assinatura HMAC, não por cookie). Cada exceção tem
  justificativa registrada — nenhuma rota fica isenta "porque sim".
- **Validação:** confirmado em múltiplas mutações reais (criar/apagar
  destinatário, salvar número WhatsApp) — header presente e coerente
  com o cookie em cada uma.

## P3 — CSP com nonce por requisição + headers completos

**Severidade original:** Média | **CWE:** CWE-1021, CWE-693

- **Implementação:** nonce único gerado a cada requisição, usado para
  permitir apenas scripts explicitamente marcados — sem depender de
  `unsafe-inline` ou `unsafe-eval` em produção. Headers de segurança
  complementares (`X-Frame-Options`, `X-Content-Type-Options`, HSTS,
  Permissions-Policy, Referrer-Policy) passaram a cobrir tanto a API
  quanto as páginas HTML renderizadas pelo frontend — uma lacuna que a
  avaliação anterior não tinha identificado, já que os dois níveis
  (backend e frontend) precisam de proteção própria; um não cobre o
  outro.
- **Validação:** console limpo em desenvolvimento e em build de
  produção; teste de clickjacking confirmado — um iframe externo
  tentando embutir a aplicação foi bloqueado por duas camadas
  independentes ao mesmo tempo (política de frame-ancestors e o header
  clássico de X-Frame-Options), não só uma.

## Por que nessa ordem

Cookie httpOnly sozinho resolve XSS lendo o token, mas abre superfície
para CSRF (o navegador ainda envia o cookie automaticamente em
requisições cross-site). CSRF sozinho não faz sentido sem antes ter um
cookie de sessão para proteger. E CSP é o que fecha a lacuna de onde um
script malicioso poderia sequer ser injetado em primeiro lugar. As três
peças são interdependentes — não fazia sentido implementar fora dessa
ordem.
