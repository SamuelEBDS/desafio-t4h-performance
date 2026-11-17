# Experimentos

## 01 — Índice dos itens por pedido

**Hipótese.** Indexar `itens_pedido(pedido_id)` deve reduzir a latência de `GET /clientes/:id/pedidos`, que busca os itens de cada pedido do cliente. O índice está em [03-indice-itens-pedido.sql](banco/03-indice-itens-pedido.sql).

**Método.** Em 16/11/2026, executei três rodadas por versão, com 20 usuários virtuais por 60 segundos. A tabela apresenta a mediana das três rodadas. Para CPU e memória, usei a média de cada rodada antes de calcular a mediana.

| Métrica | Linha de base | Com índice |
|---|---:|---:|
| Vazão | 2,36 req/s | 3,17 req/s |
| p50 da API | 1.852,0 ms | 2.400,5 ms |
| p95 da API | 26.974,4 ms | 18.812,0 ms |
| p99 da API | 31.846,4 ms | 27.208,2 ms |
| p95 da rota de pedidos do cliente | 31.306,6 ms | 17.471,8 ms |
| Erros | 0% | 0% |
| CPU da API | 6,6% | 13,1% |
| Memória da API | 37,9 MiB | 41,5 MiB |
| CPU do banco | 100,3% | 99,6% |
| Memória do banco | 288,8 MiB | 324,2 MiB |
| Rodadas com contrato preservado | 3/3 | 3/3 |

**Diagnóstico.** Na consulta isolada dos itens do pedido `1`, o PostgreSQL passou de `Parallel Seq Scan` para `Index Scan`. O tempo de execução caiu de 558,222 ms para 0,650 ms. [Plano anterior](diagnostico/pedidos-do-cliente-antes.txt) · [Plano com índice](diagnostico/itens-pedido-depois.txt)

**Decisão.** Manter o índice para o próximo teste. A vazão subiu 34,3% e o p95 da rota-alvo caiu 44,2%. O p50 global subiu 29,6%, e o consumo de memória do banco aumentou. Esses resultados se referem à carga testada.

Os arquivos brutos estão em [resultados/](resultados/), nas rodadas `00-linha-de-base` e `01-indice-itens-pedido`.

## 02 — Índice dos pedidos por cliente

**Hipótese.** Um índice em `pedidos(cliente_id)` deve acelerar a busca dos pedidos de um cliente. O índice foi adicionado em [04-indice-pedidos-cliente.sql](banco/04-indice-pedidos-cliente.sql).

**Método.** Comparei três rodadas por configuração, com 20 usuários virtuais por 60 segundos. A tabela mostra a mediana das rodadas. Para CPU e memória, calculei a mediana do consumo médio de cada rodada.

| Métrica | Linha de base | Índice de itens | Dois índices |
|---|---:|---:|---:|
| Vazão | 2,36 req/s | 3,17 req/s | 5,07 req/s |
| p50 da API | 1.852,0 ms | 2.400,5 ms | 1.338,2 ms |
| p95 da API | 26.974,4 ms | 18.812,0 ms | 11.245,8 ms |
| p99 da API | 31.846,4 ms | 27.208,2 ms | 26.525,4 ms |
| p95 da rota de pedidos do cliente | 31.306,6 ms | 17.471,8 ms | 11.563,9 ms |
| Taxa de erros | 0% | 0% | 0% |
| CPU da API | 6,6% | 13,1% | 11,8% |
| Memória da API | 37,9 MiB | 41,5 MiB | 44,6 MiB |
| CPU do banco | 100,3% | 99,6% | 98,8% |
| Memória do banco | 288,8 MiB | 324,2 MiB | 366,8 MiB |
| Rodadas com contrato preservado | 3/3 | 3/3 | 3/3 |

**Diagnóstico.** Na consulta isolada dos cinco pedidos do cliente `48272`, o plano passou de `Parallel Seq Scan` para `Index Scan` usando `idx_pedidos_cliente_id`. O tempo de execução passou de 611,030 ms para 0,333 ms. [Plano inicial](diagnostico/pedidos-do-cliente-antes.txt) · [Plano com índice](diagnostico/pedidos-do-cliente-depois.txt)

**Decisão.** Manter os dois índices. Em relação à linha de base, a vazão aumentou 114,8%, o p95 da API caiu 58,3% e o p95 da rota-alvo caiu 63,1%. A memória do banco aumentou 78,0 MiB e o p99 da API ainda ficou em 26,53 s na carga testada.

Dados brutos das três rodadas em [21h02](resultados/20261116-210200-02-indice-pedidos-cliente/), [21h03](resultados/20261116-210329-02-indice-pedidos-cliente/) e [21h04](resultados/20261116-210459-02-indice-pedidos-cliente/).
