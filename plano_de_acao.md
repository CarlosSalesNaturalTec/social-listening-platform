# PLANO DE AÇÃO

Objetivo : Construir uma Plataforma de Social Listening capaz de realizar monitoramento de redes sociais e análise de dados de forma automática. As fontes a serem pesquisadas são: Web em geral utilizando Google CSE, Google Trends, Youtube, Instagran, Facebook, Mensagens de Grupos do Whatsapp, Tik tok, Kwaii, Linkedin e Twitter. A plataforma deve possuir um frontend no qual o usuário poderá: definir os parâmetros do sistema, iniciar o processo de coleta de dados, acompanhar o andamento destas coletas , realizar as respectivas análises de social listening, e autorizar a publicação automática em redes sociais de conteúdo gerado pelo sistema. Para cada uma das fontes a serem pesquisadas e scrapeadas deverá ser construída uma respectiva API isolada. O processo de construção desta plataforma deve ser feito em etapas com a utilização do gemini cli e moderadoração de um humano. O plano de ação geral e o estado de construção da plataforma deve ser armazenado em arquivo json a ser consultado pelo gemini cli durante a criação do código, permitindo que o processo seja continuado mesmo após interrupções. Ambiente de produção: GCP, Cloud Run, Firebase, Storage/Buckets, Cloud Schedules. Ambiente de desenvolvimento Windows. O processo de coleta será inicializado pelo usuário, a partir daí a plataforma deve ser capaz de funcionar de forma automática, realizando coletas e análises diárias inicializadas por chamadas do cloud schedules. Deve ser considerado o código existente como ponto de partida. Utilizar o mesmo banco de dados criado no módulo de search_google_cse nos demais módulos, compartilhando as suas coleções.

# Módulos/Etapas da plataforma:

* - **FRONTEND** 
  * - Etapa inicial de inicio de monitoramento já construída. Disponível na pasta: /frontend . 
  * - Detalhes técnicos, instruções de uso e relação com os demais módulos estão descritas no arquivo README.md
  * - Funcionalidades
    * - Usuário Define os termos da busca (marca e concorrentes).
    * - (NOVO) Usuário Define termos-chave (ex: nomes, pautas) para monitoramento contínuo no Google Trends e suas respectivas geografias (ex: Brasil, Estado de SP).
    * - Define data histórica inicial para buscas históricas.
    * - Inicia etapa de buscas: relevante e histórica.

* - **SEARCH_GOOGLE_CSE** 
  * - Etapa já construída. Disponível na pasta : /backend .
  * - Detalhes técnicos, instruções de uso e relação com os demais módulos estão descritas no arquivo README.md
  * - Funcionalidades
    * - Endpoint acionado diariamente às 12:00hs e as 20:00hs via Cloud Schedule para buscas de dados contínuos.
    * - Endpoint acionado diariamente às 21:00hs via Cloud Schedule para buscas de dados históricos.
    * - Pesquisa urls mais relevantes no momento.
    * - Pesquisa urls mais relevantes no período histórico definido. 
    * - Salva urls em banco de dados indicando a fonte=google_cse.
    * - Dispõe de endpoint que realiza pesquisa por urls novas (dateRestrict=d1). 


* - **SEARCH_GOOGLE_TRENDS** 
* Este novo módulo será uma API isolada (Cloud Run) construída em Python, utilizando a biblioteca pytrends para interagir com o Google Trends.
* Detalhes técnicos: O módulo irá ler os termos definidos no Frontend do banco de dados para realizar as consultas.
* Funcionalidades:
  * Endpoint acionado diariamente (ex: 08:00hs) via Cloud Schedule para buscas de dados de interesse ao longo do tempo (últimos 7 dias) para os termos definidos, salvando a série temporal no banco de dados.
  * Endpoint acionado a cada hora via Cloud Schedule para monitorar "buscas em ascensão" (rising queries) relacionadas aos termos principais, salvando picos e oportunidades no banco.
  * Endpoint para consultas comparativas entre o parlamentar e seus concorrentes.
  * Salva os dados (séries temporais, buscas relacionadas, buscas em ascensão) em uma coleção específica no banco de dados (ex: google_trends_data), indicando o termo, a data, a geografia e o tipo de dado.

