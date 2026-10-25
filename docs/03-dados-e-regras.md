# Dados e regras de tratamento

Volumes de origem: 12 projetos, 123 linhas de reuniões (120 únicas), 119 tarefas (32 Atrasado, 27 Em Andamento, 60 Concluído).
Depois das regras: 115 tarefas válidas. Os dados de origem não são alterados; o tratamento fica nas views.

## Problemas encontrados

| # | Problema | Onde | Efeito se ignorado | Tratamento |
| --- | --- | --- | --- | --- |
| 1 | Categoria com espaço no fim ("Desenvolvimento e IA ") | Vórtex e Sideral | Vórtex some do top 3 de atrasadas em Desenvolvimento e IA | Espaços removidos na view de projetos |
| 2 | Reuniões duplicadas (linha idêntica) | REU-014, REU-029, REU-052 | Alpha dá R$ 20.050 (11 reuniões) em vez de R$ 18.400 (10) | Duplicadas contam uma vez na view de reuniões |
| 3 | Custo com vírgula decimal ("1150,00") | 6 reuniões | Valor lido como texto | Conversão de formato na carga |
| 4 | Tarefas de projetos inexistentes | PRJ-998 e PRJ-999 (4 tarefas) | Contagens infladas | Só entram tarefas de projetos cadastrados |
| 5 | IDs de tarefa das Atas 04-07 não batem com a base | Atas citam TAR-008 a 011 com outro conteúdo | Risco cruzado com a tarefa errada | Tarefa da ata só "confere" com o mesmo ID, projeto e responsável |
| 6 | Status desatualizado | Tarefas "(Pendente de atualização)" e "Em Andamento" com prazo vencido | Status diverge do prazo | Status da base é o oficial; um status calculado pelo prazo fica ao lado, para comparação |
| 7 | Reunião da tarefa aponta para outro projeto | 79 das 115 tarefas válidas | Juntar tarefas e reuniões atribui tarefas ao projeto errado | O projeto da tarefa vem sempre da própria tarefa; nada junta tarefas e reuniões |
| 8 | Reuniões das Atas 04-07 ausentes ou trocadas na base | REU-108 é do Quantum na base; REU-142, 175 e 210 não existem | Dados da ata "confirmados" com a reunião errada | Custo e horas vêm só do histórico de reuniões; a ata é fonte de conteúdo |
| 9 | Texto com acento | "Vórtex", "Concluído", "Média" | "Vortex" ou "Concluido" digitados sem acento não encontram nada | Comparações nas views ignoram acento e maiúsculas |

Outras observações:

- A reunião da Ata_08 (REU-241) não existe no histórico; as tarefas apontam para ela mesmo assim.
- A Ata_08 traz uma responsável nova (Larissa Moura): a validação aceita qualquer participante da reunião, não só quem já está na base.

## Data de referência

Os dados foram montados por volta de 13/10/2026 (data da Ata_08). As views usam a data atual de Brasília para dias restantes e status pelo prazo; as contagens de atrasadas usam o status oficial da base, que não depende da data.
Com o passar dos dias, tarefas "Em Andamento" ficam vencidas pelo prazo: o painel mostra essas divergências em vez de alterar o status.

## Riscos das atas x base de tarefas

| Risco | Projeto | Tarefas citadas na ata | Situação na base | Classificação |
| --- | --- | --- | --- | --- |
| 1. Gargalo de API e migração de conector | Orion | TAR-001, TAR-004 | Conferem; ambas Atrasado | Pendente |
| 2. Falta de anonimização LGPD | Vórtex | TAR-002, TAR-005 | Conferem; TAR-002 Em Andamento, TAR-005 Atrasado | Pendente |
| 3. Atraso de credenciais do banco | Alpha | TAR-003, TAR-007 | Conferem; TAR-003 Atrasado, TAR-007 Em Andamento | Pendente |
| 4. Prompt injection no chatbot | Nexus IA | TAR-008, TAR-006 | TAR-006 confere e está Em Andamento. TAR-008 na base é outra tarefa (Sideral): a trava de guardrails não está cadastrada | Pendente |
| 5. Instabilidade no broker MQTT | Pulso | TAR-009 | A ata registra conclusão em 12/10; TAR-009 na base é outra tarefa (Prisma) | Concluído (pela ata) |
| 6. Memory leak no Kubernetes | Zenith | TAR-010 | A ata não registra conclusão; TAR-010 na base é outra tarefa (Orion) | Pendente (sem registro na base) |
| 7. Divergência no simulador | Quantum | TAR-011 | A ata registra conclusão em 13/10; TAR-011 na base é outra tarefa (Orion) | Concluído (pela ata) |

Regras:

- Uma tarefa citada na ata "confere" com a base quando o mesmo ID tem o mesmo projeto e responsável.
- Um risco está concluído quando todas as suas tarefas estão concluídas, pela nota da ata ou pelo status da base.

Cruzar só pelo ID levaria a respostas erradas, como "TAR-008 concluída, risco 4 resolvido" ou "riscos 5 e 7 pendentes".
