# Plano de Delivery v1 — Lembrete Inteligente de Retorno e Renovação via WhatsApp

**Produto X · Squad Retenção · Delivery Lead**
**Data:** 2026-06-12

---

## 1. Épicos e Tarefas por Papel

### Épico 0 — Pré-requisitos e Decisões (Sprint 0)
**Objetivo:** Eliminar todos os blockers antes do primeiro commit.

| # | Tarefa | Papel | Depende de |
|---|--------|-------|------------|
| E0-01 | Responder P1–P5 (PM + stakeholders) | PM / Negócio | — |
| E0-02 | Aprovação jurídica LGPD (opt-in, opt-out, retenção) | Jurídico / PM | E0-01 |
| E0-03 | Submeter templates à Meta | PM / Design | E0-01, copy aprovada |
| E0-04 | Contratar e configurar BSP + credenciais sandbox | Tech Lead / DevOps | E0-01 (P4, P5) |
| E0-05 | Definir número(s) WABA | PM / Tech Lead | E0-01 (P5) |
| E0-06 | Fechar decisões técnicas D1–D6 | Tech Lead / PM | E0-01 |
| E0-07 | Setup de ambiente de homologação com sandbox BSP | DevOps / Backend | E0-04 |
| E0-08 | Definir estratégia de coleta de baseline analytics (30 dias) | Data | E0-01 |
| E0-09 | Kick-off de alinhamento com toda a squad | Delivery Lead | E0-01 a E0-08 |

**Duração:** 1–2 semanas (paralelo ao planejamento técnico).

---

### Épico 1 — Fundação de Dados e Infraestrutura
**Objetivo:** Criar a base de dados, configurar filas e garantir a infraestrutura de agendamento.

| # | Tarefa | Papel | Depende de | Tam |
|---|--------|-------|------------|-----|
| B-01 | Migrations (4 tabelas + índices) | Backend | E0-06 | P |
| B-02 | Modelos e repositórios | Backend | B-01 | M |
| B-03 | Setup job queue + cron 5 min | Backend | E0-07 | M |
| B-04 | Setup de logging imutável e hash SHA-256 (LGPD) | Backend | B-01, E0-02 | M |
| DA-01 | Instrumentação dos 8 eventos base (schema + pipeline) | Data | B-01 | M |
| DA-02 | Coleta de baseline (30 dias pré-feature) | Data | E0-08 | — |

---

### Épico 2 — Integração com BSP e Envio de Mensagens
**Objetivo:** Enviar e receber mensagens via WhatsApp Business API.

| # | Tarefa | Papel | Depende de | Tam |
|---|--------|-------|------------|-----|
| B-06 | Integração com BSP (client HTTP, auth, envio de template) | Backend | E0-04, E0-07, B-02 | G |
| B-07 | Worker de envio (3 camadas de idempotência, retry) | Backend | B-03, B-06 | G |
| B-08 | Job cron de agendamento (UPDATE atômico, fuso, silêncio) | Backend | B-03, B-07 | M |
| B-09 | Trigger de criação de schedule (nova consulta/receita) | Backend | B-08 | M |
| B-10 | Webhook `/webhooks/whatsapp/status` (HMAC-SHA256) | Backend | B-06 | M |
| B-11 | Retry com backoff exponencial (5 tentativas) | Backend | B-07 | P |
| B-12 | Janela de silêncio por fuso horário da clínica | Backend | B-08 | P |

---

### Épico 3 — APIs de Configuração, Histórico e Opt-out
**Objetivo:** Expor os contratos de API para o frontend consumir.

| # | Tarefa | Papel | Depende de | Tam |
|---|--------|-------|------------|-----|
| B-13 | GET/PUT `/reminder-config` | Backend | B-02 | P |
| B-14 | GET `/reminders` com filtros | Backend | B-02 | M |
| B-15 | GET `/reminders/{id}` | Backend | B-02 | P |
| B-16 | POST `/patients/{id}/opt-out` + scope | Backend | B-02, E0-02, E0-06 | M |
| B-17 | POST `/internal/reminders/dispatch` | Backend | B-07 | P |
| B-18 | Testes de integração dos endpoints | Backend / QA | B-13 a B-17 | M |

