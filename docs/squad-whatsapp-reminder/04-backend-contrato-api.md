# Contrato de API + Lista de Tarefas Backend
## Feature: Lembrete Inteligente de Retorno e Renovação via WhatsApp

---

## 1. Contrato de API REST

### GET /clinics/{clinicId}/reminder-config

**Auth:** Clínica autenticada (JWT com claim `clinicId`)

**Response 200:**
```json
{
  "clinicId": "uuid",
  "returnReminder": {
    "enabled": true,
    "daysBeforeReturn": 2,
    "templateId": "return_reminder_v1",
    "sendHour": 9
  },
  "prescriptionReminder": {
    "enabled": true,
    "daysBeforeExpiry": 5,
    "templateId": "prescription_renewal_v1",
    "sendHour": 9
  },
  "updatedAt": "2026-06-11T10:00:00Z"
}
```

**Errors:** `404` clínica não encontrada · `403` token não pertence à clínica

---

### PUT /clinics/{clinicId}/reminder-config

**Auth:** Clínica autenticada

**Request body:**
```json
{
  "returnReminder": {
    "enabled": true,
    "daysBeforeReturn": 2,
    "templateId": "return_reminder_v1",
    "sendHour": 9
  },
  "prescriptionReminder": {
    "enabled": false,
    "daysBeforeExpiry": 5,
    "templateId": "prescription_renewal_v1",
    "sendHour": 9
  }
}
```

**Response 200:** mesmo schema do GET · **Errors:** `400` validação · `403` · `404`

---

### GET /clinics/{clinicId}/reminders

**Auth:** Clínica autenticada

**Query params:**
```
type      = RETURN | PRESCRIPTION
status    = SCHEDULED | SENT | FAILED | CANCELLED
patientId = uuid
from      = ISO date
to        = ISO date
page      = int, default 1
pageSize  = int, default 20, max 100
```

**Response 200:**
```json
{
  "data": [
    {
      "reminderId": "uuid",
      "patientId": "uuid",
      "patientName": "João Silva",
      "type": "RETURN",
      "status": "SENT",
      "scheduledAt": "2026-06-11T09:00:00Z",
      "sentAt": "2026-06-11T09:00:04Z",
      "referenceDate": "2026-06-13T00:00:00Z",
      "whatsappMessageId": "wamid.xxx"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 84 }
}
```

---

### GET /reminders/{reminderId}

**Auth:** Clínica autenticada (valida que o lembrete pertence à clínica do token)

**Response 200:**
```json
{
  "reminderId": "uuid",
  "clinicId": "uuid",
  "patientId": "uuid",
  "type": "RETURN",
  "status": "FAILED",
  "scheduledAt": "2026-06-11T09:00:00Z",
  "sentAt": null,
  "referenceDate": "2026-06-13T00:00:00Z",
  "whatsappMessageId": null,
  "attempts": 2,
  "lastError": "WHATSAPP_UNREACHABLE",
  "deliveryStatus": null,
  "logs": [
    {
      "attemptNumber": 1,
      "attemptedAt": "2026-06-11T09:00:00Z",
      "outcome": "FAILED",
      "errorCode": "WHATSAPP_UNREACHABLE"
    }
  ]
}
```

---

### POST /patients/{patientId}/opt-out

**Auth:** Clínica autenticada

**Request body:**
```json
{
  "channel": "WHATSAPP",
  "reason": "PATIENT_REQUEST",
  "source": "CLINIC_PANEL"
}
```

**Response 200:**
```json
{
  "patientId": "uuid",
  "channel": "WHATSAPP",
  "optedOutAt": "2026-06-11T10:00:00Z",
  "cancelledReminders": 3
}
```

`cancelledReminders`: quantidade de SCHEDULED cancelados imediatamente.

**Errors:** `400` · `403` · `404` · `409` já em opt-out

---

### POST /webhooks/whatsapp/status

**Auth:** Verificação de assinatura HMAC (`X-Hub-Signature-256`). Sem JWT. IP allowlist recomendado.

**Request body (payload Meta/WhatsApp):**
```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "changes": [{
      "value": {
        "statuses": [{
          "id": "wamid.xxx",
          "status": "delivered",
          "timestamp": "1718096404",
          "recipient_id": "5511999999999"
        }],
        "messages": [{
          "from": "5511999999999",
          "type": "text",
          "text": { "body": "SAIR" }
        }]
      }
    }]
  }]
}
```

**Response:** `200 { "received": true }` — sempre responder 200 rápido; processar async.

