Este módulo será um serviço de **ingestão e parsing**.

**1. Alteração no `FRONTEND`:**

*   Criar uma nova área segura no frontend chamada "WhatsApp - Grupos".
*   Criar uma sub opção chamada "Ingestão de Dados".
*   Nessa área, o usuário (um "agente" ou analista de confiança) poderá fazer o upload de arquivos `.ZIP` exportados de conversas do WhatsApp. 
*   Criar uma sub opção chamada "Dashboard". 

**2. Micro-serviço `WHATSAPP_GROUPS`:**
    *   Recebe o `.zip`.
    *   **Descompacta o arquivo** em um ambiente temporário seguro.
    *   Obtem 'group_name' a partir do nome do arquivo .TXT . Exemplo: 'Conversa do WhatsApp com APG CGOTIC.txt' , 'group_name' = APG CGOTIC.

    *   **Processa o `.txt`:** Parseia o arquivo de texto.
          * no parsing do .TXT o serviço deve ser capaz de distinguir :
                * mensagens com autor. Exemplo: DD/MM/AAAA HH:MM - Nome do Autor: Conteúdo da mensagem.
                * Mensagens de Sistema (Sem Autor). Exemplo: DD/MM/AAAA HH:MM - Conteúdo da mensagem.
                * Quebras de Linha na Mesma Mensagem com autor.
                * mensagens com mídia: DD/MM/AAAA HH:MM - Contato: IMG-20250904-WA0001.jpg (arquivo anexado)`. 
                * Placeholders de Mídia:O parser precisa identificar essas strings para marcar a mensagem com has_media: true e extrair o nome do arquivo para vinculá-lo ao arquivo correspondente no .zip
                * Captions em mensagens contendo mídias
                * menções de contatos do grupo: Exemplo: DD/MM/AAAA HH:MM - Nome do Autor: {@contato} Conteúdo da mensagem

    *   **Processa as Mídias:** Para cada arquivo de mídia no `.zip`:
        *   Faz o **upload do arquivo para um bucket do Google Cloud Storage** (ex: `gs://[seu-projeto]-whatsapp-media/`). Isso é essencial para escalabilidade e segurança.
        *  Calcular o "hash" (uma assinatura digital única) de cada arquivo de mídia.
        *   **Atualiza o Documento no Firestore:** O documento da mensagem correspondente no Firestore é enriquecido com a URL do arquivo no Cloud Storage.
        ```json
        {
          "timestamp": "...",
          "author": "Nome do Contato",
          "message_text": "IMG-20250904-WA0001.jpg (arquivo anexado)",
          "has_media": true,
          "media_type": "image/jpeg", // Inferido da extensão do arquivo
          "media_gcs_url": "gs://[seu-projeto]-whatsapp-media/IMG-20250904-WA0001.jpg",
          "media_analysis_status": "pending" // Novo status para um novo job
          "hash": hash da midia
        }
        ```

**3. Dashboard**
  * sugerir artefatos de dashboard baseados nos dados enriquecidos das conversas de grupos do whatsapp. Exemplos:
  * Detectar Narrativas Emergentes: Identificar pautas, boatos e teses que estão ganhando força em grupos específicos (apoiadores, opositores, sociedade civil) antes que se tornem mainstream.
  * Mapear Apoiadores e Opositores: Entender os principais argumentos, as dores e as pautas mobilizadoras de cada grupo.
  * Análise de Sentimento Realista: O sentimento expresso em grupos fechados tende a ser mais autêntico e menos performático do que em redes abertas como o Twitter. A análise aqui revela a convicção real dos membros.
  * Identificar Influenciadores de Nicho: Dentro de cada grupo, existem membros cuja opinião tem mais peso, cujas mensagens são mais encaminhadas. Identificá-los é chave para entender a dinâmica do grupo.
  * Ativar o AGENT_MODE com Precisão Cirúrgica: Ao detectar uma crise em gestação ou uma onda de fake news se formando, o sistema pode acionar o AGENT_MODE para preparar uma contra-narrativa ou um esclarecimento antes que o dano seja massivo.
  