# Monitor Diário de Clima com n8n, Open-Meteo, Google Sheets e Gmail

## Visão geral

Este projeto é uma automação desenvolvida em n8n para consultar diariamente a previsão do tempo de uma cidade utilizando a API pública da Open-Meteo, registrar os dados em uma planilha do Google Sheets e enviar um alerta por Gmail quando determinadas condições climáticas forem identificadas.

O processo foi desenvolvido como parte de um portfólio de automações com foco em integração de APIs REST, ferramentas Google, rastreabilidade, observabilidade e boas práticas de manutenção em workflows n8n.

## Objetivo

Automatizar a coleta diária de informações climáticas, mantendo um histórico estruturado em Google Sheets e notificando por e-mail quando houver necessidade de atenção, como previsão de chuva ou temperatura elevada.

## Tecnologias utilizadas

- n8n
- Open-Meteo API
- Google Sheets
- Gmail
- HTTP Request
- Schedule Trigger
- IF Node
- Error Trigger

## API utilizada

A automação utiliza a API pública da Open-Meteo para consulta de previsão do tempo.

Endpoint base:

```text
https://api.open-meteo.com/v1/forecast
```

A Open-Meteo foi escolhida por ser uma API gratuita para uso não comercial e por não exigir API key, o que facilita a execução do projeto em ambiente de portfólio.

## Arquitetura do processo

Fluxo principal:

```text
Schedule Trigger
   ↓
Definição da cidade e parâmetros
   ↓
Consulta à API Open-Meteo
   ↓
Formatação dos dados climáticos
   ↓
Validação das regras de alerta
   ↓
Registro no Google Sheets
   ↓
Envio de alerta por Gmail, se necessário
```

## Regras de negócio

O workflow consulta diariamente a previsão do tempo e avalia se alguma condição de alerta foi atingida.

Regras iniciais:

- Enviar alerta se a temperatura máxima prevista for maior que 30°C.
- Enviar alerta se a probabilidade de chuva for maior que 60%.
- Registrar todas as execuções no Google Sheets, mesmo quando não houver alerta.

## Estrutura da planilha

A planilha Google Sheets possui três abas principais:

### Aba `weather_logs`

Responsável por armazenar os dados climáticos coletados diariamente.

Colunas:

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

### Aba `execution_logs`

Responsável por armazenar informações gerais da execução.

Colunas:

```text
execution_id
execution_date
workflow_name
status
message
```

### Aba `error_logs`

Responsável por armazenar erros capturados durante a execução.

Colunas:

```text
execution_id
error_date
workflow_name
node_name
error_message
raw_error
```

## Workflows previstos

### 1. Workflow principal

Arquivo:

```text
workflows/main-weather-monitor.json
```

Responsável por:

- iniciar a execução diariamente;
- consultar a API de clima;
- tratar os dados retornados;
- validar regras de alerta;
- registrar os dados no Google Sheets;
- enviar e-mail quando houver alerta.

### 2. Workflow global de erros

Arquivo:

```text
workflows/global-error-handler.json
```

Responsável por:

- capturar falhas de execução;
- registrar o erro em uma aba específica do Google Sheets;
- enviar alerta por Gmail para acompanhamento.

## Nodes principais

| Node | Responsabilidade |
|---|---|
| `SCHEDULE_DAILY_WEATHER_CHECK` | Inicia o workflow diariamente |
| `SET_CITY_COORDINATES` | Define cidade, coordenadas e limites de alerta |
| `API_GET_WEATHER_FORECAST` | Consulta a API Open-Meteo |
| `FORMAT_WEATHER_DATA` | Organiza os dados retornados pela API |
| `IF_ALERT_CONDITION` | Verifica se há necessidade de alerta |
| `GSHEETS_APPEND_WEATHER_LOG` | Registra o resultado no Google Sheets |
| `GMAIL_SEND_WEATHER_ALERT` | Envia alerta por e-mail |
| `LOG_EXECUTION_SUCCESS` | Registra sucesso da execução |

## Pré-requisitos

Para executar este projeto, é necessário ter:

- n8n em ambiente local ou servidor;
- conta Google com acesso ao Google Sheets e Gmail;
- credenciais Google OAuth configuradas no n8n;
- uma planilha Google Sheets criada para armazenar os logs;
- acesso à internet para consumir a API Open-Meteo.

## Variáveis e campos que precisam ser configurados

Os seguintes valores devem ser ajustados antes da execução:

```text
SUBSTITUA_PELO_ID_DA_PLANILHA
SUBSTITUA_PELO_EMAIL_DESTINATARIO
SUBSTITUA_PELA_CIDADE
SUBSTITUA_PELA_LATITUDE
SUBSTITUA_PELA_LONGITUDE
SUBSTITUA_PELO_TIMEZONE
```

Exemplo para Recife:

```text
cidade: Recife
latitude: -8.05
longitude: -34.9
timezone: America/Recife
```

## Tratamento de erros

O projeto prevê um workflow global de tratamento de erros utilizando o node `Error Trigger`.

Em caso de falha, o erro deve ser registrado na aba `error_logs` e um e-mail deve ser enviado para acompanhamento.

Exemplos de erros tratados:

- falha na chamada da API;
- erro de autenticação com Google Sheets;
- erro de autenticação com Gmail;
- estrutura inesperada no retorno da API;
- falha ao registrar dados na planilha.

## Observabilidade

A observabilidade do processo é feita por meio de logs estruturados em Google Sheets.

São registrados:

- data e hora da execução;
- ID da execução;
- cidade monitorada;
- dados climáticos retornados;
- status da execução;
- necessidade de alerta;
- motivo do alerta;
- erros capturados.

## Cenários de teste

### Cenário 1 — Execução sem alerta

Dado que a temperatura máxima é menor ou igual a 30°C  
E a probabilidade de chuva é menor ou igual a 60%  
Quando o workflow for executado  
Então os dados devem ser registrados no Google Sheets  
E nenhum e-mail de alerta deve ser enviado.

### Cenário 2 — Alerta por temperatura

Dado que a temperatura máxima é maior que 30°C  
Quando o workflow for executado  
Então os dados devem ser registrados no Google Sheets  
E um e-mail de alerta deve ser enviado.

### Cenário 3 — Alerta por chuva

Dado que a probabilidade de chuva é maior que 60%  
Quando o workflow for executado  
Então os dados devem ser registrados no Google Sheets  
E um e-mail de alerta deve ser enviado.

### Cenário 4 — Falha na API

Dado que a API esteja indisponível ou retorne erro  
Quando o workflow for executado  
Então o erro deve ser registrado na aba `error_logs`  
E um e-mail de erro deve ser enviado.

## Melhorias futuras

- Permitir monitoramento de múltiplas cidades.
- Criar uma aba de configuração no Google Sheets.
- Gerar resumo semanal por e-mail.
- Criar relatório em Google Docs.
- Salvar relatórios no Google Drive.
- Criar subworkflow reutilizável para logs.
- Adicionar controle de retry.
- Separar configurações por ambiente DEV, HML e PRD.

## Status do projeto

Em desenvolvimento.

## Autora

Projeto desenvolvido por **Paula Vasconcelos** como parte do portfólio de automações com n8n.