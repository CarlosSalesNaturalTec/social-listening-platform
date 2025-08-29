### **Documento de Contexto e Especificação: Módulo de Analytics**

**ID do Documento:** `context-analytics-v1.0`
**Autor:** Especialista Híbrido (Estratégia de Comunicação & Arquitetura de Software)
**Objetivo:** Especificar os requisitos funcionais, técnicos, endpoints de API e lógica de negócio para a construção do módulo `ANALYTICS` da plataforma de Social Listening. Este documento traduz as necessidades estratégicas de análise política em entregáveis técnicos, baseando-se no schema de dados já processado pelos módulos de scraping e NLP.

---

### **1. Visão Geral e Princípios de Arquitetura**

O módulo `ANALYTICS` será desenvolvido como um micro-serviço independente em Python (utilizando FastAPI), containerizado com Docker e implantado no Google Cloud Run. Ele não fará scraping nem processamento de linguagem natural; sua única função é ler os dados já enriquecidos do banco de dados (Firebase/Firestore), realizar agregações, cálculos e expor essa inteligência através de uma API RESTful para ser consumida pelo frontend.

*   **Fonte de Dados Primária:** Banco de dados Firestore, coleção `monitor_results`. O módulo irá consultar documentos com status `nlp_ok`.
*   **Performance:** Para dashboards e análises recorrentes, os resultados agregados podem ser cacheados ou materializados em uma coleção separada (`analytics_summary`) para otimizar os custos de leitura e a latência da API. Este processo de agregação pode ser acionado por um Cloud Scheduler.
*   **Escalabilidade:** A arquitetura serverless do Cloud Run garante que o serviço escale automaticamente com a demanda de requisições do frontend.

---

### **2. Estrutura de Dados de Entrada (Schema de Referência)**

O módulo `ANALYTICS` irá operar sobre documentos JSON com a seguinte estrutura, conforme exemplificado em `monitor_result.json`:

```json
{
  "link": "https://...",
  "title": "...",
  "scraped_content": "...",
  "publish_date": { "_seconds": 1756339200, ... },
  "search_group": "competitors", // ou "brand"
  "status": "nlp_ok",
  "google_nlp_analysis": {
    "sentiment": "neutro", // "positivo", "negativo"
    "entities": [ "Congresso Nacional", "terras indígenas", ... ],
    "moderation_results": [ { "category": "Politics", "confidence": 0.23 }, ... ]
  }
}
```
**Campos-chave para a Análise:**
*   `publish_date`: Essencial para toda análise de séries temporais.
*   `search_group`: Permite a segmentação entre a marca monitorada e seus concorrentes/opositores.
*   `google_nlp_analysis.sentiment`: A base para a medição de percepção pública.
*   `google_nlp_analysis.entities`: O núcleo para identificar pautas, temas, pessoas e organizações mencionadas.
*   `displayLink`: Para identificar as fontes (mídia) que mais repercutem os assuntos.

---

### **3. Funcionalidades e Endpoints da API**

A seguir, a especificação dos endpoints que o serviço `ANALYTICS` deverá prover. Cada endpoint mapeia diretamente para uma necessidade descrita no documento `analytics.md`.

#### **3.1. Monitoramento de Imagem e Reputação**

**Endpoint 1: `GET /api/analytics/sentiment-over-time`**
*   **Descrição:** Retorna uma série temporal do volume de menções classificadas por sentimento (positivo, negativo, neutro). Crucial para entender a evolução da percepção pública e identificar picos de crise ou repercussão positiva.
*   **Parâmetros:**
    *   `startDate` (string, formato ISO)
    *   `endDate` (string, formato ISO)
    *   `target` (string, opcional, ex: "brand", "competitor_name") - Filtra por `search_group`.
    *   `granularity` (string, opcional, "day", "week", "month") - Define o agrupamento da série temporal.
*   **Resposta (JSON):**
    ```json
    {
      "results": [
        { "date": "2025-08-20", "positive": 15, "negative": 25, "neutral": 60 },
        { "date": "2025-08-21", "positive": 18, "negative": 78, "neutral": 85 }
      ]
    }
    ```

