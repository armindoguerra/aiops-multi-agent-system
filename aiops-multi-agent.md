# Sistema Multiagentes para Resposta e Mitigação de Incidentes

**Versão:** 1.0\
**Status:** Proposta de arquitetura\
**Objetivo:** receber dados de um incidente, investigar sua provável
causa, produzir um runbook rastreável e executar, com controles de
segurança, as ações aprovadas para mitigar ou resolver o incidente.

------------------------------------------------------------------------

## 1. Visão geral

O sistema é uma plataforma de resposta a incidentes baseada em agentes
especializados. A partir de um alerta ou de um relato operacional, ela
reúne evidências, formula hipóteses, consulta procedimentos conhecidos,
gera um runbook específico para o caso e coordena sua execução.

O fluxo é dividido em duas fases:

1.  **Preparação:** entender o incidente, coletar evidências,
    identificar a causa provável e gerar um runbook revisável.
2.  **Mitigação:** validar permissões e riscos, executar as etapas
    autorizadas, verificar os resultados e atualizar o registro do
    incidente.

O sistema não deve tratar a saída de um LLM como autorização para
alterar ambientes. A execução é realizada por ferramentas
determinísticas, com políticas explícitas, controle de acesso, aprovação
humana quando necessária e trilha de auditoria.

### Resultados esperados

-   Reduzir o tempo entre detecção, diagnóstico e mitigação (MTTR).
-   Padronizar a investigação e a documentação de incidentes.
-   Reutilizar conhecimento operacional de incidentes anteriores.
-   Automatizar ações repetitivas e de baixo risco.
-   Evitar ações destrutivas ou não autorizadas.
-   Produzir evidências de cada decisão e ação executada.

------------------------------------------------------------------------

## 2. Escopo e premissas

### Entradas aceitas

O sistema pode receber incidentes por meio de:

-   Alertas de monitoramento (Datadog, Prometheus, Grafana, CloudWatch
    etc.).
-   Eventos de observabilidade e logs.
-   Tickets de ITSM (ServiceNow, Jira Service Management etc.).
-   Webhooks de pipelines, aplicações e infraestrutura.
-   Relatos manuais de operadores.

O payload normalizado deve conter, quando disponível:

  Campo                 Descrição
  --------------------- ----------------------------------------------
  `incident_id`         Identificador único do incidente
  `title`               Resumo do problema
  `description`         Sintomas e contexto informado
  `severity`            Severidade e impacto
  `detected_at`         Data/hora da detecção
  `affected_services`   Serviços, aplicações ou recursos afetados
  `environment`         Produção, homologação ou desenvolvimento
  `signals`             Métricas, logs, traces, alertas e eventos
  `related_changes`     Deploys, configurações ou mudanças recentes
  `constraints`         Restrições operacionais e janelas de mudança

Campos ausentes devem permanecer explicitamente desconhecidos. Os
agentes não podem preencher lacunas com fatos inventados.

### Limites

-   O sistema recomenda e executa apenas ações dentro de um catálogo de
    ferramentas e políticas previamente autorizadas.
-   Acesso a produção é concedido por identidade de workload, com
    privilégio mínimo e escopo limitado.
-   Ações irreversíveis, destrutivas, de alto impacto ou que alterem
    dados exigem aprovação humana explícita.
-   O sistema não deve declarar um incidente resolvido apenas porque uma
    ação foi executada; é necessário verificar sinais objetivos de
    recuperação.
-   A mitigação pode reduzir o impacto sem eliminar a causa raiz. O
    status deve distinguir mitigação, resolução e causa confirmada.

------------------------------------------------------------------------

## 3. Arquitetura lógica

``` mermaid
flowchart TD
    A[Alertas / ITSM / Webhooks / Relato manual] --> B[Incident Gateway]
    B --> C[Normalização e enriquecimento]
    C --> D[Orquestrador de incidentes]

    D --> E[Agente de triagem]
    E --> F[Agente de investigação]
    F <--> G[Observabilidade: logs, métricas e traces]
    F <--> H[CMDB, deploys e inventário]
    F --> I[Agente de diagnóstico]

    I <--> J[Base de conhecimento e incidentes anteriores]
    I --> K[Agente gerador de runbook]
    K --> L[Validador de políticas e riscos]
    L --> M{Aprovação necessária?}

    M -->|Sim| N[Operador / Incident Commander]
    N -->|Aprovado| O[Executor de ações]
    M -->|Não| O
    N -->|Rejeitado| P[Registrar decisão e encerrar execução]

    O <--> Q[Tool Gateway / APIs autorizadas]
    Q <--> R[Cloud / Kubernetes / CI-CD / ITSM]
    O --> S[Agente verificador]
    S <--> G
    S --> T{Mitigação verificada?}
    T -->|Não| U[Replanejar ou escalar]
    U --> D
    T -->|Sim| V[Agente documental]
    V --> W[Runbook final + timeline + relatório]
    W --> X[Atualizar ITSM e base de conhecimento]
```

