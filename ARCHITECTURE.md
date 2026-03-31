# Hybrid Intelligence Squad (HIS)

> Um framework de automação baseado em **GitHub Actions** e **GitHub Copilot** que organiza trabalho complexo em fases sequenciais, alternando entre execução autônoma por IA e revisão humana. O HIS orquestra o ciclo completo — da criação de tarefas até a entrega — com rastreabilidade total via GitHub Projects.

---

## 1. Visão Geral — Como Funciona

O HIS funciona como um **ciclo autônomo**: uma issue é criada, o Copilot trabalha nela, um humano revisa, e ao fazer merge a próxima fase é criada automaticamente.

```mermaid
flowchart LR
    subgraph HUMANO["👤 Humano"]
        direction TB
        H1[Inicia o Bootstrap]
        H2[Revisa o Pull Request]
        H3[Faz Merge do PR]
    end

    subgraph AUTOMAÇÃO["🤖 Automação GitHub Actions"]
        direction TB
        A1[Cria Issue da Fase]
        A2[Atribui Copilot à Issue]
        A3[Copilot Trabalha no Código]
        A4[Detecta PR Pronto]
        A5[Atualiza Status → Review]
        A6[Fecha Issue e Cria Próxima]
    end

    H1 -->|1. Dispara| A1
    A1 -->|2. Automático| A2
    A2 -->|3. Automático| A3
    A3 -->|4. Automático| A4
    A4 -->|5. Automático| A5
    A5 -->|6. Notifica| H2
    H2 -->|7. Aprova| H3
    H3 -->|8. Automático| A6
    A6 -.->|Ciclo repete| A1
```

---

## 2. Arquitetura de Componentes

Todos os arquivos vivem dentro de `.github/` e se dividem em **4 camadas**:

```mermaid
flowchart TB
    subgraph CONFIG["Camada de Configuração"]
        SC["squad-config.json<br/>Org, Projeto, Labels,<br/>Bot Login, Timeout"]
        TM["templates/default/templates-mapping.json<br/>Labels + Grafo de dependências das fases"]
        IT["templates/default/0N-*.md<br/>Templates de Issue por fase"]
    end

    subgraph ACTIONS["Camada de Ações Reutilizáveis"]
        LSC["load-squad-config<br/>Lê config → outputs"]
        AC["assign-copilot<br/>GraphQL: addAssignees"]
        CI["create-phase-issue<br/>REST: criar issue + sub-issue + projeto"]
        UPS["update-project-status<br/>GraphQL: update field"]
        FWI["find-wip-issues<br/>REST: busca paginada"]
        SL["sync-labels<br/>REST: sync labels"]
        SM["sync-milestones<br/>REST: sync milestones"]
        ATP["add-to-project<br/>GraphQL: add item"]
        IET["import-external-task<br/>REST: importar task individual"]
    end

    subgraph WORKFLOWS["Camada de Workflows"]
        W1["bootstrap-squad.yml<br/>Setup inicial manual"]
        W2["on-issue-opened-start-work.yml<br/>Issue aberta → WIP"]
        W3["on-copilot-pr-transition-to-review.yml<br/>PR aberto → Review"]
        W4["scheduled-wip-monitor.yml<br/>Monitor a cada 15min"]
        W5["on-pr-merge-close-issues.yml<br/>PR mergeado → Done"]
        W6["create-next-phase-issues.yml<br/>Issue fechada → próxima"]
        W7["manual-sync-labels.yml<br/>Sync manual de labels"]
        W8["import-from-external-board.yml<br/>Importar tasks Jira/Kanbanize"]
        W9["export-completion-report.yml<br/>Relatório CSV/Markdown"]
    end

    subgraph AGENTS["Camada de Agentes IA"]
        AG1["coder.agent.md<br/>Implementa código"]
        AG2["planner.agent.md<br/>Cria planos"]
        AG3["reviewer.agent.md<br/>Revisa qualidade"]
        AG4["orchestrator.agent.md<br/>Coordenação"]
    end

    SC --> LSC
    TM --> SL
    TM --> SM
    TM --> W6
    IT --> W6
    IT --> W1

    LSC --> W1 & W2 & W3 & W4 & W5 & W6 & W8
    AC --> W2
    CI --> W1 & W6
    UPS --> W2 & W3 & W4 & W5
    FWI --> W4
    SL --> W1 & W7
    SM --> W1 & W6
    ATP --> W6 & W8
    IET --> W8

    AG4 -.->|delega| AG1 & AG2 & AG3
```

