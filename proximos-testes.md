# Próximos testes

## Índice composto para os pedidos

- **Fonte.** [PostgreSQL 16, Indexes and ORDER BY](https://www.postgresql.org/docs/16/indexes-ordering.html).
- **Hipótese.** Um índice em `(cliente_id, criado_em DESC, id DESC)` pode atender ao filtro e à ordem dos pedidos na mesma leitura.
- **Como testar.** Na configuração com dois índices, substituir somente o índice simples de pedidos pelo composto. Conferir o contrato, comparar os planos e executar três rodadas com a mesma carga. Registrar também o tamanho dos índices.
- **O que precisa.** Um novo SQL de teste e consultas com quantidades de pedidos representativas.

## Busca de itens e produtos em lote

- **Fonte.** [Microsoft, Efficient Querying](https://learn.microsoft.com/en-us/ef/core/performance/efficient-querying).
- **Hipótese.** Buscar os itens de vários pedidos e seus produtos de uma vez pode reduzir as chamadas ao banco dentro dos laços da rota.
- **Como testar.** Alterar somente a consulta da rota, medir o número de chamadas ao banco e comparar o p95, a vazão e os recursos com a configuração de dois índices. Conferir pedidos vazios, totais, ordem e o contrato completo.
- **O que precisa.** Uma consulta em lote e evidência de que o JSON reconstruído coincide com as respostas gravadas.

## Aumento de CPU do banco

- **Fonte.** [Docker, Compose Deploy Specification](https://docs.docker.com/reference/compose-file/deploy/).
- **Hipótese.** Aumentar o limite de 1 para 2 CPUs pode reduzir a espera quando o banco estiver limitado por CPU.
- **Como testar.** Alterar apenas o limite de CPU do banco. Comparar três rodadas por configuração e avaliar a vazão, o p95 e o consumo diante da capacidade provisionada.
- **O que precisa.** Uma máquina com capacidade disponível para o novo limite e a mesma configuração para as demais variáveis.

## Carga com taxa de chegada controlada

- **Fonte.** [Grafana, Open and closed models](https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/open-vs-closed/) e [Schroeder et al., Open Versus Closed](https://www.usenix.org/legacy/events/nsdi06/tech/full_papers/schroeder/schroeder.pdf).
- **Hipótese.** Os índices podem permitir que a API sustente uma taxa maior de chegada dentro de uma meta de p95.
- **Como testar.** Em ensaio exploratório separado, comparar a linha de base e a versão com dois índices sob as mesmas taxas de chegada. Registrar p95, erros, memória, uso de CPU e ponto de saturação de cada configuração.
- **O que precisa.** Uma meta de p95, taxas de chegada representativas e um gerador de carga separado.
