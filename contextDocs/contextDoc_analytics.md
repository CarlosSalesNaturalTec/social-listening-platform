# **Documento de Contexto: Módulo ANALYTICS**

## 1. Visão Geral e Objetivo

O módulo **ANALYTICS** é o cérebro da plataforma de Social Listening. Sua função principal é consumir os dados pré-processados pelos módulos de `SEARCH_*`, `SCRAPER` e `NLP`, e, crucialmente, correlacioná-los com os dados de interesse de busca do `SEARCH_GOOGLE_TRENDS`.

O objetivo é transformar essa massa de dados em dashboards visuais, insights estratégicos e gatilhos de ação (para o `AGENT_MODE`), permitindo que a equipe do parlamentar compreenda a percepção pública, antecipe crises, identifique pautas emergentes e meça o impacto de suas ações de comunicação de forma integrada.

Este módulo será uma API (Python/Flask ou FastAPI) que servirá os dados agregados para o frontend e executará as lógicas de correlação.

## 2. Dependências de Dados

Para funcionar, o ANALYTICS depende fundamentalmente de duas coleções no Firestore:

1.  `monitor_results`: Contém os dados de menções coletadas de todas as fontes (Google CSE, Instagram, YouTube, etc.), já enriquecidos com análise de sentimento e extração de entidades pelo módulo `NLP`. A estrutura base é a do `monitor_result.json`.
2.  `google_trends_data`: **(Proposta de Aquisição de Novos Dados)** Esta coleção será populada pelo módulo `SEARCH_GOOGLE_TRENDS`. É **essencial** para o sucesso do ANALYTICS. Sua estrutura deve ser planejada para armazenar tanto séries temporais quanto dados de picos de interesse.

### **Proposta de Schema para a Coleção `google_trends_data`**

```json
// Documento na coleção 'google_trends_data'
{
  "term": "Nome do Parlamentar", // Termo que foi buscado
  "geo": "BR-SP", // Geografia da busca (País, Estado)
  "type": "interest_over_time", // ou "rising_queries", "related_topics"
  "run_id": "uuid_da_execucao",
  "created_at": "timestamp",
  "data": [ // Payload varia com o 'type'
    // Exemplo para 'interest_over_time'
    { "date": "2025-08-20T00:00:00Z", "value": 75 },
    { "date": "2025-08-21T00:00:00Z", "value": 82 }
  ]
}
```

## 3. Especificação do Frontend e Visualização de Dados

O frontend apresentará os insights em uma interface com abas temáticas, permitindo uma navegação clara e focada. Todas as abas compartilharão filtros globais:

*   **Seleção de Entidade:** Dropdown para escolher entre a "Marca" (o parlamentar) ou "Concorrentes".
*   **Seleção de Período:** Botões para filtrar os dados por "Últimos 7 dias", "Últimos 30 dias", "Últimos 90 dias" e um seletor de intervalo customizado.

### **Plano de Implementação Incremental**

Implementar um gráfico/artefato de cada vez, garantindo a qualidade e validação de cada componente antes de passar para o próximo.

---

### **Aba 1: Dashboard Principal (Visão Geral)**

O pulso da reputação digital em tempo real.

*   **Artefato 1: Gráfico de Correlação: Menções & Interesse de Busca**
    *   **Tipo:** Gráfico de Linhas Combinado.
    *   **Descrição:** Este é o principal gráfico da plataforma. Ele exibirá duas séries de dados no mesmo eixo de tempo:
        1.  **Volume de Menções:** Uma linha mostrando a contagem diária de menções (soma de todos os `origin` em `monitor_results`).
        2.  **Interesse de Busca (Google Trends):** Uma segunda linha, com cor distinta, mostrando o índice de interesse de busca (de 0 a 100) para o termo selecionado (`marca` ou `concorrente`).
    *   **Biblioteca Sugerida:** `Plotly.js` (para interatividade, como tooltips que mostram os valores exatos e eventos importantes naquele dia).
    *   **Interpretação (Texto no Frontend):** "Analise a correlação entre o que é dito nas redes e o que é buscado no Google. Picos de menções que são seguidos por picos de busca indicam que a narrativa digital está transbordando para o interesse do público geral. Um pico de busca sem um aumento nas menções pode indicar um evento offline (como uma entrevista na TV) cujo impacto online pode ser medido aqui."

*   **Artefato 2: KPIs (Key Performance Indicators)**
    *   **Tipo:** Cartões de Destaque.
    *   **Descrição:** Exibir números consolidados para o período selecionado:
        *   **Volume Total de Menções:** Contagem total de documentos em `monitor_results`.
        *   **Sentimento Médio:** Média do score de sentimento, exibido com um ícone (positivo, neutro, negativo).
        *   **Alcance Estimado:** (Futuro) Soma de seguidores/visualizações das menções. **Proposta:** Requer que os scrapers coletem dados de perfil do autor da menção.
    *   **Interpretação (Texto no Frontend):** "Tenha uma visão rápida da sua saúde digital. Acompanhe o volume de conversas e a percepção geral sobre você ou seus concorrentes."

---

### **Aba 2: Análise de Pautas e Narrativas**

Entenda *sobre o que* as pessoas estão falando.

*   **Artefato 1: Nuvem de Entidades Principais**
    *   **Tipo:** Nuvem de Palavras (Word Cloud).
    *   **Descrição:** Uma representação visual das entidades mais frequentemente extraídas dos conteúdos (`google_nlp_analysis.entities`). O tamanho de cada termo é proporcional à sua frequência.
    *   **Biblioteca Sugerida:** `WordCloud` (Python) para gerar a imagem no backend, ou `d3-cloud` no frontend.
    *   **Interpretação (Texto no Frontend):** "Visualize os temas e termos mais associados à sua marca ou concorrente no período. Termos grandes representam as pautas dominantes na conversa."

