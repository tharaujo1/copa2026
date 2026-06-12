# Mini-PRD de Delivery — Lembrete Inteligente de Retorno e Renovação via WhatsApp

**Produto X · Squad Retenção · v1**
**Status:** Pronto para refinamento técnico

---

## 1. Escopo v1 — MoSCoW

> **(P)** = premissa assumida com base em evidências do discovery; **(I)** = inferência do PM; itens sem marcação = requisito explícito do discovery.

### Must (obrigatório para lançar)

| # | Funcionalidade | Observação |
|---|---|---|
| M1 | Envio de lembrete de retorno via WhatsApp Business API oficial | Gatilho: data de retorno cadastrada no agendamento |
| M2 | Envio de lembrete de renovação de receita via WhatsApp Business API oficial | Gatilho: data de validade da receita (regra a definir — ver P1) |
| M3 | Templates de mensagem aprovados pela Meta, sem conteúdo clínico sensível | Requisito LGPD + política WhatsApp |
| M4 | Opt-out funcional via resposta na conversa ("responda SAIR" ou equivalente) | Base legal LGPD; formato exato a definir — ver P2 |
| M5 | Registro de status de cada disparo (entregue, falha, opt-out, número inválido) | Necessário para fallback e para métricas |
| M6 | Fallback visível na UI: alerta para recepção quando lembrete falhar | (P) Recepção precisa saber para agir manualmente |
| M7 | Configuração por clínica: ativar/desativar lembrete por tipo (retorno / renovação) | Requisito explícito do discovery |
| M8 | Configuração por clínica: janela de antecedência do disparo | Antecedência ideal a validar — ver P3 |
| M9 | Idempotência: garantia de não duplicar o envio para o mesmo paciente/evento | Requisito não-funcional explícito |
| M10 | Consentimento (opt-in) explícito do paciente registrado antes do primeiro envio | LGPD — base legal para contato |

### Should (importante, mas não bloqueia o lançamento)

| # | Funcionalidade | Observação |
|---|---|---|
| S1 | Opt-out também pelo cadastro do paciente no sistema (além do canal WhatsApp) | Alternativa de controle para a recepção |
| S2 | Log auditável de todos os disparos por paciente (data, status, template usado) | Conformidade LGPD + suporte |
| S3 | Preview do template de mensagem na tela de configuração da clínica | (I) Reduz erros de expectativa |
| S4 | Relatório simples de disparos (enviados, entregues, falhas, opt-outs) por período | Necessário para medir métricas de sucesso |

### Could (desejável, entra se houver folga)

| # | Funcionalidade | Observação |
|---|---|---|
| C1 | Múltiplos lembretes em cascata (ex.: 3 dias antes + 1 dia antes) | (I) Antecedência ideal ainda em aberto; pode ser v1.1 |
| C2 | Regra de antecedência configurável por especialidade | Complexidade extra; avaliar com Tech Lead |
| C3 | Notificação in-app para a recepção quando paciente confirmar leitura | (I) Dependente de capacidade da API |

### Won't (fora de escopo v1)

| # | Funcionalidade | Justificativa |
|---|---|---|
| W1 | Chatbot bidirecional | Discovery explícito |
| W2 | Confirmação ou remarcação automática de consulta | Discovery explícito |
| W3 | Campanhas de marketing ou comunicação promocional | Discovery explícito + risco LGPD |
| W4 | Canal de lembrete por e-mail ou SMS nesta versão | (I) Foco no canal de maior resposta |

---

## 2. Hipótese de Produto

> Se clínicas com retornos agendados e/ou receitas de uso contínuo ativarem o lembrete automático via WhatsApp (com opt-in do paciente e template aprovado), então o paciente receberá o aviso no momento certo sem depender da recepção, resultando em aumento da taxa de retorno em 90 dias e redução do churn de clínicas que citam "falta de integração/retorno" como motivo de cancelamento — medidos pela comparação entre clínicas com a feature ativa e clínicas sem a feature ativa no mesmo período (baseline: **a validar** antes do lançamento).

---

## 3. Métricas de Sucesso

### Primária

| Métrica | Definição | Como medir | Baseline |
|---|---|---|---|
| Taxa de retorno em 90 dias | % de pacientes com retorno previsto que efetivamente voltaram à clínica dentro de 90 dias | Cohort: clínicas com feature ativa vs. grupo controle | **A validar** — levantamento pré-lançamento obrigatório |