**Endpoint 2: `GET /api/analytics/crisis-detector`**
*   **Descrição:** Analisa as últimas 24/48 horas para identificar anomalias, como um aumento súbito no volume de menções negativas. Essencial para a "Gestão de Crises".
*   **Lógica:** Compara o volume de menções negativas das últimas 24h com a média das 72h anteriores. Se o aumento percentual ultrapassar um limiar (ex: 150%) e o volume absoluto for significativo (ex: > 50 menções), retorna um alerta.
*   **Parâmetros:** Nenhum.
*   **Resposta (JSON):**
    ```json
    {
      "alert_status": "high", // "none", "medium", "high"
      "reason": "Aumento de 210% em menções negativas nas últimas 24h.",
      "volume_increase": 2.1,
      "negative_mentions_24h": 78,
      "top_crisis_entities": ["PEC 48/2023", "Marco Temporal", "Dinamam Tuxá"],
      "sample_links": ["http://..."]
    }
    ```

#### **3.2. Mapeamento de Pautas e Relacionamento com Eleitores**

**Endpoint 3: `GET /api/analytics/top-entities`**
*   **Descrição:** Retorna as entidades (temas, pessoas, organizações) mais frequentemente mencionadas nos artigos, permitindo o "Mapeamento de interesses" do eleitorado.
*   **Parâmetros:**
    *   `startDate`, `endDate`, `target`.
    *   `sentimentFilter` (string, opcional, "positive", "negative") - Filtra entidades por sentimento associado.
*   **Resposta (JSON):**
    ```json
    {
      "entities": [
        { "name": "Congresso Nacional", "count": 150, "sentiment_distribution": {"positive": 10, "negative": 80, "neutral": 60} },
        { "name": "Povos Indígenas", "count": 120, "sentiment_distribution": {"positive": 50, "negative": 40, "neutral": 30} }
      ]
    }
    ```

#### **3.3. Inteligência de Oposição e Concorrência**

**Endpoint 4: `GET /api/analytics/competitor-snapshot`**
*   **Descrição:** Fornece uma comparação lado a lado do volume de menções e sentimento entre a marca e seus concorrentes/opositores.
*   **Parâmetros:**
    *   `startDate`, `endDate`.
*   **Resposta (JSON):**
    ```json
    {
      "comparison": [
        {
          "target": "brand",
          "total_mentions": 1250,
          "avg_sentiment": { "positive": 0.2, "negative": 0.4, "neutral": 0.4 }
        },
        {
          "target": "competitor_A", // Nome do concorrente
          "total_mentions": 980,
          "avg_sentiment": { "positive": 0.3, "negative": 0.3, "neutral": 0.4 }
        }
      ]
    }
    ```

#### **3.4. Medição de Desempenho e Estratégia**

**Endpoint 5: `GET /api/analytics/top-sources`**
*   **Descrição:** Identifica os veículos de mídia (`displayLink`) que mais publicam sobre os termos monitorados, ajudando a encontrar influenciadores e entender a paisagem midiática.
*   **Parâmetros:**
    *   `startDate`, `endDate`, `target`.
*   **Resposta (JSON):**
    ```json
    {
      "sources": [
        { "domain": "agenciabrasil.ebc.com.br", "mentions": 45 },
        { "domain": "g1.globo.com", "mentions": 32 }
      ]
    }
    ```
---

### **4. Conclusão para Desenvolvimento**

Este documento estabelece a base para a criação do módulo `ANALYTICS`. O próximo passo é traduzir estes endpoints em um código Python robusto, com funções claras para consultar o Firestore, processar os dados (preferencialmente com a biblioteca Pandas para agregação) e formatar as respostas JSON conforme especificado. A implementação deve priorizar a clareza, a testabilidade e a eficiência das consultas ao banco de dados para garantir um serviço performático e de baixo custo operacional no ambiente GCP.