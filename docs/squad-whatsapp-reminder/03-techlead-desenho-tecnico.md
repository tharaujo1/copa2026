# Desenho Técnico — Lembrete Inteligente de Retorno e Renovação via WhatsApp

**Produto X | Feature: Smart Reminder v1**
**Tech Lead | Data: 2026-06-12**

---

## 1. Abordagem de Integração com WhatsApp Business API

### 1.1 Arquitetura Recomendada: BSP Intermediário

```
Produto X Backend
┌──────────────┐    HTTPS/REST    ┌──────────────────────┐
│  Reminder    │ ───────────────► │  BSP (ex: Twilio,    │
│  Service     │ ◄─────────────── │  Zenvia, 360dialog)  │
└──────────────┘   webhook/status └──────────┬───────────┘
                                             │ Meta Cloud API
                                             ▼
                                   ┌─────────────────┐
                                   │  Meta WhatsApp  │
                                   │  Business API   │
                                   └────────┬────────┘
                                            │
                                            ▼
                                   ┌─────────────────┐
                                   │   Dispositivo   │
                                   │   do Paciente   │
                                   └─────────────────┘
```

**BSP vs. Meta Cloud API Direta:**

| Critério | BSP | Meta Cloud API Direta |
|---|---|---|
| Onboarding | 1–3 dias | 2–4 semanas |
| Suporte à aprovação de templates | BSP auxilia | Self-service |
| Infraestrutura de webhook | Gerenciada pelo BSP | Você provisiona |
| SLA | Contratual com BSP | Apenas SLA da Meta |
| Custo | Markup BSP (~10–30%) | Tarifa Meta direta |

**Recomendação v1:** BSP. Revisitar Meta Cloud API direta se volume > 50k conversas/mês.

---

### 1.2 Aprovação de Templates

**Processo:**
```
PM/Design redige template
        │
        ▼
Tech Lead revisa (sem conteúdo clínico, sem dados sensíveis)
        │
        ▼
Submit via BSP dashboard (categoria: UTILITY)
        │
        ▼
Meta analisa (24–72h úteis)
        │
   APPROVED         REJECTED
        │                 │
   Ativo p/ uso     PM revisa copy → novo submit
```

**Templates necessários para v1:**
```
retorno_lembrete_v1
"Olá {{1}}, a clínica {{2}} lembra que você tem retorno em {{3}}.
Horário: {{4}}. Dúvidas? {{5}}. Responda SAIR para cancelar."

renovacao_receita_v1
"Olá {{1}}, a clínica {{2}} informa que você tem um documento
com vencimento próximo em {{3}}. Agende: {{4}}. Responda SAIR."
```

---

### 1.3 Janela de 24h

- **Fora da janela** (nenhuma mensagem prévia do paciente): **obrigatório usar template HSM**.
- **Dentro da janela** (paciente respondeu nos últimos 24h): pode enviar texto livre.
- Para lembretes, o cenário é **sempre fora da janela** — toda mensagem proativa é template.
- A sessão aberta (após resposta do paciente) é usada apenas para confirmar o opt-out.

---

### 1.4 Custo por Conversa

- Categoria `UTILITY`: ~USD 0,02–0,04 por conversa (varia por país).
- Modelo de custo para o produto (decisão de negócio — ver P4):
  - **Opção A — SaaS incluso:** Produto X arca, repassa no plano com cap.
  - **Opção B — Pay-as-you-go:** clínica conecta WABA própria.
  - **Opção C — Créditos de mensagem.**
- **Recomendação:** Opção A para v1 com cap configurável por plano.

---

## 2. Modelo de Dados

