# Dois índices no PostgreSQL mais que dobraram a vazão da API de uma loja

## 1. Resumo

A rota de pedidos da loja consultava tabelas inteiras para retornar poucos pedidos e itens. Testei um índice para cada filtro, em mudanças separadas. Com os dois índices, a vazão foi de 2,36 para 5,07 requisições por segundo e o p95 geral caiu de 26,97 para 11,25 segundos, nos limites originais de CPU e memória. As 42 respostas do contrato foram preservadas nas nove rodadas. A memória medida do banco aumentou 78,0 MiB.

## 2. Contexto e diagnóstico

A API usa Node.js 20 e PostgreSQL 16 no Docker Compose. O banco sintético contém 500 mil pedidos e 1,5 milhão de itens. A API tem limite de 0,5 CPU e 256 MiB, e o banco tem 1 CPU e 512 MiB. O teste combina sete rotas de leitura. Na linha de base, `GET /clientes/:id/pedidos` teve p95 de 31,31 s e respondia por 20% do peso previsto na carga. Nesse cenário, a CPU média do banco ficou perto de uma CPU. [Configuração](../docker-compose.yml) · [Carga](../carga/teste-carga.js) · [Medições](../experimentos.md)

A rota busca o cliente e consulta seus pedidos por `cliente_id`, ordenados por data e ID decrescentes. Para cada pedido, busca os itens por `pedido_id`. A seguir, consulta o produto de cada item. No `EXPLAIN (ANALYZE, BUFFERS)` da versão original, a busca dos cinco pedidos do cliente 48272 levou 611,030 ms. A busca dos dois itens do pedido 1 levou 558,222 ms. Os dois planos mostraram `Parallel Seq Scan`, uma leitura paralela da tabela com filtro. [Código da rota](../api/src/rotas.js) · [Planos iniciais](../diagnostico/pedidos-do-cliente-antes.txt) · [2][explain]

**Hipótese de causa.** A busca por cliente custava uma varredura de `pedidos`. Na busca de itens, esse custo tendia a se repetir porque a aplicação executava o filtro de `itens_pedido` para cada pedido. Índices nos filtros deveriam reduzir o trabalho dessas consultas e a latência da rota durante o teste.

## 3. Estado da arte

### Índices para filtros seletivos

O PostgreSQL pode usar índices B-tree para comparações por igualdade. Quando um filtro encontra poucas linhas em uma tabela grande, esse acesso pode reduzir a leitura necessária para localizar os registros. O índice ocupa espaço e precisa ser mantido quando a tabela recebe escritas. A equipe do GitLab recomenda analisar tanto o tempo quanto o plano da consulta, pois o caminho de acesso influencia o comportamento à medida que os dados crescem. Um B-tree com várias colunas também pode entregar o resultado na ordem da consulta, se as chaves e a ordenação forem compatíveis. [1][indices] [Custo de manutenção][custo-indices] [3][ordenacao] [4][gitlab]

### Consultas em lote

Uma alternativa é buscar os itens de vários pedidos em uma consulta e montar a resposta por pedido na API. O guia do EF Core ilustra o problema N+1, em que um laço gera uma consulta adicional por registro. O padrão desta aplicação é semelhante, com chamadas explícitas no JavaScript. Buscar itens e nomes de produtos juntos pode reduzir as idas ao banco. Em uma junção que também traga os pedidos, os dados do pedido podem se repetir em cada linha de item. É preciso conferir a ordem das listas, os totais e os casos de resultado vazio. [5][lote]

### Capacidade e latência de cauda

O Compose permite alterar a capacidade disponível por contêiner. Faz sentido investigar mais CPU quando há evidência de fila e uso persistente próximo do limite. Dean e Barroso mostram por que respostas lentas na cauda da distribuição importam em sistemas interativos. Por isso, considero o p95 junto da vazão e do consumo. No modelo fechado do k6, cada usuário inicia sua próxima iteração após a anterior. Schroeder e coautores mostram que esse modelo pode produzir resultados diferentes de uma carga com chegadas independentes. [6][docker] [7][cauda] [8][k6] [9][modelos]

## 4. Alternativas consideradas

