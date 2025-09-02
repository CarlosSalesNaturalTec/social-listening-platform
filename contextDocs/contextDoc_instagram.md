## **Documento de Contexto Técnico: Módulo Search_Instagram - versão 5.0.0**

### 1. Visão Geral e Arquitetura do Módulo

O **Search_Instagram** é um módulo de inteligência e escuta ativa, projetado como um micro-serviço desacoplado. Sua responsabilidade principal é a coleta, enriquecimento primário e armazenamento de dados públicos do Instagram. Ele operará de forma autônoma, acionado por agendamentos, e exporá os dados processados para o restante da plataforma através de coleções no Firestore.

**Pilha Tecnológica:**

*   **Linguagem:** Python 3.9+
*   **Framework API:** FastAPI (para criação de endpoints e gerenciamento de Background Tasks)
*   **Biblioteca de Coleta:** Instaloader
*   **Hospedagem:** Google Cloud Run (conteneurizado com Docker)
*   **Banco de Dados (Metadados):** Google Firestore
*   **Armazenamento de Mídia (Data Lake):** Google Cloud Storage (GCS)
*   **Agendamento:** Google Cloud Scheduler
*   **Gerenciamento de Segredos:** Google Secret Manager (para armazenar arquivos de sessão)

**Componentes Principais:**

1.  **Endpoint de API (Cloud Run):** Um único serviço FastAPI com endpoints específicos para cada tipo de job (ex: `/jobs/start-daily-scan`).
2.  **Mecanismo de Coleta (Instaloader Core):** O coração do módulo, contendo a lógica de interação com o Instaloader.
3.  **Conectores de Persistência:** Módulos responsáveis por salvar metadados no Firestore, mídias no Cloud Storage e gerenciar sessões via Secret Manager.

---

### 2. Gerenciamento de Contas e Sessões

#### 2.1. Gerenciamento Seguro de Sessões com Google Secret Manager

A autenticação no Instagram é o ponto mais sensível do módulo. O arquivo de sessão gerado pelo Instaloader contém tokens de acesso que devem ser tratados como segredos de alta prioridade. Armazená-los em banco de dados ou no código é inaceitável. A solução correta é utilizar o Google Secret Manager.

**Fluxo de Armazenamento (via Frontend/Admin):**

1.  **Geração Local:** Um administrador gera o arquivo de sessão localmente usando o Instaloader (`instaloader --login=SUA_CONTA`).
2.  **Upload Seguro:** No frontend da plataforma, haverá uma área de "Gerenciamento de Contas de Serviço". Ao cadastrar uma nova conta, o administrador fará o upload do arquivo de sessão (`<username>`) através de um formulário.
3.  **Endpoint de Ingestão:** O frontend envia o arquivo para um endpoint seguro no backend (ex: `/service-accounts/upload-session`).
4.  **Criação do Segredo:** O backend recebe o arquivo, lê seu conteúdo binário e utiliza a biblioteca cliente do Google Cloud para criar uma nova versão de um *secret* no Secret Manager. O nome do segredo será padronizado, como `instagram-session-seu-projeto-username`.
5.  **Persistência do Caminho:** Após a criação bem-sucedida, o backend armazena o **caminho completo do recurso** do segredo (ex: `projects/ID_PROJETO/secrets/NOME_SEGREDO/versions/1`) no campo `secret_manager_path` do documento correspondente na coleção `service_accounts` do Firestore.

**Fluxo de Utilização (pelo Worker de Coleta):**

