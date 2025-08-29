## **Documento de Contexto e Especificação: Módulo ANALYTICS**

### 1. Visão Geral e Filosofia de Design

O módulo **ANALYTICS** será o cérebro estratégico da plataforma. Sua função não é apenas exibir dados, mas transformá-los em *insights acionáveis* para a equipe do parlamentar. Cada gráfico, métrica e relatório deve responder a uma pergunta fundamental delineada no arquivo `analytics.md`, como "Nossa reputação está melhorando?" ou "Qual narrativa nossos oponentes estão impulsionando esta semana?".

Adotaremos três princípios de design:

1.  **Comparabilidade:** Toda análise deve permitir uma comparação direta e intuitiva entre a "marca" (o parlamentar) e os "concorrentes" (`search_group`).
2.  **Temporalidade:** A dimensão de tempo é crucial em política. A análise da evolução de métricas ao longo do tempo será um recurso central. O campo `publish_date` é a chave para isso.
3.  **Escalabilidade:** A arquitetura deve prever a entrada de novas fontes de dados (`origin`: Instagram, YouTube, etc.) sem a necessidade de grandes refatorações. A segmentação por `origin` será um filtro padrão.

### 2. Estrutura de Dados de Referência (`monitor_result.json`)

Para construir as análises, utilizaremos os seguintes campos-chave do documento processado (`status: "nlp_ok"`):

*   `search_group`: (String) Essencial para separar e comparar "marca" vs. "concorrentes".
*   `origin`: (String) Permite segmentar a análise por fonte (hoje `google_cse`, amanhã `instagram`, `youtube`, etc.).
*   `publish_date`: (Timestamp) O eixo X para todas as análises temporais.
*   `google_nlp_analysis.sentiment`: (String: "positivo", "negativo", "neutro") A base para a análise de reputação.
*   `google_nlp_analysis.entities`: (Array de Strings) O "DNA" do conteúdo. Revela os principais temas, pessoas, locais e pautas mencionados.
*   `google_nlp_analysis.moderation_results`: (Array de Objetos) Útil para identificar automaticamente conteúdo potencialmente tóxico ou sensível, servindo de base para alertas de crise.
*   `displayLink`: (String) Identifica o veículo ou site de publicação, permitindo a análise de "Share of Voice" por fonte.

### 3. Features e Métricas Estratégicas (O "Quê")

A seguir, detalhamos as features do dashboard, mapeando-as diretamente aos objetivos estratégicos.

#### **3.1. Dashboard Principal: Pulso da Reputação Digital**

*   **Objetivo Estratégico:** Monitoramento de Imagem e Reputação.
*   **Componentes:**
    1.  **Sentimento Geral (Gráfico de Pizza/Donut):**
        *   **Descrição:** Visão geral da proporção de menções positivas, negativas e neutras para a marca e para cada concorrente em um período selecionado (ex: últimos 7 dias).
        *   **Dados Utilizados:** `google_nlp_analysis.sentiment`, `search_group`.
    2.  **Evolução do Sentimento (Gráfico de Linhas):**
        *   **Descrição:** O gráfico mais importante do dashboard. Mostra a evolução diária do volume de menções negativas, positivas e neutras ao longo do tempo. Permite identificar picos de crise ou repercussão positiva de uma ação específica.
        *   **Dados Utilizados:** `publish_date` (eixo X), contagem de documentos (eixo Y), `google_nlp_analysis.sentiment` (série/cor da linha).
    3.  **Nuvem de Entidades por Sentimento (Word Cloud):**
        *   **Descrição:** Duas nuvens de palavras lado a lado. Uma exibe as entidades mais frequentes em menções **negativas**, e a outra, em menções **positivas**. Responde rapidamente a: "Sobre o que falam quando falam mal de nós?".
        *   **Dados Utilizados:** `google_nlp_analysis.entities`, agregados e filtrados por `google_nlp_analysis.sentiment`.

#### **3.2. Análise de Pautas e Território de Narrativas**

