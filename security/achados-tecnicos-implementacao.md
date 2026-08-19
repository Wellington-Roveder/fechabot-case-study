# Achados técnicos descobertos durante a implementação

Nenhum destes estava previsto no escopo original de segurança — todos
apareceram testando o fluxo real de ponta a ponta, não em revisão
estática de código. Documentados separadamente porque são o tipo de
problema que pode se repetir se outro desenvolvedor (ou eu mesmo, no
futuro) adicionar algo parecido sem essa lição em mente.

## Ordem de registro de middleware é invertida na execução real

Frameworks baseados em Starlette (como FastAPI) registram middlewares
numa ordem que **não é** a ordem em que eles processam a requisição —
é invertida. Registrar o middleware de CORS antes dos middlewares de
CSRF e headers de segurança fazia o CORS ficar mais "interno" do que o
esperado na pilha de execução real. Como os outros dois middlewares
reconstróem a resposta internamente, múltiplos `Set-Cookie` na resposta
de login estavam sendo mesclados ou perdidos.

**Correção:** o middleware de CORS precisa ser o último a ser
registrado, para ficar mais externo na execução real. Regra registrada
para qualquer middleware novo adicionado no futuro — esse tipo de bug
não aparece em teste unitário isolado de cada middleware, só testando a
pilha completa.

## CSS importado sem escopo vaza globalmente

Um arquivo CSS importado dentro de um componente de rota específica
não fica automaticamente restrito a essa rota no Next.js — o bundler
injeta no bundle global da aplicação. Um reset de estilo mantido por
precaução durante a migração de estilos inline para Tailwind estava
sobrescrevendo silenciosamente os estilos base do Tailwind em **todas
as outras rotas** (login, dashboard, configurações) — não só na rota
onde o import foi escrito.

**Correção:** removido — era redundante desde o início, já que o
Tailwind já cobria a mesma função.

## CSP com nonce não alcança páginas pré-renderizadas estaticamente

Em build de produção, páginas sem dependência dinâmica são
pré-renderizadas em tempo de build pelo Next.js — o HTML fica "gerado
uma vez" sem o nonce, que por definição só existe por requisição (não
existe ainda no momento do build). Isso derrubava o CSP dessas páginas
especificamente em produção — o problema não aparecia em ambiente de
desenvolvimento, só em build de produção real.

**Correção:** forçar renderização dinâmica em todas as rotas do
frontend. Trade-off aceito conscientemente: perda de uma otimização de
performance (páginas estáticas são mais rápidas de servir) em troca de
CSP funcionando de forma consistente em toda a aplicação. Registrado
como item de otimização futura, não como pendência de segurança.

## Middleware de CSRF bloqueava a rota nova de verificação de MFA

Ao adicionar a rota de verificação de código MFA, ela não foi incluída
na lista de exceções do middleware de CSRF. Nesse ponto específico do
fluxo de login, o usuário só tem o token intermediário de MFA (não a
sessão completa com o cookie de CSRF correspondente) — então toda
tentativa de verificar um código MFA estava sendo bloqueada antes de
chegar à lógica de negócio. Para piorar o diagnóstico, o erro aparecia
no navegador como uma falha de CORS, o que mascarava a causa raiz real.

**Correção:** rota adicionada à lista de exceções, com a mesma
justificativa já documentada para a rota de login — sem sessão completa
neste ponto do fluxo, a proteção vem do token de vida curta e do rate
limiting, não do CSRF.

---

O padrão comum entre os quatro: nenhum apareceria em uma revisão
estática de código linha por linha. Todos exigiram rodar o fluxo
completo — incluindo build de produção, não só o servidor de
desenvolvimento — para se manifestar.
