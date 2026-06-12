# Spec de Fluxo + Wireframe em Texto
## Feature: Lembrete Inteligente de Retorno e Renovação via WhatsApp
**Produto X · Squad Retenção · v1**

---

## 1. Fluxo do Usuário — Configuração na Clínica

### Onde fica no painel

```
Menu lateral principal
└── Configurações
    └── Comunicação com Pacientes   ← nova seção
        └── Lembretes via WhatsApp  ← nova tela
```

Acesso secundário: banner de ativação no Dashboard principal enquanto a feature não estiver configurada.

---

### Estado A — Configuração nunca feita

```
┌─────────────────────────────────────────────────────────────┐
│  ← Configurações / Comunicação com Pacientes                │
│                                                             │
│  Lembretes via WhatsApp                                     │
│  ─────────────────────────────────────────────────────────  │
│  [ícone whatsapp]                                           │
│  Envie lembretes automáticos de retorno e renovação         │
│  de receita pelo WhatsApp oficial.                          │
│                                                             │
│  Para começar, conecte sua conta do WhatsApp Business.      │
│                                                             │
│  [ Conectar WhatsApp Business ]   ← CTA primário           │
│  Saiba como funciona ↗                                      │
└─────────────────────────────────────────────────────────────┘
```

---

### Wizard de Integração WhatsApp Business (3 passos)

**Passo 1 — Autorizar conta Meta**
```
┌─────────────────────────────────────────────────────────────┐
│  Conectar WhatsApp Business                        [X fechar]│
│  ● Passo 1 de 3   ○ Passo 2   ○ Passo 3                    │
│                                                             │
│  PASSO 1 — Autorizar conta Meta                             │
│  Número que será usado:                                     │
│  [ +55 (__)  _____ - ______ ]                               │
│  ! Este número não pode ser o mesmo usado em WA pessoal.    │
│                                                             │
│  [ Cancelar ]          [ Continuar para o Meta → ]         │
└─────────────────────────────────────────────────────────────┘
```

**Passo 2 — Verificar número**
```
┌─────────────────────────────────────────────────────────────┐
│  ○ Passo 1   ● Passo 2 de 3   ○ Passo 3                    │
│  Aguardando confirmação do Meta...  [spinner]               │
│  Se não receber o código em 5 minutos: [ Reenviar código ]  │
└─────────────────────────────────────────────────────────────┘
```

**Passo 3 — Templates aprovados**
```
┌─────────────────────────────────────────────────────────────┐
│  ○ Passo 1   ○ Passo 2   ● Passo 3 de 3                    │
│  STATUS DOS TEMPLATES:                                      │
│  [•] Lembrete de retorno         Aguardando aprovação       │
│  [•] Lembrete de renovação       Aguardando aprovação       │
│                                                             │
│  Você receberá um e-mail quando os templates forem          │
│  aprovados.                                                 │
│  [ Fechar e aguardar aprovação ]                            │
└─────────────────────────────────────────────────────────────┘
```

---

### Estado B — Integração pendente (templates em análise)

```
┌─────────────────────────────────────────────────────────────┐
│  [⚠ amarelo]  Templates em análise pela Meta               │
│  Você receberá um e-mail de confirmação em até 24 h.        │
│  As configurações abaixo estão salvas e serão ativadas      │
│  assim que aprovado.                                        │
│  [seção de configurações — campos desabilitados/cinza]      │
└─────────────────────────────────────────────────────────────┘
```

### Estado C — Integração com erro

```
┌─────────────────────────────────────────────────────────────┐
│  [✕ vermelho]  Falha na conexão com o WhatsApp Business     │
│  Motivo: Token expirado / número não verificado.            │
│  [ Reconectar WhatsApp Business ]   [ Ver detalhes ]        │
└─────────────────────────────────────────────────────────────┘
```

---

### Estado D — Configuração salva e ativa

