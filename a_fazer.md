# scraper_newspapper3k
endpoint temporario para voltar status to pending (ambiente de desenvolvimento)
Realizar tarefa em segundo plano e liberar requisição
verificar porque alguns não veio com o corpo do artigo. validar no codigo
Resumo de Scrappers no frontend // status da URL na tab DADOS
deploy
Definir horários de schedule de todos os modulos
criar schedule

# Agendamento de consulta de dados contínuos com Cloud Scheduler
* Idempotência: Desenvolva seus serviços de destino de forma que múltiplas execuções da mesma tarefa não causem resultados indesejados. O Cloud Scheduler garante a entrega "pelo menos uma vez", o que significa que, em raras ocasiões, uma tarefa pode ser executada mais de uma vez.
* Monitoramento e Logs: Fique de olho nos logs do Cloud Scheduler no Cloud Logging para verificar o status de execução dos seus jobs e diagnosticar possíveis falhas.
* Gerenciamento de Erros: Configure as políticas de repetição de acordo com a necessidade da sua aplicação. Para tarefas críticas, um número maior de tentativas pode ser apropriado.

# FrontENd
No Login, sempre dá erro na primeira vez, tem que tentar 2 vezes

