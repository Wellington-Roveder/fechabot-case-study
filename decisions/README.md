# Decisões de Arquitetura (ADRs)

Registro das decisões técnicas do FechaBot — contexto, alternativas
consideradas e trade-offs assumidos, não só o resultado final.

| # | ADR | Resumo |
|---|---|---|
| 001 | [Modelo de sessão: cookie httpOnly + CSRF + allowlist Redis](./001-modelo-sessao-cookie-csrf-redis.md) | Por que sair de JWT em `localStorage` para um modelo com revogação real de sessão |
| 002 | [Categoria de template: `requested_category` vs `effective_category`](./002-categoria-template-requested-effective.md) | Por que a reclassificação automática de template pela Meta não pode sobrescrever silenciosamente a escolha do cliente |
| 003 | [Envio ao WhatsApp sempre via fila assíncrona (Celery)](./003-envio-assincrono-celery.md) | Por que o pipeline de envio nasceu assíncrono, sem passar por uma versão síncrona |
| 004 | [MFA via TOTP, não SMS ou e-mail](./004-mfa-totp.md) | Por que o segundo fator de autenticação ficou contido no próprio sistema, sem dependência de canal externo |
| 005 | [Isolamento multi-tenant lógico, não físico](./005-isolamento-tenant-logico.md) | Por que `tenant_id` em coluna foi escolhido sobre schema separado, e o risco assumido nessa escolha |
| 006 | [CI via GitHub Actions, com gate de cobertura incremental](./006-ci-github-actions.md) | Por que o gate de cobertura começou baixo (40%) deliberadamente, como uma catraca que sobe com o tempo |

---

Cada ADR segue a mesma estrutura: **Contexto** → **Decisão** →
**Alternativas consideradas** → **Consequências** (positivas e
trade-offs assumidos, não só o lado bom). Quando aplicável, uma seção
de **Aprendizado técnico** ou **Validação** fecha o documento.
