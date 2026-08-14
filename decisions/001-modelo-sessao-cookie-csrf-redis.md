# ADR 001 — Modelo de sessão: cookie httpOnly + CSRF double-submit + allowlist de sessão via Redis

**Status:** Aceito e implementado (Sprint 10.6)

---

## Contexto

O modelo original de autenticação do FechaBot guardava o JWT no
`localStorage` do navegador. Funcionalmente resolvia o login, mas
carregava três problemas que só ficam visíveis quando se pensa em
segurança de sessão como um todo, não só em "o usuário consegue
logar":

1. **Superfície de ataque a XSS.** Qualquer token acessível via
   JavaScript é um token que um script malicioso injetado na página
   também consegue ler.
2. **Nenhum mecanismo de revogação real.** Uma vez emitido, o JWT
   permanecia válido até a expiração natural — não havia como
   invalidar uma sessão comprometida ou fazer logout "de verdade"
   antes do token expirar por conta própria.
3. **Ausência de proteção CSRF e CSP coerente.** Essas três frentes
   (armazenamento de token, CSRF, CSP) são interdependentes: resolver
   uma sem as outras deixa lacuna.

A pergunta de arquitetura por trás desta decisão não foi só "onde
guardar o token", mas **"como uma sessão pode ser revogada de forma
confiável, antes da sua expiração natural, sem reintroduzir os
problemas que motivaram usar JWT em primeiro lugar"**.

## Decisão

Reconstruir o modelo de sessão em três peças que se sustentam juntas:

- **Cookie `httpOnly`, `Secure`, `SameSite=Lax`** para o token de
  sessão — inacessível a JavaScript no navegador, eliminando a via
  mais direta de exfiltração via XSS.
- **CSRF via double-submit cookie** — um segundo cookie, não-`httpOnly`,
  comparado a um header enviado pelo frontend em toda mutação. Cookie
  sozinho não autentica mais uma escrita sem esse par.
- **Allowlist de sessão via Redis** — a peça que resolve o problema
  central: a validade de uma sessão não depende só da assinatura e
  expiração do JWT. Ela também precisa existir como entrada ativa no
  Redis. Logout remove essa entrada — revogando de verdade, não só
  limpando o cookie do lado do cliente. `POST /auth/logout-all` invalida
  todas as sessões de um usuário de uma vez (usado, por exemplo, se o
  usuário suspeitar de acesso indevido à própria conta).

## Alternativas consideradas

### JWT de vida curta + refresh token

Essa foi a alternativa séria, avaliada antes da allowlist. É o padrão
mais comum na indústria: um access token de vida curta (minutos) e um
refresh token de vida mais longa, trocado por um novo access token
periodicamente.

**Por que foi descartada:** o ganho de segurança de um refresh token
rotativo depende de implementar rotação corretamente — detecção de
reuso do refresh token, invalidação em cadeia se um token antigo for
reutilizado, etc. Isso significa raciocinar sobre **dois tokens com
ciclos de vida diferentes** e sua interação. Uma consulta (ou
verificação de existência) no Redis a cada requisição autenticada é
conceitualmente mais simples: uma sessão existe ou não existe, sem
estado intermediário para gerenciar.

## Consequências

**Positivas:**
- Revogação de sessão é imediata e real — testada especificamente
  reutilizando um cookie de sessão após logout para confirmar que o
  acesso é negado, não só client-side.
- Modelo mental mais simples de raciocinar e de auditar: uma sessão
  existe no Redis ou não existe.
- `logout-all` se torna trivial de implementar — é uma operação sobre
  as entradas do usuário no Redis, não uma lógica especial separada.

**Trade-offs assumidos:**
- **Nova dependência crítica no caminho de autenticação.** Se o Redis
  cair, a validação de sessão para junto — diferente de um JWT
  puramente stateless, que continuaria validando localmente. Esse
  trade-off foi aceito conscientemente: a garantia de revogação real
  vale mais, para este produto, do que a resiliência a uma
  indisponibilidade do Redis (que já é dependência crítica de outras
  partes do sistema, como a fila Celery).
- **Uma consulta a mais por requisição autenticada.** Custo de latência
  pequeno, mas existe — não é "gratuito" como validar só a assinatura
  do JWT localmente.
- **CSRF exige disciplina contínua.** Toda rota nova precisa decidir
  conscientemente se entra ou não na exceção de CSRF, e essa decisão
  precisa ficar documentada — não é algo que se resolve uma vez e
  esquece.

## Aprendizado técnico

Durante a implementação, a ordem de registro de middleware no Starlette
se mostrou um ponto de atenção real: a ordem em que middlewares são
adicionados via `app.add_middleware(...)` não é a ordem em que eles
executam — é invertida. Isso já causou um bug onde um middleware
esperado para rodar "por fora" (mais externo na pilha) na prática
rodava por dentro, afetando o comportamento de headers de resposta.
A regra prática adotada depois: `CORSMiddleware` sempre deve ser o
último `add_middleware` chamado, para garantir que fique mais externo
na execução real.

Esse tipo de detalhe não aparece em nenhum tutorial de "como configurar
CORS" — só aparece depurando um comportamento inesperado em produção
local, o que reforça a importância de testar a pilha completa de
middlewares, não só cada um isoladamente.
