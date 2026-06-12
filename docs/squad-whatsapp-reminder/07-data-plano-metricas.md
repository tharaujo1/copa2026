# Plano de Métricas — Lembrete Inteligente de Retorno e Renovação via WhatsApp

**Produto X | Feature: Smart Return & Renewal Reminder**
**Data:** 2026-06-12 | **Analytics Squad**

---

## 1. Eventos a Instrumentar

### Critérios gerais
- Nenhum evento carrega dado clínico individual (diagnóstico, medicamento, nome do paciente).
- Identificadores de paciente são pseudonimizados via hash SHA-256 com salt rotacionado (`patient_hash`).
- Identificadores de clínica são IDs internos opacos (`clinic_id`).

---

### `reminder_configured`
**Trigger:** Clínica salva ou edita as configurações do lembrete.
**Quem dispara:** Frontend.

| Propriedade | Tipo | Descrição |
|---|---|---|
| `clinic_id` | string | ID interno da clínica |
| `action` | enum | `activated` / `deactivated` / `edited` |
| `reminder_type` | enum | `return` / `renewal` / `both` |
| `days_before_due` | integer | Janela configurada |
| `template_id` | string | ID do template selecionado |
| `channel` | enum | `whatsapp` |
| `configured_by_role` | enum | `admin` / `receptionist` |
| `timestamp` | ISO 8601 | — |

---

### `reminder_scheduled`
**Trigger:** Job enfileira um lembrete para um paciente elegível.
**Quem dispara:** Backend job.

| Propriedade | Tipo | Descrição |
|---|---|---|
| `clinic_id` | string | — |
| `reminder_id` | string | UUID do lembrete |
| `reminder_type` | enum | `return` / `renewal` |
| `scheduled_send_at` | ISO 8601 | Horário programado |
| `days_since_last_visit` | integer | Sem identificação do paciente |
| `patient_hash` | string | SHA-256 pseudonimizado com salt |
| `timestamp` | ISO 8601 | — |

---

### `reminder_sent`
**Trigger:** Sistema dispara chamada à WhatsApp Business API.
**Quem dispara:** Backend job.

| Propriedade | Tipo |
|---|---|
| `clinic_id` | string |
| `reminder_id` | string |
| `reminder_type` | enum |
| `template_id` | string |
| `patient_hash` | string |
| `waba_message_id` | string |
| `attempt_number` | integer |
| `timestamp` | ISO 8601 |

---

### `reminder_delivered`
**Trigger:** Webhook WhatsApp confirma status `delivered`.
**Quem dispara:** Webhook handler.

| Propriedade | Tipo |
|---|---|
| `reminder_id` | string |
| `waba_message_id` | string |
| `patient_hash` | string |
| `delivery_latency_seconds` | integer |
| `timestamp` | ISO 8601 |

---

### `reminder_read`
**Trigger:** Webhook confirma status `read`.
**Quem dispara:** Webhook handler.

> **Nota:** Cobertura parcial — pacientes com "confirmações de leitura" desligadas não geram este status. Toda métrica baseada nele deve carregar essa ressalva.

| Propriedade | Tipo |
|---|---|
| `reminder_id` | string |
| `patient_hash` | string |
| `read_latency_seconds` | integer |
| `timestamp` | ISO 8601 |

---

### `reminder_failed`
**Trigger:** API retorna erro ou timeout após esgotamento de tentativas.
**Quem dispara:** Backend job / Webhook handler.

| Propriedade | Tipo |
|---|---|
| `clinic_id` | string |
| `reminder_id` | string |
| `patient_hash` | string |
| `failure_reason` | enum: `invalid_number` / `opt_out` / `template_rejected` / `rate_limit` / `timeout` / `unknown` |
| `waba_error_code` | string |
| `attempt_number` | integer |
| `is_final_attempt` | boolean |
| `timestamp` | ISO 8601 |

---

### `opt_out_received`
**Trigger:** Paciente responde com palavra-chave de opt-out.
**Quem dispara:** Webhook handler.

| Propriedade | Tipo |
|---|---|
| `clinic_id` | string |
| `patient_hash` | string |
| `opt_out_channel` | enum: `whatsapp_keyword` / `link` |
| `reminder_type_opted_out` | enum: `return` / `renewal` / `all` |
| `was_reminder_sent_within_24h` | boolean |
| `timestamp` | ISO 8601 |

---

### `patient_returned`
**Trigger:** Paciente agendou/compareceu à clínica com correlação a um lembrete enviado nos últimos 90 dias.
**Quem dispara:** Backend (trigger no agendamento/check-in).

> **Restrição crítica:** só existe se o módulo de agendamento puder correlacionar `patient_hash` com histórico de lembretes. Se agendamento for externo, este evento fica como **dependência a validar com Backend/PM**.