| Alternativa | Custo | Desempenho | Risco | Esforço |
|---|---|---|---|---|
| B-tree simples em `pedido_id` e `cliente_id` | Espaço e manutenção dos índices | Localiza linhas pelos filtros dos dois planos | Custo adicional nas escritas | Baixo, uma alteração SQL por índice |
| B-tree composto em `(cliente_id, criado_em DESC, id DESC)` | Chave mais larga | Pode atender ao filtro e à ordenação de pedidos | Ganho pequeno quando o resultado tem poucas linhas | Baixo, com nova comparação de planos |
| Buscar itens e produtos em lote | Alteração e manutenção do código da rota | Pode reduzir consultas e esperas por chamadas sequenciais | Erros na ordem, nos totais ou na montagem da resposta | Médio, com testes de contrato e de borda |
| Aumentar CPU do banco | Maior capacidade de CPU provisionada | Pode reduzir filas se a carga estiver limitada por CPU | Mais recursos sem ganho proporcional | Baixo na configuração, com nova medição |

Um índice composto mereceria prioridade se a ordenação fosse uma parcela relevante da latência. No caso analisado, a ordenação processava cinco linhas e o plano mostrava trabalho na varredura. A busca em lote atacaria outro componente do custo, o número de chamadas ao PostgreSQL, e exigiria revalidar a montagem do JSON. Aumentar CPU seria um teste útil para separar capacidade de eficiência do plano, com um orçamento de recursos diferente. [2][explain] [3][ordenacao] [5][lote] [6][docker]

## 5. Solução escolhida e por quê

Comecei com o B-tree em `itens_pedido(pedido_id)` porque essa consulta se repete para cada pedido da rota. Depois apliquei o B-tree em `pedidos(cliente_id)` para tratar a busca que antecede esse laço. As duas mudanças foram registradas em SQL separado, aplicadas na recriação do banco sintético e medidas em sequência. O `ORDER BY` explícito determina a ordem que o verificador compara. [Índice de itens](../banco/03-indice-itens-pedido.sql) · [Índice de pedidos](../banco/04-indice-pedidos-cliente.sql)

Escolhi manter os dois índices no cenário avaliado. Essa configuração elevou a quantidade de requisições atendidas dentro dos limites originais dos contêineres. A escolha considera a alta de memória medida e a manutenção dos índices em operações de escrita. As demais propostas têm testes separados para estimar o ganho adicional. [Registro das medições](../experimentos.md) · [Próximos testes](../proximos-testes.md)

## 6. Experimento e resultado

Executei três rodadas por versão na mesma máquina, com 20 usuários virtuais e 60 s de medição por rodada. Usei a mediana de cada métrica entre as três rodadas. Para CPU e memória, primeiro considerei a média amostrada de cada contêiner por rodada. Comparei o primeiro índice com a linha de base e o segundo com a versão que já tinha o índice de itens. O p95 da tabela é o percentil 95 da latência de toda a API. [Script de medição](../scripts/medir.sh) · [Registro](../experimentos.md)

| Mudança | Hipótese | p95 antes → depois | req/s antes → depois | CPU e memória antes → depois | Pastas de resultados | Conclusão |
|---|---|---:|---:|---|---|---|
| Índice de itens | Reduzir o custo da busca de itens por pedido | 26.974,4 → 18.812,0 ms | 2,36 → 3,17 | API 6,6%/37,9 MiB → 13,1%/41,5 MiB. Banco 100,3%/288,8 MiB → 99,6%/324,2 MiB. | 00 [1][base1] [2][base2] [3][base3] → 01 [1][itens1] [2][itens2] [3][itens3] | Menor p95 geral e piora do p50 |
| Índice de pedidos | Reduzir o custo da busca de pedidos por cliente | 18.812,0 → 11.245,8 ms | 3,17 → 5,07 | API 13,1%/41,5 MiB → 11,8%/44,6 MiB. Banco 99,6%/324,2 MiB → 98,8%/366,8 MiB. | 01 [1][itens1] [2][itens2] [3][itens3] → 02 [1][pedidos1] [2][pedidos2] [3][pedidos3] | Maior vazão, menor p95 geral e mais memória no banco |

