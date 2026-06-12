# Plano de QA — Lembrete Inteligente de Retorno e Renovação via WhatsApp

**Produto X · Feature v1 · Data: 2026-06-12**

---

## 1. Cenários de Teste — Caminho Feliz

### CT-01 — Clínica configura lembrete de retorno pela primeira vez

**Given** que o usuário está autenticado como administrador da clínica
**And** a feature de lembretes está habilitada no plano
**And** nenhuma configuração foi criada anteriormente

**When** o usuário acessa "Configurações > Lembretes", seleciona "Lembrete de Retorno", define antecedência e salva

**Then** a configuração é persistida corretamente
**And** o painel exibe status "Ativo"
**And** nenhum lembrete é disparado imediatamente
**And** log de criação registrado com timestamp e usuário responsável

---

### CT-02 — Lembrete de retorno enviado com sucesso

**Given** configuração de lembrete de retorno ativa
**And** paciente com número válido com WhatsApp e opt-in registrado
**And** retorno agendado dentro da janela configurada

**When** o job de agendamento executa no horário programado

**Then** a API do WhatsApp Business é chamada uma única vez com o template correto
**And** mensagem é entregue ao paciente
**And** status no painel muda para "Enviado"
**And** a mensagem não contém dado clínico sensível

---

### CT-03 — Lembrete de renovação de receita enviado com sucesso

**Given** configuração de lembrete de renovação ativa
**And** paciente com número válido, WhatsApp e opt-in
**And** receita dentro da janela configurada

**When** o job executa

**Then** template de renovação é usado (distinto do de retorno)
**And** mensagem não menciona medicamento, diagnóstico ou especialidade
**And** log de envio criado com tipo "Renovação de Receita"

---

### CT-04 — Paciente realiza opt-out via mensagem "SAIR"

**Given** paciente com opt-in ativo que já recebeu ao menos um lembrete

**When** paciente envia "SAIR" para o número WhatsApp Business da clínica

**Then** sistema registra opt-out com timestamp
**And** nenhum lembrete futuro é enviado para esse paciente
**And** lembretes já agendados na fila são cancelados
**And** log de opt-out auditável (origem, data, número)

---

### CT-05 — Recepção visualiza status "Enviado" no painel

**Given** lembrete processado e enviado com sucesso

**When** usuário acessa "Lembretes > Histórico"

**Then** lembrete aparece com status "Enviado"
**And** são exibidos: nome do paciente (sem dado clínico), tipo, data/hora
**And** filtros por data, tipo e status funcionam
**And** dados de outras clínicas não aparecem

---

## 2. Edge Cases Críticos

### EC-01 — Número de telefone inválido

**Given** cadastro do paciente com número inválido (formato errado)
**And** lembrete agendado para esse paciente

**When** o job tenta processar o envio

**Then** validação de formato executada antes de chamar a API
**And** chamada à API não é feita
**And** status marcado como "Falha — Número Inválido"
**And** alerta gerado no painel para correção do cadastro

---

### EC-02 — Paciente sem WhatsApp no número

**Given** número válido mas sem conta WhatsApp ativa
**And** BSP retorna erro (ex: 131047)

**When** o job tenta enviar

**Then** status marcado como "Falha — Sem WhatsApp"
**And** sem retry (erro não transitório)
**And** recepção visualiza status e motivo no painel

---

### EC-03 — Template rejeitado pela Meta

**Given** template com status "Rejeitado" na Meta

**When** o job tenta processar usando esse template

**Then** sistema verifica status do template antes de enfileirar
**And** envio não é executado
**And** status marcado como "Bloqueado — Template Rejeitado"
**And** alerta exibido ao administrador da clínica

---

### EC-04 — Tentativa de envio duplicado

**Given** lembrete já com status "Enviado"
**And** job re-executado por falha de infra

**When** job tenta processar o mesmo lembrete novamente

**Then** mecanismo de idempotência detecta status final
**And** chamada à API não é feita segunda vez
**And** paciente não recebe mensagem duplicada
**And** evento de tentativa duplicada registrado em log de auditoria

