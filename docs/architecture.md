# Arquitetura do Workflow

## Visão geral

Este documento descreve a arquitetura do workflow **Monitor Diário de Clima**, desenvolvido em n8n.

O processo tem como objetivo consultar diariamente a previsão do tempo por meio da API pública da Open-Meteo, registrar os dados em uma planilha Google Sheets e enviar um alerta por Gmail quando determinadas condições climáticas forem identificadas.

A arquitetura foi pensada para ser simples, legível e adequada para um projeto de portfólio, mantendo boas práticas importantes de automação corporativa, como rastreabilidade, observabilidade e tratamento de erros.

## Objetivo arquitetural

O objetivo da arquitetura é demonstrar um fluxo básico, porém bem estruturado, seguindo a sequência:

```text
Agendamento
   ↓
Definição de parâmetros
   ↓
Consulta em API externa
   ↓
Tratamento dos dados
   ↓
Validação de regra de negócio
   ↓
Registro em log
   ↓
Notificação, se necessário
```

Essa estrutura pode ser reutilizada em diferentes contextos corporativos, como monitoramento de indicadores, coleta de dados externos, alertas operacionais e atualização de bases em planilhas.

## Princípios utilizados

A arquitetura considera os seguintes princípios:

- simplicidade;
- baixo acoplamento;
- clareza para manutenção;
- uso prioritário de nodes nativos do n8n;
- rastreabilidade das execuções;
- tratamento de erros;
- facilidade de evolução futura.

## Desenho lógico do fluxo principal

```text
SCHEDULE_DAILY_WEATHER_CHECK
   ↓
SET_CITY_COORDINATES
   ↓
API_GET_WEATHER_FORECAST
   ↓
FORMAT_WEATHER_DATA
   ↓
IF_ALERT_CONDITION
   ├── true  → GMAIL_SEND_WEATHER_ALERT
   │              ↓
   │          GSHEETS_APPEND_WEATHER_LOG
   │              ↓
   │          LOG_EXECUTION_SUCCESS
   │
   └── false → GSHEETS_APPEND_WEATHER_LOG
                  ↓
              LOG_EXECUTION_SUCCESS
```

## Responsabilidade dos nodes

### `SCHEDULE_DAILY_WEATHER_CHECK`

**Tipo:** Schedule Trigger

**Responsabilidade:**  
Iniciar o workflow automaticamente em uma frequência definida.

**Entrada:**  
Não recebe dados externos. O próprio agendamento dispara a execução.

**Saída esperada:**  
Um item inicial contendo metadados da execução.

**Configuração esperada:**

```text
Frequência: diária
Horário sugerido: 08:00
Timezone: America/Recife
```

**Motivo da escolha:**  
O Schedule Trigger é um node nativo do n8n e atende bem ao objetivo de executar o processo uma vez por dia sem intervenção manual.

---

### `SET_CITY_COORDINATES`

**Tipo:** Set / Edit Fields

**Responsabilidade:**  
Centralizar os parâmetros da cidade monitorada e os limites usados nas regras de alerta.

**Entrada:**  
Item gerado pelo Schedule Trigger.

**Saída esperada:**  
Objeto com dados de configuração da cidade.

**Campos esperados:**

```text
city
latitude
longitude
timezone
alert_temperature_limit
alert_rain_probability_limit
```

**Exemplo:**

```text
city: Recife
latitude: -8.05
longitude: -34.9
timezone: America/Recife
alert_temperature_limit: 30
alert_rain_probability_limit: 60
```

**Motivo da escolha:**  
Manter esses valores em um node separado facilita manutenção e evita deixar parâmetros espalhados pelo workflow.

**Evolução futura:**  
Esses parâmetros podem ser movidos para uma aba `config` no Google Sheets, permitindo alterar a cidade e os limites sem editar o workflow.

---

### `API_GET_WEATHER_FORECAST`

**Tipo:** HTTP Request

**Responsabilidade:**  
Consultar a API Open-Meteo para buscar a previsão do tempo da cidade configurada.

**Entrada:**  
Dados de cidade e coordenadas vindos do node `SET_CITY_COORDINATES`.

**Saída esperada:**  
Resposta JSON da API contendo dados climáticos diários.

**Endpoint base:**

```text
https://api.open-meteo.com/v1/forecast
```

**Parâmetros esperados:**

```text
latitude
longitude
daily
timezone
```

**Exemplo de campos solicitados:**

```text
temperature_2m_max
temperature_2m_min
precipitation_probability_max
windspeed_10m_max
```

**Motivo da escolha:**  
O HTTP Request é o node nativo mais adequado para consumo de APIs REST no n8n.

**Riscos:**  

- indisponibilidade temporária da API;
- alteração inesperada na estrutura da resposta;
- falha de conectividade;
- timeout.

**Boas práticas:**  

- manter o endpoint separado dos parâmetros;
- validar a estrutura retornada antes de usar os dados;
- registrar falhas no workflow global de erros.