1.  **Seleção da Conta:** O job de coleta seleciona uma conta ativa da coleção `service_accounts`.
2.  **Recuperação do Caminho:** O worker lê o valor do campo `secret_manager_path`.
3.  **Acesso ao Segredo:** Utilizando a biblioteca cliente, o worker acessa a versão específica do segredo e baixa seu payload (o conteúdo do arquivo de sessão).
4.  **Escrita Temporária:** O conteúdo é escrito em um arquivo temporário no ambiente de execução do Cloud Run (ex: `/tmp/<username>`). O sistema de arquivos `/tmp` é montado na memória, garantindo que o arquivo não persista após a execução da instância.
5.  **Carga pelo Instaloader:** A instância do Instaloader é instruída a carregar a sessão a partir deste arquivo temporário (`L.load_session_from_file(username, '/tmp/username')`).
6.  **Limpeza:** Em um bloco `finally`, o arquivo temporário é obrigatoriamente deletado para garantir que nenhuma credencial permaneça no sistema, mesmo em caso de erro.

#### 2.2. Detecção e Tratamento de Sessões Inválidas 
Sessões de autenticação são frágeis e podem ser invalidadas a qualquer momento (expiração de tempo, mudança de senha, ação de segurança). O sistema deve ser projetado para detectar essa falha, sinalizá-la e facilitar a recuperação, em vez de simplesmente falhar silenciosamente.

**Fluxo de Detecção e Sinalização (pelo Worker de Coleta):**

A lógica de coleta no backend será encapsulada em um tratamento de exceções robusto, capaz de diferenciar um bloqueio temporário de uma falha de autenticação permanente.

1. **Gatilho**: Durante a inicialização ou execução de um job, ao tentar realizar uma ação que exija login (ex: L.test_login()), a biblioteca Instaloader levantará uma exceção específica, como LoginRequiredException ou PrivateModeNotPossibleException, caso a sessão seja inválida.

2. **Captura da Exceção**: O worker de coleta irá capturar essa exceção específica.

3. **Ação Imediata**: Ao capturar o erro de autenticação, o worker executará as seguintes ações críticas:
    *   **Logar o Evento:** Registrará um erro de alta prioridade na coleção `system_logs`, **seguindo o padrão definido na Seção 3**. O log incluirá, no mínimo: `service: "Search_Instagram"`, `job_type: "session_validation"`, `status: "error"`, `message: "Sessão para a conta {username} expirou..."` e `error_message` com a exceção capturada.
    * Atualizar o Estado no Firestore: Esta é a etapa mais importante. O worker fará um update no documento da conta correspondente na coleção service_accounts, alterando o valor do campo status para session_expired.
    * Abortar o Job: O job de coleta para aquela conta será encerrado graciosamente, pois não há como prosseguir sem uma sessão válida.

---

### 3. Modelo de Dados: Coleções do Firestore

| Coleção | ID do Documento | Campos Notáveis | Propósito |
| :--- | :--- | :--- | :--- |
| **`service_accounts`** | `UUID` | `username` (string), `secret_manager_path` (string), `status` (string: active/banned), `last_used_at` (timestamp) | **(Frontend)** Gerenciar o pool de contas do Instagram. Valores possíveis para status: <br> - active: Conta operacional e pronta para uso. <br> - session_expired: A sessão é inválida e requer a geração e upload de um novo arquivo. <br> - banned: A conta foi detectada como bloqueada/banida pelo Instagram (requer intervenção manual externa). |
| **`monitored_profiles`** | `instagram_username` | `type` (string: parlamentar, concorrente, midia), `is_active` (boolean), `last_scanned_at` (timestamp) | **(Frontend)** Cadastro dos perfis-alvo. |
| **`monitored_hashtags`** | `hashtag_sem_cerquilha` | `is_active` (boolean), `last_scanned_at` (timestamp) | **(Frontend)** Cadastro das hashtags-alvo. |
| **`instagram_posts`** | `post.shortcode` | `owner_username`, `caption`, `post_date_utc`, `likes_count`, `comments_count`, `media_type`, `gcs_media_path`, `collected_at`, <br>---<br> *`nlp_status` (string)* <br> *`sentiment_score` (float)* <br> *`entities` (array)* | Armazena metadados de cada post. Os campos em itálico são adicionados pelo `api_nlp`. |
| **`instagram_comments`** | `comment.id` | `text`, `username`, `likes_count`, `comment_date_utc`, <br>---<br> *`nlp_status` (string)* <br> *`sentiment_score` (float)* <br> *`entities` (array)* | Sub-coleção de `instagram_posts`. Os campos em itálico são adicionados pelo `api_nlp`. |
| **`instagram_stories`** | `story.mediaid` | `owner_username`, `story_date_utc`, `expires_at_utc`, `media_type`, **`gcs_media_path` (string)**, `collected_at` | Armazena metadados de cada Story. |
| **`system_logs`** | `UUID` | **`run_id`** (string): ID único para uma execução de job.<br>**`service`** (string): Nome do micro-serviço (ex: "Search\_Instagram").<br>**`job_type`** (string): Tarefa específica (ex: "daily\_profile\_scan").<br>**`status`** (string): `started`, `completed`, `error`, `warning`.<br>**`start_time`** (timestamp): Início da execução.<br>**`end_time`** (timestamp): Fim da execução.<br>**`message`** (string): Mensagem legível.<br>**`error_message`** (string): Detalhes do erro, se houver.<br>**`metrics`** (map): Objeto com métricas numéricas (ex: `{"processed": 5, "new_posts": 150}`). | Coleção centralizada para log de auditoria, performance e depuração de todos os micro-serviços da plataforma. |