```
┌──────────────┐         ┌─────────────────────┐
│    Clinic    │ 1     * │   ReminderConfig     │
│ id (PK)      │─────────│ id (PK)              │
│ name         │         │ clinic_id (FK)       │
│ waba_number  │         │ reminder_type        │
│ timezone     │         │ days_before          │
└──────────────┘         │ send_window_start    │
                         │ send_window_end      │
                         │ is_active            │
                         │ template_name        │
                         └────────┬────────────┘
                                  │
┌──────────────┐         ┌────────▼────────────┐
│   Patient    │         │  ReminderSchedule   │
│ id (PK)      │◄────────│ id (PK)              │
│ clinic_id    │         │ config_id (FK)       │
│ name         │         │ patient_id (FK)      │
│ phone_e164   │         │ appointment_id (FK)  │
└──────────────┘         │ scheduled_for        │
                         │ status               │
┌──────────────┐         │ idempotency_key      │
│ Appointment/ │◄────────│ attempt_count        │
│ Prescription │         └──────────┬──────────┘
│ id (PK)      │                    │
│ patient_id   │         ┌──────────▼──────────┐
│ clinic_id    │         │    ReminderLog       │
│ date         │         │ id (PK)              │
└──────────────┘         │ schedule_id (FK)     │
                         │ sent_at              │
                         │ provider_message_id  │
                         │ status               │
                         │ error_code           │
                         │ template_vars_hash   │
                         └─────────────────────┘

┌──────────────────────────────────┐
│            OptOut                │
│ id (PK)                          │
│ patient_id (FK)                  │
│ clinic_id (FK — nullable)        │
│ scope (GLOBAL | PER_CLINIC)      │
│ opted_out_at                     │
│ channel (WHATSAPP_REPLY | UI)    │
│ raw_message                      │
└──────────────────────────────────┘
```

**Campos críticos:**
```sql
-- ReminderConfig
reminder_type   ENUM('RETURN', 'PRESCRIPTION_RENEWAL')
days_before     SMALLINT NOT NULL
send_window_start  TIME NOT NULL DEFAULT '08:00'
send_window_end    TIME NOT NULL DEFAULT '20:00'
is_active       BOOLEAN NOT NULL DEFAULT TRUE

-- ReminderSchedule
status          ENUM('PENDING','PROCESSING','SENT','FAILED','CANCELLED','OPT_OUT_SKIPPED')
idempotency_key VARCHAR(64) UNIQUE  -- hash(clinic_id+patient_id+appointment_id+scheduled_date)

-- ReminderLog
template_vars_hash  VARCHAR(64)  -- hash dos vars (não os valores — LGPD)
error_code      VARCHAR(64)      -- código de erro do BSP/Meta
```

---

## 3. Agendador e Fila

### Fluxo de Agendamento

```
Evento gerador (novo agendamento ou receita criada)
        │
        ▼
Schedule Writer ── upsert em ReminderSchedule com idempotency_key
        │
        ▼
CRON — a cada 5 minutos:
  SELECT * FROM ReminderSchedule
  WHERE status = 'PENDING'
  AND scheduled_for <= NOW()
  FOR UPDATE SKIP LOCKED   ← chave para idempotência
        │
        ▼
Job Queue (BullMQ / Sidekiq / AWS SQS)
        │
        ▼
WhatsApp Send Worker:
  1. Checa opt-out
  2. Valida janela de envio (fuso da clínica)
  3. Chama BSP API
  4. Grava ReminderLog
  5. Atualiza status
```

### Idempotência (3 camadas)

**Camada 1 — banco:**
```sql
idempotency_key = SHA256(clinic_id || patient_id || ref_id || scheduled_date)
INSERT INTO reminder_schedule (...) ON CONFLICT (idempotency_key) DO NOTHING;
```

**Camada 2 — FOR UPDATE SKIP LOCKED:**
```sql
SELECT * FROM reminder_schedule
WHERE status = 'PENDING' AND scheduled_for <= NOW()
FOR UPDATE SKIP LOCKED LIMIT 50;
-- Atualiza para 'PROCESSING' antes de enviar
```

**Camada 3 — verificação no worker:**
```python
if log_exists(schedule_id) and log.status in ('DELIVERED', 'QUEUED'):
    skip()  # já foi enviado
```

### Retry Policy

| Tentativa | Delay |
|---|---|
| 1 (inicial) | — |
| 2 | 2 min |
| 3 | 10 min |
| 4 | 30 min |
| 5 | 2h |
| Após 5 falhas | status → FAILED, alerta para ops |

- Falhas retryáveis: timeout, rate limit, 5xx do BSP.
- Falhas não-retryáveis: número inválido, template rejeitado, opt-out → CANCELLED.

---

## 4. Pontos de Falha e Fallbacks

