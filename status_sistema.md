# Status possíveis para um documento na coleção `monitor_results`, com a descrição e o módulo responsável por cada um:

## Módulo: backend / search

   * `pending`
       * Descrição: Este é o status inicial de um resultado de busca assim que ele é encontrado e salvo no banco de dados. Indica que a URL está na fila, aguardando para ser processada pelo próximo serviço no pipeline (o scraper).
       * Definido em: backend/schemas/monitor_schemas.py (como valor padrão no schema MonitorResultItem).

## Módulo: scraper_newspaper3k

   * `scraper_skipped`
       * Descrição: O scraper ignorou a URL porque ela pertence a um domínio de rede social (ex: YouTube, Instagram, Facebook) que não deve ser processado para extração de texto.
       * Definido em: scraper_newspaper3k/main.py.

   * `relevance_failed`
       * Descrição: O scraper analisou a URL e seus metadados (título, snippet) e calculou uma pontuação de relevância. Este status é atribuído quando a pontuação é muito baixa, indicando que o conteúdo provavelmente não é relevante e, portanto, a extração completa não é realizada.
       * Definido em: scraper_newspaper3k/main.py.

   * `scraper_failed`
       * Descrição: Ocorreu um erro durante a tentativa de baixar ou extrair o conteúdo do artigo da URL. Isso pode acontecer por diversos motivos, como a página não estar mais disponível, bloqueios ou falhas na biblioteca de extração.
       * Definido em: scraper_newspaper3k/main.py.

   * `scraper_ok`
       * Descrição: O scraper processou a URL com sucesso, extraiu o conteúdo principal do artigo (título, texto, autores, etc.) e o salvou no banco de dados. O item está pronto para a próxima etapa, que é a análise de NLP.
       * Definido em: scraper_newspaper3k/main.py.

## Módulo: api_nlp

   * `nlp_error`
       * Descrição: Ocorreu um erro durante a tentativa de analisar o texto extraído usando a API do Google Cloud Natural Language. O processo de análise de sentimento, entidades ou moderação falhou.
       * Definido em: api_nlp/main.py.

   * `nlp_ok`
       * Descrição: O conteúdo do artigo foi analisado com sucesso pela API de NLP. Os resultados (sentimento, entidades, etc.) foram salvos no banco de dados. Este é geralmente o estado final de um item processado com sucesso.
       * Definido em: api_nlp/main.py.

## Status Adicional (Indefinido nos Módulos, mas Presente na Lógica)

   * `reprocess`
       * Descrição: Este status, embora não seja definido diretamente por nenhuma das lógicas de background, é listado como um status válido que o scraper pode buscar (['pending', 'reprocess']). Isso sugere que é um status que pode ser definido manualmente (provavelmente através da interface do frontend) para forçar que uma URL seja processada novamente, mesmo que já tenha falhado ou sido ignorada anteriormente.
      