---

### Épico 4 — Frontend: Configuração e Integração WABA
**Objetivo:** Permitir que a clínica configure lembretes e conecte WABA.

| # | Tarefa | Papel | Depende de | Tam |
|---|--------|-------|------------|-----|
| FE-0.1 | Types TypeScript + stubs de API | Frontend | B-13 (contrato) | P |
| FE-0.2 | Mock de API local (MSW) | Frontend | Contrato de API | M |
| FE-1.1 | Tela Configurações > Comunicação (wizard WABA) | Frontend | FE-0.1, Design spec | M |
| FE-1.2 | ReminderConfigBlock + LeadTimeSelector + toggles | Frontend | FE-1.1 | M |
| FE-1.3 | WhatsAppTemplatePreview + WhatsAppIntegrationBanner | Frontend | FE-1.2, templates (E0-03) | P |
| FE-1.4 | Integração real com GET/PUT reminder-config | Frontend | B-13, FE-1.2 | P |

---

### Épico 5 — Frontend: Histórico, Agenda e Perfil do Paciente
**Objetivo:** Dar visibilidade de status dos lembretes.

| # | Tarefa | Papel | Depende de | Tam |
|---|--------|-------|------------|-----|
| FE-2.1 | Tela histórico + ReminderHistoryFilters | Frontend | B-14, FE-0.1 | M |
| FE-2.2 | ReminderStatusBadge inline na agenda | Frontend | B-14, FE-2.1 | P |
| FE-2.3 | ReminderDetailDrawer (detalhes de falha) | Frontend | B-15, FE-2.2 | P |
| FE-2.4 | Banner agregado de falhas no painel | Frontend | B-14, FE-2.1 | P |
| FE-2.5 | Polling de status (30s, apenas registros pending) | Frontend | FE-2.1, Tech Lead (D6) | M |
| FE-3.1 | Aba "Comunicação" no perfil do paciente | Frontend | B-14, B-16 | M |
| FE-3.2 | PatientConsentToggle + OptOutConfirmModal | Frontend | B-16, FE-3.1 | P |

---

### Épico 6 — QA e Qualidade
**Objetivo:** Validar todos os cenários antes do lançamento.

| # | Tarefa | Papel | Depende de |
|---|--------|-------|------------|
| QA-01 | Testes E2E caminho feliz (5 cenários) | QA | Épicos 2, 3, 4, 5 completos |
| QA-02 | Testes edge cases (10 cenários) | QA | QA-01 |
| QA-03 | Testes LGPD/segurança (4 cenários) | QA | QA-01, E0-02 |
| QA-04 | Teste de carga no agendador | QA / Backend | B-08, B-11 |
| QA-05 | Teste de idempotência (duplicata + race condition) | QA / Backend | B-07, B-11 |
| QA-06 | Validação de isolamento multi-tenant | QA / Backend | B-04 |
| QA-07 | Regression geral + sign-off DoD | QA | QA-01 a QA-06 |

---

### Épico 7 — Analytics, Rollout e Monitoramento
**Objetivo:** Lançar com A/B controlado, monitoramento e dashboards operacionais.

| # | Tarefa | Papel | Depende de |
|---|--------|-------|------------|
| DA-03 | Dashboards operacionais (M2, M4, M5, G1, G2) | Data | DA-01, Épico 2 em prod |
| DA-04 | Setup A/B (20% da base, randomização por clínica) | Data / PM | DA-02, feature completa |
| DA-05 | Validar ausência de PII nos eventos | Data / QA | DA-01, E0-02 |
| OPS-01 | Alertas de falha BSP + dead-letter queue | DevOps / Backend | B-10, B-11 |
| OPS-02 | Runbook de rollback e playbook de incidente | DevOps / Tech Lead | Épicos 1–5 |
| OPS-03 | Rollout gradual (10% → 50% → 100%) | Delivery Lead / DevOps | QA-07, DA-04 |

