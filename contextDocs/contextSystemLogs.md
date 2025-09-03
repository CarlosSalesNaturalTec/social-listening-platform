## **Documento de Contexto Técnico: Refatoração e Padronização do Módulo de Logs (`system_logs`)**

### 1. Objetivo

Este documento descreve o plano de ação para refatorar e padronizar a estrutura de logs em toda a plataforma. O objetivo é migrar do modelo de log heterogêneo existente para um **padrão único e enriquecido**, centralizado na coleção `system_logs` do Firestore. Esta padronização é crucial para melhorar a observabilidade, simplificar a depuração e permitir a criação de dashboards de monitoramento de performance da plataforma.

### 2. Situação Atual e Problemas

Atualmente, os módulos existentes (`api_nlp`, `scraper_newspaper3k`, `search_google_trends`) geram logs com estruturas variadas. Um exemplo típico é:

```json
{
  "task": "Google Trends - Hourly Rising",
  "start_time": "...",
  "processed_count": 0,
  "status": "completed"
}
```

Esta abordagem apresenta as seguintes limitações:
*   **Falta de Identificação de Serviço:** É difícil filtrar todos os logs de um único micro-serviço.
*   **Falta de Estrutura para Métricas:** Dados numéricos importantes estão misturados em campos de texto livre, impedindo a análise quantitativa.
*   **Ausência de um ID de Execução:** Não há uma maneira fácil de agrupar todos os eventos de log de uma única execução de job.

### 3. Solução Proposta: O Novo Padrão `system_logs`

Todos os módulos deverão adotar a seguinte estrutura para cada documento de log inserido na coleção `system_logs`:

| Campo | Tipo de Dado | Obrigatório | Descrição |
| :--- | :--- | :--- | :--- |
| **`run_id`** | String (UUID) | Sim | ID único gerado no início da execução de um job. Deve ser o mesmo para todos os logs (`status: started`, `status: completed`, etc.) da mesma execução. |
| **`service`** | String | Sim | Nome padronizado do micro-serviço. Ex: `api_nlp`, `scraper_newspaper3k`. |
| **`job_type`** | String | Sim | Identificador da tarefa específica dentro do serviço. Ex: `web_scraping_nlp`, `instagram_nlp`. |
| **`status`** | String | Sim | Estado do evento: `started`, `completed`, `error`, `warning`. |
| **`start_time`** | Timestamp | Sim | Momento de início da execução do job. |
| **`end_time`** | Timestamp | Não | Momento de término. Presente em logs de `status: completed` ou `status: error`. |
| **`message`** | String | Sim | Mensagem descritiva e legível para humanos sobre o evento. |
| **`error_message`** | String | Não | Detalhes da exceção ou do erro, presente apenas quando `status: error`. |
| **`metrics`** | Map (Objeto) | Não | Objeto para agrupar todas as métricas numéricas relevantes. Ex: `{"processed": 10, "targets": 12, "success_rate": 0.83}`. |

### 4. Plano de Ação para Migração

Para cada módulo listado abaixo, as seguintes etapas devem ser executadas:

1.  **Localizar Funções de Log:** Identificar todas as funções ou trechos de código responsáveis por escrever na coleção de logs.
2.  **Modificar a Função:** Alterar a assinatura da função de log para aceitar os novos campos padronizados (`run_id`, `service`, `job_type`, `metrics`, etc.).
3.  **Atualizar Chamadas:** Percorrer o código e atualizar todas as chamadas à função de log para fornecer os novos dados estruturados. Um `run_id` (gerado com `uuid.uuid4()`) deve ser criado no início de cada job e passado para todas as chamadas de log subsequentes dentro daquela execução.

#### **Exemplo de Migração (Antes e Depois)**

**Antes (Código Hipotético):**

```python
# Em algum lugar no scraper_newspaper3k
def scrape_job():
    start_time = datetime.now()
    # ... lógica do job ...
    log_to_firestore({
        "task": "Scraping de URLs",
        "start_time": start_time,
        "processed_count": 42,
        "status": "completed",
        "message": "Scraping concluído."
    })
```

**Depois (Código Refatorado):**

```python
# Em algum lugar no scraper_newspaper3k
import uuid

def scrape_job():
    run_id = str(uuid.uuid4())
    start_time = datetime.now()
    
    # Log de início
    log_system_event(
        run_id=run_id,
        service="scraper_newspaper3k",
        job_type="url_scraping",
        status="started",
        start_time=start_time
    )

    # ... lógica do job ...
    
    # Log de conclusão
    log_system_event(
        run_id=run_id,
        service="scraper_newspaper3k",
        job_type="url_scraping",
        status="completed",
        start_time=start_time,
        end_time=datetime.now(),
        message="Scraping de 50 URLs concluído. 42 com sucesso, 8 com erro.",
        metrics={"targets": 50, "processed": 42, "errors": 8}
    )
```

### 5. Módulos a Serem Atualizados

*   `api_nlp`
*   `scraper_newspaper3k`
*   `search_google_trends`