---

### 4. Fluxo de Persistência de Mídia (Google Cloud Storage)

Coletar apenas os metadados é insuficiente; a análise do conteúdo visual (imagens/vídeos) é fundamental. Esses arquivos binários não devem ser armazenados no Firestore. O Google Cloud Storage (GCS) servirá como nosso *Data Lake* para esses ativos.

**Estratégia de Armazenamento:**

O processo será integrado ao loop de coleta de posts e stories. Para cada item de mídia encontrado, o worker executará os seguintes passos:

1.  **Download para Memória:** Em vez de salvar o arquivo no disco do Cloud Run, o worker fará o download da mídia (imagem ou vídeo) diretamente para um buffer na memória (um objeto `BytesIO` em Python). Isso é mais eficiente e adequado para ambientes sem estado.
2.  **Definição do Caminho de Destino:** Uma convenção de nomenclatura clara e organizada é essencial para a governança dos dados. Usaremos a seguinte estrutura de caminho no GCS:
    `instagram/[TIPO_CONTEUDO]/[NOME_PERFIL]/[ANO]-[MES]/[ID_UNICO].[EXTENSAO]`
    *   **Exemplo Post:** `instagram/posts/nome_parlamentar/2025-09/Cq2x5v3jR4p.jpg`
    *   **Exemplo Story:** `instagram/stories/nome_parlamentar/2025-09/3154879_5487895.mp4`
    *   `[TIPO_CONTEUDO]` será `posts` ou `stories`.
    *   `[ID_UNICO]` será o `shortcode` para posts e o `mediaid` para stories.
3.  **Upload para o GCS:** Usando a biblioteca cliente do Google Cloud Storage para Python, o worker fará o upload do buffer da memória para o bucket designado com o caminho definido no passo anterior.
4.  **Atualização do Firestore:** Após a confirmação do upload bem-sucedido, o worker obterá o caminho completo do GCS para o objeto (ex: `gs://seu-bucket-de-midia/instagram/.../Cq2x5v3jR4p.jpg`). Este caminho será salvo no campo `gcs_media_path` do respectivo documento no Firestore (`instagram_posts` ou `instagram_stories`).

Esta abordagem garante que o Firestore contenha apenas metadados leves e referências, enquanto o GCS lida com o armazenamento escalável e de baixo custo dos arquivos pesados. A ligação entre os dois é a chave `gcs_media_path`.

---

### 5. Dados a Serem Coletados pelo Instaloader

A coleta será dividida por tipo de conteúdo, extraindo os seguintes atributos:

#### **A. Objeto `Post` (Feed e Reels)**
*   `shortcode`: Código único do post. **(Chave Primária)**
*   `owner_username`: Nome do usuário que publicou.
*   `caption`: Legenda do post.
*   `date_utc`: Data da publicação.
*   `likes`: Quantidade de curtidas.
*   `comments`: Quantidade de comentários.
*   `typename`: Identificador do tipo de post ("GraphImage", "GraphVideo", "GraphSidecar").
*   `is_video`: Booleano para identificar se é vídeo.
*   `video_url` / `url`: URL direta da mídia para download.

#### **B. Objeto `Comment` (dentro de um Post)**
*   `id`: ID único do comentário. **(Chave Primária)**
*   `owner.username`: Nome do usuário que comentou.
*   `owner.followers`: Número de seguidores do autor do comentário.
*   `owner.followees`: Número de contas que o autor segue.
*   `owner.biography`: Biografia do autor do comentário.
*   `owner.is_private`: Se o perfil do autor é privado.
*   `text`: O texto do comentário.
*   `created_at_utc`: Data do comentário.
*   `likes_count`: Quantidade de curtidas no comentário.

#### **C. Objeto `StoryItem`**
*   `mediaid`: ID único do Story. **(Chave Primária)**
*   `owner_username`: Nome do usuário que publicou.
*   `date_utc`: Data da publicação do Story.
*   `is_video`: Booleano para identificar se é vídeo.
*   `video_url` / `url`: URL direta da mídia para download.

---

### 6. Features do Frontend

O frontend deverá prover as seguintes interfaces para gerenciar e visualizar os dados coletados:

1.  **Gerenciamento de Contas de Serviço:**
    *   Formulário para cadastrar uma nova conta (`username`).
    *   Mecanismo para upload do arquivo de sessão do Instaloader, que o backend salvará de forma segura no Secret Manager e associará à conta.
    *   Listagem das contas com seus status (Ativa, Banida), permitindo editar e remover.
    *   Listagem de Status em Tempo Real:
        * A listagem das contas deverá exibir visualmente o status atual de cada uma, lido diretamente do Firestore.
        * Verde (active): Conta saudável.
        * Alerta Laranja/Amarelo (session_expired): Indica um problema que pode ser resolvido pela interface. Acompanhado de um texto "Sessão Expirada".
        * Alerta Vermelho (banned): Indica um problema grave que requer ação fora da plataforma. Acompanhado de um texto "Conta Bloqueada".
    * Fluxo de Renovação de Sessão:
        * Para qualquer conta com o status session_expired, um botão "Atualizar Sessão" ficará visível.
        * Ao clicar, o usuário será apresentado a um formulário de upload de arquivo.
        * Ação do Usuário: O administrador deverá gerar um novo arquivo de sessão localmente (executando instaloader --login=CONTA_EXPIRADA) e fazer o upload do novo arquivo.
        * Ação do Backend: Um endpoint (/service-accounts/update-session) receberá o arquivo, criará uma nova versão do secret no Google Secret Manager e, após o sucesso, atualizará o status da conta no Firestore de volta para active.

2.  **Gerenciamento de Alvos:**
    *   CRUD completo para `monitored_profiles`, permitindo adicionar/remover perfis e classificá-los.
    *   CRUD completo para `monitored_hashtags`.

3.  **Dashboard do Instagram:**

#### 3.1 Pilha Tecnológica de Frontend (Libs e Ferramentas)

*   **Framework Principal:** React (via Next.js para performance e renderização otimizada).
*   **Bibliotecas de Gráficos:** **ApexCharts.js** ou **Chart.js**. Ambas são excelentes.
    *   **ApexCharts:** Oferece gráficos SVG modernos, interativos e com ótima aparência por padrão. Minha recomendação principal.
    *   **Chart.js:** Mais simples, usa Canvas, extremamente popular e com vasta documentação. Ótima para começar.
