# Guia de Configuração

## Visão geral

Este documento descreve os passos necessários para configurar o projeto **Monitor Diário de Clima com n8n, Open-Meteo, Google Sheets e Gmail**.

O objetivo é permitir que o workflow seja configurado de forma organizada, segura e reproduzível em outro ambiente n8n.

## Pré-requisitos

Antes de configurar o workflow, é necessário ter:

- n8n instalado e em execução;
- acesso a uma conta Google;
- acesso ao Google Sheets;
- acesso ao Gmail;
- credenciais Google OAuth configuradas no n8n;
- acesso à internet para consumir a API Open-Meteo;
- repositório do projeto clonado ou disponível localmente.

## Estrutura esperada do repositório

```text
portfolio-n8n-weather-monitor/
│
├── README.md
├── .gitignore
│
├── docs/
│   ├── architecture.md
│   ├── setup.md
│   ├── test-scenarios.md
│   └── troubleshooting.md
│
├── workflows/
│   ├── main-weather-monitor.json
│   └── global-error-handler.json
│
└── assets/
    ├── workflow-overview.png
    ├── google-sheets-example.png
    └── gmail-alert-example.png
```

## 1. Criar a planilha no Google Sheets

Crie uma nova planilha no Google Sheets com o nome:

```text
portfolio_n8n_weather_monitor
```

Essa planilha será usada para registrar:

- dados climáticos coletados diariamente;
- logs gerais de execução;
- erros capturados pelo workflow.

## 2. Criar as abas da planilha

A planilha deve conter três abas:

```text
weather_logs
execution_logs
error_logs
```

Atenção: os nomes das abas devem ser mantidos exatamente como documentado, porque serão usados na configuração dos nodes do Google Sheets no n8n.

## 3. Criar a aba `weather_logs`

Na aba `weather_logs`, crie a primeira linha com os seguintes cabeçalhos:

```text
execution_id
execution_date
city
latitude
longitude
temperature_min
temperature_max
precipitation_probability
wind_speed_max
alert_required
alert_reason
status
```

Essa aba será responsável por armazenar o histórico diário dos dados climáticos.

## 4. Criar a aba `execution_logs`

Na aba `execution_logs`, crie a primeira linha com os seguintes cabeçalhos:

```text
execution_id
execution_date
workflow_name
status
message
```

Essa aba será responsável por armazenar um resumo das execuções realizadas pelo workflow.

## 5. Criar a aba `error_logs`

Na aba `error_logs`, crie a primeira linha com os seguintes cabeçalhos:

```text
execution_id
error_date
workflow_name
node_name
error_message
raw_error
```

Essa aba será responsável por armazenar erros capturados pelo workflow global de tratamento de erros.

## 6. Obter o ID da planilha

O ID da planilha fica na URL do Google Sheets.

Exemplo de URL:

```text
https://docs.google.com/spreadsheets/d/1AbCDefGhIJkLmNoP1234567890/edit#gid=0
```

Neste exemplo, o ID da planilha é:

```text
1AbCDefGhIJkLmNoP1234567890
```

No workflow, esse valor deve substituir o placeholder:

```text
SUBSTITUA_PELO_ID_DA_PLANILHA
```

Importante: não publique IDs reais de planilhas sensíveis no GitHub.

## 7. Configurar credenciais Google no n8n

O projeto utiliza integrações com:

- Google Sheets;
- Gmail.

Por isso, é necessário configurar credenciais Google OAuth no n8n.

### Permissões necessárias

As credenciais devem permitir:

```text
ler e escrever em Google Sheets
enviar e-mails pelo Gmail
```

No n8n, essas permissões normalmente são configuradas durante a criação das credenciais dos nodes Google Sheets e Gmail.

## 8. Configurar credencial do Google Sheets

No n8n:

1. Abra ou crie o node Google Sheets.
2. Selecione uma credencial existente ou crie uma nova.
3. Faça autenticação com sua conta Google.
4. Confirme que a conta tem acesso à planilha criada.
5. Teste a credencial usando uma operação simples, como buscar ou adicionar uma linha.

### Pontos de atenção

- A conta autenticada precisa ter permissão de edição na planilha.
- Se a planilha estiver em outro Google Drive, compartilhe a planilha com a conta usada no n8n.
- Não exponha prints com e-mail pessoal ou ID real da planilha, se o repositório for público.

## 9. Configurar credencial do Gmail

No n8n:

1. Abra ou crie o node Gmail.
2. Selecione uma credencial existente ou crie uma nova.
3. Faça autenticação com sua conta Google.
4. Confirme as permissões solicitadas.
5. Teste com envio para um e-mail seu.

### Pontos de atenção

- Use um destinatário de teste no início.
- Evite disparos repetidos durante os testes.
- Caso use uma conta pessoal, tome cuidado para não expor o endereço no GitHub.

## 10. Configurar cidade monitorada

Na primeira versão do workflow, a cidade será configurada diretamente em um node do tipo Set / Edit Fields.

Campos esperados:

```text
city
latitude
longitude
timezone
alert_temperature_limit
alert_rain_probability_limit
```

Exemplo usando Recife:

```text
city: Recife
latitude: -8.05
longitude: -34.9
timezone: America/Recife
alert_temperature_limit: 30
alert_rain_probability_limit: 60
```

## 11. Configurar consulta à API Open-Meteo

O workflow utiliza o node HTTP Request para consultar a API Open-Meteo.

Endpoint base:

```text
https://api.open-meteo.com/v1/forecast
```

