### **DOCUMENTO DE CONTEXTO: Módulo `WHATSAPP_GROUPS`**

**Versão:** 1.0
**Autor:** Especialista Híbrido (Análise de Mídia, Estratégia Digital, Arquitetura de Software)
**Módulo Associado:** `WHATSAPP_GROUPS` (Sugestão de nome para o micro-serviço: `whatsapp_ingestion_service`)

---

### 1. Visão Estratégica: A Fronteira da Inteligência de Fontes Fechadas

Enquanto as redes sociais abertas nos mostram o palco, os grupos de WhatsApp nos dão acesso aos bastidores. É o ambiente onde a militância se organiza, narrativas são forjadas e testadas, e o sentimento real, sem filtros, é expresso.

O módulo `WHATSAPP_GROUPS` é a nossa porta de entrada para esse ecossistema crítico. Sua função não é apenas processar arquivos de texto, mas sim transformar um fluxo de dados não estruturados e de alta volatilidade em uma fonte de inteligência estruturada, contínua e acionável. Ele servirá como o principal termômetro da base, permitindo detectar crises e oportunidades antes que cheguem ao debate público.

### 2. Arquitetura e Fluxo de Dados

Seguindo o padrão de micro-serviços da plataforma, este módulo será autônomo, desacoplado e focado em uma única responsabilidade: **ingestão, parsing e enriquecimento primário de dados do WhatsApp**.

O fluxo completo se dará em três grandes etapas:

**Etapa 1: Ingestão via Frontend**

1.  **Entrada do Usuário:** Será criada uma nova seção no `FRONTEND` chamada "**WhatsApp - Grupos**".
2.  **Interface de Upload:** Nesta seção, o agente poderá fazer o upload de um ou mais arquivos `.ZIP` exportados diretamente do WhatsApp. A interface deve ser clara, indicando que o nome do arquivo de texto dentro do zip será usado para identificar o grupo.
3.  **Disparo:** Ao confirmar o upload, o `FRONTEND` enviará o arquivo para um endpoint seguro no micro-serviço `whatsapp_ingestion_service`.

**Etapa 2: Processamento pelo Micro-serviço `whatsapp_ingestion_service`**

Este serviço, construído em FastAPI e Python, orquestrará todo o pipeline de processamento.

1.  **Endpoint de Recepção (`POST /ingest/upload`):**
    *   Recebe o arquivo `.ZIP`.
    *   Valida o tipo de arquivo e o tamanho.
    *   Inicia uma **Tarefa em Background** (`BackgroundTasks`) para processar o arquivo, retornando uma resposta imediata `202 Accepted` para o frontend. Isso evita timeouts e melhora a experiência do usuário.
    *   Cria um log inicial na coleção `system_logs` com `source: 'whatsapp_ingestion'`, `status: 'running'`.

2.  **Processamento em Background:**
    *   **Descompressão Segura:** O `.ZIP` é descompactado em um diretório temporário e isolado no ambiente do Cloud Run.
    *   **Identificação do Grupo:** O serviço localiza o arquivo `.txt`. O nome do grupo (`group_name`) é extraído do nome do arquivo. Ex: `Conversa do WhatsApp com Apoiadores Master.txt` -> `group_name: "Apoiadores Master"`. Um `group_id` (hash do nome, por exemplo) é gerado para uso como ID do documento no Firestore.
    *   **Parsing do `.txt`:** Este é o núcleo do serviço. Um parser robusto, provavelmente baseado em expressões regulares (Regex), irá iterar linha por linha do arquivo `.txt` para extrair:
        *   **Timestamp:** Data e hora da mensagem, convertidas para o formato UTC.
        *   **Autor:** O nome do contato que enviou a mensagem.
        *   **Texto da Mensagem:** O conteúdo em si.
        *   **Mensagens de Sistema:** Identificar mensagens sem autor (ex: "Fulano entrou no grupo") e marcá-las com `is_system_message: true`.
        *   **Continuidade de Mensagem:** Lidar com quebras de linha que pertencem à mesma mensagem.
        *   **Detecção de Mídia:** Identificar placeholders como `(arquivo anexado)` ou nomes de arquivos (ex: `IMG-20250904-WA0001.jpg`). Para estas, definir `has_media: true` e extrair o `media_filename`.
        *   **Captions de Mídia:** Associar o texto que acompanha uma mídia à mesma mensagem.
        *   **Menções:** Detectar e extrair menções a outros contatos (ex: `@55719...` ou `@Nome Salvo`).

