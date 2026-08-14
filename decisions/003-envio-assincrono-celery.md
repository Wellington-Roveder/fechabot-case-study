# ADR 003 — Envio ao WhatsApp sempre via fila assíncrona (Celery), nunca chamada síncrona

**Status:** Aceito e implementado (Sprint 4, endurecido na Sprint 10)

---

## Contexto

Cada fechamento processado pode ter **múltiplos destinatários** — um
mesmo envio dispara N mensagens para N números diferentes. Além disso,
a API da Meta (WhatsApp Cloud API) é uma dependência de terceiro: sua
latência e disponibilidade não estão sob controle do FechaBot.

Isso coloca duas restrições sobre como o envio deveria funcionar:

1. Bloquear a requisição HTTP do cliente até que N chamadas sequenciais
   à Meta terminem não escala — o tempo de resposta cresceria
   linearmente com o número de destinatários, e o cliente ficaria
   esperando por algo que não deveria depender dele estar com o
   navegador aberto.
2. Acoplar a resposta ao usuário à latência (ou instabilidade) de um
   serviço externo é uma decisão de arquitetura ruim por si só — uma
   lentidão momentânea da Meta não deveria travar a experiência de
   quem está usando o FechaBot.

Por essas duas razões, a decisão de usar fila assíncrona foi tomada
**antes** de qualquer implementação síncrona — não houve uma versão
anterior síncrona que depois foi trocada; o pipeline já nasceu
assíncrono.

## Decisão

Todo envio ao WhatsApp passa por fila (Celery), nunca por chamada
síncrona dentro do ciclo de requisição-resposta:

- O endpoint que processa um fechamento (`POST /upload/process`)
  enfileira uma task por destinatário e retorna imediatamente — não
  espera nenhuma chamada à Meta terminar.
- Cada task tem **retry automático: 3 tentativas, intervalo de 60s**
  entre elas, para lidar com falhas transitórias (timeout, instabilidade
  momentânea da Meta).
- **Toda tentativa de envio deixa rastro em `sends`** — sucesso ou
  falha — nunca é descartada silenciosamente. Essa regra foi reforçada
  depois que um bug real foi encontrado (Sprint 10.5): tentativas que
  falhavam *antes* de qualquer chamada à Meta não estavam sendo
  gravadas. Corrigido e coberto por teste.
- Um bug de duplicidade nos retries também foi corrigido na Sprint 10:
  o registro de falha (`failed`) estava sendo salvo a cada tentativa
  de retry, gerando duplicatas — corrigido para gravar apenas na
  primeira tentativa.

## Alternativas consideradas

Nenhuma alternativa síncrona foi implementada ou avaliada a sério antes
desta decisão — o que existe para registrar aqui não é uma comparação
entre duas implementações, mas o raciocínio que descartou a opção
síncrona antes mesmo de código ser escrito: com múltiplos destinatários
por envio e uma dependência externa de latência variável, bloquear a
resposta do usuário nunca foi uma opção viável o suficiente para
justificar prototipar.

## Consequências

**Positivas:**
- Tempo de resposta do endpoint de processamento não depende do número
  de destinatários nem da velocidade da Meta.
- Falhas transitórias da Meta se recuperam sozinhas via retry, sem
  intervenção manual.
- Rastro completo de toda tentativa de envio — inclusive falhas —
  vira histórico auditável, não just "sumiu".

**Trade-offs assumidos:**
- **Complexidade operacional adicional.** Celery + broker (Redis) é
  mais peça de infraestrutura para manter no ar do que uma chamada
  HTTP direta seria.
- **Feedback não é instantâneo para o usuário.** Como o envio é
  assíncrono, o cliente não sabe no mesmo instante se a mensagem
  chegou — precisa consultar o histórico depois. Esse é um trade-off
  consciente de UX em troca de resiliência.
- **Retry duplicado é uma classe de bug real, não hipotética.** A
  correção da Sprint 10 (duplicata de `failed` nos retries) é prova de
  que sistemas de retry introduzem sua própria categoria de bugs que
  não existiriam numa chamada síncrona simples — o preço de resiliência
  é superfície adicional para acertar.

## Regra de arquitetura resultante

Esta decisão virou regra permanente do projeto: **fila para qualquer
envio externo — nunca chamada síncrona ao WhatsApp**, documentada como
princípio de arquitetura, não só como escolha pontual desta feature.
