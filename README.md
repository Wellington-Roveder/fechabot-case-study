<p align="center">
  <img src="./banner.svg" alt="FechaBot — estudo de caso técnico" width="100%"/>
</p>

<p align="center">
  Estudo de caso sobre a arquitetura, a evolução e a validação de uma
  plataforma de fechamentos operacionais integrada ao WhatsApp.
</p>

<p align="center">
  <a href="https://wellington-roveder.github.io/portifolio-Wellington-Roveder/">
    <img src="https://img.shields.io/badge/PORTFÓLIO-00FF7F?style=for-the-badge&logoColor=black" alt="Portfólio"/>
  </a>
  <a href="https://www.linkedin.com/in/wellington-roveder-04637b37b/">
    <img src="https://img.shields.io/badge/LINKEDIN-0077B5?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn"/>
  </a>
  <a href="https://github.com/Wellington-Roveder">
    <img src="https://img.shields.io/badge/GITHUB-1a1a1a?style=for-the-badge&logo=github&logoColor=white" alt="GitHub"/>
  </a>
</p>

> **Transparência:** este repositório é um estudo de caso. O código-fonte
> do FechaBot é privado; estão disponíveis aqui a documentação técnica,
> as decisões de arquitetura, o histórico de evolução e as evidências
> visuais de uso. As imagens e os vídeos utilizam dados de teste.

## Visão geral

O FechaBot transforma planilhas operacionais em fechamentos distribuídos
por WhatsApp. O usuário envia a planilha, aplica filtros configuráveis,
seleciona um modelo de mensagem e acompanha o processamento e o histórico
de cada envio.

O sistema foi concebido para pequenos e médios negócios que precisam
distribuir informações operacionais com rapidez, mas não dispõem de uma
estrutura dedicada de dados ou automação.

## Principais capacidades

| Capacidade | Descrição |
|---|---|
| Processamento de planilhas | Leitura da fonte original e aplicação de filtros configuráveis |
| Modelos de mensagem | Variáveis de sistema, filtro, lista e preenchimento manual |
| Envio assíncrono | Processamento por fila, retentativas automáticas e registro de cada tentativa |
| Integração com WhatsApp | Entrega por meio da Meta Cloud API |
| Histórico operacional | Consulta de envios, destinatários e resultados do processamento |
| Consulta dos dados | Interação em linguagem natural com os dados do próprio tenant |
| Isolamento por tenant | Separação dos dados e das operações entre clientes |

<p align="center">
  <img src="./media/videos/landing.gif" alt="Página inicial do FechaBot" width="700"/>
</p>

## Tecnologias e integrações

<p align="left">
  <img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python"/>
  <img src="https://img.shields.io/badge/FastAPI-009688?style=for-the-badge&logo=fastapi&logoColor=white" alt="FastAPI"/>
  <img src="https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=nextdotjs&logoColor=white" alt="Next.js"/>
  <img src="https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white" alt="PostgreSQL"/>
  <img src="https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white" alt="Redis"/>
  <img src="https://img.shields.io/badge/Celery-37814A?style=for-the-badge&logo=celery&logoColor=white" alt="Celery"/>
  <img src="https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker"/>
  <img src="https://img.shields.io/badge/Cloudflare_R2-F38020?style=for-the-badge&logo=cloudflare&logoColor=white" alt="Cloudflare R2"/>
</p>

As integrações externas são a Meta Cloud API, para envio pelo WhatsApp,
e a Claude API, para a consulta conversacional dos dados. Os testes de
integração utilizam PostgreSQL real, sem mocks de banco.

## Demonstração

As demonstrações abaixo mostram fluxos do produto em funcionamento. As
mensagens e os destinatários exibidos são de teste.

### Acesso e autenticação em duas etapas

Sessão baseada em cookie `httpOnly`, proteção CSRF e autenticação TOTP
como segunda etapa. A decisão está documentada no
[ADR 001](./decisions/001-modelo-sessao-cookie-csrf-redis.md).

<p align="center">
  <img src="./media/videos/login_mfa.gif" alt="Login com autenticação em duas etapas" width="500"/>
</p>

### Dashboard

Visão do dashboard, consulta dos dados, segurança da conta e configurações.

<p align="center">
  <img src="./media/videos/dashboard_tour.gif" alt="Tour pelo dashboard" width="600"/>
</p>

### Configuração de filtros

Upload da planilha de referência, identificação de colunas e configuração
de condições compostas.