3.  **Processamento e Armazenamento de Mídia:**
    *   Para cada arquivo de mídia (imagem, vídeo, áudio, documento) encontrado no diretório descompactado:
        *   **Cálculo de Hash:** Um hash `SHA-256` do arquivo é calculado. Isso servirá como um identificador único para a mídia, permitindo a desduplicação futura.
        *   **Upload para o Cloud Storage:** O arquivo é enviado para um bucket dedicado: `gs://[seu-projeto]-whatsapp-media/`. O nome do arquivo no bucket pode ser o próprio hash para garantir unicidade.
        *   **Persistência no Firestore:** O documento da mensagem correspondente (identificado pelo `media_filename`) é criado ou atualizado no Firestore com os metadados da mídia.

**Etapa 3: Persistência no Firestore**

O serviço irá estruturar os dados em coleções no Firestore, servindo como fonte para os demais módulos da plataforma.

| Coleção/Sub-coleção | ID do Documento | Descrição |
| :--- | :--- | :--- |
| **`whatsapp_groups`** | `group_id` (hash do nome do grupo) | Armazena metadados sobre cada grupo, como `group_name`, `first_ingestion_date`, `last_ingestion_date`. |
| **`whatsapp_groups/{group_id}/messages`** | `message_id` (hash do timestamp + autor) | Armazena cada mensagem individual do grupo, seja de texto ou de mídia. |

**Exemplo de Documento de Mensagem de Texto:**
```json
// Coleção: whatsapp_groups/{group_id}/messages
{
  "timestamp_utc": "2025-09-04T18:30:00Z",
  "author": "Nome do Contato",
  "message_text": "Pessoal, atenção a essa pauta que está subindo na mídia. Precisamos alinhar o discurso. @Coordenador",
  "is_system_message": false,
  "has_media": false,
  "mentions": ["Coordenador"],
  "nlp_status": "pending" // Gancho para o módulo de NLP
}
```

**Exemplo de Documento de Mensagem com Mídia:**
```json
// Coleção: whatsapp_groups/{group_id}/messages
{
  "timestamp_utc": "2025-09-04T18:35:10Z",
  "author": "Nome do Contato",
  "message_text": "Olhem o print que recebi. IMG-20250904-WA0002.jpg (arquivo anexado)",
  "is_system_message": false,
  "has_media": true,
  "media": {
    "original_filename": "IMG-20250904-WA0002.jpg",
    "gcs_uri": "gs://[seu-projeto]-whatsapp-media/a1b2c3d4...e5f6.jpg",
    "hash_sha256": "a1b2c3d4...",
    "media_type": "image/jpeg" // Inferido da extensão
  },
  "nlp_status": "pending",
  "media_analysis_status": "pending" // Gancho para o futuro módulo MEDIA_ANALYSIS
}
```

#### **2.1. Estratégia de Desduplicação e Idempotência**

Considerando que os dados serão ingeridos diariamente a partir de exportações que contêm sobreposição de mensagens, o sistema é projetado para ser **idempotente**, prevenindo a duplicidade de dados através de uma estratégia dupla:

*   **Desduplicação de Mensagens:**
    *   Para cada mensagem parseada, um `message_id` único e determinístico é gerado a partir de um hash combinando seu `timestamp`, `autor` e um `preview` do conteúdo.
    *   Este `message_id` é usado como o ID do documento no Firestore.
    *   **Lógica:** O serviço **verifica a existência do documento por ID antes de qualquer escrita**. Se a mensagem já existe, ela é ignorada, garantindo que cada mensagem seja armazenada apenas uma vez.