| Propriedade | Tipo |
|---|---|
| `clinic_id` | string |
| `patient_hash` | string |
| `attributed_reminder_id` | string (pode ser null) |
| `return_type` | enum: `scheduled` / `attended` |
| `days_since_reminder_sent` | integer |
| `attribution_confidence` | enum: `direct` / `inferred` |
| `reminder_type` | enum: `return` / `renewal` / `none` (grupo controle) |
| `timestamp` | ISO 8601 |

---

## 2. Definição das Métricas

| # | Métrica | Fórmula | Frequência | Janela | Fonte | Status |
|---|---|---|---|---|---|---|
| M1 | **Taxa de retorno 90D** (primária) | `retornaram em ≤90 dias` / `receberam lembrete` | Semanal (coorte) | 90 dias por coorte mensal | `patient_returned` + `reminder_sent` | **A validar** — baseline inexistente |
| M2 | **Taxa de entrega** | `count(delivered)` / `count(sent)` | Diária | Últimos 7/30 dias | `reminder_sent` + `reminder_delivered` | Disponível após instrumentação |
| M3 | **Taxa de leitura** | `count(read)` / `count(delivered)` | Diária | Últimos 7/30 dias | `reminder_delivered` + `reminder_read` | **Disponível parcialmente** — limitado pela privacidade do WA do usuário |
| M4 | **Taxa de opt-out** | `count(opt_out)` / `count(sent)` | Diária | Últimos 7/30 dias | `opt_out_received` + `reminder_sent` | Disponível após instrumentação |
| M5 | **Ativação de clínicas** | `clínicas com ≥1 sent` / `clínicas elegíveis` | Semanal | Acumulado desde lançamento | `reminder_configured` + cadastro | Disponível — requer flag de elegibilidade |
| M6 | **Impacto no churn** | `churn(clínicas ativas) − churn(clínicas sem feature)` | Mensal | Janela mínima 90 dias | Eventos de cancelamento + `reminder_configured` | **A validar** — depende de integração com billing |
| G1 | **Guard-rail: reclamação por spam** | `reclamações` / `sent` × 1.000 | Semanal | Últimos 30 dias | Tickets de suporte + relatório WABA | **Depende de** integração com sistema de suporte |
| G2 | **Guard-rail: taxa de bloqueio** | `failed com bloqueio` / `sent` | Diária | Últimos 7 dias | `reminder_failed` + relatório WABA | Disponível após instrumentação |

### Limiares de alerta (a confirmar com PM)

| Métrica | Alerta | Crítico (parar feature) |
|---|---|---|
| Taxa de entrega | < 85% | < 70% |
| Taxa de opt-out | > 5% | > 10% |
| Reclamação por spam | > 0,5‰ | > 2‰ |
| Taxa de bloqueio | > 1% | > 3% |

---

## 3. Esboço do Dashboard de Sucesso

### Seção 1 — Saúde dos Envios (Operacional / Quasi real-time)
- **Funil de envio:** `scheduled → sent → delivered → read` (volumes e percentuais), atualização horária.
- **Taxa de entrega ao longo do tempo:** linha diária, últimos 30 dias.
- **Distribuição de falhas por motivo:** pie/donut de `failure_reason`, últimos 7 dias.
- **Latência de entrega (p50/p95):** histograma de `delivery_latency_seconds`.
- **Tabela de erros recentes:** reminder_id, clinic_id, failure_reason, timestamp (sem dados de paciente).
- **Filtros:** período, `reminder_type`, `clinic_id`, `template_id`.

### Seção 2 — Impacto no Retorno (Resultado / Coorte)
- **Curva de retorno por coorte:** % de pacientes que retornaram em X dias vs. grupo controle.
- **Taxa de retorno em 30/60/90 dias:** tabela comparativa por coorte mensal com IC.
- **Distribuição de `days_since_reminder_sent`:** histograma de timing de retorno.
- **Breakdown por `attribution_confidence`** (direct vs. inferred).
- **Filtros:** coorte (mês de envio), `reminder_type`, segmento de clínica.

> Esta seção só se torna confiável após período mínimo de coleta (ver Seção 4).

### Seção 3 — Adoção da Clínica
- **Taxa de ativação acumulada (M5):** linha ao longo do tempo.
- **Funil de configuração:** `elegíveis → configuraram → enviaram ≥1 → enviaram >10`.
- **Distribuição de `days_before_due`** configurado (preferências das clínicas).
- **Clínicas ativas por semana** (novos vs. recorrentes).
- **Tabela exportável** de clínicas elegíveis sem ativação (para CS fazer outreach).