### Componentes de plataforma

  -----------------------------------------------------------------------
  Componente                          Responsabilidade
  ----------------------------------- -----------------------------------
  Incident Gateway                    Receber eventos, autenticar origem,
                                      aplicar rate limit e criar o
                                      contexto do incidente

  Normalizador                        Converter diferentes formatos em um
                                      esquema comum

  Orquestrador                        Controlar estados, dependências,
                                      retries, timeouts e handoffs entre
                                      agentes

  Agent Runtime                       Executar agentes isolados, com
                                      ferramentas e contexto limitados

  Tool Gateway                        Intermediar chamadas a APIs
                                      operacionais, aplicar autorização e
                                      registrar ações

  Policy Engine                       Avaliar risco, ambiente, escopo,
                                      aprovação e limites de execução

  Event Store                         Persistir eventos, decisões,
                                      evidências e resultados

  Knowledge Store                     Armazenar runbooks, postmortems,
                                      arquitetura e incidentes históricos

  Observability                       Monitorar agentes, ferramentas,
                                      latência, custos e resultados
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 4. Agentes especializados

Os agentes devem ter responsabilidades delimitadas. O orquestrador
controla o fluxo; os agentes não devem iniciar ações uns dos outros
livremente nem compartilhar credenciais.

### 4.1 Agente de triagem --- `TriageAgent`

**Objetivo:** classificar o incidente e estabelecer prioridade.

**Responsabilidades:** - Validar o payload recebido. - Identificar
serviços e ambientes afetados. - Determinar severidade preliminar com
base em regras operacionais. - Detectar incidentes duplicados ou
relacionados. - Identificar informações ausentes que impeçam a
investigação. - Criar o contexto inicial do incidente.

**Saída estruturada:** - Severidade sugerida e justificativa. - Serviços
afetados. - Escopo e impacto conhecidos. - Perguntas ou dados
faltantes. - Próxima etapa recomendada.

O agente não altera sistemas nem reduz severidade sem evidência.

### 4.2 Agente de investigação --- `InvestigationAgent`

**Objetivo:** coletar e correlacionar evidências.

**Responsabilidades:** - Consultar logs, métricas, traces e eventos. -
Comparar o comportamento atual com baselines. - Verificar deploys,
alterações de configuração e eventos recentes. - Correlacionar sintomas
entre dependências. - Construir uma linha do tempo. - Identificar
evidências contraditórias e lacunas.

**Saída estruturada:** - Evidências com timestamp e origem. - Linha do
tempo. - Componentes potencialmente envolvidos. - Hipóteses iniciais e
grau de confiança. - Consultas executadas e intervalo de tempo
analisado.

Toda hipótese deve apontar para evidências verificáveis.

### 4.3 Agente de diagnóstico --- `DiagnosisAgent`

**Objetivo:** avaliar hipóteses e apontar a causa mais provável.

**Responsabilidades:** - Formular hipóteses concorrentes. - Avaliar cada
hipótese contra as evidências. - Separar causa provável, fatores
contribuintes e sintomas. - Identificar testes adicionais de baixo
risco. - Estimar confiança e impacto de cada hipótese. - Recomendar
estratégia de mitigação e alternativa de rollback.

**Saída estruturada:** - Causa provável e nível de confiança. -
Hipóteses descartadas e justificativas. - Evidências favoráveis e
contrárias. - Testes adicionais necessários. - Mitigação proposta e
riscos conhecidos.

A causa raiz só pode ser marcada como confirmada quando houver evidência
suficiente e validação posterior. Caso contrário, usar "causa provável"
ou "ainda desconhecida".

### 4.4 Agente gerador de runbook --- `RunbookAgent`

**Objetivo:** transformar o diagnóstico em um procedimento operacional
executável.

**Responsabilidades:** - Consultar runbooks existentes e postmortems
relevantes. - Adaptar procedimentos ao serviço, ambiente e versão
envolvidos. - Dividir a mitigação em etapas pequenas e verificáveis. -
Definir pré-condições, comandos ou chamadas de ferramenta, resultado
esperado e rollback. - Identificar etapas que exigem aprovação. -
Incluir critérios objetivos de sucesso e interrupção.