---

### `FORMAT_WEATHER_DATA`

**Tipo:** Set / Edit Fields

**Responsabilidade:**  
Transformar a resposta bruta da API em um objeto limpo, padronizado e pronto para registro no Google Sheets.

**Entrada:**  
JSON retornado pelo node `API_GET_WEATHER_FORECAST`.

**Saída esperada:**  
Objeto padronizado com dados tratados.

**Campos gerados:**

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

**Exemplos de expressões esperadas:**

```text
temperature_max: {{$json.daily.temperature_2m_max[0]}}
temperature_min: {{$json.daily.temperature_2m_min[0]}}
precipitation_probability: {{$json.daily.precipitation_probability_max[0]}}
wind_speed_max: {{$json.daily.windspeed_10m_max[0]}}
```

**Motivo da escolha:**  
Separar a formatação em um node próprio melhora a legibilidade e reduz acoplamento entre a API e os nodes de saída.

**Quando evitar lógica excessiva aqui:**  
Se as regras de negócio crescerem muito, o ideal é separar validações em nodes específicos ou subworkflows.

---

### `IF_ALERT_CONDITION`

**Tipo:** IF

**Responsabilidade:**  
Avaliar se os dados climáticos atingem alguma regra de alerta.

**Entrada:**  
Dados tratados vindos do node `FORMAT_WEATHER_DATA`.

**Saída esperada:**  

- caminho `true`: quando houver alerta;
- caminho `false`: quando não houver alerta.

**Regras iniciais:**

```text
temperature_max > alert_temperature_limit
OU
precipitation_probability > alert_rain_probability_limit
```

**Motivo da escolha:**  
O IF Node é nativo, visual e fácil de manter. Para uma regra simples, ele é preferível ao Code Node.

**Boas práticas:**  

- nomear a condição com clareza;
- evitar muitas regras complexas no mesmo IF;
- documentar os limites usados.

---

### `GMAIL_SEND_WEATHER_ALERT`

**Tipo:** Gmail

**Responsabilidade:**  
Enviar um e-mail de alerta quando a condição climática definida for atingida.

**Entrada:**  
Dados tratados do clima e motivo do alerta.

**Saída esperada:**  
Confirmação de envio do e-mail.

**Exemplo de assunto:**

```text
[PORTFÓLIO n8n] Alerta climático diário - Recife
```

**Conteúdo esperado do e-mail:**

```text
Cidade monitorada
Temperatura mínima
Temperatura máxima
Probabilidade de chuva
Velocidade máxima do vento
Motivo do alerta
ID da execução
```

**Motivo da escolha:**  
O Gmail é uma integração nativa do n8n e adequada ao ecossistema Google.

**Riscos:**  

- falha de autenticação OAuth;
- permissão insuficiente;
- limite de envio do Gmail;
- e-mail cair em spam.

---

### `GSHEETS_APPEND_WEATHER_LOG`

**Tipo:** Google Sheets

**Responsabilidade:**  
Registrar os dados da execução na aba `weather_logs`.

**Entrada:**  
Dados tratados da execução.

**Saída esperada:**  
Nova linha adicionada na planilha.

**Aba de destino:**

```text
weather_logs
```

**Colunas esperadas:**

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

**Motivo da escolha:**  
O Google Sheets é simples, visual e suficiente para observabilidade de um projeto de portfólio.

**Boas práticas:**  

- manter nomes de colunas estáveis;
- evitar alteração manual na estrutura da planilha;
- registrar também execuções sem alerta;
- usar placeholders para IDs sensíveis da planilha.

---

### `LOG_EXECUTION_SUCCESS`

**Tipo:** Google Sheets ou Set / Edit Fields seguido de Google Sheets

**Responsabilidade:**  
Registrar uma linha resumida de sucesso na aba `execution_logs`.

**Entrada:**  
Resultado final da execução.

**Saída esperada:**  
Registro de sucesso da execução.

**Aba de destino:**

```text
execution_logs
```

**Colunas esperadas:**

```text
execution_id
execution_date
workflow_name
status
message
```

**Motivo da escolha:**  
Separar logs de execução dos dados climáticos ajuda a entender o histórico operacional do workflow.

---

## Workflow global de erros

Além do workflow principal, o projeto prevê um segundo workflow para tratamento global de erros.

### Desenho lógico

```text
ERROR_GLOBAL_TRIGGER
   ↓
FORMAT_ERROR_PAYLOAD
   ↓
GSHEETS_APPEND_ERROR_LOG
   ↓
GMAIL_SEND_ERROR_ALERT
```

### Responsabilidade

O workflow global de erros deve capturar falhas ocorridas no workflow principal, registrar o erro na aba `error_logs` e enviar um alerta por Gmail.

### Nodes previstos

