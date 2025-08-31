### Categoria 1: A Marca Principal (O Parlamentar)

O objetivo aqui é criar uma visão 360º do interesse de busca sobre o próprio parlamentar. É o pilar do monitoramento de reputação e a linha de base para todas as comparações.

**Por que cadastrar?**
*   Para alimentar o **"Gráfico de Correlação: Menções & Interesse de Busca"** no dashboard principal do `ANALYTICS`.
*   Para servir como termo principal na busca por **"Buscas em Ascensão (Rising Queries)"**, identificando pautas emergentes ou crises associadas diretamente ao nome do parlamentar.

**Exemplos de Termos:**

*   **Nome Completo e Cargo:** `"Deputado Federal João da Silva"`
    *   *Uso:* Captura buscas mais formais e específicas, muitas vezes ligadas a notícias ou documentos oficiais. O uso de aspas garante a busca pela frase exata.
*   **Nome Comum/Público:** `João da Silva`
    *   *Uso:* Captura a forma como a maioria das pessoas busca. É o termo mais amplo e geralmente com maior volume.
*   **Variações e Apelidos Políticos:** `"João do Agro"`, `"Senadora Maria da Educação"`
    *   *Uso:* Se o parlamentar é conhecido por um apelido ou por uma bandeira específica, é crucial monitorar essa variação para capturar a percepção de nicho.
*   **Nome sem o Cargo:** `"Maria Oliveira"`
    *   *Uso:* Importante para comparar com o nome completo e entender se o interesse é pela pessoa ou pela sua função política.

### Categoria 2: Concorrentes e Opositores

O objetivo é medir o "capital de atenção" dos adversários políticos para entender o cenário competitivo.

**Por que cadastrar?**
*   Para alimentar o gráfico de **"Análise Comparativa de Interesse"** no `ANALYTICS`, permitindo uma visualização clara de quem está ganhando ou perdendo a atenção do público.
*   Para contextualizar os picos de interesse do próprio parlamentar. Um aumento no seu interesse pode ser uma reação a uma ação de um concorrente.

**Exemplos de Termos:**

*   **Principal Opositor no Estado/Região:** `"Governador Carlos Santos"`
    *   *Uso:* Monitoramento direto do principal adversário no campo de jogo eleitoral.
*   **Líderes de Partidos Opostos:** `"Presidente do Partido X"`
    *   *Uso:* Entender o interesse em figuras de liderança que podem influenciar a narrativa contra o parlamentar.
*   **Potenciais Desafiantes Futuros:** `"Vereador Pedro Lima"`
    *   *Uso:* Inteligência de longo prazo para identificar concorrentes emergentes que começam a ganhar tração pública.
*   **Nomes Comuns dos Concorrentes:** `Carlos Santos`, `Pedro Lima`
    *   *Uso:* Assim como para a marca principal, é vital monitorar a forma mais comum de busca.

### Categoria 3: Pautas-Chave e Temas Estratégicos

Este é o cadastro mais estratégico e proativo. O objetivo é monitorar o interesse do público em temas específicos, independentemente de estarem associados diretamente a um nome.

**Por que cadastrar?**
*   Para **identificar oportunidades de pauta**, se antecipando a debates antes que eles explodam nas redes sociais.
*   Para medir o impacto das ações do parlamentar. Ex: Após um discurso sobre segurança, o interesse por "lei de segurança pública" aumentou na sua região?
*   Para entender as preocupações latentes do eleitorado, como indicado no documento `analytics.md`.

**Exemplos de Termos:**

*   **Projetos de Lei e Legislação:** `PEC 48`, `"Marco Temporal"`, `"Reforma Tributária"`
    *   *Uso:* Monitorar o interesse público em debates legislativos quentes. O exemplo do `monitor_result.json` sobre a "agenda anti-indígena" mostra a importância de monitorar termos como "Marco Temporal" e "PEC 48".
*   **Temas Regionais (com segmentação geográfica):** `Segurança Pública em São Paulo`, `Desemprego na Bahia`, `"Crise Hídrica" no Ceará`
    *   *Uso:* O frontend deve permitir associar um termo a uma **geografia específica (ex: BR-SP)**. Isso permite que o parlamentar entenda as dores reais e mais buscadas de sua base eleitoral.
*   **Termos de Crise Potencial:** `Corrupção`, `Escândalo`, `Nepotismo`
    *   *Uso:* Embora amplos, podem ser cruzados no `ANALYTICS` com o nome do parlamentar para criar alertas. O ideal é monitorar termos como `"Nome do Parlamentar" + escândalo` diretamente.
*   **Bandeiras Políticas e Setores:** `Agronegócio`, `Sustentabilidade`, `Educação Básica`, `Direitos Indígenas`
    *   *Uso:* Entender o nível de interesse geral da sociedade nos temas que o parlamentar defende, identificando o melhor momento para comunicar sobre eles.