*   **Manipulação de Datas:** `date-fns` - Leve e poderosa para lidar com as análises de séries temporais.
*   **Visualização de Nuvem de Palavras:** `react-wordcloud`.
*   **Tabelas de Dados:** `TanStack Table` (anteriormente React Table) para tabelas complexas, com ordenação e filtragem.

#### 3.2 - Estrutura do Dashboard (Layout Multi-Abas)

Para evitar poluição visual, o dashboard será dividido em abas, cada uma com um foco analítico claro.

1.  **Aba 1: Pulso do Dia (Visão Geral)**
2.  **Aba 2: Análise de Desempenho (Perfil Próprio)**
3.  **Aba 3: Inteligência Competitiva (Concorrentes)**
4.  **Aba 4: Radar de Pautas (Hashtags e Mídia)**

#### 3.3. Detalhamento dos Componentes por Aba

**Nota de Integração de Dados**: Os dashboards a seguir foram projetados para consumir uma combinação de dados. Métricas de engajamento (likes, comentários) são coletadas diretamente pelo Search_Instagram. Métricas analíticas, como sentimento, entidades nomeadas e moderação de conteúdo, são adicionadas posteriormente aos documentos do Firestore pelo micro-serviço api_nlp. O frontend deve consultar os documentos enriquecidos, assumindo a presença dos campos detalhados na tabela abaixo.

##### **Campos Adicionados pelo Módulo `api_nlp`**

| Campo no Firestore | Tipo de Dado | Origem (Campo de Texto) | Descrição | Exemplo de Uso |
| :--- | :--- | :--- | :--- | :--- |
| `nlp_status` | String | N/A | Status do processamento (`ok`, `error`). | Filtrar dados já processados. |
| `sentiment_score` | Float (-1.0 a 1.0) | `caption` / `text` | Pontuação do sentimento. < 0 é negativo, > 0 é positivo. | Gráficos de Balanço de Sentimento, filtros de apoiadores/críticos. |
| `sentiment_magnitude` | Float (0 a ∞) | `caption` / `text` | A "força" do sentimento, independentemente da polaridade. | Identificar comentários/posts com linguagem muito forte (crises/oportunidades). |
| `entities` | Array de Objetos | `caption` / `text` | Entidades (pessoas, locais, pautas) extraídas do texto. | Alimentar Nuvens de Palavras e tabelas de "Principais Termos". |
| `moderation` | Array de Objetos | `caption` / `text` | Categorias de conteúdo potencialmente sensível (discurso de ódio, etc.). | Alertas de moderação, identificar ataques coordenados. |


##### **Aba 1: Pulso do Dia (Visão Geral)**

**Objetivo Estratégico:** Fornecer ao parlamentar e sua equipe um resumo de 5 minutos sobre o status do ecossistema digital nas últimas 24 horas.

| Componente (Gráfico) | Tipo de Gráfico | Dados e Campos Utilizados | Insights Gerados |
| :--- | :--- | :--- | :--- |
| **KPIs Principais (Cards)** | Cards de Destaque | `instagram_posts` (filtrados por `post_date_utc` nas últimas 24h). `COUNT(*)`, `SUM(likes_count)`, `SUM(comments_count)`. | Visão imediata do volume de atividade e engajamento do dia. |
| **Stories Recentes (Últimas 24h)** | Carrossel Visual de Stories | `instagram_stories` filtrados por `story_date_utc` (últimas 24h) e por `owner_username` de perfis com `type` em ['parlamentar', 'concorrente', 'midia']. Utiliza `gcs_media_path`. | Qual foi a comunicação visual e efêmera do dia, tanto própria quanto dos concorrentes e da mídia? Permite uma reação rápida a narrativas de curta duração. |
| **Balanço de Sentimento (Últimas 24h)** | Gráfico de Rosca (Donut Chart) | `instagram_comments`. Agregação por faixas de `sentiment_score` (ex: < -0.25 = Negativo, > 0.25 = Positivo). | Qual foi a tonalidade geral da conversa sobre você e suas pautas hoje? |
| **Alerta de Crise / Oportunidade** | Tabela Simples | `instagram_posts` e `instagram_comments`. Filtro por picos de sentimento negativo ou posts com engajamento > 3x a média. | "O post X está recebendo um volume anormal de comentários negativos." "O post Y viralizou positivamente." |
| **Principais Termos do Dia** | Nuvem de Palavras (Word Cloud) | **`instagram_comments.entities`** e **`instagram_posts.entities`** (das últimas 24h). | Quais **pautas e nomes** dominaram a conversa hoje? (Muito mais preciso que palavras soltas). |