| Falha | Detecção | Ação imediata | Ação de longo prazo |
|---|---|---|---|
| Número inválido | Código BSP (ex: 131026) | Status FAILED, sem retry | Marcar `phone_invalid=true` no cadastro; alertar recepção |
| Paciente sem WhatsApp | Código BSP (ex: 131047) | Status FAILED, sem retry | Log + dashboard; fallback SMS em v2 |
| Template rejeitado | Código 132000 | Parar todos os envios do template | Alerta para PM/Design; notificação na UI da clínica |
| BSP/API fora do ar | HTTP 5xx ou timeout | Retry com backoff (5 tentativas) | Circuit breaker; dead letter queue; alerta para ops |
| Duplo disparo | idempotency_key conflict | ON CONFLICT DO NOTHING | Log de warning; alerta se attempt_count > 1 em curto intervalo |

---

## 5. LGPD e Segurança

### Registro de Consentimento

```sql
CREATE TABLE patient_consent (
  id              UUID PRIMARY KEY,
  patient_id      UUID NOT NULL REFERENCES patient(id),
  clinic_id       UUID NOT NULL REFERENCES clinic(id),
  channel         VARCHAR(32) NOT NULL,  -- 'WHATSAPP'
  consent_given   BOOLEAN NOT NULL,
  term_version    VARCHAR(16) NOT NULL,
  collected_at    TIMESTAMPTZ NOT NULL,
  collected_by    VARCHAR(64),
  ip_address      INET
);
```

### Processamento de Opt-out

```
Paciente responde "SAIR" → BSP entrega webhook
        │
        ▼
Inbound Handler:
  1. Identifica número E.164
  2. Normaliza texto (upper, trim)
  3. Se IN ('SAIR','STOP','CANCELAR','PARAR'):
     → INSERT INTO opt_out
     → Cancela lembretes PENDING do paciente
     → Envia confirmação (dentro da janela 24h)
  4. Caso contrário: descarta (v1 sem chatbot)
```

**Verificação no worker (antes de enviar):**
```sql
SELECT EXISTS (
  SELECT 1 FROM opt_out
  WHERE patient_id = $1
    AND (clinic_id = $2 OR scope = 'GLOBAL')
    AND opted_out_at <= NOW()
)
```

### Dados proibidos nas mensagens

| Dado | Alternativa |
|---|---|
| Diagnóstico / CID | "retorno" ou "consulta" genérico |
| Nome do medicamento | "documento com vencimento próximo" |
| CPF, plano de saúde | Não incluir |
| Link com token de paciente | Tokens opacos de curta validade |

### Retenção

- ReminderLog: 2 anos (auditoria)
- OptOut: enquanto ativo + 5 anos (prova de respeito ao opt-out)
- PatientConsent: prazo do contrato + 5 anos
- Direito ao esquecimento: anonimizar dados pessoais nos logs (phone, name → NULL / hash), não deletar fisicamente

---

## 6. Principais Decisões e Trade-offs

| # | Decisão | Opção A (recomendada) | Opção B | Risco principal |
|---|---|---|---|---|
| D1 | Integração WhatsApp | **BSP intermediário** | Meta Cloud API direta | Dependência de terceiro; markup de custo |
| D2 | Disparo de mensagens | **Cron + job queue** | Event-driven puro | Latência de até 5 min; precisa de infra de fila |
| D3 | Armazenamento opt-out | **Centralizado com scope** | Por clínica isolado | Complexidade de query; GLOBAL pode surpreender clínicas |
| D4 | Modelo de custo | **SaaS incluso com cap** | WABA por clínica | Produto X exposto a custo variável; câmbio |
| D5 | Gatilho renovação | **Data da receita (v1)** | Regra por especialidade | Regra por especialidade é mais precisa mas requer mapeamento |
| D6 | Idempotência | **Banco de dados (Postgres)** | Redis (distributed lock) | Redis adiciona dependência de infra |

---

## 7. Decisões a Fechar Antes do Dev

```
1. CUSTO: Produto X arca ou clínica conecta WABA própria? (P4)
   → Impacta arquitetura multi-tenant de autenticação com Meta.

2. GATILHO DE RENOVAÇÃO: data da receita ou regra por especialidade? (P1)
   → Impacta modelo de dados e Schedule Writer.

3. OPT-OUT SCOPE: somente "SAIR" ou também via UI da clínica? (P2)
   → Impacta fluxo de frontend.

4. ANTECEDÊNCIA: valores padrão? (sugestão: retorno = 1 dia, receita = 7 dias) (P3)
   → PM confirma com research com clínicas.

5. TEMPLATE COPY: quem redige e valida antes da submissão à Meta?
   → PM + Jurídico. Sem template aprovado não há v1.
```
