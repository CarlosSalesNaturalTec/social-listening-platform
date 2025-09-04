**3. Criação de um Novo Micro-serviço ou Job: `MEDIA_ANALYSIS`**
    *   Este novo serviço seria acionado por eventos do Cloud Storage (sempre que um novo arquivo de mídia for carregado).
    *   **Para Imagens:** Ele chama a **API do Google Cloud Vision** para fazer OCR, detecção de objetos, etc.
    *   **Para Áudios/Vídeos:** Ele chama a **API do Google Speech-to-Text** ou **Video Intelligence** para transcrever o conteúdo.
    *   Os resultados (texto extraído da imagem, transcrição do áudio) são salvos de volta no documento do Firestore, permitindo que o `API_NLP` analise esse novo texto.