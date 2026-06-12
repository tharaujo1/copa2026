# Artefato Frontend — Lembrete Inteligente de Retorno e Renovação via WhatsApp

---

## 1. Telas Necessárias

### 1.1 Tela de Configuração de Lembretes (`/settings/reminders`)
Configuração das regras de envio por clínica. Dois blocos: lembrete de retorno e de renovação de receita. Cada bloco tem toggle, seletor de antecedência e preview do template. Banner de aviso quando integração WhatsApp Business não está configurada ou template está pendente de aprovação.

### 1.2 Tela de Histórico de Lembretes (`/reminders/history`)
Lista paginada de todos os lembretes enviados/com falha. Filtrável por tipo, status e período. Cada linha: nome do paciente, tipo, data/hora, canal, status com badge colorido.

### 1.3 Status de Lembretes no Perfil do Paciente (seção em `/patients/:id`)
Seção na ficha do paciente com histórico de lembretes e status de opt-in/opt-out. Botão para alterar consentimento (LGPD).

### 1.4 Modal de Confirmação de Opt-out
Modal com texto LGPD explícito ao registrar opt-out. CTAs: confirmar / cancelar.

### 1.5 Drawer de Detalhes do Lembrete
Drawer lateral com detalhes completos: preview do template enviado, timestamp, status de entrega, número mascarado, erros da WhatsApp Business API.

---

## 2. Componentes Novos a Criar

| Componente | Responsabilidade | Props essenciais |
|---|---|---|
| `ReminderConfigBlock` | Bloco de configuração de um tipo de lembrete | `type`, `config`, `onChange`, `isLoading`, `templateStatus` |
| `LeadTimeSelector` | Seletor de antecedência (dropdown/radio: 1d/2d/3d/1 semana) | `value`, `onChange`, `disabled` |
| `WhatsAppTemplatePreview` | Card read-only com preview do template aprovado | `templateText`, `variables`, `status` |
| `ReminderStatusBadge` | Badge com cor e label para status de envio | `status: 'sent'|'failed'|'opt_out'|'pending'` |
| `WhatsAppIntegrationBanner` | Banner quando integração não está configurada | `variant: 'not_configured'|'template_pending'` |
| `PatientConsentToggle` | Toggle opt-in/opt-out com modal de confirmação | `patientId`, `hasConsent`, `onChange`, `isLoading` |
| `OptOutConfirmModal` | Modal de confirmação de opt-out com texto LGPD | `isOpen`, `onConfirm`, `onCancel`, `patientName` |
| `ReminderDetailDrawer` | Drawer lateral com detalhes de um lembrete | `reminderId`, `onClose` |
| `ReminderHistoryFilters` | Barra de filtros do histórico | `filters`, `onChange` |

---

## 3. Componentes Existentes a Reutilizar/Adaptar

> **Premissa: verificar se existem no codebase antes de criar.**

| Componente | Uso nesta feature |
|---|---|
| Tabela com paginação | Listagem do histórico e listagem no perfil do paciente |
| Toggle/Switch | Ativar/desativar tipos de lembrete; opt-in/opt-out |
| Badge de status | `ReminderStatusBadge` pode herdar de `StatusBadge` genérico |
| Formulário com validação | Tela de configuração (react-hook-form ou equivalente) |
| Toast / Snackbar | Feedback de save, erro de API, reenvio |
| Drawer genérico | `ReminderDetailDrawer` pode usar `Drawer` base existente |
| Modal genérico | `OptOutConfirmModal` pode usar `Modal`/`Dialog` existente |
| DateRangePicker | Filtro de período no histórico |

---

## 4. Estados de UI por Tela

### Tela de Configuração

| Estado | Comportamento |
|---|---|
| Loading inicial | Skeleton nos dois blocos |
| Editando | Formulário editável, botão "Salvar" habilitado quando há mudança |
| Salvando | Botão com spinner, campos desabilitados |
| Sucesso no save | Toast de sucesso |
| Erro no save | Toast de erro com opção retry |
| Integração WhatsApp não configurada | `WhatsAppIntegrationBanner` variant `not_configured`; blocos desabilitados |
| Template pendente de aprovação | `WhatsAppIntegrationBanner` variant `template_pending`; toggle desabilitado |

### Tela de Histórico

| Estado | Comportamento |
|---|---|
| Loading | Skeleton da tabela |
| Empty state (sem registros) | Ilustração + "Nenhum lembrete enviado ainda" |
| Empty state (filtros aplicados) | "Nenhum resultado" + botão "Limpar filtros" |
| Erro de API | Inline error com botão "Tentar novamente" |

### Perfil do Paciente — Seção de Lembretes

| Estado | Comportamento |
|---|---|
| Loading | Skeleton da seção |
| Paciente sem telefone | Aviso "Telefone não cadastrado — lembretes não serão enviados" |
| Opt-out ativo | Badge "Opt-out ativo" + botão para reverter |

---

## 5. Integrações com API Backend