### Secundárias

| Métrica | Definição | Baseline |
|---|---|---|
| Taxa de entrega do lembrete | % de disparos com status "entregue" / total tentados | A validar |
| Taxa de opt-out | % de pacientes que solicitaram opt-out / total que receberam ≥1 lembrete | A validar |
| Ativação da feature | % de clínicas elegíveis que ativaram ao menos um tipo de lembrete em 30 dias | A validar |
| Impacto no churn | Variação na taxa de churn mensal de clínicas com feature ativa | A validar |

### Guard-rails

| Métrica | Limite | Ação se atingir |
|---|---|---|
| Reclamações de spam | > 0,5‰ de reports (referência Meta) | Pausar disparos e investigar |
| Taxa de falha de entrega | > 20% persistente | Alerta automático; revisar cadastros |
| Violação LGPD / reclamação formal | Qualquer incidente | Parar feature, acionar jurídico |

---

## 4. Riscos e Mitigações — Top 5

| # | Risco | Prob | Impacto | Mitigação |
|---|---|---|---|---|
| R1 | Template rejeitado ou suspenso pela Meta | Média | Alto | Aprovar template ANTES do dev; ter template reserva; monitorar quality score |
| R2 | Qualidade do dado de telefone (inválidos, sem WhatsApp) | Alta | Médio | Validação de formato no cadastro; comunicar à clínica; validação via API antes do disparo |
| R3 | LGPD — ausência de opt-in válido | Média | Alto | Opt-in obrigatório como pré-condição; revisar fluxo com jurídico antes do dev |
| R4 | Custo por conversa do WhatsApp não definido | Alta | Médio | P4 precisa ser respondida ANTES do dev; sem política de cobrança = sem lançamento |
| R5 | Agendamento confiável em escala | Média | Alto | Idempotência no contrato técnico; testes de carga e reenvio em QA; monitoramento de fila |

---

## 5. Perguntas a Fechar ANTES do Dev

| # | Pergunta | Quem responde | Impacto de não responder |
|---|---|---|---|
| P1 | Gatilho da renovação de receita: data de emissão + validade, manual, ou regra por especialidade? | Tech Lead + Médico-referência | Bloqueia backend e template |
| P2 | Opt-out: "SAIR" suficiente, ou precisa também no cadastro? Ambos Must ou um é Should? | PM + Jurídico/LGPD | Impacta UX, dados e compliance |
| P3 | Antecedência ideal: 3 dias, 1 dia, ou ambos em cascata? | PM (pesquisa rápida ou benchmark) | Bloqueia modelo de configuração e template |
| P4 | Quem paga o custo por conversa do WhatsApp: clínica ou incluso no plano? | PM + Comercial/Pricing | Sem isso não há BSP e não há desenvolvimento |
| P5 | Número WABA remetente: por clínica ou compartilhado do Produto X? | Tech Lead + PM | Muda radicalmente a arquitetura de integração |

---

## 6. Dependências de Outros Papéis

### O que o PM entrega para cada papel

| Papel | Entregável do PM |
|---|---|
| Tech Lead | Este PRD + respostas às P1, P4, P5 antes do refinamento técnico |
| Designer | JTBD, fluxos em prosa, restrições de template (sem dado clínico sensível), casos de fallback |
| Backend | Requisitos de integração WhatsApp, modelo de dados necessário, regras de negócio de disparo |
| Frontend | Telas necessárias, fluxo de opt-in, especificação do relatório simples |
| QA | Critérios de aceite por funcionalidade, casos de borda prioritários |
| Data | Definição das métricas, pedido de setup de cohort para taxa de retorno 90D |

### O que o PM precisa de cada papel

| Papel | O que o PM precisa |
|---|---|
| Tech Lead | Viabilidade técnica das opções de arquitetura; estimativa de esforço; riscos técnicos |
| Designer | Protótipo navegável dos fluxos para validação com recepcionistas antes do dev |
| QA | Lista de cenários de risco que QA quer cobrir (especialmente idempotência e fallback) |
| Data | Confirmação de que os eventos são instrumentáveis com o stack atual |
| Jurídico/LGPD | Parecer sobre fluxo de opt-in/opt-out, linguagem dos templates, base legal |
| Comercial/Pricing | Decisão sobre modelo de custo da API (desbloqueador de P4) |
