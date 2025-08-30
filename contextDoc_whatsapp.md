# WHATSAPP
  
* - Criar neste novo módulo um endpoind, capaz de 
* - Recebr como parâmetro uma conversação exportada de grupos do whatsapp.
* - Realizar buscas nestes arquivos por termos de pesquisas pre-cadastrados. 
* - Se necessário realiza o respectivo scrapping de midias.

## Pesquisa
* - Analisar a questão da identificação e uso data da conversação.
* - Pesquisar por hashtags ou termos de pesquisa pre-cadastrados na coleção do firestore platform_config/search_terms.
* - Ao encontrar conteúdo que atendam aos critérios de busca, cadastrá-las na coleção monitor_results com o campo origin="whatsapp_groups".

## Scrapping
  * - Realizar a transcrição de áudios, vídeos, PDF, imagens.
  * - Salvar para posterior análise NLP. Utilizar campos : htmlSnippet, scraped_content, scraped_title, publish_date e outros novos, se necessário.    
  * - Ao concluir, define o respecivo status como "scraper_ok". Se não conseguiu realizar o scraper registra esta informação no respectivo status inclusive registrando o motivo da não realização.