```
┌─────────────────────────────────────────────────────────────┐
│  Lembretes via WhatsApp                                     │
│  Número conectado: +55 (11) 9 8765-4321  [✓ ativo]          │
│  [ Desconectar ]                                            │
│  ─────────────────────────────────────────────────────────  │
│  LEMBRETE DE RETORNO                          [toggle ON ]  │
│  Enviar lembrete com antecedência de:                       │
│  [ ] 3 dias antes   [x] 1 dia antes   [ ] Ambos            │
│  Horário de envio:  [ 09:00 ▾ ]                             │
│  ─────────────────────────────────────────────────────────  │
│  LEMBRETE DE RENOVAÇÃO DE RECEITA             [toggle OFF]  │
│  (seção colapsada — ativar toggle para expandir)            │
│  ─────────────────────────────────────────────────────────  │
│  OPT-OUT DO PACIENTE                                        │
│  [x] Permitir opt-out por mensagem ("responda SAIR")        │
│  [x] Exibir opção de opt-out no cadastro do paciente        │
│  ─────────────────────────────────────────────────────────  │
│  [ Salvar configurações ]    [ Visualizar templates ]       │
└─────────────────────────────────────────────────────────────┘
```

> **Regra de UX:** desativar o toggle não cancela lembretes agendados para as próximas 2 horas — aparece modal de confirmação (ver Edge Case 4).

---

## 2. Fluxo do Usuário — Jornada do Lembrete (perspectiva da recepção)

### Tela: Ficha do Paciente — aba "Comunicação"

```
┌─────────────────────────────────────────────────────────────┐
│  Maria da Silva    [Dados] [Histórico] [Comunicação]        │
│                                                             │
│  PREFERÊNCIAS DE CONTATO                                    │
│  WhatsApp: (11) 9 8765-4321                                 │
│  Status do opt-in:  [✓ Ativo]   [ Revogar manualmente ]    │
│                                                             │
│  HISTÓRICO DE LEMBRETES                                     │
│  Data/hora          Tipo              Status                │
│  10/06/2026 09:00   Retorno           ✓ Entregue            │
│  07/06/2026 09:00   Retorno           ✗ Falha — sem WA      │
│  02/05/2026 09:00   Renovação receita ✓ Entregue            │
│  [ Ver mais ]                  [ Enviar manualmente ]       │
└─────────────────────────────────────────────────────────────┘
```

### Card de Agendamento na Agenda (status inline)

```
┌──────────────────────────────────────┐
│  09:00  Maria da Silva               │
│         Retorno — Dr. Costa          │
│  [💬 Lembrete enviado ✓]             │  ← badge verde
└──────────────────────────────────────┘
```

Clicar no badge vermelho (falha) abre drawer:

```
┌──────────────────────────────┐
│  Falha no lembrete           │
│  Motivo: número sem WhatsApp │
│  Tentativa: 10/06 às 09:00   │
│  [ Tentar reenviar ]         │
│  [ Ligar para paciente ]     │
└──────────────────────────────┘
```

### Banner de falhas agregadas

```
┌─────────────────────────────────────────────────────────────┐
│  [⚠]  3 lembretes falharam hoje. Ver pacientes afetados →  │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Copy Neutra dos Templates de WhatsApp

> **Princípio:** zero dado clínico — sem diagnóstico, medicamento, especialidade, nome do médico ou procedimento.

### Template 1 — Lembrete de Retorno (`retorno_lembrete_v1`)

```
Olá, {{1}}! 👋

Você tem um retorno agendado na {{2}}.

📅 Data: {{3}}
🕐 Horário: {{4}}

Para reagendar ou cancelar, entre em contato conosco pelo
número {{5}} ou compareça à recepção.

Até logo!
— Equipe {{2}}

─────────────────────
Para não receber mais lembretes, responda: SAIR
```

| Variável | Valor injetado |
|---|---|
| `{{1}}` | Primeiro nome do paciente |
| `{{2}}` | Nome fantasia da clínica |
| `{{3}}` | Data do agendamento (dd/mm/aaaa) |
| `{{4}}` | Horário do agendamento (HH:MM) |
| `{{5}}` | Telefone de contato da clínica |

---

### Template 2 — Lembrete de Renovação (`renovacao_receita_v1`)

```
Olá, {{1}}! 👋

Passamos para lembrar que você tem um documento
com vencimento próximo em nossa clínica.

Agende uma consulta para renová-lo:
📞 {{2}}
🌐 {{3}}

Não deixe para a última hora!
— Equipe {{4}}

