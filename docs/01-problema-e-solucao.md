# Problema e solução

A Orion Tech é uma consultoria que conduz vários projetos ao mesmo tempo. Dois problemas tomavam tempo da equipe:

1. **Cadastro manual de tarefas.** Ao fim de cada reunião, alguém lia a ata ou a transcrição e lançava na planilha as tarefas combinadas.
2. **Informação espalhada.** Decisões ficavam em atas em PDF; andamento das tarefas e custos, em planilhas separadas. Uma pergunta simples de um gestor exigia procurar em vários arquivos.

Bases usadas: cadastro de projetos (12), histórico de reuniões (123 linhas), controle de tarefas (119), 7 atas de reuniões passadas e uma transcrição nova (Ata_08, Projeto Sideral) ainda não lançada.

## 1. Automação do cadastro de tarefas

**Critério:** receber a transcrição de uma reunião, identificar as tarefas combinadas (responsável, prazo, prioridade e projeto) e cadastrá-las na base de tarefas, sem digitação.

**Como funciona**

1. A transcrição em PDF é colocada na biblioteca Entrada do SharePoint, o que dispara um fluxo do Power Automate.
2. O Azure AI Document Intelligence extrai o texto do PDF.
3. O Azure OpenAI (gpt-4.1-mini) devolve as tarefas num JSON com esquema fixo: reunião, projeto, data, participantes e, para cada tarefa, descrição, responsável, prazo e prioridade.
4. Uma procedure no Azure SQL recebe o JSON, valida, evita duplicatas, gera os IDs (TAR-NNN) e grava, numa única transação.
5. O resultado chega no Teams em forma de tabela e o PDF vai para a biblioteca Processadas.

**Garantias**

| Situação | Comportamento |
| --- | --- |
| Projeto inexistente, data inválida ou nenhuma tarefa | O arquivo inteiro é recusado, com o motivo |
| Prazo antes da reunião, prioridade fora de Alta/Média/Baixa ou responsável que não participou da reunião | Só aquela tarefa é recusada, com o motivo |
| Mesma ata processada de novo | Tarefas reconhecidas como "já existia"; nada é duplicado |
| Dois arquivos ao mesmo tempo | IDs gerados com bloqueio; um índice único impede duplicata |
| Falha em qualquer etapa | Aviso no Teams e execução marcada como falha |

Um responsável novo é aceito desde que esteja entre os participantes (Larissa Moura, da Ata_08, não existia na base).
Cada execução fica registrada num log no banco.

**Resultado com a Ata_08** (reunião REU-241, projeto PRJ-005):

| Id | Responsável | Tarefa | Prazo | Prioridade |
| --- | --- | --- | --- | --- |
| TAR-120 | Beatriz Lima | Executar fine-tuning do modelo MobileNetV2 com lote de imagens térmicas | 19/10/2026 | Alta |
| TAR-121 | Diego Martins | Configurar container Docker de inferência no dispositivo Edge da linha de produção | 20/10/2026 | Alta |
| TAR-122 | Larissa Moura | Elaborar caderno de testes de aceitação do usuário (UAT) com P&D Industrial | 23/10/2026 | Média |

## 2. Agente de consulta

**Critério:** responder dúvidas dos gestores consultando tanto as planilhas (projetos, reuniões e tarefas) quanto o texto das atas.

**Como funciona**

- Agente no Copilot Studio com duas fontes:
  - **ConsultarBase**: conector SQL Server que executa um SELECT escrito pelo agente nas views do banco. Usado para contagens, somas, percentuais, filtros e listas.
  - **Atas** (Ata_01 a Ata_07) como conhecimento, para decisões, motivos técnicos e riscos. O texto completo das atas também está numa view.
- Busca semântica sozinha não soma nem conta, e SQL sozinho não lê PDF: por isso as duas fontes.
- Perguntas que cruzam atas e tarefas usam um catálogo dos riscos das atas já cruzado com a base, em vez de depender de a busca encontrar todas as atas.
- O usuário de banco do agente só tem leitura nas views: não altera dados e não enxerga as tabelas brutas.

**Perguntas de referência**

| Pergunta | Resposta | Fonte |
| --- | --- | --- |
| Quais os 3 projetos de "Desenvolvimento e IA" com mais tarefas atrasadas e quem são os líderes? | Orion (Lucas Mendes) 11; Vórtex (Camila Rocha) 8; Nexus IA (Rafael Souza) 5 | Base |
| Qual o custo acumulado em reuniões do projeto Alpha e quanto isso representa do orçamento? | 10 reuniões, R$ 18.400,00, 14,72% de R$ 125.000,00 | Base (bate com a Ata_03) |
| Quais as tarefas de prioridade Alta da Beatriz Lima, por prazo? | TAR-004 (06/10), TAR-005 (09/10), TAR-006 (15/10), TAR-007 (25/10); depois da automação, TAR-120 (19/10) entre TAR-006 e TAR-007 | Base |
| Segundo a ata do Orion, qual o motivo técnico da mudança na homologação e o que foi decidido sobre o fornecedor? | Rate limit da API legada (bloqueio após 120 req/min) e latência média acima de 8,5 s; migração para a instância dedicada CloudCore, sem custo adicional no trimestre | Ata_01 |
| Quais riscos críticos das atas já tiveram as tarefas concluídas e quais seguem pendentes? | Concluídos: 5 (Pulso) e 7 (Quantum). Pendentes: 1, 2, 3, 4 e 6 | Atas + base |

Detalhe da última pergunta e das divergências entre atas e base: `03-dados-e-regras.md`.

## 3. Painel de acompanhamento

**Critérios:** orçamento previsto x tempo e custo de reuniões; lista de tarefas com Atrasado no topo e as demais pelos dias restantes até o prazo; outros indicadores úteis para a gestão.

| Página | O que mostra |
| --- | --- |
| Orçamento x reuniões | Orçamento, custo e horas de reunião por projeto e % do orçamento gasto em reuniões (Horizon é o maior, 16,29%; Alpha, 14,72%) |
| Tarefas | Atrasado no topo, depois por dias restantes, com cor por status; tarefas "Em Andamento" com prazo vencido; taxa de conclusão no prazo; tarefas criadas pela automação |
| Riscos das atas | Riscos críticos concluídos x pendentes e as tarefas citadas nas atas que não batem com a base |

O painel lê as mesmas views que o agente: os números são os mesmos nos dois.
O arquivo em `powerbi/` traz os dados importados e abre sem acesso ao banco.