*   **Desduplicação de Mídias:**
    *   Para cada arquivo de mídia, um hash **SHA-256** de seu conteúdo binário é calculado (`media_hash`).
    *   Este `media_hash` é usado como o nome do arquivo no Google Cloud Storage (ex: `<hash>.jpg`). Isso cria um sistema de armazenamento endereçável por conteúdo.
    *   **Lógica:** O serviço **verifica a existência do arquivo no bucket antes de qualquer upload**. Se uma mídia com o mesmo conteúdo (mesmo hash) já foi enviada anteriormente, o novo upload é ignorado, economizando custos e evitando redundância. O documento no Firestore apenas aponta para o arquivo já existente.


### 3. Relação com Outros Módulos

Este módulo não é uma ilha; ele é um provedor de dados brutos e estruturados para o ecossistema.

*   **NLP:** O `api_nlp` terá um **novo Job** agendado para rodar após a janela de ingestão. Este job buscará por documentos na sub-coleção `messages` onde `nlp_status: 'pending'`, aplicará as análises de sentimento e entidades, e atualizará o status para `nlp_ok`.
*   **MEDIA\_ANALYSIS:** Da mesma forma, o módulo `MEDIA_ANALYSIS` irá procurar por mensagens com `media_analysis_status: 'pending'` para realizar tarefas como OCR (extração de texto de imagens), transcrição de áudio e reconhecimento de objetos/logos.
*   **ANALYTICS & Frontend:** O módulo `ANALYTICS` irá expor novos endpoints que o `FRONTEND` consumirá para popular os dashboards. Esses endpoints irão agregar os dados da coleção `whatsapp_groups` para gerar os insights.
*   **AGENT\_MODE:** A detecção de anomalias (pico de sentimento negativo, disseminação rápida de um boato) nesta coleção será um dos gatilhos mais poderosos para acionar o `AGENT_MODE`, permitindo a criação de contra-narrativas ou esclarecimentos direcionados.

### 4. Dashboard: Artefatos de Análise e Inteligência

O dashboard no `FRONTEND` será a interface onde o valor estratégico deste módulo se materializa. Ele deve ir além de simples contagens de mensagens.

1.  **Radar de Narrativas:**
    *   **O que é:** Uma nuvem de entidades e termos-chave (extraídos pelo NLP) com visualização de sua evolução temporal (últimas 24h, 7 dias).
    *   **Inteligência Gerada:** Detecta pautas, boatos e teses que estão ganhando tração em grupos específicos antes que se tornem mainstream.

2.  **Termômetro de Sentimento:**
    *   **O que é:** Um gráfico de linhas mostrando o sentimento médio (positivo, negativo, neutro) por grupo ao longo do tempo.
    *   **Inteligência Gerada:** Mede a reação da base a eventos específicos (um discurso, uma notícia, uma ação do governo), fornecendo feedback autêntico e imediato.

3.  **Mapa de Influência Interna:**
    *   **O que é:** Um ranking de membros por volume de mensagens, reações (se disponível) e menções recebidas. Pode ser visualizado como um grafo de rede.
    *   **Inteligência Gerada:** Identifica os influenciadores de nicho e os "super-propagadores" de informação dentro de cada comunidade.

4.  **Painel de Alertas:**
    *   **O que é:** Cards que destacam anomalias detectadas, como um aumento súbito de mensagens com sentimento negativo, a menção a um termo de crise, ou a rápida disseminação de uma mesma imagem (mesmo `media_hash`) em múltiplos grupos.
    *   **Inteligência Gerada:** Fornece o gatilho para a ativação do `AGENT_MODE`, permitindo uma resposta cirúrgica e proativa a crises em gestação.

---

### **Detalhamento do Item 4: Dashboard de Análise de Grupos do WhatsApp**

Este dashboard será alimentado por um conjunto de endpoints no micro-serviço `ANALYTICS`, que por sua vez realizará as agregações necessárias a partir dos dados no Firestore. A premissa é que o usuário possa filtrar por um ou mais grupos (`group_id`) e por um intervalo de tempo.