---

## 3. Ciclo de Vida Completo de uma Issue

Este é o fluxo detalhado desde a criação até o fechamento de cada fase:

```mermaid
flowchart TD
    START(("Início"))

    subgraph FASE1["FASE 1 — Criação da Issue"]
        E1["Evento: issue.opened + trigger label"]
        W2_1["on-issue-opened-start-work.yml"]
        S1["load-squad-config"]
        S2["assign-copilot action<br/>GraphQL addAssigneesToAssignable"]
        S3["Adiciona label AI-AGENT_WIP"]
        S4["update-project-status action<br/>Status: In Progress"]
        S5["Habilita scheduled-wip-monitor.yml"]
    end

    subgraph FASE2["FASE 2 — Copilot Trabalhando"]
        CP1["🤖 Copilot recebe a issue"]
        CP2["Agentes IA trabalham:<br/>coder → planner → reviewer"]
        CP3["Copilot cria Pull Request<br/>com Closes #N no body"]
    end

    subgraph FASE3["FASE 3 — Detecção de Conclusão"]
        direction TB
        E3A["Evento: pull_request.opened<br/>~30 segundos"]
        E3B["Schedule: a cada 15 minutos<br/>fallback"]
        W3_1["on-copilot-pr-transition-to-review.yml"]
        W4_1["scheduled-wip-monitor.yml"]
        D1["Verifica se autor é Copilot"]
        D2["Busca issue vinculada<br/>parse Closes #N ou timeline"]
        D3["Verifica Copilot workflow<br/>completed com sucesso"]
        D4["update-project-status<br/>Status: Review"]
        D5["Troca labels:<br/>remove WIP → adiciona REVIEW"]
        D6["Solicita review do humano"]
    end

    subgraph FASE4["FASE 4 — Review Humano"]
        H_R["👤 Humano revisa o PR"]
        H_A{{"Aprovado?"}}
        H_M["Faz merge do PR"]
    end

    subgraph FASE5["FASE 5 — Fechamento e Próxima Fase"]
        E5["Evento: pull_request.closed<br/>merged = true"]
        W5_1["on-pr-merge-close-issues.yml"]
        CL1["Verifica se autor é Copilot"]
        CL2["Busca issue vinculada"]
        CL3["update-project-status<br/>Status: Done"]
        CL4["Fecha a issue"]
        E6["Evento: issue.closed<br/>trigger label"]
        W6_1["create-next-phase-issues.yml"]
        DEP1["Lê templates-mapping.json<br/>grafo de dependências"]
        DEP2{"Todas as dependências resolvidas?"}
        DEP3["Cria issues das próximas fases"]
        DEP4["Adiciona ao Project via add-to-project"]
    end

    subgraph STUCK["Detecção de Travamento"]
        ST1["scheduled-wip-monitor.yml<br/>verifica a cada 15min"]
        ST2{"WIP há mais de 24 horas?"}
        ST3["Adiciona label stuck<br/>comentário diagnóstico"]
    end

    START --> E1
    E1 --> W2_1 --> S1 --> S2 --> S3 --> S4 --> S5

    S5 --> CP1 --> CP2 --> CP3

    CP3 --> E3A & E3B
    E3A --> W3_1
    E3B --> W4_1
    W3_1 --> D1
    W4_1 --> D1
    D1 --> D2 --> D3 --> D4 --> D5 --> D6

    D6 --> H_R --> H_A
    H_A -->|Sim| H_M
    H_A -->|Não| CP1

    H_M --> E5 --> W5_1 --> CL1 --> CL2 --> CL3 --> CL4
    CL4 --> E6 --> W6_1 --> DEP1 --> DEP2
    DEP2 -->|Sim| DEP3 --> DEP4
    DEP4 -.->|Volta ao início| E1
    DEP2 -->|Não, aguarda outras fases| END2(("Espera"))

    E3B --> ST1 --> ST2
    ST2 -->|Sim| ST3
    ST2 -->|Não| D1
```

