# AGENT_MODE
* Utilizar open-wa/wa-automate https://github.com/open-wa/wa-automate-nodejs 
* No frontend imolementar cadastro de números whatsapp para receber alertas de crise, etc (nome, número, grupo)

* Enviar alertas para contas pre-definidas
* Enviar mensagens personalizadas contendo texto e midias para contas especificas 


* - Este módulo deverá ter um sistema de gatilhos (triggers) que monitora a base de dados. Deverá ser configurado para reagir não apenas a dados das redes sociais, mas também aos dados do Google Trends. Exemplos de gatilhos:
Se (interesse_busca_trends para 'termo_negativo' > 80) E (sentimento_geral < -0.5) -> acionar_protocolo_crise()
Se (busca_em_ascensao para 'pauta_relevante' == 'BREAKOUT') -> sugerir_post_oportunidade()

* - publicações semi-automáticas, com moderação humana, em resposta a determinados cenários de crise, ou seguindo tendências da web, nas seguintes redes sociais:
    * - Instagram stories e feed
    * - Status do Whatsapp
    * - tikTok stories e feed
    * - Reels no youtube


    