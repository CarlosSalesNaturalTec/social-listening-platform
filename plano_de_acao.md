# PLANO DE AÇÃO

Objetivo : Construir uma Plataforma de Social Listening capaz de realizar monitoramento de redes sociais e análise de dados de forma automática. As fontes a serem pesquisadas são: Web em geral utilizando Google CSE, Youtube, Instagran, Facebook, Mensagens de Grupos do Whatsapp, Tik tok, Kwaii, Linkedin e Twitter. A plataforma deve possuir um frontend no qual o usuário poderá: definir os parâmetros do sistema, iniciar o processo de coleta de dados, acompanhar o andamento destas coletas e realizar as respectivas análises de social listening. Para cada uma das fontes a serem pesquisadas e scrapeadas deverá ser construída uma respectiva API isolada.  O processo de construção desta plataforma deve ser feito em etapas com a utilização do gemini cli e moderadoração de um humano. O plano de ação geral e o estado de construção da plataforma deve ser armazenado em arquivo json a ser consultado pelo gemini cli durante a criação do código, permitindo que o processo seja continuado mesmo após interrupções. Ambiente de produção: GCP, Cloud Run, Firebase, Storage/Buckets, Cloud Schedules. Ambiente de desenvolvimento Windows. O processo de coleta será inicializado pelo usuário, a partir daí a plataforma deve ser capaz de funcionar de forma automática, realizando coletas e análises diárias inicializadas por chamadas do cloud schedules. Deve ser considerado o código existente como ponto de partida. Utilizar o mesmo banco de dados criado no módulo de search_google_cse nos demais módulos, compartilhando as suas coleções.

# Módulos/Etapas da plataforma:

* - **FRONTEND** 
  * - Etapa inicial de inicio de monitoramento já construída. Disponível na pasta: /frontend . 
  * - Usuário Define os termos da busca (marca e concorrentes).
  * - Define data histórica inicial para buscas históricas.
  * - Inicia etapa de buscas.
  * - Mantenha inalterada a estrutura de pastas existente /frontend e /backend. Salvo necessidade justificada.

* - **SEARCH_GOOGLE_CSE** 
  * - Etapa já construída. Disponível na pasta : /backend .
  * - Pesquisa urls mais relevantes no momento.
  * - Pesquisa urls mais relevantes no período histórico definido. 
  * - Salva urls em banco de dados indicando a fonte=google_cse.
  * - Dispõe de endpoint que realiza pesquisa por urls novas (dateRestrict=d1). Este endpoint deve ser acionado diariamente via Cloud Schedule.
  * - Mantenha inalterada a estrutura de pastas existente /frontend e /backend. Salvo necessidade justificada.

* - **Scraper newspaper3k** 
  * - Etapa já construída. Disponível na pasta : /scraper_newspaper3k.
  * - Estabelece um filtro inicial para as urls: caso o seu dominio seja youtube, instagram ou facebook devem ser desconsideradas. As mesmas  possuirão scrapers específicos a serem desenvolvidos em etapa posterior. 
  * - Caso passe do filtro anterior, analisa a relevância da url. Critérios para determinação de relevância: Presença do termo exato no title: peso 30. Presença do termo no snippet: peso 10. Domínio confiável ou autoridade alta: peso 25. URL amigável, sem muitos parâmetros: peso 5 . Título com múltiplos termos úteis: peso 10. Data da publicação (quando disponível): Peso 20. Ponto de corte = 0.60. 
  * - Para as urls que passarão da linha de corte por relevância, tenta realizar o scraping. 
  * - Se conseguiu realizar o scraping salva resultado em banco de dados e define o respecivo status como scraper_ok para evitar reprocessamento.
  * - Se não conseguiu realizar o scraper registra esta informação no respectivo status inclusive registrando o motivo da não realização.

* - **NLP** 
  * - Etapa já construída. Disponível na pasta : /api_nlp.
  * - para cada uma das urls scrapeadas na etapa anterior realiza:
    * - Análise de sentimento.
    * - Extração de entidades.
    * - Moderação de conteúdo.
  * - Salvar análises em banco de dados e atualiza o respectivo status: nlp_ok ou nlp_erro.

* - **ANALYTICS** 
  * - Implementar módulo conforme descrito no documento de contexto: contextDoc.md
      
* - **SEARCH_INSTAGRAM** 
  * - Utilizar Instaloader ou similar.
  * - Criar neste novo módulo um endpoind, para ser acionado via Cloud Scheduler.
  * - Pesquisa por hashtags.
  * - Pesquisa em perfis específicos. 
  * - Salva urls obtidas nas pesquisas em banco de dados indicando a origem=instagram.
  * - baixar conteúdos (imagens, vídeos) e metadados.
  * - Realizar OCR e/ou imagem descrição nas imagens (jpeg, png).
  * - Transcrever vídeos.
  * - Disponibilizar material em banco de dados para análise via módulo de NLP existente.

