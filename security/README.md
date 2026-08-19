# Segurança — Revisão v3 (Sprint 10.6)

Encerramento de um roadmap de segurança conduzido em três fases
sequenciais, cada uma fechada somente após checklist de teste funcional
completo. Metodologia de referência: OWASP Top 10 (2021), OWASP ASVS,
OWASP API Security Top 10, CWE.

## Contexto

A avaliação original (v1) foi feita só com o frontend, com hipóteses
conservadoras sobre o que o backend fazia. A v2 reconciliou isso com o
backend real. Este documento (v3) fecha o ciclo: todos os achados de
severidade Alta e Média identificados foram implementados, testados e
validados — incluindo build de produção simulado, não só ambiente de
desenvolvimento.

**Resultado líquido:** saída de um modelo de sessão vulnerável a XSS
(JWT em `localStorage`, sem CSRF, sem CSP) para defesa em profundidade
completa — cookie `httpOnly` + CSRF via double-submit + CSP com nonce
por requisição + allowlist de sessão revogável + step-up authentication
+ MFA opcional (TOTP).

No processo, quatro bugs técnicos não previstos no escopo original
foram descobertos e corrigidos — documentados separadamente porque são
o tipo de problema que só aparece testando de ponta a ponta, não em
revisão estática de código.

## Linha do tempo

| Parte | Conteúdo |
|---|---|
| [Fase 1 — Modelo de sessão](./fase1-sessao-csrf-csp.md) | Cookie httpOnly, CSRF double-submit, CSP com nonce |
| [Fase 2 — Revogação e credenciais de terceiro](./fase2-redis-stepup.md) | Allowlist de sessão via Redis, step-up authentication, limite de payload da IA |
| [Fase 3 — MFA](./fase3-mfa.md) | TOTP opcional, códigos de backup, login em duas etapas |
| [Achados técnicos durante a implementação](./achados-tecnicos-implementacao.md) | 4 bugs descobertos testando ponta a ponta, não previstos no escopo original |

## Pontuação final

- **Security Score: 94/100** (subiu de 78/100 na avaliação anterior —
  todos os itens de severidade Alta/Média fechados; o não-100 reflete
  riscos residuais aceitos conscientemente, nenhum de severidade Alta)
- **OWASP Top 10 (2021):** A02 (Cryptographic/Storage Failures), A01
  (Broken Access Control) e A07 (Auth Failures) passaram de "fraco"
  para "forte" — nenhuma categoria com achado Alto ainda aberto
- **ASVS (Nível 1–2), estimado:** Session Management (V3) e
  Communications (V14), antes as áreas mais fracas, hoje estão entre as
  mais fortes do sistema

## Riscos residuais aceitos e documentados

Nem toda decisão de segurança termina em "zero risco". Estas foram
aceitas conscientemente, com justificativa registrada — não são
lacunas esquecidas:

| Item | Justificativa |
|---|---|
| `style-src 'unsafe-inline'` mantido no CSP | Acomoda estilos dinâmicos vindos de dados internos do componente, não de input de usuário. `script-src` — onde XSS de fato executa — não tem essa exceção |
| `DELETE /whatsapp-numbers/{id}` sem step-up auth | Só a criação/rotação de credencial (`POST`) exige senha. Remoção não introduz credencial maliciosa nova, é reversível e detectável pelo dono |
| MFA opt-in, não obrigatório | Decisão de produto: priorizar conversão de cadastro nesta fase, revisitável no futuro |

## Observações positivas consolidadas

Nem tudo na revisão foi achado a corrigir — alguns pontos já estavam
sólidos desde antes e foram apenas mantidos ou estendidos:

- Validação de JWT correta desde o início (algoritmo explícito, `exp`
  verificado, `bcrypt` para senha) — mantida e estendida com allowlist
  de revogação
- Isolamento multi-tenant consistente em toda a base de código, sem
  exceção encontrada
- SQL 100% parametrizado
- Rate limiting granular por tenant, estendido a todos os endpoints
  novos (MFA, logout-all)
- Criptografia consistente para credenciais de terceiro e secrets MFA,
  reaproveitando o mesmo módulo de criptografia
- Disposição de testar cada fase em profundidade antes de avançar — os
  quatro bugs documentados foram capturados justamente porque nenhuma
  fase foi dada como concluída sem checklist funcional completo

---

## Retrospectiva

Vale ser direto sobre isso: não sou especialista em segurança. Usei IA
como ferramenta de apoio ao longo desse processo — para pesquisar
padrões, validar abordagens, e às vezes gerar a implementação a partir
de uma decisão que eu já tinha tomado. Isso não é diferente de usar
qualquer outra ferramenta de produtividade no trabalho de engenharia
hoje.

O que considero o diferencial real aqui não é ter escrito cada linha
sozinho — é ter identificado os pontos de ataque certos para investigar
em primeiro lugar, saber o que pedir e por quê, e conseguir avaliar se
a solução proposta realmente fechava o problema ou só parecia fechar.
Segurança não é uma lista estática de checkboxes; é julgamento sobre
onde olhar e o que é suficiente para o contexto real do produto.

Olhando pra trás, considero que sempre há espaço pra melhoria — padrões
de mercado mudam, e tem coisa nova pra aprender continuamente nessa
área. Não vejo os 94/100 finais como um teto, mas como um retrato
honesto de onde o produto estava naquele momento, com as decisões que
fizeram sentido para a fase em que ele se encontrava. Se fosse decidir
de novo hoje, provavelmente chegaria em ajustes diferentes em pelo
menos alguns dos riscos residuais aceitos — não porque estavam errados
na época, mas porque segurança é um alvo que se move junto com o que
se aprende.

---

Pendências de baixo impacto identificadas durante esta revisão (itens
cosméticos, débitos técnicos não relacionados à segurança) estão
registradas em [`roadmap/`](../roadmap), não aqui — esta seção documenta
o que foi encontrado e corrigido, não o backlog.