---

### POST /internal/reminders/dispatch

**Auth:** Service-to-service token

**Request body:**
```json
{
  "reminderIds": ["uuid1", "uuid2"],
  "idempotencyKey": "job-run-2026-06-11T09:00:00Z"
}
```

**Response 200:**
```json
{
  "dispatched": 2,
  "skipped": 0,
  "failed": 0,
  "results": [
    { "reminderId": "uuid1", "outcome": "DISPATCHED" }
  ]
}
```

---

## 2. Schema do Banco de Dados

```sql
CREATE TABLE reminder_configs (
  clinic_id                    UUID        PRIMARY KEY REFERENCES clinics(id),
  return_enabled               BOOLEAN     NOT NULL DEFAULT false,
  return_days_before           SMALLINT    CHECK (return_days_before BETWEEN 1 AND 7),
  return_template_id           VARCHAR(64),
  return_send_hour             SMALLINT    NOT NULL DEFAULT 9 CHECK (return_send_hour BETWEEN 0 AND 23),
  prescription_enabled         BOOLEAN     NOT NULL DEFAULT false,
  prescription_days_before     SMALLINT    CHECK (prescription_days_before BETWEEN 1 AND 30),
  prescription_template_id     VARCHAR(64),
  prescription_send_hour       SMALLINT    NOT NULL DEFAULT 9 CHECK (prescription_send_hour BETWEEN 0 AND 23),
  created_at                   TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at                   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE reminder_schedules (
  id                  UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  clinic_id           UUID        NOT NULL REFERENCES clinics(id),
  patient_id          UUID        NOT NULL REFERENCES patients(id),
  type                VARCHAR(20) NOT NULL CHECK (type IN ('RETURN','PRESCRIPTION')),
  status              VARCHAR(20) NOT NULL DEFAULT 'SCHEDULED'
                                  CHECK (status IN ('SCHEDULED','DISPATCHED','SENT','DELIVERED','READ','FAILED','CANCELLED')),
  reference_date      DATE        NOT NULL,
  scheduled_at        TIMESTAMPTZ NOT NULL,
  dispatched_at       TIMESTAMPTZ,
  sent_at             TIMESTAMPTZ,
  whatsapp_message_id VARCHAR(128) UNIQUE,
  delivery_status     VARCHAR(20),
  attempts            SMALLINT    NOT NULL DEFAULT 0,
  last_error          VARCHAR(128),
  idempotency_key     VARCHAR(128) UNIQUE,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rs_clinic_status ON reminder_schedules(clinic_id, status);
CREATE INDEX idx_rs_scheduled_at  ON reminder_schedules(scheduled_at) WHERE status = 'SCHEDULED';
CREATE INDEX idx_rs_patient        ON reminder_schedules(patient_id);
CREATE INDEX idx_rs_wamid          ON reminder_schedules(whatsapp_message_id) WHERE whatsapp_message_id IS NOT NULL;

CREATE TABLE reminder_logs (
  id               UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  reminder_id      UUID        NOT NULL REFERENCES reminder_schedules(id),
  attempt_number   SMALLINT    NOT NULL,
  attempted_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  outcome          VARCHAR(20) NOT NULL CHECK (outcome IN ('SUCCESS','FAILED')),
  error_code       VARCHAR(64),
  error_detail     TEXT,
  whatsapp_payload JSONB,
  UNIQUE (reminder_id, attempt_number)
);

CREATE TABLE patient_opt_outs (
  id           UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  patient_id   UUID        NOT NULL REFERENCES patients(id),
  channel      VARCHAR(20) NOT NULL CHECK (channel IN ('WHATSAPP','SMS','EMAIL')),
  opted_out_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  reason       VARCHAR(32) NOT NULL CHECK (reason IN ('PATIENT_REQUEST','CLINIC_REQUEST','SYSTEM')),
  source       VARCHAR(32) NOT NULL CHECK (source IN ('CLINIC_PANEL','WHATSAPP_REPLY','IMPORT')),
  opted_in_at  TIMESTAMPTZ,
  UNIQUE (patient_id, channel)
);
```

---

## 3. Job de Agendamento

### Lógica (cron a cada hora)

```
1. Para cada clínica com algum tipo de lembrete ativo:
   a. Buscar reminder_schedules WHERE:
      - status = 'SCHEDULED'
      - scheduled_at BETWEEN now() AND now() + 1h
      - NÃO existe patient_opt_outs ativo para patient_id + channel=WHATSAPP

   b. Para cada lembrete selecionado:
      - UPDATE atômico: status → 'DISPATCHED' (se status ainda = 'SCHEDULED')
      - Enfileirar na job queue
      - Registrar em reminder_logs
```

