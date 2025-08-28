# Google AI Studio - System Instructions:

Assuma a persona de um especialista híbrido e multifacetado. Seu núcleo técnico é o de um:
* Analista de Mídias Sociais com especialização em política.
* Estrategista de Comunicação Digital.
* Consultor de Marketing Político Digital.
No entanto, sua expertise vai além. Você também possui as habilidades e a mentalidade de um:
* Engenheiro de Software Sênior e Arquiteto de Nuvem
* Especialista em Python, Google Cloud Platform (GCP) e micro-serviços

# Prompt
Analise as informações fornecidas no arquivo analytics.md e outros em anexo.
A partir delas gostaria de gerar um documento de contexto que servirá exclusivamente como base para construção do módulo de **ANALYTICS** da plataforma de social listening descrita no documento plano_de_acao_md. O arquivo monitor_result.json possui a estrutura dos artigos analisados pela plataforma, utilizar estes dados para produzir as análises. Se surgirem dúvidas, ambiguidades ou quaisquer outro problema me pergunte. 


# Diversos
* Analisar: Erro geral na tarefa de scraping : '_UnaryStreamMultiCallable' object has no attribute '_retry'
* Estratégia para:  scraper_failed, nlp_error, reprocess 


# FrontENd
No Login, sempre dá erro na primeira vez, tem que tentar 2 vezes

# API´s
Colocar acesso não público no Gcloud Deploy 

# Diversos para os novos módulos a criar
* Analisar, remover e evitar deprecad
* Perguntar qual stack quando houver várias opções / PEsquisar manualmenete antes de iniciar um módulo
* Módulo dashboard, começar com a visão do analista, depois developer

# Futuros - Técnicas de Desenvolvimento de novos sistemas
* Documento de Contexto - gerar através de prompt e utilizar no projeto
* Fontes de Research - gerar pesquisa profunda sobre o tema e usar como referencia no projeto
* Arquitetura da aplicação - gerar através de prompt e utilizar no projeto

# Schedule
ok - 12:00hs e 20:00hs  - search
ok - 21:00hs            - historical_search
ok - 22:00hs            - scraper
ok - 23:00              - nlp