---

##### **Aba 2: Análise de Desempenho (Perfil Próprio)**

**Objetivo Estratégico:** Entender a fundo a performance da própria comunicação, identificando o que funciona e o que não funciona.

| Componente (Gráfico) | Tipo de Gráfico | Dados e Campos Utilizados | Insights Gerados |
| :--- | :--- | :--- | :--- |
| **Evolução do Engajamento** | Gráfico de Linhas (Multi-série) | **Eixo X:** `instagram_posts.post_date_utc`. **Eixo Y:** `SUM(likes_count)` e `SUM(comments_count)`. | Nosso engajamento está crescendo ou diminuindo ao longo do tempo (semana, mês, trimestre)? |
| **Performance por Tipo de Conteúdo** | Gráfico de Barras Agrupadas | Agregação de `AVG(likes_count)` e `AVG(comments_count)` por `instagram_posts.media_type` (Imagem, Vídeo, Carrossel). | Qual formato de post gera mais engajamento? Devemos investir mais em Reels? Nossos carrosséis didáticos estão funcionando? |
| **Ranking de Posts** | Tabela Interativa | `instagram_posts` (ordenável por `likes_count`, `comments_count`). | Quais foram nossos posts de maior sucesso? E os de menor sucesso? Permite analisar os melhores e piores casos. |
| **Análise de Sentimento dos Comentários** | Gráfico de Barras Empilhadas | **Eixo X:** `instagram_posts.shortcode`. **Eixo Y:** `COUNT(instagram_comments)` empilhado por `sentiment_score`. | Permite ver visualmente quais posts geraram mais reações positivas ou negativas. |
| **Top 5 Apoiadores** | Tabela | `instagram_comments` (agrupado por `username`, `COUNT(*)`, filtrado por sentimento positivo). | Quem são nossos maiores defensores? Podemos engajá-los ou amplificar suas vozes? |
| **Top 5 Críticos** | Tabela | `instagram_comments`. Agrupado por `username`, ordenado por `COUNT(*) * AVG(ABS(sentiment_score))` onde `sentiment_score < -0.25`. | Quem são nossos detratores mais impactantes (frequentes e intensos)? |
| **Mapa de Influência dos Comentaristas** | Gráfico de Dispersão (Scatter Plot) | **Eixo X:** `COUNT(comments) por username`. **Eixo Y:** `AVG(user_followers)`. | Identifica visualmente os atores mais importantes: quem comenta muito com poucos seguidores (militante) vs. quem comenta pouco mas tem muitos seguidores (influenciador). |

---

##### **Aba 3: Inteligência Competitiva (Concorrentes)**

**Objetivo Estratégico:** Fazer um benchmarking claro e direto contra os adversários políticos, identificando suas estratégias, forças e fraquezas.

