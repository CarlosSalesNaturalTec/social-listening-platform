# PLANO DE AÇÃO

**Objetivo**: Construir uma Plataforma de Social Listening capaz de realizar monitoramento de redes sociais e análise de dados de forma automática. As fontes a serem pesquisadas são: Web em geral utilizando Google CSE, Google Trends, Youtube, Instagram, Facebook, Mensagens de Grupos do Whatsapp, TikTok, Kwai, Linkedin e Twitter. A plataforma deve possuir um frontend no qual o usuário poderá: definir os parâmetros do sistema, iniciar o processo de coleta de dados, acompanhar o andamento destas coletas, realizar as respectivas análises de social listening, e gerenciar o AGENT MODE. Para cada uma das fontes a serem pesquisadas e scrapeadas deverá ser construída uma respectiva API isolada (micro-serviço). O processo de construção desta plataforma deve ser feito em etapas com a utilização do gemini cli e moderação de um humano. O plano de ação geral e o estado de construção da plataforma deve ser armazenado em arquivo json a ser consultado pelo gemini cli durante a criação do código, permitindo que o processo seja continuado mesmo após interrupções. Ambiente de produção: GCP, Cloud Run, Firebase, Storage/Buckets, Cloud Schedules. Ambiente de desenvolvimento Windows. O processo de coleta será inicializado pelo usuário, a partir daí a plataforma deve ser capaz de funcionar de forma automática, realizando coletas e análises diárias inicializadas por chamadas do cloud schedules. Deve ser considerado o código existente como ponto de partida. Utilizar o mesmo banco de dados criado no módulo de search_google_cse nos demais módulos, compartilhando as suas coleções.

# Módulos/Etapas da plataforma:

*   **FRONTEND**
    *   Etapa inicial de inicio de monitoramento já construída. Disponível na pasta: `/frontend` .
    *   Detalhes técnicos, instruções de uso e relação com os demais módulos estão descritas no arquivo README.md
    *   Funcionalidades
        *   Usuário Define os termos da busca (marca e concorrentes).
        *   (NOVO) Usuário define termos-chave (ex: nomes, pautas) para monitoramento contínuo no Google Trends e suas respectivas geografias (ex: Brasil, Estado de SP).
        *   Define data histórica inicial para buscas históricas.
        *   Inicia etapa de buscas: relevante e histórica.

*   **SEARCH_GOOGLE_CSE**
    *   Etapa já construída. Disponível na pasta : `/backend` .
    *   Detalhes técnicos, instruções de uso e relação com os demais módulos estão descritas no arquivo README.md
    *   Funcionalidades
        *   Endpoint acionado diariamente às 12:00hs e as 20:00hs via Cloud Schedule para buscas de dados contínuos.
        *   Endpoint acionado diariamente às 21:00hs via Cloud Schedule para buscas de dados históricos.
        *   Pesquisa urls mais relevantes no momento.
        *   Pesquisa urls mais relevantes no período histórico definido.
        *   Salva urls em banco de dados indicando a fonte=google\_cse.
        *   Dispõe de endpoint que realiza pesquisa por urls novas (dateRestrict=d1).

*   **SEARCH_GOOGLE_TRENDS**
    *   Implementar módulo conforme descrito no documento de contexto: `/contextDocs/contextDoc_SearchGoogleTrends.md`.

*   **Scraper_newspaper3k**
    *   Etapa já construída. Disponível na pasta : `/scraper_newspaper3k`.
    *   Detalhes técnicos, instruções de uso e relação com os demais módulos estão descritas no arquivo README.md
    *   Funcionalidades
        *   Este módulo possui um Endpoint que deve ser acionado diariamente às 22:00hs via Cloud Schedule para realizar os respectivos scrapers.
        *   Estabelece um filtro inicial para as urls: caso o seu dominio seja youtube, instagram ou facebook devem ser desconsideradas. As mesmas possuirão scrapers específicos a serem desenvolvidos em etapa posterior.
        *   Caso passe do filtro anterior, analisa a relevância da url. Critérios para determinação de relevância: Presença do termo exato no title: peso 30. Presença do termo no snippet: peso 10. Domínio confiável ou autoridade alta: peso 25. URL amigável, sem muitos parâmetros: peso 5 . Título com múltiplos termos úteis: peso 10. Data da publicação (quando disponível): Peso 20. Ponto de corte = 0.60.
        *   Para as urls que passarão da linha de corte por relevância, tenta realizar o scraping.
        *   Se conseguiu realizar o scraping salva resultado em banco de dados e define o respecivo status como scraper\_ok para evitar reprocessamento.
        *   Se não conseguiu realizar o scraper registra esta informação no respectivo status inclusive registrando o motivo da não realização.

