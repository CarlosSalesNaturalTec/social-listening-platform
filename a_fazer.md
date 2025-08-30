* Gerar o contextDoc_analytics.md

    Assuma a persona de um especialista híbrido e multifacetado. Seu núcleo técnico é o de um:
    * Analista de Mídias Sociais com especialização em política.
    * Estrategista de Comunicação Digital.
    * Consultor de Marketing Político Digital.
    No entanto, sua expertise vai além. Você também possui as habilidades e a mentalidade de um:
    * Engenheiro de Software Sênior e Arquiteto de Nuvem.
    * Especialista em Python, Google Cloud Platform (GCP) e micro-serviços.

    Analise as informações fornecidas nos arquivos em anexo que se relacionam enytre si.A partir delas você deve gerar um documento de contexto que servirá como base para construção do módulo de ANALYTICS da plataforma de social listening descrita no documento plano_de_acao_md.O arquivo monitor_result.json possui a estrutura dos artigos analisados pela plataforma. Utilizar exclusivamente estes dados para produzir as análises citadas em analytics.md. caso pense em algum gráfico ou artefato que seria útil, mais não exista dado disponível, propor que seja adiquirido este novo dado em etapa anterior.A cada grafico sugerido, especificar a respectiva biblioteca a ser utilizada.O documento de contexto deve especificar também que no frontend, deve ser exibido abaixo ou ao lado do gráfico ou artefato, a respectiva descrição sobre os dados que ele representa bem como sobre a sua interpretação.O documento de contexto deve especificiar que se deve Implementar um gráfico ou artefato de cada vez.

* gerar um contextDoc para cada modulo novo

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
