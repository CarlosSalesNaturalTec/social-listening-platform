# SEARCH_INSTAGRAM

### 1. A Visão Estratégica e Analítica (O "Quê")

No contexto de um parlamentar, o Instagram não é apenas um feed de fotos. É um campo de batalha de narrativas, um termômetro de popularidade e um canal direto (e muitas vezes bruto) de feedback do cidadão.
A plataforma deve usar o Instagram para responder às seguintes perguntas:

#### **A. Monitoramento do Próprio Mandato:**

*   **Análise de Sentimento nos Comentários:** Qual é a reação predominante nas postagens do parlamentar? É positiva, negativa, neutra? 
*   **Mapeamento de Apoiadores e Críticos:** Quem são os usuários mais engajados? É possível identificar os defensores mais vocais (que podem ser amplificados) e os detratores mais consistentes (cujas críticas podem sinalizar uma crise em potencial)?
*   **Desempenho de Conteúdo:** Quais formatos (Reels, Stories, Carrossel, Post estático) e quais temas (prestação de contas, vida pessoal, ataque a adversários) geram mais engajamento positivo? Isso informa a futura estratégia de comunicação.
*   **Menções e Marcações:** O que outras contas (cidadãos, influenciadores, imprensa) estão postando e marcando o perfil do parlamentar? Isso é fundamental para captar percepções fora da "bolha" do próprio perfil.

#### **B. Análise de Concorrentes Políticos:**

*   **Benchmarking de Estratégias:** Fazer exatamente a mesma análise acima, mas para os perfis dos concorrentes. Qual a estratégia de comunicação deles? Onde eles são fortes e onde são fracos?
*   **Identificação de Vulnerabilidades:** Os comentários nos posts de um concorrente revelam uma pauta na qual ele está sendo muito criticado? Essa é uma vulnerabilidade que pode ser explorada politicamente.
*   **Oportunidades de Contraste:** Se um concorrente posta sobre o tema X e recebe aplausos, enquanto o seu parlamentar é criticado no mesmo tema, isso é um *insight* valioso para ajustar a mensagem.

#### **C. Escuta das Pautas do Cidadão:**

*   **Monitoramento de Hashtags Relevantes:** Rastrear hashtags locais (`#SegurançaEm[SuaCidade]`, `#[SeuBairro]Abandonado`) e temáticas (`#ReformaTributaria`, `#PLdaSaude`). O que está sendo dito e, mais importante, *sentido* sobre esses temas?
*   **Análise de Perfis de Mídia e Influenciadores Locais:** O que os portais de notícia locais, as páginas de bairro e os influenciadores digitais da região estão postando? Os comentários nessas publicações são uma mina de ouro para entender as preocupações reais e a linguagem do cidadão comum.
*   **Mapeamento de Geotags:** Monitorar publicações marcadas em locais estratégicos do distrito eleitoral do parlamentar (hospitais, praças, escolas). Isso pode revelar problemas e sentimentos localizados que não aparecem em discussões mais amplas.


### 2. Observações gerais

* - Utilizar o Instaloader.
* - O sistema precisa gerenciar uma sessão de login. Isso envolve salvar o arquivo de sessão (L.save_session_to_file) em um local seguro e carregá-lo (L.load_session_from_file) a cada execução do seu coletor. Você também precisa de um mecanismo para detectar se a sessão expirou e alertar um operador para fazer o login novamente.
* - Criar neste novo módulo um endpoind, para ser acionado via Cloud Scheduler e a partir daí realizar os scripts.

* - Análise das publicações no perfil do próprio parlamentar.
  * - Publicações ocorridas no dia corrente.
  * - Quantidade de curtidas e comentários
  * - Obter últimos comentários (limite de 100) incluindo: comment.owner.followers, comment.owner.followees,comment.owner.biography,comment.owner.is_private

* - Busca por hashtags e em perfis pre-cadastrados.
* - Publicações ocorridas no dia corrente.
* - Dados a serem extraídos:
  * - O código único do post 
  * - O nome de usuário de quem publicou
  * - O texto da legenda da publicação (string)
  * - A data e hora da publicação em UTC (objeto datetime)
  * - A QUANTIDADE de curtidas (inteiro)
  * - A QUANTIDADE de comentários
  * - Obter últimos comentários (limite de 100) incluindo: comment.owner.followers, comment.owner.followees,comment.owner.biography,comment.owner.is_private

  * - Fazer o donwload de Videos, imagens e/ou carrossel. salvar em bucket do GCP. para posterior processamento. 
  
* - Stories : criar endpoint específico para obtenção de stories de contas pre-cadastradas. será acionado várias vezes ao dia via cloud schedule. Fazer o donwload de Videos, imagens e/ou carrossel. salvar em bucket do GCP. para posterior processamento. 

* - utilizar Background Tasks 
* - Registrar em system_logs andamento dos scripts: iniciado, concluido, etc. Conforme padrão dos demais módulos da plataforma.

### 3. Frontend 
* No módulo de frontend existente, criar Cadastros de contas de serviço do instagram com as respectivas credenciais de seção para uso no processo de coleta.
* No módulo de frontend existente, criar cadastros de perfis/contas a serem pesquisadas. Ex: Parlamentar X, Jornal Y.
* Gerar no no múdulo de frontEnd um dashboard exclusivo para análises do Instagram.

### 4. Estratégias anti-bloqueios 

A Regra de Ouro: Aja como um Humano. Um usuário humano não visualiza 50 perfis por segundo nem baixa 10.000 comentários em 3 minutos. Ele clica, lê, rola a página, pausa. Seu código deve imitar essa cadência.

Introduza Pausas Aleatórias (Jitter): Nunca use pausas fixas como time.sleep(5). Um padrão repetitivo é fácil de detectar. Use pausas aleatórias dentro de um intervalo razoável entre ações significativas (ex: após processar um post, antes de pegar o próximo).

Pausas Maiores entre Lotes: Após processar um lote de, digamos, 20-30 posts, introduza uma pausa significativamente maior, de vários minutos.

Gerencie a Sessão Corretamente: Use o método load_session_from_file para não precisar fazer login com usuário e senha a cada execução.

Roteamento de Contas: Use um pool de várias contas de serviço. Seu sistema pode escolher aleatoriamente uma conta para cada tarefa de coleta, distribuindo a carga e tornando mais difícil para o Instagram identificar um padrão vindo de um único usuário.

Seu código precisa ser resiliente e reagir de forma inteligente quando um bloqueio acontece.

Capture as Exceções Corretas: O instaloader levanta exceções específicas para diferentes tipos de problemas. A mais importante para nós é a TooManyRequestsException.

Implemente uma Estratégia de "Backoff": Quando você receber um erro TooManyRequestsException, não tente novamente de imediato. Isso só vai piorar a situação. Espere um tempo significativamente longo (ex: 15-30 minutos) antes de tentar novamente.