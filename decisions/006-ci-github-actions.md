# ADR 006 — CI via GitHub Actions, com gate de cobertura incremental

**Status:** Aceito e implementado

---

## Contexto

Até este ponto, testes rodavam manualmente antes de cada push — sem
garantia de que rodavam sempre, nem de que uma alteração não quebrava
algo que já estava funcionando antes. Faltava um pipeline que rodasse
lint e testes automaticamente a cada mudança, contra um banco Postgres
real (consistente com a prática de teste já adotada no projeto — ver
`tests/conftest.py`, sem mocks de banco).

## Decisão

Pipeline de CI via GitHub Actions, acionado em push e pull request:

1. **Setup do ambiente** — dependência de sistema (`libmagic`, usada
   por `python-magic`), Python, dependências do projeto.
2. **Aplicação do schema do banco** — contra um Postgres real
   provisionado no próprio job, não um banco mockado.
3. **Lint** (`ruff`).
4. **Testes com cobertura** (`pytest --cov`), com **gate mínimo de
   cobertura configurado em 40%**.

O fluxo de trabalho passou a usar branches próprias por fix ou feature
(ex: `fix/webhook-malformed-payload`, `feature/utils-context-manager`),
com Pull Request para `develop` — o CI roda tanto na abertura quanto a
cada novo commit no PR, antes do merge.

## Alternativas consideradas

### Outra ferramenta de CI (GitLab CI, CircleCI, Jenkins)

Não foi avaliada a sério. O repositório já está hospedado no GitHub —
GitHub Actions não exige nenhuma configuração de conta adicional nem
integração externa para começar a funcionar. Para o estágio atual do
projeto, o custo de avaliar alternativas não se justificava frente a
uma opção que já "nasce funcionando" no mesmo lugar onde o código já
vive.

## O gate de cobertura como catraca incremental, não valor definitivo

O número 40% não foi escolhido como meta final — foi escolhido através
de um processo prático: a intenção original era travar em 80%, mas ao
testar contra a suíte de testes existente, a cobertura real bateu 44%.
Em vez de manter um gate inatingível (que bloquearia todo PR até a
suíte de testes crescer significativamamente), o piso foi ajustado para
40% — abaixo da cobertura atual, mas alto o suficiente para pegar uma
queda real de cobertura.

A ideia é subir esse piso aos poucos, conforme mais testes forem
escritos — o gate funciona como uma **catraca**: impede regressão
(cobertura não pode cair abaixo do piso atual), sem travar o
desenvolvimento no presente esperando por uma cobertura que ainda não
existe. É uma alternativa mais realista do que travar um número alto
"aspiracional" desde o início e ter que reduzir ou ignorar o gate toda
vez que ele bloqueia um PR legítimo.

## Consequências

**Positivas:**
- Toda mudança passa por lint e teste automaticamente antes do merge —
  não depende mais de lembrar de rodar localmente.
- Testes já rodam contra Postgres real no pipeline, mantendo a mesma
  garantia que a suíte já tinha localmente.
- O histórico de execuções do CI (visível na aba Actions) vira também
  um registro de que PRs que corrigiram bugs reais — como os dois
  itens de robustez documentados no [roadmap](../roadmap) — foram
  validados por teste antes de entrar na branch principal, não só
  declarados como corrigidos.

**Trade-offs assumidos:**
- 40% de cobertura mínima é um piso baixo — não é uma garantia forte
  de qualidade por si só, é o começo deliberado de um processo que
  sobe com o tempo.
- Pipeline atual não inclui deploy (CD) — fica restrito a lint e
  teste; deploy automatizado é item de roadmap futuro (Sprint 11).