─────────────────────
Para não receber mais lembretes, responda: SAIR
```

| Variável | Valor injetado |
|---|---|
| `{{1}}` | Primeiro nome do paciente |
| `{{2}}` | Telefone de contato da clínica |
| `{{3}}` | Link de agendamento online (opcional) |
| `{{4}}` | Nome fantasia da clínica |

> **Nota de compliance:** "documento com vencimento próximo" é a formulação mais neutra aprovável pela Meta sem mencionar "receita". Validar com jurídico antes da submissão.

---

### Confirmação de Opt-out

```
Tudo certo, {{1}}!

Removemos você da lista de lembretes automáticos da {{2}}.
Se mudar de ideia, avise a recepção na sua próxima visita.

— Equipe {{2}}
```

---

## 4. Estados da Feature — Diagrama em Texto

```
SISTEMA AGENDA LEMBRETE
         │
         ▼
  [Verificações pré-envio]
         │
    ┌────┴──────────────────────────────────────────┐
    │                                               │
Paciente tem número?                    Paciente em opt-out?
   NÃO → [SEM NÚMERO]                       SIM → [OPT-OUT ATIVO]
   SIM                                      (não envia, registra)
    │
    ▼
Clínica com integração ativa?
   NÃO → [INTEGRAÇÃO INATIVA] → notifica admin
   SIM
    │
    ▼
[Envia para API do WhatsApp Business]
    │
    ├──→ SUCESSO → [ENTREGUE ✓]
    │
    └──→ FALHA
              │
         ┌────┴──────────────────┐
         │                       │
    Número inválido        Sem WhatsApp
    [NÚMERO INVÁLIDO]      [SEM WHATSAPP]
         │
    Template rejeitado?
    SIM → [TEMPLATE REJEITADO] → bloqueia TODOS os envios
         │
    Janela 24h expirou?
    SIM → [JANELA EXPIRADA]

ESTADOS FINAIS (o que a recepção vê):
✓ Entregue              Verde
✗ Falha — Número inválido       Vermelho
✗ Falha — Sem WhatsApp          Vermelho
✗ Falha — Template rejeitado    Vermelho (bloqueio global)
✗ Falha — Janela expirada       Cinza
⊘ Não enviado — Opt-out         Cinza
⊘ Não enviado — Sem número      Cinza
⊘ Não enviado — Feature inativa Cinza
```

---

## 5. Edge Cases de UX

| # | Situação | Comportamento |
|---|---|---|
| EC-1 | Paciente sem número cadastrado | Lembrete não agendado; badge cinza com sugestão "Adicione o WhatsApp no cadastro"; não bloqueia agendamento |
| EC-2 | Paciente bloqueou o número da clínica | Status "Falha — mensagem bloqueada"; sem retry automático; opção "Ligar para paciente" no drawer |
| EC-3 | Lembrete duplicado (falha de idempotência) | Segundo envio bloqueado na camada de aplicação; log de warning; UI exibe apenas o primeiro envio |
| EC-4 | Clínica desativa feature com lembretes nas próximas 2h | Modal: "Há 4 lembretes agendados. Cancelar agora ou enviar e desativar após?" |
| EC-5 | Paciente sem WhatsApp no número | Status "Falha — número sem WhatsApp"; badge vermelho; sem fallback SMS em v1 |

---

## 6. Decisões de Design em Aberto

| # | Decisão | Dono |
|---|---|---|
| D1 | Antecedência padrão: 3d, 1d ou ambos? | PM |
| D2 | Opt-in: passivo (cadastro) ou ativo (confirmação via WA)? | PM + Jurídico |
| D3 | Lookups de WA no cadastro entra em v1? | Tech Lead |
| D4 | Fallback SMS entra em v1? | PM |
| D5 | SLA de notificação de falha para admin (imediato ou diário)? | PM |

---

## 7. Dependências de Outros Papéis

| Papel | O que Design precisa |
|---|---|
| PM | Decisão D1–D5; aprovação da copy dos templates |
| Tech Lead | Viabilidade de Lookups API de WA; estratégia de job scheduling |
| Backend | Endpoint de agendamento; webhook de status; processamento de "SAIR" |
| Frontend | Toggle com estado desabilitado; badge inline na agenda; drawer de detalhes |
| Jurídico | Validação da copy "documento com vencimento próximo"; fluxo de opt-in/opt-out |
