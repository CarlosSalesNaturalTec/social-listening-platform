### **DOCUMENTO DE CONTEXTO: Módulo `API_NLP` (O Cérebro Analítico)**

**Versão:** 2.0
**Autor:** Especialista Híbrido (Análise de Mídia, Estratégia Digital, Arquitetura de Software)
**Módulo Associado:** `api_nlp`

---

### 1. Visão Estratégica: Transformando Dados em Inteligência Acionável

O módulo `api_nlp` é o cérebro analítico da plataforma. Sua função transcende o simples processamento de texto; ele é responsável por decodificar a linguagem, o sentimento e as intenções por trás de cada mensagem, convertendo o ruído bruto das conversas em sinais de inteligência claros e estruturados.

Para os dados do WhatsApp, onde a comunicação é direta e sem filtros, a velocidade e a precisão dessa análise são cruciais. Este módulo nos permite identificar narrativas em formação, medir a temperatura emocional da base e mapear os principais atores e pautas em tempo real, fornecendo os insumos necessários para os dashboards de análise e para a ativação do `AGENT_MODE`.

### 2. Arquitetura Orientada a Eventos: Processamento em Tempo Real

Para garantir a máxima agilidade, o processamento de NLP para as mensagens do WhatsApp não será feito em lotes agendados. Adotaremos uma arquitetura serverless e orientada a eventos, que dispara a análise no exato momento em que uma nova mensagem é ingerida na plataforma.

**Fluxo de Operação:**

`[Nova Mensagem no Firestore]` -> `[Gatilho do Cloud Function]` -> `[Chamada HTTP para API_NLP]` -> `[Documento Enriquecido]`

**Etapa 1: O Gatilho (Cloud Functions for Firebase)**
*   Será implantada uma **Cloud Function** configurada para ser acionada pelo evento `onCreate` na coleção de mensagens do WhatsApp.
*   **Caminho do Gatilho:** `whatsapp_groups/{groupId}/messages/{messageId}`.
*   Isso significa que a função será executada **automaticamente** para cada nova mensagem que o `whatsapp_ingestion_service` salva no Firestore.

**Etapa 2: O Orquestrador Leve (A Cloud Function)**
*   A responsabilidade da Cloud Function é mínima e específica: agir como um "despachante".
*   Ao ser acionada, ela extrai os parâmetros `groupId` and `messageId` do contexto do evento.
*   Em seguida, ela faz uma chamada HTTP segura (autenticada via token de serviço) para um novo endpoint no micro-serviço `api_nlp`, passando esses IDs no corpo da requisição.

**Etapa 3: O Executor (Micro-serviço `api_nlp`)**
*   **Novo Endpoint:** Será criado um endpoint dedicado para esta tarefa: `POST /process/whatsapp-message`.
*   **Payload da Requisição:** O endpoint receberá um JSON simples contendo os identificadores da mensagem a ser processada:
    ```json
    {
      "group_id": "a1b2c3d4...",
      "message_id": "e5f6g7h8..."
    }
    ```
*   **Lógica de Processamento Focado:** Ao receber a requisição, o serviço executa uma operação atômica e direcionada:
    1.  Usa os IDs recebidos para buscar **um único documento** no Firestore.
    2.  Verifica se o documento realmente precisa de processamento (ex: `nlp_status === 'pending'`).
    3.  Extrai o campo `message_text` e o envia para a Google Cloud Natural Language API.
    4.  Recebe os resultados da análise (sentimento, entidades, etc.).
    5.  **Atualiza o mesmo documento original** no Firestore com os dados enriquecidos e altera o `nlp_status` para `nlp_ok`.

### 3. Detalhamento da Análise NLP

Para cada `message_text` processado, o serviço realiza as seguintes análises:

*   **Análise de Sentimento:** Avalia o tom emocional da mensagem, resultando em um score numérico (de -1.0 a 1.0) e um rótulo categórico (`Positivo`, `Negativo`, `Neutro`).
*   **Extração de Entidades:** Identifica e categoriza substantivos e nomes próprios. O foco será em entidades de alto valor político e estratégico:
    *   `PERSON`: Nomes de políticos (o parlamentar, aliados, opositores), assessores, figuras públicas.
    *   `ORGANIZATION`: Partidos, ministérios, órgãos governamentais, empresas, movimentos sociais.
    *   `LOCATION`: Cidades, estados, regiões relevantes para a atuação política.
    *   `EVENT`: Nomes de projetos de lei (PLs), CPIs, eleições, eventos políticos.
    *   `WORK_OF_ART`: Títulos de notícias, livros, reportagens.
*   **Classificação de Conteúdo (Tópicos):** A API também categoriza o conteúdo do texto em tópicos gerais (ex: `/Politics`, `/Finance`, `/Law & Government`), permitindo uma visão macro das pautas discutidas.
*   **Moderação de Conteúdo:** Opcionalmente, pode ser ativada para identificar toxicidade, discurso de ódio ou outros conteúdos sensíveis.

### 4. Modelo de Dados Enriquecido: O Resultado Final

A operação transforma um documento de dados brutos em um registro de inteligência estruturada.

**ANTES (Documento vindo do `whatsapp_ingestion_service`):**
```json
// Coleção: whatsapp_groups/a1b2c3d4/messages/e5f6g7h8
{
  "timestamp_utc": { "_seconds": 1756888620, "_nanoseconds": 0 },
  "author": "Daniel Quadros",
  "message_text": "Estou na SESAB, apresentação da ferramenta de monitoramento do Zabbix.",
  "nlp_status": "pending",
  "is_system_message": false,
  "has_media": false,
  "media_analysis_status": "not_applicable"
}
```

**DEPOIS (Documento enriquecido pelo `api_nlp`):**
```json
// Coleção: whatsapp_groups/a1b2c3d4/messages/e5f6g7h8
{
  "timestamp_utc": { "_seconds": 1756888620, "_nanoseconds": 0 },
  "author": "Daniel Quadros",
  "message_text": "Estou na SESAB, apresentação da ferramenta de monitoramento do Zabbix.",
  "nlp_status": "ok",
  "is_system_message": false,
  "has_media": false,
  "media_analysis_status": "not_applicable",
  "nlp_analysis": {
    "sentiment": {
      "score": 0.1,
      "magnitude": 0.1,
      "label": "Neutro"
    },
    "entities": [
      { "text": "SESAB", "type": "ORGANIZATION", "salience": 0.6 },
      { "text": "Zabbix", "type": "OTHER", "salience": 0.4 }
    ],
    "content_categories": [
      { "name": "/Computers & Electronics/Software", "confidence": 0.9 }
    ]
  }
}
```

| Campo Adicionado | Descrição | Valor Estratégico |
| :--- | :--- | :--- |
| `nlp_analysis.sentiment` | Objeto contendo `score`, `magnitude` e `label` do sentimento. | Mede a temperatura emocional da conversa. |
| `nlp_analysis.entities` | Array de objetos, cada um com o texto, tipo e relevância (`salience`) da entidade. | Mapeia quem está falando de quem, sobre o quê e onde. |
| `nlp_analysis.content_categories` | Array de tópicos classificados pela IA, com nível de confiança. | Agrupa mensagens por pautas temáticas automaticamente. |
| `nlp_status` | Status atualizado para `ok` (ou `error` em caso de falha). | Controla o fluxo de trabalho, indicando que a mensagem está pronta para análise. |

### 5. Relação com Módulos Consumidores

Os dados enriquecidos pelo `api_nlp` são a matéria-prima para:
*   **`ANALYTICS`:** O serviço de analytics irá consumir esses campos estruturados para realizar as agregações que alimentarão os dashboards (Nuvem de Palavras, Gráfico de Sentimento, etc.).
*   **`AGENT_MODE`:** Regras de negócio poderão ser criadas para monitorar os dados enriquecidos em tempo real. Por exemplo: "Se mais de 10 mensagens com `sentiment.label == 'Negativo'` e mencionando a entidade `PERSON: 'Nome do Parlamentar'` forem detectadas em 5 minutos, dispare um alerta."