**Saída:** documento Markdown e representação estruturada do runbook.

O agente não pode criar comandos operacionais com base apenas em
suposições. Se faltarem dados essenciais, deve solicitar investigação
adicional ou produzir um runbook de diagnóstico, não de execução.

### 4.5 Agente de política e risco --- `SafetyPolicyAgent`

**Objetivo:** impedir execução insegura ou não autorizada.

**Responsabilidades:** - Classificar o risco de cada etapa. - Verificar
ambiente, escopo, permissões e janela operacional. - Exigir aprovação
para ações classificadas como sensíveis. - Bloquear ações fora do
catálogo permitido. - Validar limites de impacto, concorrência e
frequência. - Verificar existência de plano de rollback.

Este agente aplica regras determinísticas por meio do Policy Engine. O
LLM pode explicar riscos, mas não pode sobrepor a decisão de política.

### 4.6 Agente executor --- `ExecutionAgent`

**Objetivo:** executar as etapas aprovadas.

**Responsabilidades:** - Invocar ferramentas exclusivamente pelo Tool
Gateway. - Executar uma etapa por vez, respeitando dependências. -
Aplicar timeout, retry limitado e idempotência. - Capturar saída, código
de retorno e evidências. - Interromper a execução diante de divergência,
falha ou aumento de impacto. - Acionar rollback somente quando previsto
e autorizado.

O executor não recebe credenciais permanentes. O Tool Gateway obtém
credenciais temporárias e escopadas conforme a identidade do workload e
a política aplicável.

### 4.7 Agente verificador --- `VerificationAgent`

**Objetivo:** confirmar se a mitigação produziu o resultado esperado.

**Responsabilidades:** - Consultar novamente métricas, logs, traces e
health checks. - Comparar indicadores com os critérios definidos no
runbook. - Verificar regressões ou efeitos colaterais. - Determinar se a
execução deve continuar, parar, reverter ou escalar. - Recomendar
observação adicional antes do encerramento.

**Saída:** resultado por critério, evidências, confiança e status de
mitigação.

### 4.8 Agente documental --- `DocumentationAgent`

**Objetivo:** consolidar o registro operacional.

**Responsabilidades:** - Atualizar o runbook com o que efetivamente
ocorreu. - Registrar ações planejadas, aprovadas, executadas, rejeitadas
ou revertidas. - Criar timeline do incidente. - Produzir resumo para o
ticket e material para postmortem. - Atualizar a base de conhecimento
após revisão.

O agente deve distinguir claramente o plano original do histórico real
de execução.

------------------------------------------------------------------------

## 5. Fluxo operacional detalhado

### Fase A --- Recepção e triagem

1.  O Incident Gateway recebe o alerta ou relato.
2.  A origem é autenticada e o payload é validado.
3.  O normalizador converte o evento para o esquema comum.
4.  O sistema cria ou associa um `incident_id`.
5.  O TriageAgent classifica o incidente e identifica o escopo.
6.  O orquestrador registra o estado `TRIAGED`.

### Fase B --- Investigação e diagnóstico

1.  O InvestigationAgent coleta dados no intervalo temporal relevante.
2.  O agente consulta inventário, dependências e mudanças recentes.
3.  As evidências são armazenadas com origem, timestamp e referência.
4.  O DiagnosisAgent avalia hipóteses e recomenda testes.
5.  Se a confiança for insuficiente, o fluxo retorna à investigação ou
    solicita intervenção humana.
6.  O orquestrador registra `DIAGNOSED` ou `NEEDS_HUMAN_INPUT`.

### Fase C --- Geração e validação do runbook

1.  O RunbookAgent consulta procedimentos existentes.
2.  Gera as etapas específicas do incidente.
3.  Cada etapa recebe risco, pré-condições, timeout, validação e
    rollback.
4.  O SafetyPolicyAgent verifica as etapas.
5.  O sistema publica uma versão candidata do runbook.
6.  O orquestrador registra `RUNBOOK_READY`.

### Fase D --- Aprovação

A política determina se a execução pode ser automática ou se exige
aprovação.

A aprovação deve apresentar ao operador: - Resumo do incidente e
impacto. - Diagnóstico e evidências. - Etapas exatas que serão
executadas. - Recursos afetados. - Riscos e efeitos colaterais. - Plano
de rollback. - Identidade que executará as ações.