---

## 4. Grafo de Dependências das Fases

As fases são definidas em `templates-mapping.json` e seguem um grafo de dependências configurável. Fases paralelas são criadas simultaneamente quando suas dependências são atendidas.

> **Nota:** O diagrama abaixo ilustra o grafo do perfil `modernization` (8 fases). Outros perfis possuem grafos diferentes — `default` (4 fases), `feature` (3 fases), `bugfix` (2 fases). A estrutura é sempre configurável via `templates-mapping.json` do perfil ativo.

```mermaid
flowchart TD
    PA["Fase A"]
    PB["Fase B"]
    PC["Fase C<br/>(paralela com D)"]
    PD["Fase D<br/>(paralela com C)"]
    PE["Fase E"]
    PF["Fase F<br/>(paralela com G)"]
    PG["Fase G<br/>(paralela com F)"]
    PH["Fase H - Final"]

    PA -->|"dependsOn"| PB
    PB -->|"dependsOn (paralelo)"| PC
    PB -->|"dependsOn (paralelo)"| PD
    PD -->|"dependsOn"| PE
    PE -->|"dependsOn (paralelo)"| PF
    PE -->|"dependsOn (paralelo)"| PG
    PF -->|"dependsOn"| PH
    PG -->|"dependsOn"| PH

    style PA fill:#0075ca,color:#fff
    style PB fill:#d4c5f9,color:#000
    style PC fill:#fbca04,color:#000
    style PD fill:#006b75,color:#fff
    style PE fill:#0e8a16,color:#fff
    style PF fill:#1d76db,color:#fff
    style PG fill:#e99695,color:#000
    style PH fill:#c2e0c6,color:#000
```

Cada fase é um objeto em `templates-mapping.json` com:
- **id**: identificador único da fase
- **dependsOn**: lista de fases que devem estar concluídas antes
- **parallelizable / canRunWith**: permite execução simultânea com outra fase
- **template**: arquivo `.md` com o corpo da issue

### Milestones (Opcional)

Perfis podem agrupar fases em **milestones** para organizar entregas no GitHub. Milestones são definidas no `templates-mapping.json` e sincronizadas automaticamente durante o bootstrap e criação de fases.

```mermaid
flowchart LR
        subgraph MS1["Milestone: Foundation"]
                P0["Fase 0<br/>Discovery"]
                P1["Fase 1<br/>UX Design"]
                P2["Fase 2<br/>Security"]
                P3["Fase 3<br/>Gateway"]
        end

        subgraph MS2["Milestone: Build"]
                P4["Fase 4<br/>Backend"]
                P5["Fase 5<br/>Frontend"]
        end

        subgraph MS3["Milestone: Validation"]
                P6["Fase 6<br/>Testing"]
                P7["Fase 7<br/>Release"]
        end

        MS1 --> MS2 --> MS3

        style MS1 fill:#0075ca22,stroke:#0075ca
        style MS2 fill:#1d76db22,stroke:#1d76db
        style MS3 fill:#0e8a1622,stroke:#0e8a16
```

Cada template em `templates-mapping.json` pode incluir um campo `milestone` referenciando o título da milestone:

```json
{
    "milestones": [
        { "title": "Foundation", "description": "Fases 0-3", "state": "open" }
    ],
    "templates": [
        { "id": "discovery", "milestone": "Foundation", "..." }
    ]
}
```

