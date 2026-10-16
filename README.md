# Desafio Motiva — Escadas rolantes (Dynamic Programming)

**Aluno:** Bruno Santos Castilho — RM 566799

Dataset sintético dos sensores das escadas rolantes da ViaQuatro (Motiva, Linha 4-Amarela) usado em três atividades: buscas, percurso na linha e plano de turno. Tudo em Python puro (`csv`, `random`, `datetime`), com busca linear e binária feitas à mão e fila/pilha com listas (`append`, `pop(0)`, `pop()`). Sem grafos, sem `deque`, sem bibliotecas de busca.

## Arquivos

| Arquivo | Conteúdo |
|---|---|
| `notebook.ipynb` | Código completo, já executado |
| `sensores.csv` | Dataset sintético (14.400 leituras) |
| `linha.csv` | Estações da Linha 4-Amarela em ordem, com minutos até a seguinte |
| `tempos_base.csv` | Estação, minutos de ida e minutos de ida e volta a partir da base |
| `comparacoes_buscas.csv` | Resumo das comparações (linear × binária) |
| `alertas_5_por_escada.csv` | Leitura do 5º alerta acumulado de cada escada e comparações de cada busca |
| `plano_turnos.csv` | Agenda e pendentes dos turnos de 4, 6 e 8 h, normal e com perda de 1 h |

Para reproduzir: abrir o `notebook.ipynb` e executar todas as células (Python 3, sem dependências externas).

## Regras da geração (Etapa 0)

* Função: `gerar_dataset(n_escadas, n_leituras, semente)`, com `random.seed(semente)`.
* **Semente usada: `42`** — 60 escadas × 240 leituras (uma por hora, de 01/09/2026 00:00 a 10/09/2026 23:00).
* Colunas: `id_escada, estacao, grupo, data_hora, vibracao, temperatura, corrente, tensao, nivel_oleo, velocidade`.
* Estação sorteada entre as 11 da Linha 4; grupo `crítica` com probabilidade de 30%, senão `não crítica`.
* Leituras de cada escada em ordem de horário; a ordem das escadas no arquivo é embaralhada.
* Faixas normais: vibração 1–4 mm/s · temperatura 35–65 °C · corrente 10–25 A · tensão 361–399 V · óleo 60–100 % · velocidade 0,48–0,52 m/s. Uma leitura está **em alerta** quando algum sensor sai da faixa.
* Perfis (o perfil não vai para o CSV, pois não está entre as colunas pedidas):
  * **Saudável (60%)** — ruído gaussiano em torno do centro da faixa, sempre dentro dela.
  * **Desgaste gradual (30%)** — a partir de um ponto sorteado entre 30% e 60% da série, vibração (+3 a +5), temperatura (+12 a +20), corrente (+4 a +8) sobem e o óleo cai (−20 a −35) de forma linear até o fim.
  * **Falhas pontuais (10%)** — leituras normais com 4 a 8 picos isolados, cada um em um sensor sorteado.

### Resumo do dataset

| Sensor | Mínimo | Média | Máximo |
|---|---|---|---|
| vibração (mm/s) | 1,07 | 2,84 | 8,49 |
| temperatura (°C) | 35,8 | 51,21 | 81,4 |
| corrente (A) | 10,4 | 18,01 | 32,8 |
| tensão (V) | 330,8 | 380,05 | 418,5 |
| nível de óleo (%) | 35,8 | 77,67 | 99,0 |
| velocidade (m/s) | 0,449 | 0,500 | 0,570 |

| Perfil | Escadas | Leituras em alerta |
|---|---|---|
| Saudável | 36 | 0 |
| Desgaste gradual | 18 | 1.499 |
| Falhas pontuais | 6 | 39 |

As médias ficam perto do centro das faixas porque 60% das escadas são saudáveis; os mínimos e máximos fora da faixa vêm dos perfis de desgaste e de falha.

## Atividade 1 — Busca linear × busca binária

Só duas funções, `busca_linear` e `busca_binaria`, usadas nos dois testes. Contagem: cada teste entre o alvo e um elemento conta 1 comparação (na binária, até 2 por passo: `==` e `<`).

Para o 5º alerta, cada escada vira a lista de alertas acumulados (0, 0, 1, 1, 2, ...), que nunca diminui e cresce de 1 em 1; a primeira posição com valor 5 é a leitura procurada. A binária usa `primeira=True` para continuar à esquerda até a primeira ocorrência.

