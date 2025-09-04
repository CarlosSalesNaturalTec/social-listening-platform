
**3. Integração com o Ecossistema (A Orquestração):**

*   **`api_nlp` (O Cérebro Central):**
    
    * Um novo endpoint será criado e acionado por eventos do Firestore (usando Cloud Functions for Firebase) sempre que um novo documento for criado em whatsapp_messages com nlp_status: 'pending'.

    * Este job irá executar as mesmas análises (sentimento, entidades, moderação) no campo message_text e enriquecer o documento original na coleção whatsapp_messages, atualizando o status para nlp_ok.

    *   Esta é a principal ferramenta para essa tarefa. Para cada mensagem que entra, o API_NLP vai usar a biblioteca Google Cloud Natural Language para: analisar sentimento, moderação de conteúdo, além de identificar e categorizar "entidades" no texto. As entidades que nos interessam são: 
      * PERSON: O nome do parlamentar, apelidos, nomes de aliados e opositores importantes.
      * ORGANIZATION: Seu partido, partidos rivais, ministérios, comissões ("CPI da Pandemia"), empresas, etc.
      * LOCATION: Cidades, estados ou regiões de interesse para o mandato.
      * EVENT: Nomes de projetos de lei ("PL 2630"), eventos políticos, etc.
      
      * A mensagem original no Firestore:
      ```
      {
        "timestamp": "2025-09-04T15:10:00Z",
        "author": "Liderança Comunitária",
        "message_text": "Pessoal, o Deputado Sicrano vai visitar Campinas na sexta para discutir a reforma tributária.",
        "nlp_status": "pending"
      }
      ```
      * Após o processamento pelo API_NLP, o documento é enriquecido (inferindo TYPE em relevant_entities):
      {
        "timestamp": "2025-09-04T15:10:00Z",
        "author": "Liderança Comunitária",
        "message_text": "Pessoal, o Deputado Sicrano vai visitar Campinas na sexta para discutir a reforma tributária.",
        "nlp_status": "nlp_ok",
        "sentiment_score": 0.8,
        "sentiment_label": "Positivo",
        "relevant_entities": [
          {"text": "Deputado Sicrano", "type": "PERSON"},
          {"text": "Campinas", "type": "LOCATION"}
        ],
        "topics": ["reforma_tributaria", "agenda_parlamentar"]
      }



