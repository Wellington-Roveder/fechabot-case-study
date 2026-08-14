# Decisões de Arquitetura (ADRs)

Registro das decisões técnicas do FechaBot — contexto, alternativas
consideradas e trade-offs assumidos, não só o resultado final.

| # | ADR | Resumo |
|---|---|---|
| 001 | [Modelo de sessão: cookie httpOnly + CSRF + allowlist Redis](./001-modelo-sessao-cookie-csrf-redis.md) | Por que sair de JWT em `localStorage` para um modelo com revogação real de sessão |
| 002 | [Categoria de template: `requested_category` vs `effective_category`](./002-categoria-template-requested-effective.md) | Por que a reclassificação automática de template pela Meta não pode sobrescrever silenciosamente a escolha do cliente |
| 003 | [Envio ao WhatsApp sempre via fila assíncrona (Celery)](./003-envio-assincrono-celery.md) | Por que o pipeline de envio nasceu assíncrono, sem passar por uma versão síncrona |

---

Cada ADR segue a mesma estrutura: **Contexto** → **Decisão** →
**Alternativas consideradas** → **Consequências** (positivas e
trade-offs assumidos, não só o lado bom). Quando aplicável, uma seção
de **Aprendizado técnico** ou **Validação** fecha o documento.