A aprovação é vinculada ao hash da versão do runbook. Qualquer alteração
posterior nas etapas invalida a aprovação anterior.

### Fase E --- Execução e verificação

1.  O ExecutionAgent solicita ao Tool Gateway a execução da próxima
    etapa.
2.  O Policy Engine valida novamente a autorização no momento da
    execução.
3.  O Tool Gateway executa a ação e registra o resultado.
4.  O ExecutionAgent verifica o retorno técnico da ferramenta.
5.  O VerificationAgent mede os indicadores de recuperação.
6.  Se os critérios forem atendidos, a etapa é marcada como concluída.
7.  Se houver falha, o sistema interrompe a sequência e avalia retry,
    rollback ou escalonamento.
8.  Após todas as etapas, o sistema realiza uma verificação final.

### Fase F --- Encerramento e aprendizado

1.  O DocumentationAgent atualiza o runbook com os resultados reais.
2.  O ticket é atualizado com timeline e evidências.
3.  O incidente é marcado como mitigado ou resolvido, conforme os
    critérios.
4.  Um postmortem pode ser gerado como rascunho.
5.  A inclusão do runbook na base de conhecimento depende de revisão
    humana.

------------------------------------------------------------------------

## 6. Máquina de estados do incidente

``` mermaid
stateDiagram-v2
    [*] --> RECEIVED
    RECEIVED --> TRIAGED
    TRIAGED --> INVESTIGATING
    INVESTIGATING --> DIAGNOSED
    INVESTIGATING --> NEEDS_HUMAN_INPUT
    NEEDS_HUMAN_INPUT --> INVESTIGATING
    DIAGNOSED --> RUNBOOK_DRAFTED
    RUNBOOK_DRAFTED --> POLICY_REVIEW
    POLICY_REVIEW --> AWAITING_APPROVAL
    POLICY_REVIEW --> RUNBOOK_READY
    AWAITING_APPROVAL --> RUNBOOK_READY
    AWAITING_APPROVAL --> REJECTED
    RUNBOOK_READY --> EXECUTING
    EXECUTING --> VERIFYING
    VERIFYING --> MITIGATED
    VERIFYING --> REPLANNING
    REPLANNING --> POLICY_REVIEW
    EXECUTING --> ROLLBACK
    ROLLBACK --> VERIFYING
    VERIFYING --> ESCALATED
    MITIGATED --> RESOLVED
    MITIGATED --> MONITORING
    MONITORING --> RESOLVED
    MONITORING --> REOPENED
    REOPENED --> INVESTIGATING
    REJECTED --> [*]
    RESOLVED --> [*]
    ESCALATED --> [*]
```

**Definições importantes:** - `MITIGATED`: impacto controlado, mas a
causa pode continuar presente. - `RESOLVED`: critérios de recuperação
atendidos e estabilidade confirmada. - `MONITORING`: período de
observação após a mitigação. - `REOPENED`: sintomas retornaram ou a
verificação detectou regressão. - `ESCALATED`: intervenção humana ou de
outra equipe é necessária.

------------------------------------------------------------------------

## 7. Contrato do runbook

O runbook deve existir em dois formatos sincronizados:

1.  **Markdown:** legível por operadores e anexável ao ticket.
2.  **JSON estruturado:** validável e consumível pelo orquestrador.

### 7.1 Exemplo de runbook em Markdown

``` markdown
# Runbook — Aumento de latência no serviço de pagamentos

## Contexto
- Incident ID: INC-12345
- Serviço: payments-api
- Ambiente: production
- Severidade: SEV-2
- Diagnóstico: saturação do pool de conexões com o banco
- Confiança: alta
- Evidências: links para dashboards, logs e traces

## Objetivo
Restabelecer a latência dentro do SLO sem perda de dados.

## Pré-condições
- Confirmar que o banco está saudável.
- Confirmar capacidade disponível.
- Validar aprovação para alteração de capacidade.

## Etapas
### 1. Coletar estado atual
- Ação: consultar conexões ativas, fila e latência.
- Sucesso: métricas registradas.
- Risco: baixo.

### 2. Aplicar mitigação autorizada
- Ação: executar a operação aprovada de ajuste do pool.
- Sucesso: fila reduzida e conexões dentro do limite.
- Risco: médio.
- Aprovação: obrigatória.

### 3. Verificar recuperação
- Ação: consultar latência, taxa de erro e saturação.
- Sucesso: indicadores dentro dos limites por 10 minutos.

## Rollback
Restaurar os parâmetros anteriores se houver aumento de erros ou saturação.

## Critérios de parada
- Erros aumentam após a mudança.
- Banco apresenta sinais de instabilidade.
- O recurso ultrapassa o limite de capacidade definido.

## Resultado
- Status: pendente
- Evidências: a preencher pelo executor
```