---

### EC-05 — Race condition: opt-out antes do envio

**Given** lembrete na fila aguardando processamento
**And** paciente envia "SAIR" no intervalo entre enfileiramento e processamento

**When** o job processa o lembrete enfileirado

**Then** job consulta status de consentimento antes de chamar a API
**And** detecta opt-out ativo
**And** cancela envio sem chamar a API
**And** status atualizado para "Cancelado — Opt-out"

---

### EC-06 — Clínica desativa feature após agendamento

**Given** lembretes na fila para pacientes da clínica
**And** administrador desativa a feature

**When** job tenta processar lembretes dessa clínica

**Then** job verifica status da feature antes de processar
**And** todos os lembretes pendentes são cancelados sem envio
**And** status marcado como "Cancelado — Feature Desativada"

---

### EC-07 — Retorno remarcado após agendamento do lembrete

**Given** lembrete agendado para o retorno original
**And** retorno remarcado para nova data

**When** a remarcação é registrada no sistema

**Then** lembrete original é cancelado
**And** novo lembrete é criado para a nova data
**And** paciente não recebe lembrete referente à data antiga
**And** histórico registra cancelamento + criação

---

### EC-08 — Número do paciente alterado após agendamento

**Given** lembrete enfileirado com o número antigo
**And** recepção atualiza o número no cadastro

**When** job processa o lembrete

**Then** job usa o número atual do cadastro (não o snapshot)
**And** se número é válido com WhatsApp, envio é realizado com sucesso
**And** se número foi removido/invalidado, lembrete falha com status adequado

> **Nota:** comportamento depende da decisão de arquitetura (snapshot vs. número atual). Definir com Tech Lead.

---

### EC-09 — BSP fora do ar durante o envio

**Given** lembrete sendo processado
**And** BSP retorna 5xx ou timeout

**When** job tenta enviar

**Then** sistema identifica erro como transitório
**And** retry agendado com backoff exponencial (3 tentativas: 5min, 15min)
**And** se todas falharem: status "Falha — Serviço Indisponível" + alerta ops
**And** sem mensagem duplicada em retry bem-sucedido após falha parcial

---

### EC-10 — Múltiplos retornos do mesmo paciente na mesma semana

**Given** paciente com dois retornos para datas diferentes na mesma semana
**And** ambos dentro da janela de antecedência

**When** job executa

**Then** lembrete distinto criado para cada retorno
**And** paciente recebe duas mensagens separadas referenciando datas corretas
**And** sem duplicatas (cada lembrete enviado exatamente uma vez)

---

## 3. Cenários de Segurança / LGPD

### SG-01 — Mensagem não contém dado clínico sensível

**Given** lembrete configurado para envio

**When** conteúdo da mensagem é gerado a partir do template aprovado

**Then** texto não contém diagnóstico, medicamento, especialidade, CID, resultado de exame
**And** teste automatizado valida o conteúdo gerado contra lista de campos proibidos

---

### SG-02 — Opt-out respeitado em 100% dos casos

**Given** paciente com opt-out registrado (por qualquer canal)

**When** qualquer job de envio é executado

**Then** nenhum lembrete é enviado ao paciente em nenhuma circunstância
**And** bloqueio verificado no início do processamento de cada lembrete individualmente
**And** teste de regressão cobre: opt-out antes do agendamento, após o agendamento (race condition), com múltiplos lembretes na fila

---

### SG-03 — Log de consentimento auditável

**Given** paciente concede opt-in

**Then** sistema registra: timestamp, canal de coleta, usuário responsável, versão do texto de consentimento
**And** log é imutável (append-only)
**And** disponível para exportação pelo DPO

---

### SG-04 — Acesso restrito à clínica correta (IDOR)

**Given** registros de lembretes de múltiplas clínicas no sistema

**When** usuário autenticado de uma clínica acessa o histórico via painel ou API

