# scraper_newspapper3k
ok - endpoint temporario para voltar status to pending (ambiente de desenvolvimento)
ok - Realizar tarefa em segundo plano e liberar requisição
ok - log de scrapers realizados
deploy

# FrontEnd
Resumo de Scrappers realizados (system_logs)
status da URL na tab DADOS
verificar porque alguns não veio com o corpo do artigo. validar no codigo

# Schedule
Google_CSE registrar em System_logs
Definir horários de schedule de todos os modulos
Criar schedule para scraper

# Agendamento de consulta de dados contínuos com Cloud Scheduler
* Idempotência: Desenvolva seus serviços de destino de forma que múltiplas execuções da mesma tarefa não causem resultados indesejados. O Cloud Scheduler garante a entrega "pelo menos uma vez", o que significa que, em raras ocasiões, uma tarefa pode ser executada mais de uma vez.
* Monitoramento e Logs: Fique de olho nos logs do Cloud Scheduler no Cloud Logging para verificar o status de execução dos seus jobs e diagnosticar possíveis falhas.
* Gerenciamento de Erros: Configure as políticas de repetição de acordo com a necessidade da sua aplicação. Para tarefas críticas, um número maior de tentativas pode ser apropriado.

# FrontENd
No Login, sempre dá erro na primeira vez, tem que tentar 2 vezes