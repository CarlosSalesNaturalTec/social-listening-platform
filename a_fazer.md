* gerar um contextDoc para cada modulo novo


Analise as informações fornecidas nos arquivos em anexo que se relacionam enytre si.A partir delas você deve gerar um documento de contexto que servirá como base para construção do módulo de SEARCH_GOOGLE_TRENDS da plataforma de social listening descrita no documento plano_de_acao_md. Este novo módulo será uma API isolada (Cloud Run) construída em Python, utilizando a biblioteca pytrends para interagir com o Google Trends.
* Detalhes técnicos: O módulo irá ler os termos definidos no Frontend do banco de dados para realizar as consultas.
* Funcionalidades:
  * Endpoint acionado diariamente (ex: 08:00hs) via Cloud Schedule para buscas de dados de interesse ao longo do tempo (últimos 7 dias) para os termos definidos, salvando a série temporal no banco de dados.
  * Endpoint acionado a cada hora via Cloud Schedule para monitorar "buscas em ascensão" (rising queries) relacionadas aos termos principais, salvando picos e oportunidades no banco.
  * Endpoint para consultas comparativas entre o parlamentar e seus concorrentes.
  * Salva os dados (séries temporais, buscas relacionadas, buscas em ascensão) em uma coleção específica no banco de dados (ex: google_trends_data), indicando o termo, a data, a geografia e o tipo de dado.

# Diversos
* Scraper / coluna Snippet ... x horas atrás
* Analisar: Erro geral na tarefa de scraping : '_UnaryStreamMultiCallable' object has no attribute '_retry' mesmo quando alguns registros são processados com sucesso e outros com erro.
* Estratégia para:  scraper_failed, nlp_error, reprocess ? 


# Melhorias
* No Login, sempre dá erro na primeira vez, tem que tentar 2 vezes
* Acesso externo bloqueado para as APIs
* Analisar, remover e evitar deprecad

# Schedule
ok - 12:00hs e 20:00hs  - search
ok - 21:00hs            - historical_search
ok - 22:00hs            - scraper
ok - 23:00              - nlp