| Tela / Ação | Endpoint | Método |
|---|---|---|
| Carregar config | `GET /api/v1/clinics/:clinicId/reminder-settings` | GET |
| Salvar config | `PUT /api/v1/clinics/:clinicId/reminder-settings` | PUT |
| Status integração WA | `GET /api/v1/clinics/:clinicId/whatsapp-integration` | GET |
| Listar histórico | `GET /api/v1/reminders?clinicId=&type=&status=&from=&to=&page=` | GET |
| Detalhes lembrete | `GET /api/v1/reminders/:reminderId` | GET |
| Histórico do paciente | `GET /api/v1/patients/:patientId/reminders` | GET |
| Alterar consent | `PATCH /api/v1/patients/:patientId/whatsapp-consent` | PATCH |

### Tratamento de Erros

- Erros de validação (4xx): inline abaixo do campo.
- Erros de save/ação: toast com mensagem humanizada.
- Erros de listagem: error state inline com botão retry.
- Timeout/rede: toast persistente "Falha de conexão".

### Atualização de Status — DECISÃO A ALINHAR COM TECH LEAD

- **Polling (30s recomendado):** mais simples; aceitável se atualização não precisa ser em tempo real. Ativar apenas com lembretes com status `pending` na tela.
- **WebSocket/SSE:** menor latência; mais complexo. Avaliar com Tech Lead se volume justifica.

---

## 6. Lista de Tarefas Frontend

### Fase 0 — Fundação (sem bloqueio de backend)

| # | Tarefa | Tam | Dependência |
|---|--------|-----|------------|
| 0.1 | Types TypeScript para `ReminderConfig`, `ReminderRecord`, `ReminderStatus`, `PatientConsent` | P | Contrato de API do Backend |
| 0.2 | Mock de API local (MSW ou similar) com fixtures para todos os endpoints | M | Contrato de API definido |
| 0.3 | Componente `ReminderStatusBadge` com todos os estados | P | Designer (spec de cores) |
| 0.4 | Componente `WhatsAppIntegrationBanner` (ambas variantes) | P | Designer (spec) |
| 0.5 | Componente `WhatsAppTemplatePreview` (read-only) | P | Designer + Backend (estrutura do template) |

### Fase 1 — Tela de Configuração

| # | Tarefa | Tam | Dependência |
|---|--------|-----|------------|
| 1.1 | Componente `LeadTimeSelector` | P | Designer (opções disponíveis) |
| 1.2 | Componente `ReminderConfigBlock` | M | 0.3, 0.4, 0.5, 1.1, Designer spec |
| 1.3 | Montar tela `/settings/reminders` com lógica de formulário e validação | M | 1.2 |
| 1.4 | Integrar com GET/PUT reminder-settings + estados | M | 1.3, Backend endpoint |
| 1.5 | Integrar banner de integração (GET /whatsapp-integration) | M | 1.4, Backend endpoint |

### Fase 2 — Histórico de Lembretes

| # | Tarefa | Tam | Dependência |
|---|--------|-----|------------|
| 2.1 | Componente `ReminderHistoryFilters` | M | Designer spec |
| 2.2 | Tela `/reminders/history` com tabela paginada, filtros e estados | G | 2.1, 0.3, Backend endpoint |
| 2.3 | Componente `ReminderDetailDrawer` | M | 0.5, Backend `GET /reminders/:id` |
| 2.4 | Lógica de polling de status (30s, apenas com registros `pending`) | M | 2.2, Tech Lead (decisão polling vs. WS) |

### Fase 3 — Perfil do Paciente

| # | Tarefa | Tam | Dependência |
|---|--------|-----|------------|
| 3.1 | Componente `OptOutConfirmModal` | P | Designer spec + texto LGPD aprovado |
| 3.2 | Componente `PatientConsentToggle` | M | 3.1, Backend endpoint |
| 3.3 | Seção de lembretes no perfil do paciente | M | 3.2, 0.3, Backend endpoints |

### Fase 4 — Qualidade

| # | Tarefa | Tam | Dependência |
|---|--------|-----|------------|
| 4.1 | Testes unitários dos componentes novos | M | Fases 1–3 concluídas |
| 4.2 | Testes E2E: salvar config, histórico, opt-out | G | 4.1, QA (cenários) |
| 4.3 | Acessibilidade (aria-labels, foco em modais, contraste) | M | Fases 1–3 concluídas |
| 4.4 | Handoff para QA: estados, edge cases, mocks disponíveis | P | 4.1 |

**Estimativa total:** ~19 dias (1 dev) · ~12–13 dias (2 devs com Fases 1 e 2 em paralelo após Fase 0)

---

## 7. Dependências de Outros Papéis

### O que Frontend precisa antes de começar

**De Designer:** specs das 3 telas + modal + drawer; cores do `ReminderStatusBadge`; texto de opt-out (revisado por jurídico); comportamento dos blocos sem integração; localização da seção no perfil do paciente.

**De Backend:** contrato OpenAPI completo; estrutura do objeto de template; campos de status de entrega e códigos de erro da WhatsApp API; ambiente de staging com sandbox.

**De Tech Lead:** decisão polling vs. WebSocket; estratégia de autenticação para novos endpoints; reenvio manual em v1 ou v2; mascaramento do número de telefone.

### O que Frontend entrega para QA

- Build de staging apontando para backend de staging.
- Documento de estados de UI implementados.
- Mocks MSW disponíveis para testes automatizados.
