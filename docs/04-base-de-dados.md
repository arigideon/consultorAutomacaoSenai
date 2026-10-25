# Base de dados

Azure SQL com dois schemas: `dados` guarda o dado como veio da origem; `consulta` expõe views com as regras de tratamento. Agente e painel só leem `consulta`.

## Tabelas (`dados`)

| Tabela | Conteúdo | Quem alimenta | Quando |
| --- | --- | --- | --- |
| `projetos` | Cadastro de projetos | Carga inicial dos CSVs | Uma vez |
| `reunioes` | Histórico de reuniões, com as duplicadas da origem | Carga inicial dos CSVs | Uma vez |
| `tarefas` | Controle de tarefas, com a origem de cada linha (carga ou automação) | Carga inicial + automação | Uma vez + a cada transcrição nova |
| `atas` | Texto completo das Atas 01-07 | Carga, lendo os PDFs | Na carga |
| `atas_acoes` | Riscos críticos e tarefas citadas em cada ata | Carga, lendo os PDFs | Na carga |
| `log_execucoes` | Registro de cada carga e de cada transcrição processada | Carga e procedure | A cada execução |

Na carga há só conversão de formato (datas e vírgula decimal). Um índice único impede que a automação grave a mesma tarefa duas vezes (mesma reunião, responsável e prazo).

## Views (`consulta`)

| View | Uso |
| --- | --- |
| `vw_projetos` | Projetos com categoria limpa |
| `vw_reunioes` | Reuniões sem duplicadas |
| `vw_tarefas` | Tarefas de projetos válidos, status oficial e pelo prazo, dias restantes e ordem do painel |
| `vw_custo_projeto` | Orçamento x reuniões por projeto: quantidade, horas, custo e % do orçamento |
| `vw_resumo_tarefas_projeto` | Tarefas por projeto e status |
| `vw_atas` | Texto completo das atas |
| `vw_riscos_tarefas` | Riscos críticos das atas x base de tarefas |
| `vw_proximo_id_tarefa` | Próximo ID livre (informativo) |

Regras aplicadas:

| Regra | O que faz |
| --- | --- |
| R1 | Categoria sem espaços nas pontas |
| R2 | Reuniões duplicadas contam uma vez |
| R3 | Tarefas de projetos inexistentes ficam de fora |
| R4 | Status da base é o oficial; status pelo prazo é calculado para comparação |
| R5 | "Hoje" é a data de Brasília |
| R6 | Textos comparados sem diferenciar acento e maiúsculas |
| R7 | Tarefa citada na ata só confere com a base se ID, projeto e responsável batem |
| R8 | Risco concluído = todas as tarefas do risco concluídas (nota da ata ou base) |

## Procedure da automação

`dados.registrar_tarefas_ata(@Arquivo, @Json)` recebe o JSON extraído da transcrição e devolve uma linha por tarefa com a situação (inserida, já existia ou rejeitada) e o motivo:

1. Confere o cabeçalho: JSON válido, reunião no padrão REU-NNN, projeto cadastrado, data válida, ao menos uma tarefa.
2. Confere cada tarefa: campos preenchidos, prazo depois da reunião, prioridade Alta/Média/Baixa, responsável entre os participantes, sem repetição no arquivo.
3. Marca como "já existia" o que já está na base (mesma reunião, responsável e prazo).
4. Gera os IDs em sequência com bloqueio, grava com origem "automação" e status "Em Andamento", e registra no log.

## Como os dados se atualizam

As views não guardam dados: são consultas salvas, calculadas a cada leitura. Não há trigger nem reprocessamento.

- Uma transcrição nova grava tarefas em `dados.tarefas`; a próxima pergunta ao agente já as enxerga.
- O painel importa os dados: tarefas novas aparecem depois de **Atualizar** no Power BI.
- O catálogo das atas (`atas` e `atas_acoes`) vem das Atas 01-07; uma transcrição nova gera apenas tarefas.