Quando `milestones` não está presente, o framework funciona normalmente sem criar milestones - a feature é **totalmente opcional**.

---

## 5. Máquina de Estados — Labels e Status

Cada issue passa por estados bem definidos, controlados por labels e pelo campo "Status" no GitHub Project:

```mermaid
stateDiagram-v2
    [*] --> Criada: issue.opened + trigger label

    state "Criada" as Criada {
        [*] --> trigger_label: trigger label aplicada
    }

    state "Em Progresso (WIP)" as WIP {
        [*] --> wip_label: label += AI-AGENT_WIP
        wip_label --> status_ip: Status = In Progress
        status_ip --> copilot: Copilot atribuído
    }

    state "Em Revisão (Review)" as Review {
        [*] --> review_label: label WIP→REVIEW
        review_label --> status_review: Status = Review
        status_review --> human: Humano notificado
    }

    state "Concluído (Done)" as Done {
        [*] --> status_done: Status = Done
        status_done --> closed: Issue fechada
    }

    state "Travado (Stuck)" as Stuck {
        [*] --> stuck_label: label += stuck
        stuck_label --> diagnostic: Comentário diagnóstico
    }

    Criada --> WIP: on-issue-opened-start-work.yml
    WIP --> Review: on-copilot-pr-transition-to-review.yml ou scheduled-wip-monitor.yml
    WIP --> Stuck: scheduled-wip-monitor.yml (WIP > 24h)
    Review --> Done: on-pr-merge-close-issues.yml
    Done --> [*]: create-next-phase-issues.yml (cria próxima fase)
    Stuck --> WIP: Intervenção manual
```

---

## 6. Mapa de Arquivos — Referência Rápida

```mermaid
flowchart LR
    subgraph ROOT[".github/"]
        direction TB

        subgraph CFG["Configuração"]
            F1["squad-config.json"]
        end

        subgraph TPL["Templates"]
            F3["templates/default/templates-mapping.json"]
            F4["templates/default/0N-*.md"]
        end

        subgraph ACT["Ações Reutilizáveis"]
            F5["actions/load-squad-config/"]
            F6["actions/assign-copilot/"]
            F7["actions/create-phase-issue/"]
            F8["actions/update-project-status/"]
            F9["actions/find-wip-issues/"]
            F10["actions/sync-labels/"]
            F10b["actions/sync-milestones/"]
            F11["actions/add-to-project/"]
            F20["actions/import-external-task/"]
        end

        subgraph WKF["Workflows"]
            F12["workflows/bootstrap-squad.yml<br/>Manual — setup inicial"]
            F13["workflows/on-issue-opened-start-work.yml<br/>issue.opened — atribui Copilot"]
            F14["workflows/on-copilot-pr-transition-to-review.yml<br/>pull_request — detecta PR"]
            F15["workflows/scheduled-wip-monitor.yml<br/>schedule 15min — fallback"]
            F16["workflows/on-pr-merge-close-issues.yml<br/>PR merged — fecha issue"]
            F17["workflows/create-next-phase-issues.yml<br/>issue.closed — próxima fase"]
            F18["workflows/manual-sync-labels.yml<br/>Manual — sync labels"]
            F21["workflows/import-from-external-board.yml<br/>Manual — importar Jira/Kanbanize"]
            F22b["workflows/export-completion-report.yml<br/>Manual — relatório"]
        end

        subgraph AGT["Agentes IA"]
            F19["agents/coder.agent.md<br/>Implementação"]
            F20["agents/planner.agent.md<br/>Planejamento"]
            F21["agents/reviewer.agent.md<br/>Revisão"]
            F22["agents/orchestrator.agent.md<br/>Orquestração"]
        end
    end

    F1 --> F5
    F5 --> F12 & F13 & F14 & F15 & F16 & F17
    F22 -.-> F19 & F20 & F21
```

---

