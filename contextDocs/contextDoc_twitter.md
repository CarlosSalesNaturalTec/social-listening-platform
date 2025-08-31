# SEARCH_TWITTER

* - Criar neste novo módulo um endpoind, para ser acionado via Cloud Scheduler. Ao ser acionado inicia pesquisa pelos termos de busca e hashtags e na sequência realiza o respectivo scrapping.
* - Utilizar preferencialmente o Instaloader.
* - Se necessário utilizar a API do Facebook Graph API.

## Pesquisa
* - Fazer pesquisa de publicações ocorridas no dia corrente.
* - Pesquisar por hashtags ou termos de pesquisa pre-cadastrados na coleção do firestore platform_config/search_terms.
* - Pesquisar também em perfis pre-cadastrados. 
* - Criar estratégia para pesquisar por perfis relevantes ao contexto do parlamentar e se possível cadastra-los automaticamente na lista de perfis pre-cadastrados a serem monitorados.
* - Ao encontrar publicações que atendam aos critérios de busca, cadastrá-las na coleção monitor_results com o campo origin="twitter".

## Scrapping
  * - De textos, mídias, comentários, likes.
  * - Realizar a transcrição de áudios, vídeos, PDF, imagens, bem como OCR de eventuais slides que surjam em vídeos.
  * - Salvar para posterior análise NLP. Utilizar campos : htmlSnippet, scraped_content, scraped_title, publish_date e outros novos, se necessário.    
  * - Ao concluir, define o respecivo status como "scraper_ok". Se não conseguiu realizar o scraper registra esta informação no respectivo status inclusive registrando o motivo da não realização.