# **Documento de Contexto: Módulo SEARCH_GOOGLE_TRENDS**

## 1. Visão Geral e Objetivo

O módulo **SEARCH_GOOGLE_TRENDS** é uma API de microserviço, construída em Python sobre o Google Cloud Run, cuja responsabilidade central é coletar e estruturar dados de intenção de busca do Google Trends. Ele atua como o "termômetro do interesse público", capturando o que o eleitorado está ativamente pesquisando, em contraste com o que está sendo discutido passivamente nas redes sociais.

O objetivo principal deste módulo é alimentar a coleção `google_trends_data` no Firestore com dados consistentes e atualizados, que serão consumidos primariamente pelo módulo `ANALYTICS` para gerar insights correlacionados, identificar pautas emergentes, detectar crises de imagem precocemente e realizar análises comparativas de "capital de atenção" entre o parlamentar e seus opositores.

## 2. Posicionamento na Arquitetura

Este módulo opera como um coletor de dados agendado e também como um serviço sob demanda, ocupando uma posição fundamental no pipeline de dados da plataforma.

*   **Fonte de Instrução:** O módulo lê os termos a serem monitorados (marca, concorrentes, pautas-chave) e suas respectivas geografias de uma coleção de configurações no Firestore, que é gerenciada pelo **FRONTEND**.
*   **Processamento:** Utilizando a biblioteca `pytrends`, o serviço é acionado por **Cloud Scheduler** para execuções periódicas (diárias e horárias) e também expõe endpoints para consultas diretas.
*   **Saída de Dados:** Todos os dados coletados são padronizados e salvos na coleção `google_trends_data` no Firestore.
*   **Consumidor Principal:** O módulo **ANALYTICS** é o principal consumidor dos dados gerados, utilizando-os para construir os gráficos e artefatos visuais definidos em seu documento de contexto.

**Fluxo de Dados:**

`FRONTEND` -> `Firestore (monitoring_settings)` -> `Cloud Scheduler` -> **`SEARCH_GOOGLE_TRENDS (Cloud Run)`** -> `Firestore (google_trends_data)` -> `ANALYTICS`

## 3. Lógica de Negócio e Funcionalidades

As funcionalidades do módulo são projetadas para atender às necessidades específicas do `ANALYTICS`, conforme detalhado no plano de ação.

### a) Monitoramento de Interesse ao Longo do Tempo (Execução Diária)

*   **Gatilho:** Cloud Scheduler, diariamente às 08:00.
*   **Lógica:**
    1.  Lê a lista de termos principais (marca, concorrentes) da coleção de configurações.
    2.  Para cada termo, consulta o Google Trends para obter a série temporal de "interesse de busca" dos últimos 7 dias.
    3.  Formata e salva os dados na coleção `google_trends_data` com o `type: 'interest_over_time'`.
*   **Propósito:** Fornecer a linha de base para o "Gráfico de Correlação: Menções & Interesse de Busca", o principal artefato do dashboard do `ANALYTICS`.

### b) Detecção de Pautas Emergentes (Execução Horária)

*   **Gatilho:** Cloud Scheduler, a cada hora.
*   **Lógica:**
    1.  Lê a lista de termos principais.
    2.  Para cada termo, executa uma consulta de "buscas em ascensão" (`rising queries`).
    3.  Se forem encontradas buscas com crescimento notável (especialmente "Breakout"), elas são formatadas e salvas na coleção `google_trends_data` com o `type: 'rising_queries'`.
*   **Propósito:** Agir como um sistema de alerta precoce. Alimenta o artefato "Buscas em Ascensão" no `ANALYTICS`, permitindo que a equipe de comunicação se antecipe a crises ou capitalize em pautas virais.

### c) Análise Competitiva Comparativa (Endpoint Sob Demanda)

*   **Gatilho:** Chamada de API direta, provavelmente vinda do backend do `ANALYTICS`.
*   **Lógica:**
    1.  Recebe uma lista de até 5 termos (ex: parlamentar e 4 concorrentes), um período de tempo e uma geografia via parâmetros da API.
    2.  Executa uma única consulta comparativa no Google Trends.
    3.  Retorna um objeto JSON estruturado com as séries temporais de interesse para cada termo.
*   **Propósito:** Fornecer os dados para o artefato "Análise Comparativa de Interesse" do `ANALYTICS`, permitindo a comparação direta do "capital de atenção" no mercado político.