*   **Artefato 2: Tabela de Menções Relevantes**
    *   **Tipo:** Tabela Paginada e Pesquisável.
    *   **Descrição:** Lista as menções individuais, ordenadas por data ou relevância. Inclui colunas para: Conteúdo (`snippet`), Fonte (`origin`), Data (`publish_date`) e Sentimento. Clicar em uma entidade na nuvem de palavras deve filtrar esta tabela para mostrar apenas as menções que contêm aquela entidade.
    *   **Interpretação (Texto no Frontend):** "Mergulhe nas conversas individuais. Filtre por pautas específicas para entender o contexto e a origem de cada narrativa."

---

### **Aba 3: Inteligência de Google Trends**

Insights diretos das intenções de busca do eleitorado.

*   **Artefato 1: Buscas em Ascensão (Rising Queries)**
    *   **Tipo:** Tabela Simples.
    *   **Descrição:** Exibe os termos de busca relacionados ao parlamentar que tiveram um crescimento súbito e significativo, classificados como "BREAKOUT". Os dados virão de `google_trends_data` onde `type == 'rising_queries'`.
    *   **Interpretação (Texto no Frontend):** "Este é seu sistema de alerta precoce. Termos 'em ascensão' indicam pautas emergentes ou o início de uma crise de imagem. Aja rapidamente para se posicionar ou preparar uma resposta."

*   **Artefato 2: Análise Comparativa de Interesse**
    *   **Tipo:** Gráfico de Linhas Múltiplas.
    *   **Descrição:** Compara o interesse de busca (série temporal do Google Trends) entre a "Marca" e até 3 "Concorrentes" selecionados.
    *   **Biblioteca Sugerida:** `Chart.js` ou `Plotly.js`.
    *   **Interpretação (Texto no Frontend):** "Compare seu 'capital de atenção' com o de seus opositores. Identifique quem está dominando o interesse público em diferentes momentos e analise os eventos que causaram picos ou quedas para cada um."

---

### **Aba 4: Análise de Sentimento**

Um mergulho profundo na percepção do público.

*   **Artefato 1: Distribuição de Sentimento**
    *   **Tipo:** Gráfico de Pizza ou Rosca (Donut Chart).
    *   **Descrição:** Mostra a porcentagem de menções classificadas como Positivas, Negativas e Neutras no período selecionado.
    *   **Biblioteca Sugerida:** `Chart.js`.
    *   **Interpretação (Texto no Frontend):** "Entenda a proporção da percepção pública. Uma grande fatia negativa é um sinal claro de crise ou desgaste de imagem."

*   **Artefato 2: Evolução do Sentimento no Tempo**
    *   **Tipo:** Gráfico de Linhas Empilhadas.
    *   **Descrição:** Mostra a evolução diária do volume de menções, com as barras de cada dia segmentadas por cor para representar a quantidade de menções positivas, negativas e neutras.
    *   **Biblioteca Sugerida:** `Plotly.js`.
    *   **Interpretação (Texto no Frontend):** "Acompanhe como o sentimento do público flutua ao longo do tempo. Identifique os dias exatos em que ocorreram picos de negatividade para investigar as causas e o impacto de eventos específicos."

## 4. Especificação do Backend (Endpoints da API)

O módulo ANALYTICS deverá expor os seguintes endpoints para alimentar o frontend:

*   `GET /api/analytics/combined_view`
    *   **Parâmetros:** `term`, `start_date`, `end_date`
    *   **Resposta:** JSON contendo duas arrays: uma para o volume de menções por dia e outra para os dados de interesse do Google Trends por dia.

*   `GET /api/analytics/kpis`
    *   **Parâmetros:** `term`, `start_date`, `end_date`
    *   **Resposta:** JSON com os valores consolidados: `total_mentions`, `average_sentiment`.

*   `GET /api/analytics/entities_cloud`
    *   **Parâmetros:** `term`, `start_date`, `end_date`
    *   **Resposta:** JSON com uma lista de entidades e suas frequências: `[{"text": "PEC 48", "value": 150}, ...]`

*   `GET /api/analytics/mentions`
    *   **Parâmetros:** `term`, `start_date`, `end_date`, `page`, `entity` (opcional)
    *   **Resposta:** Lista paginada de documentos de menções.

*   `GET /api/analytics/trends_comparison`
    *   **Parâmetros:** `terms` (lista de termos), `start_date`, `end_date`
    *   **Resposta:** JSON com uma série temporal para cada termo.

*   *E assim por diante para cada artefato visual...*

## 5. Integração com o AGENT_MODE

O módulo ANALYTICS também rodará processos em background (via Cloud Scheduler, por exemplo, a cada hora) para verificar as condições de gatilho definidas em `contextDoc_agentMode.md`.

*   **Exemplo de Lógica:**
    1.  Consultar os dados mais recentes de `google_trends_data` para termos negativos pré-cadastrados.
    2.  Consultar os dados mais recentes de `monitor_results` para calcular o sentimento geral na última hora.
    3.  Se `(interesse_busca_trends para 'termo_negativo' > 80) E (sentimento_geral < -0.5)`, então fazer uma chamada HTTP para um endpoint do módulo `AGENT_MODE` para `acionar_protocolo_crise()`, passando os dados relevantes como payload.