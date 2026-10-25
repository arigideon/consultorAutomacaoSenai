# Diagramas da solução

Quatro visões do funcionamento, em Mermaid.

## 1. Automação do cadastro de tarefas

Uma transcrição nova cai na biblioteca Entrada; o fluxo extrai o texto e as tarefas e entrega o JSON para a procedure, que valida, evita duplicatas, gera os IDs e grava.

### 1a. Caminho entre os sistemas

```mermaid
sequenceDiagram
    autonumber
    actor G as Gestor
    participant SP as SharePoint<br/>Entrada e Processadas
    participant PA as Power Automate<br/>fluxo de cadastro
    participant DI as Document Intelligence<br/>prebuilt-read
    participant AO as Azure OpenAI<br/>gpt-4.1-mini
    participant DB as Azure SQL<br/>fluxo_escrita
    participant T as Teams

    G->>SP: Sobe o PDF da transcrição em Entrada
    SP->>PA: Gatilho When a file is created (concorrência 1)
    PA->>SP: Obter arquivo
    SP-->>PA: Conteúdo em base64
    PA->>DI: POST analyze com o PDF
    DI-->>PA: 202 + Operation-Location
    loop Aguardar DI, a cada 2 s, até 2 min
        PA->>DI: GET Operation-Location
        DI-->>PA: status (running ou succeeded)
    end
    alt status diferente de succeeded
        PA->>T: Falha na leitura do PDF
        PA->>PA: Terminate Failed
    else succeeded
        PA->>DB: Listar projetos (SELECT em consulta.vw_projetos)
        DB-->>PA: Id_Projeto - Nome
        PA->>AO: Prompt + projetos + texto, com JSON Schema estrito
        AO-->>PA: JSON com reunião, projeto, data, participantes e tarefas
        PA->>DB: EXEC dados.registrar_tarefas_ata(Arquivo, Json)
        DB-->>PA: Uma linha por tarefa: inserida, ja existia ou rejeitada
        PA->>T: Tabela com o resultado
        PA->>SP: Move o PDF para Processadas
    end
    Note over PA,T: Se qualquer ação falhar, Avisar erro no Teams e Terminate Failed.
```

### 1b. O que a procedure decide

```mermaid
flowchart TD
    IN(["JSON do modelo + nome do arquivo"]) --> CABD["Confere o cabeçalho<br/>JSON válido, REU-NNN, projeto existe,<br/>data AAAA-MM-DD, tem tarefas"]
    CABD --> CAB{"Cabeçalho<br/>ok?"}
    CAB -- "não" --> ERRO["Linha única: erro + motivo"]
    CAB -- "sim" --> VALD["Confere cada tarefa<br/>descrição, responsável e prazo preenchidos,<br/>prazo depois da reunião, prioridade Alta, Média ou Baixa,<br/>responsável entre os participantes, sem repetir no arquivo"]
    VALD --> VAL{"Tarefa<br/>ok?"}
    VAL -- "não" --> REJ["rejeitada + motivo"]
    VAL -- "sim" --> DUPD["Busca na base<br/>mesma reunião + responsável + prazo"]
    DUPD --> DUP{"Já<br/>existe?"}
    DUP -- "sim" --> EXI["ja existia + Id_Tarefa"]
    DUP -- "não" --> ID["Próximo TAR-NNN<br/>com bloqueio UPDLOCK"]
    ID --> INS[("INSERT em dados.tarefas<br/>Origem automacao, Em Andamento,<br/>Data_Criacao = data da reunião")]
    ERRO & REJ & EXI & INS --> LOG[("dados.log_execucoes<br/>sucesso, ignorado ou erro")]
    LOG --> OUT(["Result set para o fluxo"])
```

Rodar o mesmo PDF de novo devolve as tarefas como "ja existia": a deduplicação está na procedure e num índice único filtrado (`Origem = 'automacao'`).

## 2. O agente fazendo uma consulta

O agente decide, a cada pergunta, se consulta a base (cálculo, listas, riscos x tarefas), o texto das atas ou os dois. A base só é acessada por SELECT nas views, com um usuário que só tem leitura.

```mermaid
sequenceDiagram
    autonumber
    actor G as Gestor
    participant A as Agente Copilot Studio
    participant K as Knowledge<br/>Atas 01 a 07 enviadas
    participant C as ConsultarBase<br/>conector SQL Server<br/>Shared account agente_leitura
    participant V as Azure SQL<br/>views do schema consulta

    G->>A: Pergunta em português
    A->>A: Escolhe a fonte pelas instruções

    alt Cálculo, lista ou riscos x tarefas
        A->>C: query = um SELECT nas views
        C->>V: Executa com agente_leitura
        alt Consulta válida
            V-->>C: Linhas
            C-->>A: JSON com as linhas
        else Erro de SQL
            V-->>C: Mensagem de erro
            C-->>A: Erro
            A->>C: SELECT corrigido
            C->>V: Executa de novo
            V-->>C: Linhas
            C-->>A: JSON com as linhas
        end
    else Conteúdo das atas
        A->>K: Busca semântica nas atas
        K-->>A: Trechos relevantes
        opt Precisa do texto integral
            A->>C: SELECT Texto FROM consulta.vw_atas
            C->>V: Executa
            V-->>C: Texto das atas
            C-->>A: JSON
        end
    end

    A-->>G: Resposta direta, detalhes e fonte (view ou ata)

    Note over C,V: agente_leitura só tem SELECT no schema consulta.<br/>Escrita e tabelas brutas são negadas pelo banco.
```

Exemplos de consulta que o agente gera:

