# Arquitetura

Diagrama geral: `Diagrama/overview.drawio`. Fluxos detalhados: `05-diagramas.md`.

## Visão geral

```
CSVs de origem + Atas 01-07                     SharePoint, biblioteca Entrada
   │                                               │ transcrição nova
   │ carga inicial (Python, admin)                 ▼
   │                                             Power Automate
   │                                               │  Document Intelligence (texto)
   │                                               │  → Azure OpenAI (JSON com esquema fixo)
   │                                               │  → procedure: valida, deduplica, gera IDs, log
   ▼                                               ▼
        Azure SQL: schema dados (bruto) + schema consulta (views com as regras)
   │                                  │
   ▼ importação (só leitura)          ▼ ConsultarBase: só SELECT (só leitura)
Power BI (painel)              Agente Copilot Studio ◄── Atas 01-07 (conhecimento)
                                      │
                                   Gestores
```

## Componentes

| Componente | Responsabilidade | Tecnologia |
| --- | --- | --- |
| Bibliotecas de atas | Entrada (transcrições novas) e Processadas | SharePoint |
| Base de acompanhamento | Projetos, reuniões, tarefas e log de execuções, como vieram da origem | Azure SQL, schema `dados` |
| Camada semântica | Views com as regras de tratamento e os cálculos | Azure SQL, schema `consulta` |
| Catálogo das atas | Texto completo, riscos críticos e tarefas citadas nas Atas 01-07, cruzados com a base | Tabelas `dados.atas` e `dados.atas_acoes`; views `vw_atas` e `vw_riscos_tarefas` |
| Extrator | PDF → texto → JSON de tarefas | Power Automate, Document Intelligence, Azure OpenAI |
| Registro | Validação, deduplicação, IDs sequenciais com bloqueio, gravação e log numa transação | Procedure `dados.registrar_tarefas_ata` |
| Agente | Decide entre a base (cálculos, listas, catálogo) e as atas (conteúdo), ou os dois | Copilot Studio |
| ConsultarBase | Executa o SELECT do agente nas views, com usuário só de leitura | Conector SQL Server |
| Painel | Orçamento x reuniões, tarefas com Atrasado no topo, riscos e indicadores | Power BI |

## Decisões

- **Azure SQL em vez de SharePoint Lists:** as perguntas dos gestores são cálculos (contagem, soma, percentual, ordenação) e cruzamentos; Lists não faz agregação nem junção.
- **SQL escrito pelo agente em vez de um fluxo por pergunta:** fluxos fixos quebrariam com variações das perguntas.
- **Regras de tratamento nas views, não nas tabelas:** os dados de origem ficam intactos e a mesma regra vale para agente, painel e automação. Mudar uma regra é alterar a view; o efeito é imediato para todos.
- **Catálogo estruturado das atas além da busca semântica:** a busca devolve os trechos mais parecidos com a pergunta, o que basta para "o que a ata do Orion diz". Para "considerando todas as atas" é preciso cobertura total e cruzamento exato com a base; por isso riscos e tarefas citadas viram linhas no banco.
- **Leitura determinística das atas históricas:** as 7 atas têm o mesmo layout e são um lote fechado; um parser em Python é reprodutível. Transcrições novas passam pelo fluxo com IA.
- **Lógica de cadastro na procedure, não no fluxo:** validar, deduplicar e gerar IDs em ações do Power Automate exigiria vários loops e não seria atômico. Na procedure fica numa transação, com bloqueio para os IDs. O fluxo cuida de gatilho, IA e avisos.
- **Conhecimento por upload em vez de SharePoint:** permite publicar o agente sem login. Com usuários autenticados, o SharePoint seria a fonte, respeitando a permissão de cada um.
- **Fora do escopo:** Azure Functions, Logic Apps, Data Lake e Dataverse. O volume e o processo não pediram essas peças.

## Segurança

| Identidade | Pode | Não pode |
| --- | --- | --- |
| `agente_leitura` (agente e painel) | SELECT nas views de `consulta` | Ler as tabelas brutas, gravar ou apagar |
| `fluxo_escrita` (automação) | Executar a procedure; ler e inserir em `dados`; ler as views | Apagar dados |
| Admin | Carga inicial e manutenção | — |

- O agente é público, mas só lê views com dados fictícios. Em uso interno, autenticação Microsoft e canal Teams.
- Chaves de API ficam em variáveis de ambiente da solução Power Platform; as ações HTTP usam Secure Inputs, então as chaves não aparecem no histórico.
- A automação é idempotente: reprocessar uma ata não duplica tarefas, inclusive com execuções simultâneas.