**Then** apenas lembretes da própria clínica são retornados
**And** manipulação de ID na API retorna HTTP 403 ou 404
**And** teste cobre acesso via interface e via chamada direta à API

---

## 4. Dados de Teste Necessários

| ID | Dado | Detalhes |
|---|---|---|
| DT-01 | Paciente com número válido com WhatsApp | Sandbox BSP |
| DT-02 | Paciente com número válido sem WhatsApp | BSP deve retornar erro 131047 |
| DT-03 | Paciente com número inválido | Formato errado |
| DT-04 | Paciente com opt-out ativo | Registro pré-existente |
| DT-05 | Paciente com opt-in ativo | Consentimento registrado |
| DT-06 | Clínica com feature ativada | Config + template aprovado |
| DT-07 | Clínica com feature desativada | Feature flag = false |
| DT-08 | Template aprovado pela Meta | Status "APPROVED" no sandbox |
| DT-09 | Template pendente | Status "PENDING" |
| DT-10 | Template rejeitado | Status "REJECTED" |
| DT-11 | Retorno dentro da janela de antecedência | Agendamento em D-2 |
| DT-12 | Retorno fora da janela | Agendamento em D-10 |
| DT-13 | Lembrete com status "Enviado" (para idempotência) | Registro pré-existente |
| DT-14 | Múltiplos retornos do mesmo paciente na semana | Dois agendamentos distintos |
| DT-15 | BSP simulando timeout/5xx | Mock/stub configurado |

---

## 5. Definition of Done da v1

### DoD-01 — Configuração de lembrete
- [ ] Admin cria, edita e desativa configuração via painel
- [ ] Campos obrigatórios validados
- [ ] Não aplica retroativamente
- [ ] Log de alteração registrado

### DoD-02 — Envio de lembrete de retorno
- [ ] Enviado dentro da janela configurada (tolerância: ± 15 min)
- [ ] Template correto utilizado
- [ ] Mensagem sem dado clínico sensível (validado por teste automatizado)
- [ ] Status atualizado corretamente após resposta da API

### DoD-03 — Envio de lembrete de renovação
- [ ] Mesmo critério do DoD-02
- [ ] Template de renovação distinto do de retorno
- [ ] Nenhum dado de prescrição na mensagem

### DoD-04 — Opt-out funcional
- [ ] Processado em até 5 minutos após recebimento
- [ ] Nenhum lembrete enviado após opt-out
- [ ] Lembretes na fila cancelados
- [ ] Race condition (EC-05) coberta por teste automatizado
- [ ] Log imutável e auditável
- [ ] Teste de regressão obrigatório após qualquer deploy no fluxo de envio

### DoD-05 — Status de entrega visível
- [ ] Painel exibe: Agendado, Enviado, Falha (com motivo), Cancelado
- [ ] Atualização com lag máximo de 1 minuto
- [ ] Filtros por data, tipo e status funcionando
- [ ] Isolamento por clínica garantido
- [ ] Nenhum dado clínico exibido

### DoD-06 — Idempotência garantida
- [ ] Execução dupla do job não gera envio duplicado (EC-04 por teste automatizado)
- [ ] Chave de idempotência definida e documentada
- [ ] Teste de carga com re-execução concorrente sem duplicatas

---

## 6. Dependências de Outros Papéis

| Papel | O que QA precisa |
|---|---|
| Backend | Endpoint de status (`GET /reminders/{id}`); sandbox/mock BSP com cenários de erro; mecanismo para forçar re-execução do job; documentação da política de retry; endpoint para simular recebimento de "SAIR" |
| Frontend | Tela de configuração em homologação; tela de histórico/status; indicação visual de feature ativada/desativada |
| Tech Lead | Decisão: snapshot do número vs. número atual (EC-08); comportamento de lembretes ao remarcar retorno (EC-07); janela de tolerância de envio (± 15 min?) |
| PM | Confirmação dos critérios de aceite de cada DoD; texto exato dos templates aprovados; regra para múltiplos retornos na mesma semana (EC-10); prazo máximo de processamento do opt-out (proposto: 5 min) |