Parâmetros esperados:

```text
latitude
longitude
daily
timezone
```

Valor esperado para o parâmetro `daily`:

```text
temperature_2m_max,temperature_2m_min,precipitation_probability_max,windspeed_10m_max
```

Exemplo de chamada completa para teste no navegador:

```text
https://api.open-meteo.com/v1/forecast?latitude=-8.05&longitude=-34.9&daily=temperature_2m_max,temperature_2m_min,precipitation_probability_max,windspeed_10m_max&timezone=America%2FRecife
```

## 12. Configurar regras de alerta

As regras iniciais do projeto são:

```text
temperature_max > 30
OU
precipitation_probability > 60
```

No workflow, esses limites devem vir dos campos:

```text
alert_temperature_limit
alert_rain_probability_limit
```

Isso facilita manutenção, porque evita deixar valores fixos diretamente no node IF.

## 13. Configurar node de alerta por Gmail

O node Gmail deve enviar e-mail apenas quando a condição de alerta for verdadeira.

Assunto sugerido:

```text
[PORTFÓLIO n8n] Alerta climático diário - Recife
```

Corpo sugerido:

```text
Olá,

Foi identificado um alerta climático para a cidade monitorada.

Cidade: {{$json.city}}
Temperatura mínima: {{$json.temperature_min}}°C
Temperatura máxima: {{$json.temperature_max}}°C
Probabilidade de chuva: {{$json.precipitation_probability}}%
Velocidade máxima do vento: {{$json.wind_speed_max}} km/h

Motivo do alerta:
{{$json.alert_reason}}

ID da execução:
{{$json.execution_id}}

Este e-mail foi gerado automaticamente por um workflow n8n.
```

Campo que precisa ser substituído:

```text
SUBSTITUA_PELO_EMAIL_DESTINATARIO
```

## 14. Configurar registro no Google Sheets

O node Google Sheets responsável pelo registro climático deve adicionar uma nova linha na aba:

```text
weather_logs
```

Mapeamento esperado:

```text
execution_id → execution_id
execution_date → execution_date
city → city
latitude → latitude
longitude → longitude
temperature_min → temperature_min
temperature_max → temperature_max
precipitation_probability → precipitation_probability
wind_speed_max → wind_speed_max
alert_required → alert_required
alert_reason → alert_reason
status → status
```

O log de execução deve ser registrado na aba:

```text
execution_logs
```

Mapeamento esperado:

```text
execution_id → execution_id
execution_date → execution_date
workflow_name → workflow_name
status → status
message → message
```

## 15. Configurar workflow global de erros

O projeto prevê um workflow separado para tratamento global de erros.

Esse workflow deve conter:

```text
ERROR_GLOBAL_TRIGGER
FORMAT_ERROR_PAYLOAD
GSHEETS_APPEND_ERROR_LOG
GMAIL_SEND_ERROR_ALERT
```

Após criar o workflow global de erros, configure o workflow principal para usar esse Error Workflow nas configurações do próprio workflow.

### Dados esperados no log de erro

```text
execution_id
error_date
workflow_name
node_name
error_message
raw_error
```

## 16. Placeholders usados no projeto

Antes de executar o workflow, substitua todos os placeholders abaixo:

```text
SUBSTITUA_PELO_ID_DA_PLANILHA
SUBSTITUA_PELO_EMAIL_DESTINATARIO
SUBSTITUA_PELA_CIDADE
SUBSTITUA_PELA_LATITUDE
SUBSTITUA_PELA_LONGITUDE
SUBSTITUA_PELO_TIMEZONE
```

## 17. Segurança

Não devem ser versionados no GitHub:

- credenciais Google;
- tokens OAuth;
- arquivos `.env`;
- banco local do n8n;
- e-mails pessoais ou corporativos;
- IDs reais de planilhas sensíveis;
- prints com dados sensíveis.

Use placeholders sempre que documentar valores privados.

## 18. Teste inicial recomendado

Antes de ativar o agendamento, execute o workflow manualmente.

Valide:

- se a API retorna dados;
- se os dados são formatados corretamente;
- se uma linha é registrada em `weather_logs`;
- se uma linha é registrada em `execution_logs`;
- se o Gmail é enviado quando a regra de alerta é verdadeira;
- se não há credenciais expostas no JSON exportado.

## 19. Ativação do agendamento

Depois que os testes manuais forem concluídos, configure o Schedule Trigger.

Configuração sugerida:

```text
Frequência: diária
Horário: 08:00
Timezone: America/Recife
```

Ative o workflow apenas depois de validar:

- credenciais;
- planilha;
- mapeamento de colunas;
- condição de alerta;
- envio de e-mail;
- tratamento de erro.

## Checklist de configuração

- [ ] Planilha criada.
- [ ] Aba `weather_logs` criada.
- [ ] Aba `execution_logs` criada.
- [ ] Aba `error_logs` criada.
- [ ] Cabeçalhos configurados corretamente.
- [ ] ID da planilha identificado.
- [ ] Credencial Google Sheets configurada.
- [ ] Credencial Gmail configurada.
- [ ] Cidade configurada.
- [ ] Coordenadas configuradas.
- [ ] Timezone configurado.
- [ ] API Open-Meteo testada.
- [ ] Regras de alerta configuradas.
- [ ] Registro no Google Sheets testado.
- [ ] Envio de Gmail testado.
- [ ] Workflow global de erros configurado.
- [ ] Schedule Trigger ativado.
- [ ] Dados sensíveis removidos antes do commit.