### Seção 4 — Guard-rails
- **Taxa de opt-out (M4):** linha diária com linhas de alerta (5%) e crítico (10%).
- **Taxa de bloqueio (G2):** linha diária com limiar em 1%.
- **Reclamações por spam (G1):** barras semanais por mil envios.
- **Top clínicas por taxa de opt-out:** tabela para identificar uso inadequado.
- **Painel de alertas ativos:** últimos 7 dias com severidade.

---

## 4. Estratégia de Baseline e Experimento

### 4.1 Como medir o baseline atual

**Retrospectivo (imediato):** analisar dados históricos do módulo de agendamento — para cada paciente com consulta registrada, calcular se houve nova consulta em ≤90 dias. Agregar por clínica. **Limitação:** viés de seleção (documentar).

**Prospectivo (30–60 dias pré-feature):** rodar o evento `patient_returned` sem lembrete associado (`attributed_reminder_id = null`, `reminder_type = none`) antes de ativar a feature para qualquer clínica.

### 4.2 Design experimental recomendado

**Híbrido: A/B com rollout gradual**

1. **Fase 0 (30 dias antes):** coletar baseline prospectivo.
2. **Fase 1 — A/B 20%:** aleatorizar clínicas (não pacientes) entre grupo tratamento e controle, 50/50 dentro do pool.
3. **Fase 2 — rollout gradual:** 50% → 100% se guard-rails dentro dos limiares após 4–6 semanas.

**Por que randomizar por clínica:** evita contaminação — o comportamento do paciente é influenciado por operações da clínica inteira.

### 4.3 Prazo mínimo para M1 significativo

> **Premissas a validar com PM antes de confirmar.**

| Variável | Valor assumido | Status |
|---|---|---|
| Taxa de retorno baseline | ~30–40% em 90 dias | A validar na Fase 0 |
| Efeito mínimo detectável (MDE) | +5 pp | A definir com PM |
| Poder estatístico | 80% | Padrão |
| Nível de significância | α = 0,05 | Padrão |

**Prazo total estimado:**
- Fase 0 (baseline): 30 dias
- Fase 1 (A/B): 90 dias de envio + 90 dias de janela de retorno
- **Total: ~6–7 meses do início até primeira leitura confiável de M1**

**O que pode ser avaliado antes:** M2, M3, M4 (disponíveis em dias); M5 (semanas); G1, G2 (monitoramento contínuo desde o dia 1).

---

## 5. Restrições LGPD para Analytics

### O que pode estar nos eventos

- IDs opacos internos (`clinic_id`, `reminder_id`, `waba_message_id`).
- `patient_hash` (pseudônimo — nunca o número real).
- Atributos comportamentais não identificáveis (`days_since_last_visit`, `return_type`).
- Timestamps sem associação direta a um indivíduo identificável.

### O que nunca pode estar nos eventos

- Nome, CPF, data de nascimento, telefone, endereço do paciente.
- Qualquer dado clínico (diagnóstico, prescrição, especialidade).
- Combinação que permita re-identificação (ex: `clinic_id` + `timestamp_exato` + `specialty` em clínicas pequenas — usar granularidade de dia, não hora).

### Mecanismos técnicos obrigatórios

1. `patient_hash` com salt por clínica, rotacionado a cada 6 meses.
2. Tabela de mapeamento `patient_hash → patient_id` exclusivamente no banco transacional — nunca no data warehouse.
3. Retenção: eventos de analytics por máximo 2 anos; depois apenas agregados.
4. Acesso: dashboards nunca exibem dados em nível de paciente individual.
5. `opt_out_received` dispara imediatamente supressão do número na fila de envios.

---

## 6. Dependências de Outros Papéis

### O que Analytics precisa

| Papel | O que Analytics precisa |
|---|---|
| Backend | Instrumentar os 8 eventos com as propriedades especificadas; `patient_hash` com salt por clínica; `reminder_id` como chave de correlação confiável |
| PM | Confirmar MDE para M1; definir critério de sucesso formal; confirmar clínicas elegíveis |
| Tech Lead / Data Infra | Destino dos eventos (BigQuery/Snowflake); pipeline de ingestão de webhooks; ferramenta de dashboard |
| Jurídico / DPO | Validar pseudonimização; aprovar política de retenção |

### O que Analytics entrega

| Destinatário | Entregável |
|---|---|
| PM | Relatório semanal de saúde dos envios; relatório de coorte de retorno a cada 30 dias; alerta imediato se guard-rail crítico for atingido |
| Delivery Lead | Status de instrumentação (checklist); alertas de anomalia; estimativa de data para M1 confiável |
| CS / Growth | Lista de clínicas elegíveis sem ativação (sem dados de pacientes); relatório de ativação (M5) |
| Engenharia | Spec técnica de eventos (este documento, Seção 1) como contrato de instrumentação |
