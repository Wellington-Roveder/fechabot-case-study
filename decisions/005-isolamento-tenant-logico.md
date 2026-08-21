# ADR 005 — Isolamento multi-tenant lógico (`tenant_id`), não físico (schema por tenant)

**Status:** Aceito e implementado (desde a fundação do banco, Sprint 1)

---

## Contexto

O FechaBot atende múltiplos negócios (tenants) na mesma instância do
produto. Toda tabela que guarda dado de cliente precisa garantir que um
tenant nunca veja dado de outro — essa é a regra de isolamento mais
fundamental do sistema, presente desde a primeira sprint.

Existem, de forma geral, duas estratégias para isso num banco
relacional: **isolamento lógico** (uma coluna `tenant_id` em cada
tabela, com todo acesso filtrado por ela) ou **isolamento físico**
(schema — ou banco — separado por tenant, com o próprio Postgres
impedindo acesso cruzado por estrutura).

## Decisão

Isolamento lógico: `tenant_id` como coluna em toda tabela que armazena
dado de cliente, com a regra de que **esse valor sempre vem do JWT do
usuário autenticado, nunca de parâmetro de requisição**. Toda rota
protegida usa uma dependency de autenticação que expõe o tenant do
usuário logado, e toda query filtra por ele.

## Alternativas consideradas

### Schema (ou banco) separado por tenant

Isolamento físico foi considerado antes de decidir pela coluna. A
motivação é real: o próprio Postgres garantiria a separação por
estrutura, eliminando a possibilidade de uma query mal escrita vazar
dado entre tenants.

**Por que foi descartado, por agora:** o motivo decisivo não foi
técnico — foi de contexto de projeto. Manter N schemas sincronizados
(migração de schema precisa rodar em todos, backup e manutenção
multiplicam por tenant) é trabalho real de manter, e o projeto é
desenvolvido solo. Isolamento lógico permite seguir trabalhando sem
depender de processos externos de infraestrutura para cada novo
tenant, e sem multiplicar o esforço de manutenção a cada cliente novo.
Isolamento lógico também deixa mais espaço para construir sobre o
mesmo modelo — variações de query e filtro que fazem sentido para o
produto (ex: os filtros configuráveis do próprio FechaBot) partem da
mesma base de dado compartilhada, sem a fricção adicional de cruzar
schemas.

## Consequências

**Positivas:**
- Simplicidade operacional: uma migração, um schema, uma rotina de
  backup — sem multiplicar por tenant.
- Compatível com o ritmo de um projeto mantido solo, sem depender de
  automação de provisionamento de schema a cada tenant novo.
- Deixa espaço para evoluir queries e filtros sobre uma base de dado
  única, sem a complexidade extra de trabalhar através de múltiplos
  schemas.

**Trade-off assumido, e o mais importante desta decisão:**
Isolamento lógico depende de disciplina, não de garantia estrutural do
banco. Nada no Postgres impede, por natureza, uma query que esqueça o
filtro por `tenant_id` — se isso acontecer, o vazamento entre tenants é
real, não teórico. Esse risco foi reconhecido conscientemente: **um
desenvolvedor futuro que não filtre a query corretamente pode causar
um vazamento de dado entre clientes**, algo que isolamento físico
tornaria estruturalmente impossível.

A mitigação adotada não elimina o risco, mas reduz a superfície onde
ele pode aparecer: toda rota autenticada passa pela mesma dependency
de autenticação, que resolve o `tenant_id` a partir do JWT de forma
centralizada — a intenção é que nenhuma rota precise (ou consiga,
seguindo o padrão) decidir o tenant "na mão" a partir de um parâmetro
de requisição. Isso não é uma garantia do banco como um schema seria,
mas reduz a decisão de "qual tenant filtrar" a um único ponto do
código, em vez de repetida em cada endpoint.