| Componente (Gráfico) | Tipo de Gráfico | Dados e Campos Utilizados | Insights Gerados |
| :--- | :--- | :--- | :--- |
| **Head-to-Head de Engajamento** | Gráfico de Linhas (Multi-série) | **Eixo X:** `post_date_utc`. **Eixo Y:** `SUM(likes_count + comments_count)`. Cada linha representa um perfil monitorado. | Quem está ganhando a batalha do engajamento diário/semanal? Houve algum pico de um concorrente que precisamos analisar? |
| **Comparativo de Estratégia de Conteúdo** | Gráfico de Barras Empilhadas 100% | **Eixo X:** `owner_username`. **Eixo Y:** `COUNT(posts)` empilhado por `media_type`. | Qual a proporção de Vídeos vs. Imagens que meus concorrentes usam? A estratégia deles é diferente da minha? |
| **Nuvem de Palavras Comparativa** | Duas Nuvens de Palavras Lado a Lado | **`instagram_posts.entities`** de cada perfil. | Quais são as **pautas e temas centrais** usados por mim vs. meu principal concorrente? Revela foco de narrativa. |
| **Identificação de Vulnerabilidades** | Tabela | `instagram_posts` (de concorrentes) filtrados por posts com alto `comments_count` e baixo `sentiment_score` médio. | "O Concorrente X postou sobre segurança e recebeu uma avalanche de críticas. Esta é uma oportunidade para nos posicionarmos." |

---

##### **Aba 4: Radar de Pautas (Hashtags e Mídia)**

**Objetivo Estratégico:** Sair da "bolha" dos perfis políticos e entender o que o cidadão, a imprensa local e os influenciadores estão falando.

| Componente (Gráfico) | Tipo de Gráfico | Dados e Campos Utilizados | Insights Gerados |
| :--- | :--- | :--- | :--- |
| **Feed da Hashtag** | Feed de Posts (visual) | `instagram_posts` filtrado por `monitored_hashtags`. | O que as pessoas estão postando em tempo real sobre `#SegurançaEmMinhaCidade`? Permite ver fotos e vídeos "crus" da realidade. |
| **Sentimento da Pauta ao Longo do Tempo** | Gráfico de Linhas | **Eixo X:** `post_date_utc`. **Eixo Y:** `AVG(sentiment_score)` para posts/comentários de uma hashtag específica. | A percepção pública sobre a `#ReformaTributaria` está melhorando ou piorando no Instagram? |
| **Principais Influenciadores da Pauta** | Tabela | `instagram_posts` (de uma hashtag) agrupado por `owner_username`, ordenado por `SUM(likes_count)`. | Quem são as vozes mais relevantes (não-políticas) falando sobre este tema? São aliados ou opositores em potencial? |

Ao implementar este design, você entregará um produto que não apenas informa, mas capacita a ação estratégica. A equipe do parlamentar poderá, com poucos cliques, passar de uma visão macro do cenário para uma análise micro de um único comentário influente, tornando a comunicação digital muito mais proativa e baseada em dados.

---

### 7. Estratégias Anti-Bloqueio (Implementação Detalhada)

Esta é a parte mais crítica do módulo. A implementação deve seguir rigorosamente as seguintes estratégias:

1.  **Pool e Rotação de Contas de Serviço:**
    *   **Lógica:** Antes de iniciar um job de coleta, o sistema consultará a coleção `service_accounts`. Ele selecionará a conta com o `last_used_at` mais antigo e status `active`.
    *   **Implementação:** O worker principal receberá o nome de usuário da conta a ser usada, buscará seu arquivo de sessão no Secret Manager e o carregará na instância do Instaloader. Ao final do job, o `last_used_at` da conta será atualizado no Firestore.

2.  **Cadência Humana (Pausas Aleatórias):**
    *   **Lógica:** O código terá funções de "delay" inteligentes que serão chamadas em pontos estratégicos do processo de coleta.
    *   **Implementação:**
        ```python
        import random
        import time

        def human_like_pause(min_seconds=5, max_seconds=15):
            delay = random.uniform(min_seconds, max_seconds)
            time.sleep(delay)

        # Exemplo de uso:
        # for post in profile.get_posts():
        #     process_post(post)
        #     human_like_pause(8, 22) # Pausa mais longa entre posts
        ```
    *   **Pausas de Lote:** Após processar um perfil inteiro ou um lote de 20 posts de uma hashtag, uma pausa maior (ex: `human_like_pause(180, 300)`) será acionada.