### 7.2 Esquema JSON sugerido

``` json
{
  "runbook_id": "RB-INC-12345-v1",
  "incident_id": "INC-12345",
  "version": 1,
  "status": "pending_approval",
  "objective": "Mitigate the incident without data loss",
  "diagnosis": {
    "summary": "Connection pool saturation",
    "confidence": 0.91,
    "evidence_ids": ["EV-001", "EV-002"]
  },
  "steps": [
    {
      "step_id": "STEP-001",
      "description": "Collect current connection pool metrics",
      "tool": "observability.query_metrics",
      "parameters": {
        "service": "payments-api",
        "environment": "production"
      },
      "risk": "low",
      "approval_required": false,
      "preconditions": ["service_identity_authorized"],
      "timeout_seconds": 30,
      "retry_policy": {
        "max_attempts": 2,
        "backoff_seconds": 5
      },
      "success_criteria": [
        "metrics_returned"
      ],
      "rollback": null,
      "status": "pending"
    }
  ],
  "global_stop_conditions": [
    "error_rate_increases",
    "database_health_degrades"
  ],
  "final_verification": {
    "metrics": ["latency_p95", "error_rate", "connection_pool_usage"],
    "observation_window_minutes": 10
  },
  "approval": {
    "required": true,
    "approved_by": null,
    "approved_at": null,
    "approved_runbook_hash": null
  }
}
```

O exemplo é ilustrativo. Nomes de ferramentas, parâmetros, limites e
critérios devem ser definidos para cada ambiente e serviço.

------------------------------------------------------------------------

## 8. Catálogo de ferramentas

Os agentes não devem ter acesso irrestrito a shell, terminal ou APIs.
Todas as operações devem passar por ferramentas tipadas, com contratos e
permissões explícitas.

  Ferramenta                      Operação                              Perfil de risco inicial
  ------------------------------- ------------------------------------- -------------------------
  `observability.query_metrics`   Consultar métricas                    Baixo
  `observability.search_logs`     Consultar logs                        Baixo
  `observability.get_trace`       Consultar traces                      Baixo
  `cmdb.get_service`              Consultar inventário e dependências   Baixo
  `deployment.get_history`        Consultar deploys recentes            Baixo
  `kubernetes.get_status`         Consultar estado de recursos          Baixo
  `kubernetes.restart_workload`   Reiniciar workload                    Médio/alto
  `deployment.rollback`           Reverter deploy                       Alto
  `cloud.scale_resource`          Alterar capacidade                    Médio/alto
  `feature_flag.set`              Alterar feature flag                  Médio
  `itsm.update_incident`          Atualizar ticket                      Baixo
  `notification.send`             Notificar responsáveis                Baixo

O risco efetivo depende do serviço, ambiente, escopo, impacto e
reversibilidade. A tabela não substitui a avaliação do Policy Engine.

### Requisitos do Tool Gateway

-   Autenticação forte e autorização por ferramenta, recurso e ambiente.
-   Credenciais temporárias e privilégio mínimo.
-   Allowlist de operações e parâmetros.
-   Validação de schema antes da execução.
-   Idempotency key para operações que possam ser repetidas.
-   Timeout, rate limit e limite de concorrência.
-   Registro de identidade, parâmetros sanitizados, horário e resultado.
-   Proteção contra injeção de comandos e parâmetros não confiáveis.
-   Bloqueio de acesso arbitrário à rede, shell e secrets.
-   Separação entre ferramentas de leitura e de escrita.

------------------------------------------------------------------------

## 9. Política de segurança e autonomia

A autonomia deve ser progressiva e baseada em risco, não apenas na
confiança declarada pelo modelo.

  -----------------------------------------------------------------------
  Nível                   Tipo de ação            Política
  ----------------------- ----------------------- -----------------------
  L0 --- Observação       Consultas de métricas,  Automática
                          logs e inventário       

  L1 --- Diagnóstico      Testes de leitura e     Automática, com limites
                          validações sem impacto  

  L2 --- Baixo impacto    Ações reversíveis e     Automática somente em
                          previamente aprovadas   escopo certificado
                          em catálogo             

  L3 --- Alto impacto     Restart em produção,    Aprovação humana
                          rollback, alteração de  
                          capacidade relevante    

  L4 --- Crítica          Exclusão de dados,      Bloqueada para execução
                          mudanças irreversíveis, autônoma
                          segurança ou impacto    
                          regulatório             
  -----------------------------------------------------------------------