## 7. Tabela de Eventos e Gatilhos

| Evento GitHub | Workflow Acionado | O que Acontece |
|---|---|---|
| `workflow_dispatch` | `bootstrap-squad.yml` | Setup inicial: cria config, labels, primeira fase |
| `issues.opened` + trigger label | `on-issue-opened-start-work.yml` | Atribui Copilot, WIP label, Status → In Progress |
| `pull_request.opened/sync/ready` | `on-copilot-pr-transition-to-review.yml` | Detecta PR do Copilot, Status → Review (~30s) |
| `schedule: */15` | `scheduled-wip-monitor.yml` | Fallback: monitora WIP, detecta stuck (>24h) |
| `pull_request.closed` (merged) | `on-pr-merge-close-issues.yml` | Status → Done, fecha issue vinculada |
| `issues.closed` + trigger label | `create-next-phase-issues.yml` | Resolve dependências, cria próximas fases |
| `workflow_dispatch` | `manual-sync-labels.yml` | Re-sincroniza labels do repositório |
| `workflow_dispatch` | `import-from-external-board.yml` | Importa tasks do Jira/Kanbanize como GitHub Issues |
| `workflow_dispatch` | `export-completion-report.yml` | Gera relatório CSV/Markdown de tarefas importadas |

---

## 8. Fluxo Completo — Exemplo Prático

> Cenário: Um projeto com 4 fases configuradas (A → B → C+D paralelas)

```
1. 👤 Humano executa "bootstrap-squad.yml" manualmente
   └─ Gera squad-config.json com org, projeto, labels
   └─ Sincroniza labels no repositório
   └─ Cria Issue da Fase A com trigger label + WIP label

2. 🤖 on-issue-opened-start-work.yml dispara (issue.opened)
   └─ Atribui copilot-swe-agent à issue
   └─ Status do Projeto → "In Progress"
   └─ Habilita scheduled-wip-monitor.yml

3. 🤖 Copilot trabalha na tarefa
   └─ Agentes IA: planner → coder → reviewer
   └─ Cria branch, implementa a entrega
   └─ Abre PR com "Closes #N" no body

4. 🤖 on-copilot-pr-transition-to-review.yml dispara (~30 segundos)
   └─ Detecta PR do Copilot vinculado à Issue
   └─ Status → "Review"
   └─ Labels: remove WIP, adiciona REVIEW
   └─ Solicita review do humano

5. 👤 Humano revisa e aprova o PR
   └─ Faz merge

6. 🤖 on-pr-merge-close-issues.yml dispara
   └─ Status → "Done"
   └─ Fecha a Issue

7. 🤖 create-next-phase-issues.yml dispara
   └─ Fase A fechada → dependência da Fase B resolvida
   └─ Cria Issue da Fase B

8. Ciclo repete do passo 2
   └─ Fase B fechada → Fase C + Fase D criadas (paralelo!)
   └─ Fase C + Fase D fechadas → Projeto Completo!
```

---

## 9. Integração com Boards Externos (Jira / Kanbanize)

O HIS suporta a importação de tarefas de boards externos via abordagem **Import & Replicate**: as tasks são copiadas para GitHub Issues e o framework processa automaticamente.

```mermaid
flowchart LR
    subgraph EXTERNO["Board Externo"]
        JI["Jira Cloud<br/>REST API v3"]
        KA["Kanbanize<br/>REST API v2"]
    end

    subgraph IMPORT["Workflow de Importação"]
        W8["import-from-external-board.yml<br/>workflow_dispatch"]
        IET["import-external-task action<br/>Idempotência via source marker"]
    end

    subgraph HIS["Motor HIS (inalterado)"]
        GI["GitHub Issue<br/>com trigger label"]
        CICLO["Ciclo normal:<br/>WIP → Review → Done"]
    end

    subgraph REPORT["Relatório"]
        W9["export-completion-report.yml<br/>CSV + Markdown artifact"]
    end

    JI -->|"fetch tasks"| W8
    KA -->|"fetch cards"| W8
    W8 -->|"cria issues"| GI
    IET -.->|"utility"| W8
    GI -->|"automático"| CICLO
    CICLO -->|"dados"| W9
```

