## **Documento de Contexto Técnico: Módulo Search_Instagram**

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

1.  **Endpoint de API (Cloud Run):** Um único serviço FastAPI com endpoints específicos para cada tipo de job (ex: `/jobs/start-daily-scan`, `/jobs/start-stories-scan`). Esses endpoints recebem a requisição do Cloud Scheduler, validam, registram o início do job no `system_logs` e imediatamente disparam a lógica de coleta como uma **Background Task**. Isso garante que o Scheduler receba uma resposta `200 OK` rapidamente, evitando timeouts.
2.  **Mecanismo de Coleta (Instaloader Core):** O coração do módulo, contendo a lógica de interação com o Instaloader, incluindo o gerenciamento de sessões, estratégias anti-bloqueio e parsing dos dados.
3.  **Conectores de Persistência:** Módulos responsáveis por salvar os dados estruturados (metadados) no Firestore e os arquivos de mídia (imagens/vídeos) no Cloud Storage.

---

### 2. Modelo de Dados: Coleções do Firestore

As seguintes coleções serão criadas ou utilizadas para dar suporte ao módulo. O ID do Documento será a chave primária natural sempre que possível para garantir idempotência e fácil acesso.

| Coleção | ID do Documento | Campos Notáveis | Propósito |
| :--- | :--- | :--- | :--- |
| **`service_accounts`** | `UUID` | `username` (string), `secret_manager_path` (string), `status` (string: active/banned), `last_used_at` (timestamp) | **(Frontend)** Gerenciar o pool de contas do Instagram usadas para a coleta. |
| **`monitored_profiles`** | `instagram_username` | `type` (string: parlamentar, concorrente, midia), `is_active` (boolean), `last_scanned_at` (timestamp) | **(Frontend)** Cadastro dos perfis-alvo para monitoramento. |
| **`monitored_hashtags`** | `hashtag_sem_cerquilha` | `is_active` (boolean), `last_scanned_at` (timestamp) | **(Frontend)** Cadastro das hashtags-alvo para monitoramento. |
| **`instagram_posts`** | `post.shortcode` | `owner_username`, `caption`, `post_date_utc`, `likes_count`, `comments_count`, `media_type`, `gcs_media_path`, `collected_at` | Armazena metadados de cada post (Feed, Reel) coletado. |
| **`instagram_comments`** | `comment.id` | `text`, `username`, `user_followers`, `user_followees`, `user_biography`, `user_is_private`, `likes_count`, `comment_date_utc` | **Sub-coleção de `instagram_posts`**. Armazena os comentários de um post específico. |
| **`instagram_stories`** | `story.mediaid` | `owner_username`, `story_date_utc`, `expires_at_utc`, `media_type`, `gcs_media_path`, `collected_at` | Armazena metadados de cada Story coletado. |
| **`system_logs`** | `UUID` | `timestamp`, `service` (string: "Search_Instagram"), `job_type`, `status` (string: started, completed, error), `message` | Log de auditoria e depuração para as execuções dos jobs. |

---

### 3. Dados a Serem Coletados pelo Instaloader

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

### 4. Features do Frontend

O frontend deverá prover as seguintes interfaces para gerenciar e visualizar os dados coletados:

1.  **Gerenciamento de Contas de Serviço:**
    *   Formulário para cadastrar uma nova conta (`username`).
    *   Mecanismo para upload do arquivo de sessão do Instaloader, que o backend salvará de forma segura no Secret Manager e associará à conta.
    *   Listagem das contas com seus status (Ativa, Banida), permitindo editar e remover.

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

##### **Aba 1: Pulso do Dia (Visão Geral)**

**Objetivo Estratégico:** Fornecer ao parlamentar e sua equipe um resumo de 5 minutos sobre o status do ecossistema digital nas últimas 24 horas.