*   **Objetivo Estratégico:** Monitoramento de Pautas e Tendências, Apoio à Estratégia de Comunicação.
*   **Componentes:**
    1.  **Top Entidades Mencionadas (Gráfico de Barras):**
        *   **Descrição:** Ranking das entidades (temas, pessoas, projetos de lei) mais associadas à marca e aos concorrentes. Essencial para entender a pauta dominante.
        *   **Dados Utilizados:** `google_nlp_analysis.entities`, `search_group`.
    2.  **Mapa de Coocorrência de Entidades (Grafo de Rede - *Feature Avançada*):**
        *   **Descrição:** Visualização que mostra como as entidades se conectam. Se "PEC 48/2023" aparece frequentemente junto com "direitos indígenas" e o nome do parlamentar, este gráfico torna essa conexão visualmente explícita. Ajuda a entender o contexto da discussão.
        *   **Dados Utilizados:** Análise de frequência de pares de `google_nlp_analysis.entities` dentro do mesmo documento.
    3.  **Fontes Mais Ativas (Tabela ou Gráfico de Barras):**
        *   **Descrição:** Ranking dos veículos (`displayLink`) que mais publicam sobre a marca e concorrentes. Permite identificar aliados, detratores e a imprensa neutra.
        *   **Dados Utilizados:** `displayLink`, `search_group`.

#### **3.3. Inteligência Competitiva**

*   **Objetivo Estratégico:** Inteligência de Oposição e Concorrência Eleitoral.
*   **Componentes:**
    1.  **Share of Voice (Gráfico de Área Empilhada):**
        *   **Descrição:** Mostra a "fatia do debate" que pertence ao parlamentar versus cada concorrente ao longo do tempo. Mede a visibilidade e o protagonismo no cenário digital.
        *   **Dados Utilizados:** `publish_date`, contagem de documentos por `search_group`.
    2.  **Dashboard Comparativo de Sentimento (Gráfico de Barras Agrupadas):**
        *   **Descrição:** Lado a lado, para cada competidor (incluindo a marca), exibe a distribuição percentual de sentimento (Positivo/Negativo/Neutro). Uma forma rápida de comparar a reputação de todos os players.
        *   **Dados Utilizados:** `google_nlp_analysis.sentiment`, `search_group`.
    3.  **Feed de Ataques (Tabela Filtrada):**
        *   **Descrição:** Uma tabela que exibe apenas as menções com sentimento `negativo` e que contenham o nome do parlamentar E o nome de um concorrente na lista de `entities`. Permite à equipe de resposta rápida focar diretamente nas narrativas adversárias.
        *   **Dados Utilizados:** `search_group`, `google_nlp_analysis.sentiment`, `google_nlp_analysis.entities`.

### 4. Considerações de Arquitetura e Implementação (O "Como")

Para que o Frontend possa exibir esses dados de forma eficiente sem sobrecarregar o Firebase/Firestore a cada requisição, a arquitetura deve ser planejada.

1.  **Backend for Frontend (BFF) / API de Analytics:**
    *   O Frontend não deve consultar o Firestore diretamente para gerar as análises. Devemos criar um novo microsserviço (ex: `api_analytics` em Cloud Run) ou adicionar endpoints ao `backend` existente.
    *   Este serviço será responsável por conter toda a lógica de negócio das consultas, agregações e transformações dos dados.
    *   **Exemplo de Endpoint:** `GET /api/analytics/sentiment-over-time?group=marca&start_date=2025-08-01&end_date=2025-08-28&origin=google_cse`

2.  **Performance e Pré-agregação (Materialized Views):**
    *   Realizar agregações complexas em tempo real em uma grande coleção de documentos (`monitor_results`) é ineficiente e caro.
    *   **Solução:** Criar uma **estratégia de pré-agregação**. Uma Cloud Function, acionada pelo Cloud Scheduler (ex: a cada hora), pode processar os novos documentos (`status: "nlp_ok"`) e atualizar uma coleção de sumarização (ex: `analytics_summary`).
    *   **Estrutura de `analytics_summary`:**
        ```json
        // Exemplo de documento em 'analytics_summary'
        {
          "summary_id": "marca_2025-08-27", // ID composto: grupo + data
          "date": "2025-08-27",
          "search_group": "marca",
          "total_mentions": 150,
          "sentiment_counts": {
            "positivo": 40,
            "negativo": 85,
            "neutro": 25
          },
          "top_entities": [
            { "entity": "Congresso Nacional", "count": 120 },
            { "entity": "PEC 48/2023", "count": 95 }
          ],
          "mentions_by_source": {
            "google_cse": 150
            // Futuramente: "instagram": 320, "youtube": 80
          }
        }
        ```
    *   Os endpoints da `api_analytics` consultariam primariamente esta coleção `analytics_summary`, resultando em dashboards extremamente rápidos e custos operacionais menores. As consultas em tempo real na coleção principal seriam reservadas para análises mais granulares e específicas.

Este documento fornece a base estratégica e a direção técnica para construir um módulo de **ANALYTICS** robusto, relevante e performático, alinhado perfeitamente com os objetivos do projeto.