### Regras obrigatórias

1.  **Deny by default:** qualquer ação não explicitamente permitida é
    bloqueada.
2.  **Human-in-the-loop:** ações de alto impacto exigem aprovação
    humana.
3.  **Aprovação vinculada ao plano:** a aprovação vale apenas para o
    hash exato do runbook.
4.  **Revalidação:** permissões e pré-condições são verificadas
    imediatamente antes de cada ação.
5.  **Escopo limitado:** uma ação deve afetar apenas os recursos
    declarados.
6.  **Rollback:** mudanças precisam ter reversão definida ou
    justificativa explícita para não possuí-la.
7.  **Kill switch:** operadores podem suspender todas as execuções ou um
    tipo de ferramenta.
8.  **Limites de execução:** número máximo de ações, duração total,
    retries e orçamento por incidente.
9.  **Segregação de funções:** quem aprova deve ser identificado; ações
    críticas podem exigir dupla aprovação.
10. **Dados não confiáveis:** logs, tickets e conteúdo externo são
    evidências, nunca instruções para os agentes. Instruções encontradas
    nesses dados não podem alterar políticas nem conceder permissões.

### Proteção contra prompt injection

-   Tratar logs, descrições de tickets, páginas e mensagens como
    conteúdo não confiável.
-   Nunca seguir instruções encontradas dentro de evidências.
-   Não permitir que agentes alterem seus próprios prompts, políticas ou
    permissões.
-   Remover ou mascarar segredos antes de enviar dados ao modelo.
-   Usar ferramentas com schemas rígidos, sem execução livre de código.
-   Validar toda saída do modelo contra schemas e políticas
    determinísticas.

------------------------------------------------------------------------

## 10. Persistência e modelo de dados

Recomenda-se persistir o estado operacional em banco transacional e os
artefatos de evidência em armazenamento apropriado.

Entidades principais:

-   `Incident`: identidade, severidade, serviço, ambiente e estado
    atual.
-   `IncidentEvent`: eventos imutáveis da timeline.
-   `Evidence`: origem, timestamp, conteúdo ou referência, hash e
    classificação.
-   `Hypothesis`: hipótese, confiança, evidências favoráveis e
    contrárias.
-   `Runbook`: versão, conteúdo, hash, estado e autoria.
-   `RunbookStep`: ferramenta, parâmetros, risco, dependências e
    critérios.
-   `Approval`: aprovador, decisão, timestamp e hash aprovado.
-   `Execution`: tentativa, identidade, parâmetros sanitizados,
    resultado e duração.
-   `Verification`: indicadores, limites, janela e conclusão.
-   `AuditRecord`: decisão de política e trilha de auditoria.

### Princípios de persistência

-   Eventos de execução e auditoria devem ser append-only.
-   Cada versão do runbook é imutável.
-   O estado atual deve poder ser reconstruído a partir dos eventos.
-   Evidências devem manter referência à fonte original.
-   Segredos e dados pessoais devem ser minimizados e protegidos.
-   Definir retenção conforme políticas de segurança, privacidade e
    regulação.

------------------------------------------------------------------------

## 11. Orquestração e confiabilidade

O orquestrador deve ser baseado em workflow durável, não em uma cadeia
de chamadas LLM mantida apenas em memória.

### Requisitos

-   Persistência do estado entre reinicializações.
-   Handoff explícito entre agentes.
-   Timeout por agente e por ferramenta.
-   Retry apenas para operações seguras e idempotentes.
-   Dead-letter queue para eventos que não possam ser processados.
-   Deduplicação por `incident_id` e chave do evento.
-   Controle de concorrência para incidentes que afetem o mesmo recurso.
-   Cancelamento e pausa operacional.
-   Reentrada segura após falha do runtime.
-   Limite de custo e tokens por incidente.

### Pseudocódigo do fluxo