Os planos posteriores registraram `Index Scan` nas duas consultas isoladas, com tempos de 0,650 ms para itens e 0,333 ms para pedidos. O primeiro índice piorou o p50 de 1,85 para 2,40 s. Com os dois, o p50 caiu para 1,34 s e o p95 da rota-alvo caiu 63,1% frente à linha de base. A taxa de erros foi de 0% nas nove rodadas, e cada verificação preservou as 42 respostas de referência. [Plano de itens](../diagnostico/itens-pedido-depois.txt) · [Plano de pedidos](../diagnostico/pedidos-do-cliente-depois.txt) · [Medições](../experimentos.md)

## 7. Limites e testes futuros

Três rodadas curtas com dados sintéticos descrevem o cenário local. O modelo fechado permite que o volume efetivo de chamadas varie com o tempo de resposta. Os planos SQL são observações pontuais sujeitas ao estado do cache. Com os dois índices, o p99 geral ainda ficou em 26,53 s. [8][k6] [9][modelos] · [Medições](../experimentos.md)

A vazão maior sob os mesmos limites indica melhor aproveitamento da configuração nesse teste. Para estimar economia em produção, mediria memória, disco e custo de escritas com uma taxa de chegada e uma meta de p95 definidas. Eu revisaria a escolha se, nas mesmas condições de chegada e com o contrato preservado, o ganho desaparecesse ou se o custo extra de armazenamento e de escrita superasse o ganho de capacidade. As fichas detalham os experimentos que podem orientar essa revisão. [Próximos testes](../proximos-testes.md)

## 8. Referências

1. PostgreSQL Global Development Group. [Index Types][indices] e [Introduction to Indexes][custo-indices]. PostgreSQL 16. Acesso em 16/11/2026.
2. PostgreSQL Global Development Group. [Using EXPLAIN][explain]. PostgreSQL 16. Acesso em 16/11/2026.
3. PostgreSQL Global Development Group. [Indexes and ORDER BY][ordenacao]. PostgreSQL 16. Acesso em 16/11/2026.
4. GitLab. [Query performance guidelines][gitlab]. Acesso em 16/11/2026.
5. Microsoft. [Efficient Querying][lote]. Documentação do EF Core. Acesso em 16/11/2026.
6. Docker. [Compose Deploy Specification][docker]. Acesso em 16/11/2026.
7. Dean, Jeffrey e Barroso, Luiz André. [The Tail at Scale][cauda]. Communications of the ACM, 56, p. 74–80, 2013. Acesso em 16/11/2026.
8. Grafana. [Open and closed models][k6]. Documentação do k6. Acesso em 16/11/2026.
9. Schroeder, Bianca, Wierman, Adam e Harchol-Balter, Mor. [Open Versus Closed: A Cautionary Tale][modelos]. NSDI, 2006. Acesso em 16/11/2026.

## 9. Uso de IA

Usei o ChatGPT para orientar o diagnóstico, pesquisar fontes, consolidar as medianas e redigir o artigo. As medições foram executadas no meu ambiente. Os números, os planos SQL e o resultado do contrato foram conferidos com as saídas registradas no repositório.

[indices]: https://www.postgresql.org/docs/16/indexes-types.html
[custo-indices]: https://www.postgresql.org/docs/16/indexes-intro.html
[explain]: https://www.postgresql.org/docs/16/using-explain.html
[ordenacao]: https://www.postgresql.org/docs/16/indexes-ordering.html
[gitlab]: https://docs.gitlab.com/development/database/query_performance/
[lote]: https://learn.microsoft.com/en-us/ef/core/performance/efficient-querying
[docker]: https://docs.docker.com/reference/compose-file/deploy/
[cauda]: https://research.google/pubs/the-tail-at-scale/
[k6]: https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/open-vs-closed/
[modelos]: https://www.usenix.org/legacy/events/nsdi06/tech/full_papers/schroeder/schroeder.pdf
[base1]: ../resultados/20261116-124238-00-linha-de-base/
[base2]: ../resultados/20261116-124425-00-linha-de-base/
[base3]: ../resultados/20261116-124610-00-linha-de-base/
[itens1]: ../resultados/20261116-135658-01-indice-itens-pedido/
[itens2]: ../resultados/20261116-135837-01-indice-itens-pedido/
[itens3]: ../resultados/20261116-140018-01-indice-itens-pedido/
[pedidos1]: ../resultados/20261116-210200-02-indice-pedidos-cliente/
[pedidos2]: ../resultados/20261116-210329-02-indice-pedidos-cliente/
[pedidos3]: ../resultados/20261116-210459-02-indice-pedidos-cliente/