| Teste | Busca | Total | Média | Mín. | Máx. |
|---|---|---|---|---|---|
| 1.000 consultas por ID (60 escadas) | linear (ordem do arquivo) | 30.649 | 30,65 | 1 | 60 |
| | binária (lista ordenada) | 8.950 | 8,95 | 1 | 11 |
| 1ª leitura com 5 alertas acumulados (60 escadas × 240 leituras) | linear | 12.568 | 209,47 | 106 | 240 |
| | binária | 915 | 15,25 | 10 | 16 |

**Interpretação.** Na busca por ID, a linear faz em média ~n/2 comparações (30,65 para n = 60), enquanto a binária fica limitada a ~2·log₂(60) ≈ 11: cerca de 3,4× menos trabalho, e a diferença cresce com o número de escadas. No 5º alerta a vantagem é muito maior (~13,7×): a lista de alertas acumulados nunca diminui, então já está ordenada e a binária resolve em ~8 passos (10 a 16 comparações), enquanto a linear precisa varrer até o 5º alerta — e, nas 37 escadas que nunca chegam a 5, percorre as 240 leituras inteiras. A binária só vale porque existe ordem (IDs ordenados ou acumulado crescente); na lista do arquivo, embaralhada, só a linear funciona.

23 das 60 escadas chegam a 5 alertas: as 18 de desgaste (entre 06/09 e 08/09, conforme o desgaste avança) e 5 das 6 com falhas pontuais (uma teve só 4 picos).

## Atividade 2 — Percurso na linha com pilha

`linha.csv` tem as 11 estações da Linha 4-Amarela, de **Vila Sônia (base, 1ª posição)** até Luz. Os minutos entre estações são estimativas de percurso (2 a 3 min), totalizando 21 min até Luz.

* `tempo_ate(linha, estacao)` — busca linear pela estação somando os minutos desde a base.
* `rota_ida_e_volta(linha, estacao)` — na ida empilha as estações com `append`; na volta desempilha com `pop()`, o que naturalmente inverte o caminho (a pilha devolve primeiro a última estação visitada).

Ida e volta vai de 0 min (a própria base) a 42 min (Luz). Como o deslocamento é curto perto do tempo de serviço (40–60 min), a localização pesa menos que o tipo de escada no custo de cada atendimento.

## Atividade 3 — Plano do turno com fila e pilha

* Escada **em alerta** = tem ao menos uma leitura em alerta (24 escadas: 4 críticas, 20 não críticas). O primeiro alerta é achado com `busca_binaria(acumulados, 1, primeira=True)`.
* `fila_critica` e `fila_normal` na ordem do primeiro alerta; crítica atendida antes, retirando com `pop(0)`.
* Custo = ida + serviço (60 min crítica, 40 min normal) + volta. Cabe → `append` na agenda; não cabe → pendentes (a próxima da fila continua sendo testada).
* Perda de 1 hora → `planejar_turno(base, horas, perdeu_1h=True)` faz `pop()` na agenda até caber; as escadas retiradas voltam para o início dos pendentes na ordem original.
* Turno começa às 08:00.

| Turno | Atendidas | Minutos usados | Pendentes | Com 1 h a menos (desfeitas com `pop()`) |
|---|---|---|---|---|
| 4 h | 3 críticas | 228 / 240 | 21 | ESC-023 desfeita → 2 atendidas, 146 / 180 min |
| 6 h | 4 críticas + 1 normal | 350 / 360 | 19 | ESC-043 desfeita → 4 atendidas, 292 / 300 min |
| 8 h | 4 críticas + 3 normais | 468 / 480 | 17 | ESC-040 e ESC-052 desfeitas → 5 atendidas, 350 / 420 min |

A agenda completa (escada, estação, rota, início e fim) de cada cenário está no notebook e em `plano_turnos.csv`.

**Interpretação.**
* A prioridade funciona: as escadas críticas sempre entram primeiro, e só a partir do turno de 6 h sobra tempo para a fila normal. No turno de 4 h, até uma crítica (ESC-033) fica pendente.
* Um único turno não dá conta das 24 escadas em alerta (o de 8 h atende 7). Com a demanda deste dataset, seriam necessários vários turnos ou mais equipes; os pendentes viram a fila do turno seguinte.
* No turno de 8 h, ESC-007 (Paulista, 70 min) não coube, mas ESC-040 (São Paulo-Morumbi, 44 min), que vinha depois, coube — testar cada escada em vez de parar na primeira que não cabe aproveita melhor o tempo.
* Desfazer com `pop()` remove sempre o último atendimento agendado, que é o de menor prioridade: a ordem de prioridade é preservada mesmo quando o turno encolhe. Por outro lado, o desfazer não tenta encaixar de novo uma escada menor no tempo que sobra (no 8 h com perda, ficam 70 min livres), porque a regra pede apenas `pop()` até caber.