---

## 2. Sequenciamento em Sprints

| Sprint | Objetivos | Papéis | Entregáveis | Dependências críticas | Riscos |
|--------|-----------|--------|-------------|----------------------|--------|
| **Sprint 0** (–2 a 0 sem) | Eliminar todos os blockers; sem código | PM, Jurídico, Tech Lead, Design, Delivery Lead | P1–P5 respondidas; templates submetidos; BSP contratado; jurídico aprovado | Budget (P4); disponibilidade do BSP | Templates rejeitados pela Meta; demora jurídica LGPD |
| **Sprint 1** (sem 1–2) | Fundação de dados; infra de fila; início BSP | Backend, DevOps, Data | Migrations; fila; logging LGPD; schema analytics; B-06 em andamento | E0-04 (BSP sandbox pronto); E0-02 aprovado | BSP sem sandbox estável; conflito de schema |
| **Sprint 2** (sem 3–4) | BSP completo; worker de envio; APIs; tela de config | Backend, Frontend, QA (início) | B-06 a B-12 completos; APIs B-13 a B-17; FE Épico 4 completo | Templates aprovados pela Meta; B-06 funcional | Template pendente na Meta bloqueia FE real |
| **Sprint 3** (sem 5–6) | Histórico, agenda, perfil; testes E2E e edge cases | Frontend, QA, Data | FE Épico 5 completo; QA-01 a QA-05; dashboards operacionais | B-14/B-15/B-16 completos; homologação estável | Race condition opt-out descoberta tardiamente |
| **Sprint 4** (sem 7–8) | QA final; LGPD; carga; alertas; go/no-go | QA, DevOps, Data, Delivery Lead | QA-07 sign-off; OPS-01/OPS-02; DA-04 A/B setup; runbook | QA-07 sem bloqueadores; DA-02 baseline completa | Bugs críticos em QA de carga; custo BSP acima do projetado |
| **Sprint 5** (sem 9–10) | Rollout 10% → 50% → 100%; monitoramento | Delivery Lead, DevOps, PM, Data, Backend | Feature em produção para 100% da base elegível; dashboards ativos | Go/no-go Sprint 4; G1/G2 configurados | Pico inesperado; opt-out em massa; falha BSP em prod |

---

## 3. Sprint 0 — Pré-requisitos Detalhados

### Decisões de Negócio (PM + Stakeholders)

| ID | Decisão | Owner | Bloqueador de |
|----|---------|-------|---------------|
| P1 | Gatilho de renovação: data da receita ou regra por especialidade? | PM + Médico-referência | B-09, template `renovacao_receita_v1` |
| P2 | Opt-out: apenas "SAIR" (Must) ou também via cadastro? Scope global vs. por clínica? | PM + Jurídico | B-16, FE-3.2 |
| P3 | Antecedência padrão: 3 dias, 1 dia ou ambos configuráveis? | PM | LeadTimeSelector, B-12 |
| **P4** | **Custo WhatsApp: incluso no SaaS ou repassado à clínica?** | **PM + Financeiro** | **E0-04 — bloqueia tudo** |
| **P5** | **Número WABA: compartilhado ou um por clínica?** | **PM + Tech Lead** | **E0-04, E0-05, arquitetura B-06** |

### Aprovações e Contratações

| ID | Atividade | Owner | Critério de conclusão |
|----|-----------|-------|----------------------|
| E0-02 | Parecer jurídico LGPD | Jurídico / DPO | Documento assinado |
| E0-03 | Submissão + aprovação dos 2 templates na Meta | PM / Design | Status "Aprovado" no Business Manager |
| E0-04 | Contratação BSP + credenciais sandbox e produção | Tech Lead / Financeiro | Credenciais funcionais em homologação |
| E0-05 | Provisionamento do(s) número(s) WABA | PM / Tech Lead / BSP | Número verificado e ativo |