### Idempotência via UPDATE atômico

```sql
UPDATE reminder_schedules
SET    status = 'DISPATCHED', dispatched_at = now()
WHERE  id = $1
  AND  status = 'SCHEDULED'
  AND  NOT EXISTS (
    SELECT 1 FROM patient_opt_outs
    WHERE patient_id = reminder_schedules.patient_id
      AND channel = 'WHATSAPP'
      AND opted_in_at IS NULL
  )
RETURNING id;
-- Se retornar 0 linhas: outro worker já processou — ignorar
```

### Retry Policy

| Tentativa | Delay |
|---|---|
| 2 | 5 min |
| 3 | 15 min |
| Após 3 falhas | status → FAILED |

- Retryável: timeout, rate limit (`130429`), 5xx.
- Não retryável: número inválido (`131026`), template rejeitado (`132000`), opt-out.

---

## 4. Opt-out via Mensagem

### Fluxo

```
Paciente envia "SAIR"
        │
        ▼
POST /webhooks/whatsapp/status
  1. Validar assinatura HMAC (rejeitar se inválida)
  2. Responder 200 imediatamente
  3. Publicar evento raw na fila interna
        │
        ▼
Worker: WhatsappWebhookProcessor
  4. Detectar texto em {"sair","stop","cancelar","0","parar"}
  5. Resolver patient_id a partir do número E.164
  6. INSERT INTO patient_opt_outs (ON CONFLICT DO UPDATE)
  7. UPDATE reminder_schedules SET status='CANCELLED'
     WHERE patient_id=$1 AND status='SCHEDULED'
  8. Enviar template de confirmação
```

### Race Condition — 2 camadas de proteção

**Camada 1 — UPDATE do job:**
```sql
AND NOT EXISTS (
  SELECT 1 FROM patient_opt_outs
  WHERE patient_id = reminder_schedules.patient_id
    AND channel = 'WHATSAPP'
    AND opted_in_at IS NULL
)
```

**Camada 2 — worker de envio (antes de chamar a API):**
```python
if opt_out_exists(patient_id):
    status = 'CANCELLED'
    return  # não enviar
```

---

## 5. Lista de Tarefas Backend

| # | Tarefa | Tam | Depende de |
|---|--------|-----|------------|
| B-01 | Migrations das 4 tabelas + índices | P | Tech Lead: aprovação do schema |
| B-02 | CRUD `GET/PUT /clinics/{clinicId}/reminder-config` | P | B-01 |
| B-03 | `GET /clinics/{clinicId}/reminders` com filtros e paginação | P | B-01 |
| B-04 | `GET /reminders/{reminderId}` com logs agregados | P | B-01 |
| B-05 | `POST /patients/{patientId}/opt-out` | P | B-01 |
| B-06 | Integração BSP (client HTTP, autenticação, envio template, parsing de resposta) | G | Tech Lead: credenciais WABA + templates aprovados |
| B-07 | Worker de envio (consome fila, retry, atualiza schedules e logs) | G | B-01, B-06 |
| B-08 | Job cron de seleção horária (UPDATE atômico, enfileira) | M | B-01, B-07 |
| B-09 | Trigger/job de criação de lembretes a partir de agendamentos e receitas | M | B-01, modelo de dados existente |
| B-10 | `POST /webhooks/whatsapp/status`: validação HMAC, fan-out para fila interna | M | B-06 |
| B-11 | Worker de webhook: status (delivered, read, failed) → atualiza delivery_status | M | B-10, B-01 |
| B-12 | Worker de webhook: opt-out ("SAIR"), cancelamento de agendados, confirmação | M | B-10, B-05, B-06 |
| B-13 | `POST /internal/reminders/dispatch` (service-to-service) | P | B-07, B-08 |
| B-14 | Testes de integração (job + worker + banco + race condition opt-out) | M | B-07, B-08, B-12 |
| B-15 | Observabilidade: métricas, alertas de fila travada, log estruturado sem dado sensível | M | B-07, B-10 |
| B-16 | Documentação OpenAPI (swagger) dos endpoints públicos | P | B-02 a B-05, B-10 |

**Ordem crítica:** `B-01 → B-06 → B-07 → B-08 + B-09 → B-10 → B-11 + B-12 → demais`