| Node | Responsabilidade |
|---|---|
| `ERROR_GLOBAL_TRIGGER` | Captura erros de workflows configurados para usar este handler |
| `FORMAT_ERROR_PAYLOAD` | Organiza informações relevantes do erro |
| `GSHEETS_APPEND_ERROR_LOG` | Registra o erro no Google Sheets |
| `GMAIL_SEND_ERROR_ALERT` | Envia e-mail de alerta sobre falha |

### Dados esperados no log de erro

```text
execution_id
error_date
workflow_name
node_name
error_message
raw_error
```

### Boas práticas

- evitar registrar tokens ou dados sensíveis no log;
- capturar nome do workflow e node com erro;
- manter o e-mail de erro objetivo;
- registrar o erro mesmo que o alerta por e-mail falhe;
- testar o Error Trigger antes de considerar o processo pronto.

---

## Inputs do processo

O workflow principal utiliza os seguintes inputs configurados internamente:

```text
city
latitude
longitude
timezone
alert_temperature_limit
alert_rain_probability_limit
spreadsheet_id
recipient_email
```

Esses valores devem ser configurados com placeholders no repositório público.

Exemplo:

```text
SUBSTITUA_PELA_CIDADE
SUBSTITUA_PELA_LATITUDE
SUBSTITUA_PELA_LONGITUDE
SUBSTITUA_PELO_TIMEZONE
SUBSTITUA_PELO_ID_DA_PLANILHA
SUBSTITUA_PELO_EMAIL_DESTINATARIO
```

## Outputs do processo

O processo gera três tipos principais de saída:

### 1. Registro climático

Gravado na aba:

```text
weather_logs
```

### 2. Registro operacional

Gravado na aba:

```text
execution_logs
```

### 3. Alerta por e-mail

Enviado somente quando uma regra de alerta for atendida.

## Observabilidade

A observabilidade é feita por meio de logs estruturados em Google Sheets.

O objetivo é permitir responder perguntas como:

- o workflow executou hoje?
- houve alerta?
- qual foi a temperatura máxima registrada?
- a API retornou dados corretamente?
- houve falha em algum node?
- qual foi o motivo do alerta?

## Decisões técnicas

### Uso de Google Sheets como camada de log

O Google Sheets foi escolhido por ser simples, visual, gratuito e adequado para um projeto de portfólio.

Para um ambiente produtivo com alto volume, uma base de dados mais robusta poderia ser considerada. Neste projeto, o Google Sheets é suficiente.

### Uso de Gmail para alertas

O Gmail foi escolhido por estar dentro do ecossistema Google e por possuir integração nativa com o n8n.

### Uso de HTTP Request para a API

A API Open-Meteo não possui node específico no n8n. Por isso, o HTTP Request é a opção correta.

### Evitar Code Node no início

A arquitetura evita Code Node porque as regras iniciais podem ser resolvidas com nodes nativos e expressões simples.

O Code Node só deve ser considerado se as regras de alerta ficarem mais complexas.

## Riscos operacionais

| Risco | Impacto | Mitigação |
|---|---|---|
| API indisponível | Workflow falha na coleta | Registrar erro e enviar alerta |
| Mudança no retorno da API | Campos podem quebrar | Validar estrutura antes de formatar |
| Falha na credencial Google | Logs e e-mails podem falhar | Revisar credenciais OAuth |
| Alteração manual na planilha | Append pode falhar ou gravar errado | Proteger cabeçalhos |
| Limite do Gmail | Alerta pode não ser enviado | Evitar envio excessivo |
| Dados sensíveis no GitHub | Exposição de informações | Usar placeholders |

## Estratégia de evolução

A arquitetura atual foi pensada para começar simples e permitir evolução gradual.

Evoluções possíveis:

1. Ler configurações de uma aba `config` no Google Sheets.
2. Monitorar múltiplas cidades.
3. Criar subworkflow reutilizável para logs.
4. Criar resumo semanal por Gmail.
5. Salvar relatório semanal no Google Drive.
6. Criar documentação visual com prints do workflow.
7. Separar ambientes DEV, HML e PRD.

## Critérios de sucesso

O workflow será considerado funcional quando:

- executar manualmente sem erros;
- executar por agendamento;
- consultar corretamente a API Open-Meteo;
- registrar dados na aba `weather_logs`;
- registrar execução na aba `execution_logs`;
- enviar Gmail quando houver alerta;
- capturar erros no workflow global;
- não expor credenciais ou dados sensíveis no GitHub.

## Checklist de arquitetura

- [ ] Workflow principal definido.
- [ ] Workflow global de erros previsto.
- [ ] Nodes nomeados com padrão claro.
- [ ] API externa documentada.
- [ ] Inputs e outputs documentados.
- [ ] Logs definidos.
- [ ] Riscos operacionais mapeados.
- [ ] Melhorias futuras listadas.
- [ ] Code Node evitado na primeira versão.
- [ ] Credenciais e dados sensíveis protegidos.