### Fluxo do PO

1. **Admin (uma vez)**: Configura secrets no repositório
   - Jira: `JIRA_EMAIL` + `JIRA_API_TOKEN`
   - Kanbanize: `KANBANIZE_API_KEY`
2. **PO**: Actions → **Import from External Board** → Run workflow
3. **Automático**: Fetch → Idempotência → Criar Issues → HIS processa
4. **PO**: Actions → **Export Completion Report** → Download artifact

### Idempotência

Cada issue importada contém um marcador invisível no body:
```html
<!-- source: jira:PROJ-123 -->
```
O workflow verifica todos os issues existentes antes de importar. Executar múltiplas vezes é seguro — tasks já importadas são ignoradas.

### Issues Importadas vs Issues de Fase

Issues importadas são **independentes** — não disparam encadeamento de fases (`create-next-phase-issues` ignora graciosamente). Seguem um ciclo único: WIP → Review → Done.

---

## 10. Autenticação — HIS_TOKEN (PAT Fine-Grained)

Os workflows do HIS definem `GH_TOKEN` no nível do job (`env: GH_TOKEN: ${{ secrets.HIS_TOKEN }}`). Esse ambiente é herdado por todos os steps, incluindo actions compostas locais e actions compostas aninhadas.

As actions compostas do HIS não recebem mais `token` como input. Steps que usam `gh` CLI consomem `GH_TOKEN` herdado automaticamente. Steps `actions/github-script` usam `github-token: ${{ env.GH_TOKEN }}`.

| Componente | Token | Configuração |
|---|---|---|
| Workflows (nível de job) | `GH_TOKEN` com valor de `secrets.HIS_TOKEN` | Configurar `HIS_TOKEN` uma vez por repositório |
| Actions compostas do HIS (gh CLI) | `GH_TOKEN` herdado do job | Nenhum repasse via `with` |
| Actions compostas / workflows (`actions/github-script`) | `github-token: ${{ env.GH_TOKEN }}` | Usa o token já definido no job |
| Jira import | `secrets.JIRA_EMAIL` + `secrets.JIRA_API_TOKEN` | Uma vez pelo admin |
| Kanbanize import | `secrets.KANBANIZE_API_KEY` | Uma vez pelo admin |

---

## 11. Glossário

| Termo | Significado |
|---|---|
| **Squad Config** | Arquivo JSON central com todas as configurações do projeto |
| **Composite Action** | Ação reutilizável do GitHub Actions (como uma função) |
| **Workflow** | Automação que executa em resposta a eventos do GitHub |
| **Copilot (agente)** | IA que implementa código automaticamente nas issues |
| **WIP** | "Work In Progress" — Copilot está trabalhando |
| **Review** | Copilot terminou, aguardando revisão humana |
| **Done** | PR aprovado e mergeado, fase concluída |
| **Stuck** | Copilot não completou em 24h — precisa de atenção |
| **Phase (Fase)** | Uma etapa do projeto, definida em templates-mapping.json |
| **Milestone** | Agrupamento opcional de fases em marcos de entrega no GitHub, sincronizados automaticamente via `sync-milestones` |
| **Templates Mapping** | Grafo que define a ordem e dependências entre fases |
| **GraphQL Mutation** | Chamada à API do GitHub para modificar dados (projeto, assignees) |
| **Idempotência** | Garantia de não duplicar ações se executadas múltiplas vezes |
| **Import & Replicate** | Abordagem de importar tasks de boards externos (Jira/Kanbanize) para GitHub Issues, sem alterar o motor do HIS |
| **Source Marker** | Comentário HTML invisível (`<!-- source: jira:PROJ-123 -->`) usado para idempotência na importação |