3.  **Tratamento de Erros e Backoff:**
        
    *   **Lógica:** O código de coleta será envolvido em um bloco `try...except` que trata exceções de forma hierárquica e específica.
    *   **Implementação:**
    ```python
    from instaloader.exceptions import TooManyRequestsException, LoginRequiredException
    # Supondo que 'username' e um 'run_id' único estão no escopo

    try:
        # ... lógica de validação de sessão e coleta ...

    except LoginRequiredException as e:
        # ERRO FATAL DE AUTENTICAÇÃO
        log_system_event(
            run_id=run_id,
            service="Search_Instagram",
            job_type="session_validation",
            status='error', 
            message=f'Sessão para {username} é inválida.',
            error_message=str(e)
        )
        # Atualiza o status no Firestore para 'session_expired'
        update_service_account_status(username, 'session_expired') 
        # Encerra o job para esta conta
        return

    except TooManyRequestsException as e:
        # ERRO TEMPORÁRIO DE RATE LIMIT
        log_system_event(
            run_id=run_id,
            service="Search_Instagram",
            job_type="data_collection",
            status='warning', 
            message='TooManyRequestsException recebida. Iniciando backoff.',
            error_message=str(e)
        )
        # Entrar em modo de espera longa (backoff)
        time.sleep(random.uniform(900, 1800))
    ```

4.  **Coleta Defensiva:**
    *   **Lógica:** Limitar por padrão a quantidade de dados coletados para o estritamente necessário, especialmente para operações "caras" como a coleta de comentários.
    *   **Implementação:** Utilizar `itertools.islice` para limitar a coleta de comentários ao teto de 100, conforme especificado nos requisitos.
        ```python
        import itertools
        
        comments_iterator = post.get_comments()
        for comment in itertools.islice(comments_iterator, 100):
            # ... processar comentário ...
        ```

### **8. Requisitos de Infraestrutura (Google Cloud Run)**

Para garantir a estabilidade e a eficiência de custos do serviço, o deploy no Google Cloud Run deve seguir as seguintes configurações iniciais. Estas configurações são baseadas na análise da carga de trabalho, que é limitada por I/O de rede e uso de memória para buffers de arquivo.

| Parâmetro | Configuração Inicial Recomendada | Justificativa |
| :--- | :--- | :--- |
| **Memória** | **512 MiB** | Garante uma margem segura para o sistema operacional, o serviço Python e o buffer de download de mídias (vídeos), prevenindo falhas de "Out of Memory". |
| **CPU** | **1 vCPU** | A tarefa não é intensiva em CPU. 1 vCPU com capacidade de "burst" é mais do que suficiente para a carga de trabalho, otimizando os custos. |
| **Simultaneidade (Concurrency)** | **1** | Crítico para a estabilidade. Garante que cada instância processe apenas uma requisição por vez, evitando a exaustão de memória ao tentar baixar múltiplos arquivos simultaneamente. |
| **Tempo Limite da Requisição** | **600 segundos (10 minutos)** | Oferece tempo suficiente para a coleta de perfis grandes e o download de vídeos em conexões mais lentas, evitando timeouts prematuros do job. |
| **Instâncias Mínimas** | **0** | Habilita o escalonamento a zero, a principal vantagem de custo do Cloud Run. O serviço não consumirá recursos quando estiver ocioso. |
| **Instâncias Máximas** | **3** | Atua como uma salvaguarda de custos. Como os jobs são agendados, uma alta escalabilidade não é necessária e um limite baixo previne picos de custo por erros. |

**Estratégia de Otimização:** Após o deploy inicial, a utilização real de memória deve ser monitorada na aba de Métricas do serviço para validar e, se necessário, ajustar a alocação de memória (para cima ou para baixo) com base em dados de produção.