* - **Scraper newspaper3k** 
  * - Etapa já construída. Disponível na pasta : /scraper_newspaper3k.
  * - Detalhes técnicos, instruções de uso e relação com os demais módulos estão descritas no arquivo README.md
  * - Funcionalidades
    * - Este módulo possui um Endpoint que deve ser acionado diariamente às 22:00hs via Cloud Schedule para realizar os respectivos scrapers.
    * - Estabelece um filtro inicial para as urls: caso o seu dominio seja youtube, instagram ou facebook devem ser desconsideradas. As mesmas  possuirão scrapers específicos a serem desenvolvidos em etapa posterior. 
    * - Caso passe do filtro anterior, analisa a relevância da url. Critérios para determinação de relevância: Presença do termo exato no title: peso 30. Presença do termo no snippet: peso 10. Domínio confiável ou autoridade alta: peso 25. URL amigável, sem muitos parâmetros: peso 5 . Título com múltiplos termos úteis: peso 10. Data da publicação (quando disponível): Peso 20. Ponto de corte = 0.60. 
    * - Para as urls que passarão da linha de corte por relevância, tenta realizar o scraping. 
    * - Se conseguiu realizar o scraping salva resultado em banco de dados e define o respecivo status como scraper_ok para evitar reprocessamento.
    * - Se não conseguiu realizar o scraper registra esta informação no respectivo status inclusive registrando o motivo da não realização.
    
* - **NLP** 
  * - Etapa já construída. Disponível na pasta : /api_nlp.
  * - Detalhes técnicos, instruções de uso e relação com os demais módulos estão descritas no arquivo README.md
  * - Funcionalidades
    * - Este módulo possui um Endpoint que deve ser acionado diariamente às 23:00hs via Cloud Schedule para realizar as respectivos análises.
    * - para cada uma das urls scrapeadas na etapa anterior realiza:
      * - Análise de sentimento.
      * - Extração de entidades.
      * - Moderação de conteúdo.
    * - Salvar análises em banco de dados e atualiza o respectivo status: nlp_ok ou nlp_erro.

* - **ANALYTICS** 
  * - Implementar módulo conforme descrito no documento de contexto: contextDoc_analytics.md.
        
* - **SEARCH_INSTAGRAM** 
  * - Implementar módulo conforme descrito no documento de contexto: contextDoc_instagram.md
  
* - **SEARCH_YOUTUBE** 
  * - Implementar módulo conforme descrito no documento de contexto: contextDoc_youtube.md
  
* - **SEARCH_FACEBOOK** 
  * - Implementar módulo conforme descrito no documento de contexto: contextDoc_facebook.md.
  
* - **WHATSAPP_GROUPS** 
  * - Implementar módulo conforme descrito no documento de contexto: contextDoc_whatsapp.md
 
* - **AGENT MODE**
  * - Implementar módulo conforme descrito no documento de contexto: contextDoc_agentMode.md 

* - **SEMANTIC** 
  * - Implementar módulo conforme descrito no documento de contexto: contextDoc_semantic.md

* - **SEARCH_TWITER** 
  * - Implementar módulo conforme descrito no documento de contexto: contextDoc_twitter.md

* - **SEARCH_TIKTOK** 
  * - Implementar módulo conforme descrito no documento de contexto: contextDoc_tiktok.md

* - **SEARCH_KWAII** 
  * - Implementar módulo conforme descrito no documento de contexto: contextDoc_kwaii.md

* - **SEARCH_LINKEDIN** 
  * - Implementar módulo conforme descrito no documento de contexto: contextDoc_linkedin.md