``` python
async def handle_incident(event):
    incident = await incident_service.create_or_get(event)

    triage = await triage_agent.run(incident)
    await incident_service.record_triage(incident.id, triage)

    evidence = await investigation_agent.run(incident, triage)
    diagnosis = await diagnosis_agent.run(incident, evidence)

    if diagnosis.confidence < MIN_DIAGNOSIS_CONFIDENCE:
        return await request_human_investigation(incident, diagnosis)

    runbook = await runbook_agent.generate(incident, diagnosis, evidence)
    validation = await policy_engine.validate_runbook(runbook)

    if not validation.allowed:
        return await incident_service.block_execution(incident.id, validation)

    if validation.requires_approval:
        approval = await approval_service.request(runbook)
        if not approval.approved:
            return await incident_service.record_rejection(incident.id)

    for step in runbook.steps:
        await policy_engine.authorize_step(step, incident)
        result = await execution_agent.execute(step)
        await incident_service.record_execution(incident.id, step, result)

        if not result.success:
            return await incident_recovery.handle_failure(incident, runbook, step)

        verification = await verification_agent.verify_step(step, result)
        if not verification.passed:
            return await incident_recovery.handle_failed_verification(
                incident, runbook, step, verification
            )

    final_check = await verification_agent.verify_recovery(incident, runbook)
    await documentation_agent.finalize(incident, runbook, final_check)

    return await incident_service.close_or_monitor(incident.id, final_check)
```

O código é pseudocódigo conceitual: tratamento de exceções, persistência
transacional, autorização e execução real devem ser implementados nos
serviços correspondentes.

------------------------------------------------------------------------

## 12. Integrações sugeridas

A arquitetura deve manter os agentes desacoplados dos fornecedores. As
integrações são adaptadores substituíveis.

  Domínio                      Exemplos de integração
  ---------------------------- ------------------------------------------
  Observabilidade              Datadog, Prometheus, Grafana, CloudWatch
  ITSM                         ServiceNow, Jira Service Management
  Cloud                        AWS, Azure, Google Cloud
  Orquestração de containers   Kubernetes
  CI/CD                        GitHub Actions, GitLab CI, Jenkins
  Feature flags                LaunchDarkly ou solução interna
  Conhecimento                 Confluence, SharePoint, Git
  Mensageria                   Kafka, SQS, RabbitMQ
  Persistência                 PostgreSQL e object storage
  Identidade e secrets         IAM, Vault, Secrets Manager

### Exemplo de integração com Datadog e ServiceNow

-   Datadog gera o alerta e envia webhook ao Incident Gateway.
-   O gateway cria ou atualiza um incidente no ServiceNow.
-   InvestigationAgent consulta métricas, logs e traces por APIs de
    leitura.
-   O RunbookAgent gera o procedimento e o anexa ao ticket.
-   A aprovação é registrada no ServiceNow ou em serviço próprio.
-   ExecutionAgent aciona operações permitidas por adaptadores internos.
-   VerificationAgent consulta novamente Datadog.
-   DocumentationAgent atualiza o ticket e gera o resumo final.

------------------------------------------------------------------------

## 13. Observabilidade do próprio sistema

O sistema precisa ser observável como qualquer serviço crítico.

### Métricas técnicas

-   `incident_processing_duration_seconds`
-   `agent_execution_duration_seconds`
-   `agent_token_usage_total`
-   `agent_tool_calls_total`
-   `tool_execution_failures_total`
-   `policy_denials_total`
-   `approval_wait_duration_seconds`
-   `runbook_generation_duration_seconds`
-   `runbook_execution_success_total`
-   `verification_failures_total`

### Métricas de resultado

-   MTTA (Mean Time to Acknowledge).
-   MTTD (Mean Time to Diagnose).
-   MTTM (Mean Time to Mitigate).
-   MTTR (Mean Time to Resolve).
-   Percentual de incidentes mitigados automaticamente.
-   Percentual de runbooks aprovados sem alteração.
-   Taxa de rollback.
-   Taxa de reabertura de incidentes.
-   Ações bloqueadas pelo Policy Engine.
-   Incidentes com causa raiz confirmada.

A taxa de automação não deve ser otimizada isoladamente. Uma automação
que reduz MTTR, mas aumenta regressões ou mudanças indevidas, não é
bem-sucedida.

### Logs e traces

Cada execução deve possuir um `trace_id` correlacionando: - Evento
original. - Chamadas aos agentes. - Consultas às fontes de evidência. -
Decisões do Policy Engine. - Aprovações. - Chamadas ao Tool Gateway. -
Verificações e atualizações do ticket.

Não registrar tokens, senhas, chaves ou conteúdo sensível desnecessário.

------------------------------------------------------------------------

## 14. Avaliação e testes

Antes de permitir execução em produção, o sistema deve ser avaliado em
incidentes históricos e ambientes controlados.

### Estratégia de testes

1.  **Offline evaluation:** reproduzir incidentes históricos sem
    executar ações.