## 4. Estrutura de Dados (Schema - Firestore)

A estrutura de dados proposta no `contextDoc_analytics.md` é um excelente ponto de partida e será adotada com pequenos refinamentos para garantir clareza e eficiência na consulta.

**Coleção:** `google_trends_data`

### Documento Exemplo (`type: 'interest_over_time'`)

```json
{
  "term": "Nome do Parlamentar",
  "geo": "BR",
  "type": "interest_over_time",
  "timeframe": "7-d",
  "run_id": "uuid_da_execucao_diaria",
  "created_at": "2025-08-28T08:00:00Z",
  "data": [
    { "date": "2025-08-21T00:00:00Z", "value": 75, "formattedValue": "75" },
    { "date": "2025-08-22T00:00:00Z", "value": 82, "formattedValue": "82" },
    { "date": "2025-08-23T00:00:00Z", "value": 79, "formattedValue": "79" }
  ]
}
```

### Documento Exemplo (`type: 'rising_queries'`)

```json
{
  "term": "Nome do Parlamentar",
  "geo": "BR-SP",
  "type": "rising_queries",
  "timeframe": "1-h",
  "run_id": "uuid_da_execucao_horaria",
  "created_at": "2025-08-28T14:00:00Z",
  "data": [
    { "query": "nome do parlamentar projeto x", "value": 3550, "formattedValue": "+3.550%" },
    { "query": "esposa do parlamentar", "value": 900, "formattedValue": "+900%" },
    { "query": "parlamentar votou contra", "value": 250000, "formattedValue": "Breakout" }
  ]
}
```

## 5. Especificação da API (Endpoints)

O módulo exporá endpoints para serem acionados por serviços internos (Cloud Scheduler) e outros módulos (Analytics).

*   **Endpoint de Execução Diária:**
    *   **Método:** `POST`
    *   **Path:** `/tasks/run-daily-interest`
    *   **Descrição:** Acionado pelo Cloud Scheduler para buscar o interesse ao longo do tempo para os termos configurados. A lógica interna busca os termos do Firestore.
    *   **Resposta:** `200 OK` com um resumo da operação.

*   **Endpoint de Execução Horária:**
    *   **Método:** `POST`
    *   **Path:** `/tasks/run-hourly-rising`
    *   **Descrição:** Acionado pelo Cloud Scheduler para buscar por pautas em ascensão.
    *   **Resposta:** `200 OK` com um resumo da operação.

*   **Endpoint de Comparação:**
    *   **Método:** `GET`
    *   **Path:** `/api/compare`
    *   **Descrição:** Permite ao módulo `ANALYTICS` solicitar uma comparação direta entre múltiplos termos.
    *   **Parâmetros de Query:**
        *   `terms` (string, separado por vírgula): Ex: "Parlamentar A,Concorrente B,Concorrente C"
        *   `geo` (string, opcional, default='BR'): Ex: "BR-SP"
        *   `start_date` (string, formato YYYY-MM-DD)
        *   `end_date` (string, formato YYYY-MM-DD)
    *   **Resposta (JSON):**
        ```json
        {
          "Parlamentar A": [
            {"date": "2025-08-21", "value": 80}, ...
          ],
          "Concorrente B": [
            {"date": "2025-08-21", "value": 65}, ...
          ],
          "Concorrente C": [
            {"date": "2025-08-21", "value": 72}, ...
          ]
        }
        ```

## 6. Considerações Técnicas e de Operação

*   **Biblioteca:** `pytrends` será a biblioteca Python utilizada para a interface com o Google Trends.
*   **Rate Limiting:** O Google Trends pode aplicar rate limiting. A lógica do serviço deve incluir tratamento de exceções (ex: `TooManyRequestsError` da `pytrends`) e, se necessário, implementar um backoff exponencial para evitar bloqueios. As consultas devem ser otimizadas para minimizar o número de requisições.
*   **Configuração:** O serviço será containerizado (Dockerfile) e implantado no Cloud Run. As configurações, como o ID do projeto GCP , serão gerenciadas via variáveis de ambiente.
*   **Autenticação:** O serviço utilizará uma Conta de Serviço (Service Account) com as permissões IAM necessárias para ler a coleção de configurações e escrever na coleção `google_trends_data`. Os endpoints de `tasks` devem ser protegidos para aceitar chamadas apenas do Cloud Scheduler.