### Decisões Técnicas

| ID | Decisão | Recomendação TL | Status |
|----|---------|-----------------|--------|
| D1 | BSP vs. Meta Cloud API direta | BSP para v1 | A confirmar |
| D2 | Cron+queue vs. event-driven | Cron+queue para v1 | A confirmar |
| D3 | Opt-out centralizado vs. por clínica | A validar (depende de P2) | Pendente P2 |
| D4 | SaaS incluso vs. WABA por clínica | A validar (depende de P4/P5) | Pendente P4/P5 |
| D5 | Idempotência via banco vs. Redis | A validar com infra existente | A confirmar |
| D6 | Polling (30s) vs. WebSocket/SSE | A alinhar Tech Lead + Frontend | A confirmar |

---

## 4. Riscos Consolidados e Mitigações

| # | Risco | Prob | Impacto | Owner | Mitigação |
|---|-------|------|---------|-------|-----------|
| R1 | Template rejeitado ou demorado pela Meta | Alta | Alto | PM + Design | Submeter na semana 1 do Sprint 0; preparar versões alternativas; não iniciar B-06 com template pendente |
| R2 | Custo por conversa não definido (P4) | Alta | Alto | PM + Financeiro | P4 como blocker absoluto do Sprint 0; sem definição = sem BSP = sem desenvolvimento |
| R3 | Qualidade do cadastro de telefone | Alta | Médio | PM + Backend | Validação E.164 antes do envio; fallback visível na UI; edge case QA-02 coberto |
| R4 | Ausência de opt-in válido (LGPD) | Média | Crítico | Jurídico / PM / Backend | Aprovação jurídica obrigatória no Sprint 0; consentimento explícito no onboarding; logs imutáveis desde o dia 1 |
| R5 | BSP fora do ar em produção | Média | Alto | Tech Lead / DevOps | Retry com backoff (B-11); dead-letter queue; alertas OPS-01; runbook OPS-02; SLA contratual |
| R6 | Agendamento confiável em escala | Média | Alto | Tech Lead / Backend | Teste de carga no Sprint 4 (QA-04); `FOR UPDATE SKIP LOCKED`; monitoramento de fila |
| R7 | Race condition no opt-out | Média | Médio | Backend / QA | 3 camadas de idempotência (B-07); teste específico QA-05 |
| R8 | Atraso na aprovação do BSP / número WABA | Média | Alto | Tech Lead / PM | Iniciar processo no Sprint 0 semana 1; plano B com BSP alternativo |

---

## 5. Definition of Done da v1

### Funcionalidade
- [ ] Clínica ativa/desativa lembretes por tipo via tela de configuração
- [ ] Clínica configura antecedência e horário de envio
- [ ] Wizard de integração WABA funcional
- [ ] Preview do template aprovado na tela de configuração
- [ ] Lembrete de retorno enviado automaticamente com `retorno_lembrete_v1`
- [ ] Lembrete de renovação enviado com `renovacao_receita_v1`
- [ ] Opt-out via "SAIR" processado e respeitado em todos os envios subsequentes
- [ ] Opt-out disponível na ficha do paciente (scope conforme P2)
- [ ] Status de disparo visível na agenda e no histórico
- [ ] Banner de falha agregado no painel da clínica
- [ ] Fallback de falha com orientação ao operador

### Qualidade e Testes
- [ ] 5 cenários de caminho feliz passando (QA-01)
- [ ] 10 edge cases críticos testados e passando (QA-02)
- [ ] 4 cenários LGPD/segurança validados (QA-03)
- [ ] Teste de carga do agendador sem degradação (QA-04)
- [ ] Idempotência validada (QA-05)
- [ ] Isolamento multi-tenant validado (QA-06)