* - **SEARCH_YOUTUBE** 
  * - Utilizar yt-dlp.
  * - Criar neste novo módulo um endpoind, para ser acionado via Cloud Scheduler.
  * - Pesquisa por hashtags ou termos de pesquisa.
  * - Pesquisa em perfis específicos. 
  * - Salva urls obtidas nas pesquisas em banco de dados indicando a origem=youtube.
  * - baixar vídeos e metadados.
  * - Transcrever vídeos.
  * - Excluir vídeos transcritos.
  * - Disponibilizar material em banco de dados para análise via módulo de NLP existente.

* - **SEARCH_FACEBOOK** 
  * - Criar neste novo módulo um endpoind, para ser acionado via Cloud Scheduler.
  * - Pesquisa por hashtags ou termos de pesquisa.
  * - Pesquisa em perfis específicos. 
  * - Salva urls obtidas nas pesquisas em banco de dados indicando a origem=facebook.
  * - baixar vídeos e metadados.
  * - Transcrever vídeos.
  * - Excluir vídeos transcritos.
  * - Disponibilizar material em banco de dados para análise via módulo de NLP existente.

* - **WHATSAPP_GROUPS** 
  * - Novo módulo capaz de realizar buscas em conversas exportadas de grupos de whatsapp
  * - Ao encontrar menções aos termos especificados na plataforma, disponibilizar material em banco de dados para análise via módulo de NLP existente.

* - **SEARCH_TWITER** 
  * - Criar neste novo módulo um endpoind, para ser acionado via Cloud Scheduler.
  * - Pesquisa por hashtags.
  * - Pesquisa em perfis específicos. 
  * - Salva urls obtidas nas pesquisas em banco de dados indicando a origem=twitter.
  * - baixar conteúdos (imagens, vídeos) e metadados.
  * - Realizar OCR e/ou imagem descrição nas imagens (jpeg, png).
  * - Transcrever vídeos.
  * - Disponibilizar material em banco de dados para análise via módulo de NLP existente.

* - **SEARCH_TIKTOK** 
  * - Criar neste novo módulo um endpoind, para ser acionado via Cloud Scheduler.
  * - Pesquisa por hashtags.
  * - Pesquisa em perfis específicos. 
  * - Salva urls obtidas nas pesquisas em banco de dados indicando a origem=tiktok.
  * - baixar conteúdos (imagens, vídeos) e metadados.
  * - Realizar OCR e/ou imagem descrição nas imagens (jpeg, png).
  * - Transcrever vídeos.
  * - Disponibilizar material em banco de dados para análise via módulo de NLP existente.

* - **SEARCH_KWAII** 
  * - Criar neste novo módulo um endpoind, para ser acionado via Cloud Scheduler.
  * - Pesquisa por hashtags.
  * - Pesquisa em perfis específicos. 
  * - Salva urls obtidas nas pesquisas em banco de dados indicando a origem=kwaii.
  * - baixar conteúdos (imagens, vídeos) e metadados.
  * - Realizar OCR e/ou imagem descrição nas imagens (jpeg, png).
  * - Transcrever vídeos.
  * - Disponibilizar material em banco de dados para análise via módulo de NLP existente.

* - **SEARCH_LINKEDIN** 
  * - Criar neste novo módulo um endpoind, para ser acionado via Cloud Scheduler.
  * - Pesquisa por hashtags.
  * - Pesquisa em perfis específicos. 
  * - Salva urls obtidas nas pesquisas em banco de dados indicando a origem=linkedin.
  * - baixar conteúdos (imagens, vídeos) e metadados.
  * - Realizar OCR e/ou imagem descrição nas imagens (jpeg, png).
  * - Transcrever vídeos.
  * - Disponibilizar material em banco de dados para análise via módulo de NLP existente.

- **AGENT MODE**
  * - Alertas de crise e outros via WhatsApp utilizando @open-wa/wa-automate.
 
- **SEMANTIC** 
  * - Criar novo módulo, e disponiblizar no mesmo, um endpoind para ser acionado via Cloud Scheduler.
  * - Ao ser acionado realizar o Embeddings dos artigos e suas respectivas análises de NLP. 
  * - Armazenamento em Banco vetorial para posterior consultas vias chats.
  * - Por meio do Whatsapp permitir consulta aos artigos e análises da plataforma utilizando técnicas de busca semântica. Utilizar @open-wa/wa-automate para acesso ao Whatsapp.