| Pergunta | View consultada |
| --- | --- |
| 1. Top 3 de Desenvolvimento e IA com mais atrasadas | `vw_resumo_tarefas_projeto` |
| 2. Custo de reuniões do Alpha e % do orçamento | `vw_custo_projeto` |
| 3. Tarefas Alta da Beatriz por prazo | `vw_tarefas` |
| 4. Motivo técnico e fornecedor no Orion | Knowledge (Ata_01) ou `vw_atas` |
| 5. Riscos críticos concluídos e pendentes | `vw_riscos_tarefas` |

## 3. Painel de acompanhamento

O painel importa as mesmas views que o agente usa: os números batem entre os dois.

```mermaid
flowchart LR
    subgraph SQL["Azure SQL: schema consulta"]
        VP["vw_projetos"]
        VT["vw_tarefas<br/>Ordem_Painel, Dias_Restantes"]
        VR["vw_reunioes<br/>sem duplicadas"]
        VRT["vw_riscos_tarefas"]
    end

    subgraph PBI["Power BI Desktop"]
        PQ["Power Query<br/>Importar, usuário agente_leitura"]
        subgraph MOD["Modelo estrela"]
            DIM["vw_projetos<br/>dimensão"]
            FT["vw_tarefas<br/>+ 19 medidas DAX"]
            FR["vw_reunioes"]
            FRT["vw_riscos_tarefas"]
        end
        P1["Página Orçamento x reuniões<br/>orçamento, custo, horas, %"]
        P2["Página Tarefas<br/>Atrasado no topo, depois dias restantes"]
        P3["Página Riscos das atas<br/>concluídos x pendentes"]
    end

    PBIX(["painel-orion-tech.pbix<br/>dados importados"])

    VP & VT & VR & VRT --> PQ
    PQ --> DIM & FT & FR & FRT
    DIM -- "1:N por Id_Projeto" --> FT
    DIM -- "1:N por Id_Projeto" --> FR
    DIM -- "1:N por Id_Projeto" --> FRT
    DIM & FT & FR --> P1
    DIM & FT --> P2
    FRT & FT --> P3
    P1 & P2 & P3 -- "Arquivo > Salvar como" --> PBIX
```

Não existe relação por `Id_Reuniao`: na base, ele aponta para reuniões de outro projeto em 79 das 115 tarefas (armadilha 7).
Depois de processar uma transcrição nova, **Página Inicial > Atualizar** traz as tarefas novas.

## 4. Como as tabelas são alimentadas e como as views funcionam

As tabelas de `dados` guardam o dado como veio (só com conversão de formato). As views de `consulta` não guardam dados: são consultas salvas, calculadas a cada leitura, com as regras de limpeza.

```mermaid
flowchart LR
    subgraph ORIG["Origens"]
        CSV["CSVs de origem<br/>projetos, reuniões, tarefas"]
        PDF["Atas 01 a 07 em PDF"]
        NOVA["Ata nova na<br/>biblioteca Entrada"]
    end

    subgraph CARGA["Carga inicial em Python (admin)"]
        LB["Leitura dos CSVs<br/>datas ISO, vírgula decimal"]
        LA["Leitura das atas<br/>pypdf + parser de riscos e ações"]
        S2["Criação das tabelas"]
        S3["Criação das views"]
        S5["Procedure + permissão"]
    end

    S1["Segurança<br/>uma vez: schemas e usuários"]
    FLUXO["Fluxo de cadastro<br/>EXEC registrar_tarefas_ata"]

    subgraph DADOS["Schema dados: tabelas brutas"]
        TP[("projetos")]
        TR[("reunioes<br/>com duplicadas")]
        TT[("tarefas")]
        TA[("atas")]
        TAA[("atas_acoes")]
        TL[("log_execucoes")]
    end

    subgraph CONS["Schema consulta: views"]
        WP["vw_projetos<br/>R1 trim da categoria, R6 sem acento"]
        WR["vw_reunioes<br/>R2 DISTINCT"]
        WT["vw_tarefas<br/>R3 só projetos válidos, R4 status pelo prazo,<br/>R5 hoje em Brasília, Ordem_Painel"]
        WC["vw_custo_projeto"]
        WS["vw_resumo_tarefas_projeto"]
        WA["vw_atas"]
        WRT["vw_riscos_tarefas<br/>R7 confere ID + projeto + responsável,<br/>R8 risco concluído"]
    end

    subgraph USO["Quem lê"]
        AG["Agente<br/>agente_leitura"]
        BI["Power BI<br/>agente_leitura"]
        FL["Fluxo: Listar projetos<br/>fluxo_escrita"]
    end

    CSV --> LB
    PDF --> LA
    LB -- "uma vez" --> TP & TR & TT
    LA -- "a cada carga" --> TA & TAA
    S2 -. "cria" .-> DADOS
    S3 -. "cria ou altera" .-> CONS
    S1 -. "antes de tudo" .-> DADOS
    S5 -. "cria a procedure" .-> FLUXO
    NOVA --> FLUXO
    FLUXO -- "a cada ata nova" --> TT
    FLUXO --> TL

    TP --> WP
    TR --> WR
    TT --> WT
    WP --> WT
    WP & WR --> WC
    WP & WT --> WS
    TA --> WA
    TA & TAA & TT --> WRT

    CONS --> AG & BI
    WP --> FL
```

| Tabela | Quem alimenta | Quando |
| --- | --- | --- |
| `dados.projetos`, `dados.reunioes` | Carga dos CSVs | Uma vez |
| `dados.tarefas` | Carga dos CSVs + fluxo de cadastro | Uma vez + a cada ata nova |
| `dados.atas`, `dados.atas_acoes` | Carga lendo as Atas 01 a 07 | A cada execução da carga |
| `dados.log_execucoes` | Carga e procedure | A cada execução |