### LGPD e Segurança
- [ ] Nenhum dado clínico sensível nas mensagens enviadas
- [ ] Consentimento explícito registrado antes do primeiro envio
- [ ] Logs de opt-out imutáveis e auditáveis
- [ ] Hash SHA-256 com salt em `patient_id` em logs e analytics
- [ ] Webhook `/webhooks/whatsapp/status` validado com HMAC-SHA256
- [ ] Aprovação formal do DPO / Jurídico documentada

### Infraestrutura e Operações
- [ ] Retry com backoff exponencial funcionando (5 tentativas)
- [ ] Dead-letter queue configurada com alerta
- [ ] Janela de silêncio por fuso horário da clínica respeitada
- [ ] Alertas de falha BSP e fila morta configurados (OPS-01)
- [ ] Runbook de rollback documentado (OPS-02)
- [ ] Feature flag ou rollout gradual configurado

### Analytics
- [ ] 8 eventos instrumentados e validados sem PII (DA-01, DA-05)
- [ ] Dashboards operacionais com M2, M4, M5, G1 e G2 ativos (DA-03)
- [ ] A/B test configurado para 20% da base (DA-04)
- [ ] Baseline de 30 dias coletada antes do lançamento (DA-02)

---

## 6. Matriz de Dependências Críticas

| Quem entrega | Para quem | O quê | Sprint |
|---|---|---|---|
| PM + Negócio | **Todos** | Respostas P1–P5 | Sprint 0 |
| Jurídico / DPO | **Backend + PM** | Parecer LGPD | Sprint 0 |
| PM + Design | **Backend + Frontend** | Templates aprovados pela Meta | Sprint 0–1 |
| Tech Lead / DevOps | **Backend** | BSP com credenciais sandbox (E0-04, E0-07) | Sprint 0 |
| Tech Lead | **Backend + Frontend** | Decisões D1–D6 fechadas | Sprint 0 |
| **Backend** (B-01, B-02) | Frontend | Schema e contratos de API (stubs TypeScript) | Sprint 1 |
| **Backend** (B-06) | Frontend + QA | Integração BSP funcional em sandbox | Sprint 2 |
| **Backend** (B-13 a B-16) | Frontend | Endpoints de config, histórico e opt-out em homologação | Sprint 2 |
| **Backend** (B-10) | QA + Data | Webhook de status funcional | Sprint 2 |
| **Design** | Frontend | Specs de wireframes e copy dos templates | Sprint 1 |
| **Frontend** (Épicos 4+5) | QA | Telas completas em homologação | Sprint 2–3 |
| **QA** (QA-07) | Delivery Lead / PM | Sign-off para go/no-go | Sprint 4 |
| **Data** (DA-02) | PM + Data | Baseline 30 dias coletada | Pré-Sprint 1 |
| **Data** (DA-01, DA-05) | QA | Validação ausência de PII nos eventos | Sprint 3 |
| **DevOps** (OPS-01, OPS-02) | Delivery Lead | Alertas e runbook | Sprint 4 |

---

## Notas do Delivery Lead

**Premissas assumidas neste plano:**
1. Squad: 1 Backend dev sênior, 1–2 Frontend devs, 1 QA, Tech Lead (parcial), Data (parcial), suporte de DevOps.
2. Estimativa Frontend: ~19 dias (1 dev) / ~12–13 dias (2 devs) — dos especialistas, não estimativa do Delivery Lead.
3. Backend não forneceu estimativa total — tamanhos P/M/G usados para priorização, não datas.
4. Prazo de ~6–7 meses para M1 (taxa de retorno) confiável é pós-lançamento e não altera o cronograma de entrega.

**Itens que exigem decisão ativa ANTES do Sprint 1:**
- P4 e P5 resolvidas — sem elas nenhum desenvolvimento pode começar de fato.
- Submissão dos templates à Meta na semana 1 do Sprint 0 — lead time de aprovação é imprevisível.
- D6 (polling vs. WebSocket) fechado com Tech Lead e Frontend antes do Sprint 2.