<p align="center">
  <img src="./media/videos/filter_config.gif" alt="Configuração de filtros" width="600"/>
</p>

### Modelos de mensagem

Editor com variáveis disponíveis, visualização em tempo real e valores de
exemplo para validação antes do envio.

<p align="center">
  <img src="./media/videos/template_config.gif" alt="Configuração de modelo de mensagem" width="600"/>
</p>

### Destinatários e número de WhatsApp

<p align="center">
  <img src="./media/videos/recipient_config.gif" alt="Configuração de destinatário" width="500"/>
  <br/><br/>
  <img src="./media/screenshots/whatsapp_number_config.png" alt="Número de WhatsApp configurado" width="600"/>
</p>

### Validação ponta a ponta

O fluxo foi validado da planilha até a entrega de uma mensagem em um
WhatsApp real, utilizando um modelo aprovado pela Meta.

<p align="center">
  <img src="./media/screenshots/whatsapp_delivery_confirmation.png" alt="Confirmação de entrega de mensagem no WhatsApp" width="500"/>
</p>

### Novo envio e histórico

<p align="center">
  <img src="./media/videos/send_confirmation.gif" alt="Confirmação de novo envio" width="600"/>
</p>

<p align="center">
  <img src="./media/screenshots/send_history.png" alt="Histórico de envios" width="600"/>
</p>

Os destinatários foram removidos após os testes. O histórico apresenta
essa alteração como “Destinatário removido”, comportamento previsto no
produto.

## Destaques técnicos

- **Sessões revogáveis:** cookie `httpOnly`, proteção CSRF, allowlist de
  sessão no Redis e expiração controlada, sem depender apenas do JWT no
  cliente.
- **Separação de categorias de template:** `requested_category` registra
  a escolha original e `effective_category` registra a categoria vigente
  após a análise da Meta.
- **Processamento resiliente:** Celery, retentativas automáticas e
  persistência do resultado de todas as tentativas, inclusive falhas.
- **Isolamento multi-tenant:** cada registro possui `tenant_id` e as
  rotas autenticadas obtêm o escopo a partir da sessão, não de parâmetros
  enviados pelo cliente.
- **Segurança documentada:** revisão em fases, achados técnicos,
  correções e riscos residuais registrados em [`security/`](./security).

## Como o repositório está organizado

```text
fechabot-case-study/
├── README.md            documentação principal e demonstrações
├── banner.svg           imagem de apresentação
├── architecture/       arquitetura, componentes e fluxo de dados
├── decisions/           decisões técnicas, alternativas e trade-offs
├── security/            revisões de segurança e achados documentados
├── roadmap/             entregas concluídas, plano e débitos técnicos
└── media/
    ├── screenshots/     capturas do produto
    └── videos/          GIFs de fluxos completos
```

## Documentação

- [Arquitetura e fluxo de dados](./architecture/README.md)
- [Decisões de arquitetura](./decisions/README.md)
- [Revisão de segurança](./security/README.md)
- [Roadmap e débitos técnicos](./roadmap/README.md)

## Escopo atual

O produto possui um fluxo funcional validado, mas não é apresentado como
um SaaS pronto para operação comercial. Permanecem no planejamento:
onboarding self-service, cobrança recorrente, monitoramento, deploy de
produção, agendamento recorrente, LGPD formal e a integração via Embedded
Signup da Meta.

O status detalhado e os débitos conhecidos estão no
[roadmap](./roadmap/README.md). O repositório não contém instruções de
execução porque a implementação principal é privada.

## Sobre o projeto

O FechaBot foi desenvolvido como um projeto prático de arquitetura,
integração e segurança de aplicações web. O objetivo deste repositório é
tornar as decisões e os resultados verificáveis, incluindo limitações,
pendências e evidências do fluxo principal.

<p align="left">
  <a href="https://wellington-roveder.github.io/portifolio-Wellington-Roveder/">
    <img src="https://img.shields.io/badge/PORTFÓLIO-00FF7F?style=for-the-badge&logoColor=black" alt="Portfólio"/>
  </a>
  <a href="https://www.linkedin.com/in/wellington-roveder-04637b37b/">
    <img src="https://img.shields.io/badge/LINKEDIN-0077B5?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn"/>
  </a>
  <a href="https://github.com/Wellington-Roveder">
    <img src="https://img.shields.io/badge/GITHUB-1a1a1a?style=for-the-badge&logo=github&logoColor=white" alt="GitHub"/>
  </a>
</p>