#### **1. Radar de Narrativas**

*   **Objetivo Estratégico:** Entender "sobre o que estão falando *agora*?" de forma rápida e visual. Identificar os temas dominantes, os nomes mais citados e as pautas emergentes que a equipe de comunicação deve monitorar ou engajar.

*   **Artefato Visual:**
    *   **Tipo de Gráfico:** Nuvem de Palavras Interativa (Interactive Word Cloud). As palavras não são apenas texto; são clicáveis. Clicar em uma palavra (ex: "Reforma Tributária") poderia filtrar os outros gráficos do dashboard para exibir dados apenas de mensagens que contêm essa entidade.
    *   **Biblioteca Sugerida:** **D3.js** (especificamente o módulo `d3-cloud`) para máxima customização e interatividade.

*   **Fonte de Dados e Lógica de Agregação:**
    *   **Coleção:** `whatsapp_groups/{group_id}/messages`
    *   **Filtros:** O endpoint no `ANALYTICS` deve buscar documentos onde `nlp_status == 'nlp_ok'` e dentro do intervalo de tempo selecionado.
    *   **Campos Utilizados:** O campo-chave é o `entities` (que foi adicionado pelo `api_nlp`). Este é um array de objetos, ex: `[{ "text": "Deputado Sicrano", "type": "PESSOA" }, ...]`.
    *   **Lógica de Agregação:** O backend deverá:
        1.  Iterar sobre todos os documentos de mensagens que correspondem aos filtros.
        2.  Para cada mensagem, iterar sobre seu array `entities`.
        3.  Criar um dicionário ou mapa para contar a frequência de cada `entity.text`. É recomendável normalizar o texto (ex: tudo minúsculo) para agrupar variações.
        4.  O endpoint retornará uma lista de objetos, ex: `[{ "text": "reforma tributária", "value": 150 }, { "text": "deputado sicrano", "value": 125 }, ...]`.
        5.  No frontend, o `value` será usado para definir o tamanho da palavra na nuvem.

---

#### **2. Termômetro de Sentimento**

*   **Objetivo Estratégico:** Medir a "temperatura" emocional da base. Correlacionar picos de sentimento (positivo ou negativo) com eventos externos (um discurso, uma notícia, uma ação política) para entender causa e efeito em tempo real.

*   **Artefato Visual:**
    *   **Tipo de Gráfico:** Gráfico de Linhas Combinado com Gráfico de Barras (Combo Chart).
        *   **Eixo Y1 (Linha):** Score Médio de Sentimento (de -1 a 1).
        *   **Eixo Y2 (Barras):** Volume Total de Mensagens.
        *   **Eixo X:** Tempo (agrupado por hora ou dia, dependendo do intervalo selecionado).
    *   **Biblioteca Sugerida:** **ECharts** ou **Chart.js**, que lidam muito bem com gráficos combinados e têm excelente performance.

*   **Fonte de Dados e Lógica de Agregação:**
    *   **Coleção:** `whatsapp_groups/{group_id}/messages`
    *   **Filtros:** `nlp_status == 'nlp_ok'`, intervalo de tempo. Ignorar mensagens onde `is_system_message == true`.
    *   **Campos Utilizados:** `timestamp_utc` e `sentiment.score` (adicionado pelo `api_nlp`).
    *   **Lógica de Agregação:** O backend fará uma agregação temporal (GROUP BY):
        1.  Agrupar os documentos por intervalo de tempo (ex: por dia).
        2.  Para cada dia, calcular duas métricas:
            *   A média do campo `sentiment.score` de todas as mensagens daquele dia.
            *   A contagem total de mensagens (`COUNT(*)`) daquele dia.
        3.  O endpoint retornará uma série temporal, ex: `[{ "date": "2025-09-04", "avg_sentiment": 0.35, "message_count": 450 }, { "date": "2025-09-05", "avg_sentiment": -0.12, "message_count": 620 }, ...]`.

---

#### **3. Mapa de Influência Interna**