| Componente (Gráfico) | Tipo de Gráfico | Dados e Campos Utilizados | Insights Gerados |
| :--- | :--- | :--- | :--- |
| **KPIs Principais (Cards)** | Cards de Destaque | `instagram_posts` (filtrados por `post_date_utc` nas últimas 24h). `COUNT(*)`, `SUM(likes_count)`, `SUM(comments_count)`. | Visão imediata do volume de atividade e engajamento do dia. |
| **Balanço de Sentimento (Últimas 24h)** | Gráfico de Rosca (Donut Chart) | `instagram_comments` (agregados por `sentiment_score`). | Qual foi a tonalidade geral da conversa sobre você e suas pautas hoje? Positiva, negativa ou neutra? |
| **Alerta de Crise / Oportunidade** | Tabela Simples | `instagram_posts` e `instagram_comments`. Filtro por picos de sentimento negativo ou posts com engajamento > 3x a média. | "O post X está recebendo um volume anormal de comentários negativos." "O post Y viralizou positivamente." |
| **Principais Termos do Dia** | Nuvem de Palavras (Word Cloud) | `instagram_comments.text` e `instagram_posts.caption` (das últimas 24h). | Quais palavras e temas dominaram a conversa hoje? Ajuda a identificar pautas emergentes rapidamente. |

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
| **Top 5 Críticos** | Tabela | `instagram_comments` (agrupado por `username`, `COUNT(*)`, filtrado por sentimento negativo). | Quem são nossos detratores mais consistentes? Seus argumentos são válidos? Representam uma ameaça? |
| **Mapa de Influência dos Comentaristas** | Gráfico de Dispersão (Scatter Plot) | **Eixo X:** `COUNT(comments) por username`. **Eixo Y:** `AVG(user_followers)`. | Identifica visualmente os atores mais importantes: quem comenta muito com poucos seguidores (militante) vs. quem comenta pouco mas tem muitos seguidores (influenciador). |

---

##### **Aba 3: Inteligência Competitiva (Concorrentes)**

**Objetivo Estratégico:** Fazer um benchmarking claro e direto contra os adversários políticos, identificando suas estratégias, forças e fraquezas.

| Componente (Gráfico) | Tipo de Gráfico | Dados e Campos Utilizados | Insights Gerados |
| :--- | :--- | :--- | :--- |
| **Head-to-Head de Engajamento** | Gráfico de Linhas (Multi-série) | **Eixo X:** `post_date_utc`. **Eixo Y:** `SUM(likes_count + comments_count)`. Cada linha representa um perfil monitorado. | Quem está ganhando a batalha do engajamento diário/semanal? Houve algum pico de um concorrente que precisamos analisar? |
| **Comparativo de Estratégia de Conteúdo** | Gráfico de Barras Empilhadas 100% | **Eixo X:** `owner_username`. **Eixo Y:** `COUNT(posts)` empilhado por `media_type`. | Qual a proporção de Vídeos vs. Imagens que meus concorrentes usam? A estratégia deles é diferente da minha? |
| **Nuvem de Palavras Comparativa** | Duas Nuvens de Palavras Lado a Lado | `instagram_posts.caption` de cada perfil. | Quais são as palavras e temas mais usados por mim vs. meu principal concorrente? Revela foco de narrativa. |
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

### 5. Estratégias Anti-Bloqueio (Implementação Detalhada)

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
    *   **Lógica:** O código de coleta será envolvido em um bloco `try...except` robusto, especificamente para capturar `TooManyRequestsException`.
    *   **Implementação:**
        ```python
        from instaloader.exceptions import TooManyRequestsException

        try:
            # ... lógica de coleta ...
        except TooManyRequestsException:
            # 1. Logar o erro no system_logs
            log_system_event(status='error', message='TooManyRequestsException received.')
            # 2. Marcar a conta de serviço como "em espera" no Firestore (opcional)
            # 3. Entrar em modo de espera longa (backoff)
            time.sleep(random.uniform(900, 1800)) # Espera de 15 a 30 minutos
            # 4. Tentar novamente a operação ou encerrar o job graciosamente
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
