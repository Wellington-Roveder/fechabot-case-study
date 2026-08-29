# Roadmap

Status de desenvolvimento por sprint e débitos técnicos conhecidos.
Itens de segurança já corrigidos ou aceitos como risco documentado
estão em [`security/`](../security), não aqui — esta página cobre
funcionalidade e débito técnico geral.

## Concluído

| Sprint | Entrega |
|---|---|
| 1 | Fundação do banco — schema multi-tenant (tenants, users, filters, templates, sends, whatsapp_numbers, recipients) |
| 2 | Filtros dinâmicos sobre planilha |
| 3 | Autenticação JWT (bcrypt + python-jose) |
| 4 | Integração Meta Cloud API + Celery — pipeline assíncrono de envio, retry automático |
| 5 | Pipeline completo de upload → processamento → disparo, CRUD de templates/destinatários/números |
| 6 | Frontend MVP — login, dashboard, novo envio, histórico, configurações |
| 7 | Integração front + back validada, primeira mensagem real entregue via WhatsApp |
| 8 | Melhorias pós-MVP — filtros compostos (AND/AND NOT), correções de segurança pontuais |
| 9 | FechaBot IA — chat em linguagem natural sobre os dados do tenant, isolamento por tenant, janela de histórico |
| 10 | Hardening de segurança do backend — rate limiting, CORS, headers, `SecretStr` em credenciais, validações de schema reforçadas (17 itens corrigidos) |
| 10.5 | Reclassificação automática de categoria de template pela Meta — ver [ADR 002](../decisions/002-categoria-template-requested-effective.md) |
| 10.6 | Reconstrução completa do modelo de sessão + MFA — ver [`security/`](../security) |
| — | CI via GitHub Actions — lint, testes contra Postgres real, gate de cobertura incremental. Ver [ADR 006](../decisions/006-ci-github-actions.md) |

## Planejado

| Sprint | Escopo |
|---|---|
| 11 | SaaS completo — onboarding self-service, cobrança recorrente, monitoramento, deploy em produção, CD (o CI já está em produção), tela de chat da IA no frontend, tratamento de erro 429 |
| 12 | Wizard de conexão WhatsApp via Embedded Signup da Meta (substituindo o formulário manual atual) — bloqueado por pré-requisitos fora do código (verificação de Business Portfolio, status de Tech Provider junto à Meta) |
| 13 | LGPD formal — bases legais de tratamento, política de retenção, mapeamento de dados pessoais, investigação de cookies de terceiro observados em ambiente de teste |
| — | Agendamento recorrente de fechamento (Celery Beat) |

## Débitos técnicos conhecidos

Itens de baixo impacto, sem risco de segurança ativo — corretude e
polimento, não brecha:

| Item | Categoria | Status |
|---|---|---|
| Payload JSON inválido no endpoint de webhook propaga exceção não tratada em vez de responder 400 | Robustez / tratamento de erro | ✅ Corrigido, validado por CI |
| Endpoints retornam 500 em vez de 400 para ID malformado (deletes e acknowledge) | Robustez / tratamento de erro | ✅ Corrigido, validado por CI |
| Falhas de envio não registram o motivo retornado pela Meta, apenas o status HTTP | Observabilidade | ⬜ Aberto |
| `/register` ainda não implementado (link morto na landing) | Funcionalidade pendente | ⬜ Aberto |
| Sem idempotência em `POST /upload/process` — múltiplos cliques podem gerar múltiplos disparos | Robustez | ⬜ Aberto |
| UUID/timestamp ausente no nome de arquivo enviado ao storage | Cosmético | ⬜ Aberto |
| Rota de documentação (Swagger) referencia um endpoint de token inexistente | Cosmético | ⬜ Aberto |
| Fonte carregada via `@import` em tempo de execução, deveria usar otimização nativa do framework | Performance / CSP | ⬜ Aberto |
| Trackers de terceiro observados em ambiente de teste, origem ainda não confirmada — investigação prevista antes da Sprint 13 | Privacidade / LGPD | ⬜ Aberto |
| Uso de dado pessoal (número de destinatário) ainda não avaliado sob a ótica de retenção/hash exigida por LGPD | Privacidade / LGPD | ⬜ Aberto |
| Cobertura de teste no gate de CI ainda baixa (40%, catraca subindo aos poucos) | Qualidade / processo | 🔄 Em andamento — ver [ADR 006](../decisions/006-ci-github-actions.md) |

## Fora do escopo desta lista

Risco de segurança residual aceito conscientemente (ex: ausência de
step-up authentication na remoção de número WhatsApp) está documentado
com justificativa em [`security/README.md`](../security/README.md#riscos-residuais-aceitos-e-documentados),
não repetido aqui.
