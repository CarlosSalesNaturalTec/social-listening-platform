SCRAPER

   "ERROR:main:Erro inesperado no documento 9bd6cefe64a526bbc2c4773b7211f8ed7d23138263c8eab732274b47a342ccad: All strings must be XML compatible: Unicode or ASCII, no NULL bytes or control characters


"WARNING:main:Erro no scraping do documento b0d01ebd8f0013bc0cc455262099b65230638260acc384b6d73386f2789afb57: Article `download()` failed with 504 Server Error: Gateway Time-out for url: https://www.camara.leg.br/noticias/1155870-camara-aprova-projeto-que-amplia-numero-de-deputados-federais-de-513-para-531/ on URL https://www.camara.leg.br/noticias/1155870-camara-aprova-projeto-que-amplia-numero-de-deputados-federais-de-513-para-531/"

=====


Recebi o log de erro. A mensagem 403 Permission 'secretmanager.versions.add' denied é extremamente clara e aponta para um problema de configuração no Google Cloud, e não um bug no nosso código.

  1. Estado Atual e Verificação de Recuperação

  O estado atual do projeto é que estamos prontos para desenvolver a UI de gerenciamento de alvos, mas a funcionalidade de criação de contas de serviço
  está bloqueada por uma permissão de nuvem.

  2. Análise e Planejamento

  Análise do Erro:

   1. O log Secret 'instagram-session-carlossalesti' já existe, adicionando nova versão. indica que nossa lógica está correta: o código tentou criar o
      secret, viu que ele já existia, e corretamente pulou para a etapa de adicionar uma nova versão.
   2. O erro 403 Permission 'secretmanager.versions.add' denied ocorre nesta segunda etapa. Ele significa que a Conta de Serviço que o nosso backend
      utiliza para se autenticar com as APIs do Google não tem a permissão específica para adicionar versões a um secret existente no Secret Manager.
   3. Conforme nossa documentação, a conta de serviço precisa da role Cloud Datastore User para o Firestore. Claramente, esquecemos de adicionar as
      permissões para o Secret Manager.

  Plano de Ação (Instruções para Você):

  Este problema precisa ser resolvido por você no Console do Google Cloud, concedendo as permissões corretas à conta de serviço que o backend utiliza.
  O código não precisa de alterações.

  A conta de serviço, baseada no nome do arquivo de chave (...firebase-adminsdk-fbsvc...), provavelmente é:
  firebase-adminsdk-fbsvc@monitora-parlamentar-elmar.iam.gserviceaccount.com

  Você precisa adicionar duas roles a esta conta de serviço:

   1. `Secret Manager Secret Version Adder`: Concede a permissão para adicionar novas versões de segredos. (Corrige o erro atual).
   2. `Secret Manager Secret Accessor`: Concederá a permissão para ler o conteúdo dos segredos. Isso será necessário no futuro, quando o módulo de coleta
      precisar baixar os arquivos de sessão para usar. É melhor já adicionar agora.

  3. Proposta e Ponto de Pausa

  Ação Necessária (Sua Parte):

  Por favor, siga estes passos no seu Console do Google Cloud:

   1. Vá para a seção IAM e Administrador -> IAM.
   2. Encontre a conta de serviço principal na lista. O nome dela deve ser algo como firebase-adminsdk-fbsvc@....
   3. Clique no ícone de lápis (Editar principal) na linha correspondente.
   4. Clique em + ADICIONAR OUTRO PAPEL.
   5. No campo de filtro, procure por `Secret Manager Secret Version Adder` e selecione este papel.
   6. Clique em + ADICIONAR OUTRO PAPEL novamente.
   7. Procure por `Secret Manager Secret Accessor` e selecione este papel.
   8. Clique em SALVAR.

  Após salvar essas alterações, as permissões podem levar um ou dois minutos para se propagarem. Depois disso, a funcionalidade de upload de sessão


* implementar search_instagram
* Analisar: scrapper. log do servidor de produção. . porque processo iniciou e não foi concluido
* api_nlp  endpoint não encontrado

* gerar um contextDoc para cada modulo novo

# Melhorias
* Scraper / coluna Snippet ... x horas atrás
* Estratégia para:  scraper_failed, nlp_error, reprocess ? 
* No Login, sempre dá erro na primeira vez, tem que tentar 2 vezes
* Acesso externo bloqueado para as APIs
* Analisar, remover e evitar deprecad além de warnings de build/deploy

# Schedule
ok - 12:00hs e 20:00hs  - search
ok - 21:00hs            - historical_search
ok - 22:00hs            - scraper
ok - 23:00              - nlp
23:30 - instagram jobs