*   **NLP (Processamento de Linguagem Natural)**
    *   Etapa já construída. Disponível na pasta : `/api_nlp`.
    *   **ESCOPO EXPANDIDO:** Este módulo funcionará como o cérebro central de processamento de texto da plataforma, agnóstico à fonte dos dados. Ele será responsável por analisar textos provenientes tanto de artigos da web (`scraper_newspaper3k`) quanto de redes sociais (`Search_Instagram`, etc.).
    *   Detalhes técnicos, instruções de uso e relação com os demais módulos estão descritas no arquivo README.md. **(Nota: um novo documento de contexto será criado para detalhar as alterações)**
    *   Funcionalidades
        *   **Job 1 (Web Scraping):** Endpoint existente acionado diariamente às **23:00hs** via Cloud Schedule. Para cada uma das urls scrapeadas na etapa anterior (`status: scraper_ok`), realiza as análises e atualiza o status para `nlp_ok` ou `nlp_erro`.
        *   **Job 2 (Instagram - NOVO):** Um **novo Endpoint** será criado e acionado diariamente às **00:00hs** via Cloud Schedule. Este job irá buscar por documentos nas coleções `instagram_posts` e `instagram_comments` que ainda não foram processados (ex: campo `nlp_status` inexistente).
        *   Para cada documento de texto encontrado (seja de web ou Instagram), o módulo realiza:
            *   Análise de sentimento.
            *   Extração de entidades.
            *   Moderação de conteúdo.
        *   Salva as análises **enriquecendo os documentos originais** no Firestore com os resultados e atualiza o respectivo status (ex: `nlp_status: 'ok'`).

*   **ANALYTICS**
    *   Implementar módulo conforme descrito no documento de contexto: `/contextDocs/contextDoc_analytics.md`.

*   **SEARCH_INSTAGRAM**
    *   **ESCOPO DEFINIDO:** Módulo focado exclusivamente na **coleta de dados** do Instagram. Sua responsabilidade é popular o Firestore com dados brutos, que serão posteriormente consumidos por outros serviços, como o `api_nlp`.
    *   Implementar módulo conforme descrito no documento de contexto: `/contextDocs/contextDoc_instagram.md`.
    *   Endpoint acionado diariamente às 23:30hs via Cloud Schedule.
    *   Funcionalidades
        *   Coleta de posts, comentários e stories de perfis e hashtags monitoradas.
        *   Persistência dos dados brutos e metadados em coleções específicas do Firestore (ex: `instagram_posts`, `instagram_comments`).
        *   Upload de mídias (imagens/vídeos) para o Google Cloud Storage.
        *   **Os dados de texto coletados (legendas e comentários) serão processados de forma assíncrona pelo módulo `api_nlp`.**

*   **SEARCH_YOUTUBE**
    *   Implementar módulo conforme descrito no documento de contexto: `/contextDocs/contextDoc_youtube.md`

*   **SEARCH_FACEBOOK**
    *   Implementar módulo conforme descrito no documento de contexto: `/contextDocs/contextDoc_facebook.md`.

*   **WHATSAPP_GROUPS**
    *   Implementar módulo conforme descrito no documento de contexto: `/contextDocs/contextDoc_whatsapp.md`

*   **AGENT_MODE**
    *   Implementar módulo conforme descrito no documento de contexto: `/contextDocs/contextDoc_agentMode.md`

*   **SEMANTIC**
    *   Implementar módulo conforme descrito no documento de contexto: `/contextDocs/contextDoc_semantic.md`

*   **SEARCH_TWITTER**
    *   Implementar módulo conforme descrito no documento de contexto: `/contextDocs/contextDoc_twitter.md`

*   **SEARCH_TIKTOK**
    *   Implementar módulo conforme descrito no documento de contexto: `/contextDocs/contextDoc_tiktok.md`

*   **SEARCH_KWAII**
    *   Implementar módulo conforme descrito no documento de contexto: `/contextDocs/contextDoc_kwaii.md`

*   **SEARCH_LINKEDIN**
    *   Implementar módulo conforme descrito no documento de contexto: `/contextDocs/contextDoc_linkedin.md`