2.  **Runbook review:** especialistas avaliam correção, clareza,
    segurança e completude.
3.  **Simulation:** executar ferramentas simuladas e validar estados e
    decisões.
4.  **Chaos testing:** testar indisponibilidade de APIs, timeout,
    respostas incompletas e falhas parciais.
5.  **Shadow mode:** gerar diagnósticos e runbooks em paralelo ao
    processo atual, sem executar mudanças.
6.  **Canary automation:** habilitar poucas ações de baixo risco em
    serviços selecionados.
7.  **Progressive rollout:** ampliar gradualmente após evidência de
    segurança e eficácia.

### Critérios mínimos de aceitação

-   Nenhuma ação fora do catálogo autorizado é executada.
-   Toda ação de escrita tem identidade, autorização e auditoria.
-   A aprovação é invalidada quando o runbook muda.
-   Falhas de ferramenta não geram retries ilimitados.
-   O sistema consegue pausar e cancelar execuções.
-   O estado é recuperável após reinicialização.
-   O sistema não encerra incidentes sem verificação.
-   Incidentes ambíguos são escalados em vez de receber ações
    especulativas.

------------------------------------------------------------------------

## 15. Plano de implementação incremental

### Etapa 1 --- Fundação e documentação

-   Criar Incident Gateway e esquema canônico.
-   Integrar ITSM e observabilidade em modo leitura.
-   Implementar triagem, investigação e diagnóstico.
-   Gerar runbooks Markdown sem execução.
-   Armazenar evidências e versões.
-   Medir qualidade dos diagnósticos e runbooks.

**Saída:** sistema que recebe incidentes, investiga e documenta
recomendações.

### Etapa 2 --- Simulação e governança

-   Criar Tool Gateway.
-   Implementar catálogo de ferramentas e Policy Engine.
-   Simular operações em ambientes isolados.
-   Implementar aprovação humana e auditoria.
-   Validar rollback e critérios de parada.

**Saída:** runbooks estruturados e testados, com execução simulada.

### Etapa 3 --- Automação controlada

-   Habilitar ferramentas de baixo risco.
-   Automatizar ações reversíveis previamente certificadas.
-   Adicionar verificação automática e monitoramento pós-ação.
-   Implantar kill switch e limites por incidente.

**Saída:** mitigação automática de casos conhecidos e de baixo risco.

### Etapa 4 --- Expansão

-   Adicionar novos serviços e ferramentas.
-   Construir catálogo de padrões de incidentes.
-   Usar feedback de operadores para melhorar prompts, políticas e
    procedimentos.
-   Expandir automação por classe de risco, mantendo aprovação para
    ações críticas.

**Saída:** plataforma multiagentes de resposta a incidentes com
autonomia progressiva.

------------------------------------------------------------------------

## 16. Decisões arquiteturais recomendadas

1.  **Workflow durável como núcleo:** agentes são etapas especializadas
    de um processo controlado, não uma sociedade de agentes autônomos
    sem limites.
2.  **LLM para raciocínio; código para controle:** decisões de fluxo,
    autorização, limites e execução ficam em serviços determinísticos.
3.  **Runbook estruturado como contrato:** Markdown atende pessoas; JSON
    atende o executor.
4.  **Tool Gateway obrigatório:** nenhum agente acessa diretamente
    produção.
5.  **Autonomia por risco:** iniciar em modo leitura e avançar
    gradualmente.
6.  **Verificação independente:** um agente diferente do executor
    confirma o resultado por sinais observáveis.
7.  **Auditoria por padrão:** decisões, evidências, aprovações e
    execuções são rastreáveis.
8.  **Conhecimento com curadoria:** runbooks aprendidos de incidentes
    anteriores só entram na base oficial após revisão.

## 17. Definição de sucesso

O sistema será considerado bem-sucedido quando demonstrar, em operação
controlada:

-   Redução mensurável do tempo de diagnóstico e mitigação.
-   Runbooks corretos, completos e reproduzíveis.
-   Ausência de ações não autorizadas.
-   Taxa aceitável de rollback e reabertura.
-   Evidências suficientes para auditoria e postmortem.
-   Capacidade de reconhecer incerteza e escalar para humanos.
-   Operação resiliente mesmo com falha de agentes, modelos ou
    integrações.

**Princípio final:** o sistema deve ser autônomo na coleta de
evidências, na elaboração de hipóteses e na preparação de procedimentos;
a autonomia para alterar ambientes deve ser conquistada por etapas,
limitada por políticas verificáveis e acompanhada de validação
independente.