*   **Objetivo Estratégico:** Identificar quem são os formadores de opinião e os "hubs" de informação dentro de cada grupo. Mapear quem fala com quem permite entender a estrutura social da comunidade e direcionar a comunicação para os membros-chave.

*   **Artefato Visual:**
    *   **Tipo de Gráfico:** Grafo de Rede Direcionado (Directed Network Graph).
        *   **Nós (Círculos):** Representam os `autores` (membros do grupo). O tamanho do nó pode ser proporcional ao número de mensagens que o autor enviou.
        *   **Arestas (Linhas/Setas):** Representam as interações. Uma seta sai do `autor` de uma mensagem e aponta para o membro mencionado no campo `mentions`. A espessura da linha pode representar a frequência de menções entre esses dois membros.
    *   **Biblioteca Sugerida:** **vis.js** ou **D3.js (com d3-force)** são as ferramentas padrão para visualizações de rede complexas e interativas.

*   **Fonte de Dados e Lógica de Agregação:**
    *   **Coleção:** `whatsapp_groups/{group_id}/messages`
    *   **Filtros:** Intervalo de tempo. Filtrar por mensagens que tenham o campo `mentions` preenchido e não vazio.
    *   **Campos Utilizados:** `author`, `mentions`.
    *   **Lógica de Agregação:** O backend construirá a estrutura de dados do grafo:
        1.  **Criar lista de nós:** Obter todos os `autores` únicos no período e contar suas mensagens para definir o "peso" (tamanho) de cada nó.
        2.  **Criar lista de arestas:** Iterar sobre as mensagens. Para cada mensagem com menções, criar uma aresta do `autor` para cada pessoa no array `mentions`. Contar a frequência de cada par `(autor, mencionado)` para definir o peso da aresta.
        3.  O endpoint retornará a estrutura do grafo, ex: `{"nodes": [{ "id": "Autor A", "value": 80 }, ...], "edges": [{ "from": "Autor A", "to": "Autor B", "value": 15 }, ...]}`.

---

#### **4. Painel de Alertas (AGENT_MODE Triggers)**

*   **Objetivo Estratégico:** Deixar de ser reativo e passar a ser proativo. Este painel não é para análise histórica, mas para detecção de anomalias em tempo real que exigem atenção imediata da equipe.

*   **Artefato Visual:**
    *   **Tipo de Gráfico/Artefato:** Uma série de "Cards" de Indicadores Chave (KPI Cards) com cores condicionais (verde, amarelo, vermelho).
    *   **Biblioteca Sugerida:** Não é uma biblioteca de gráficos, mas um componente de UI de bibliotecas como **Material-UI (MUI)**, **Ant Design** ou **Bootstrap**.

*   **Fonte de Dados e Lógica de Agregação:**
    Cada card será alimentado por um endpoint específico no `ANALYTICS` que executa uma query de verificação de anomalia.

    *   **Card 1: Alerta de Sentimento Negativo**
        *   **Lógica:** Contar o número de mensagens com `sentiment.label == 'Negativo'` na última hora. Se o número ultrapassar um limiar pré-definido (ex: 50 mensagens ou 3 desvios padrão acima da média horária), o card fica vermelho.
        *   **Campos:** `timestamp_utc`, `sentiment.label`.

    *   **Card 2: Alerta de Viralização de Mídia**
        *   **Lógica:** Identificar se um mesmo `media.hash_sha256` apareceu em mais de X grupos diferentes (ex: 3 grupos) nos últimos 30 minutos. Isso indica uma campanha de disseminação coordenada.
        *   **Campos:** `media.hash_sha256`, `timestamp_utc`. (Requer uma query que abranja múltiplos `group_id`).

    *   **Card 3: Alerta de Termo de Crise**
        *   **Lógica:** Manter uma lista de "termos de crise" (ex: "escândalo", "denúncia", "problema grave"). Contar quantas mensagens nos últimos 15 minutos contêm uma entidade (`entities.text`) que corresponde a um desses termos. Se ultrapassar um limiar, o card fica vermelho.
        *   **Campos:** `entities.text`, `timestamp_utc`.