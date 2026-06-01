# Cenários de Teste

## Visão geral

Este documento descreve os cenários de teste previstos para o workflow **Monitor Diário de Clima com n8n, Open-Meteo, Google Sheets e Gmail**.

O objetivo dos testes é validar se o processo funciona corretamente em situações normais, em situações de alerta e em cenários de falha.

## Objetivos dos testes

Os testes devem garantir que o workflow:

- consulta corretamente a API Open-Meteo;
- formata os dados climáticos corretamente;
- aplica as regras de alerta;
- registra os dados no Google Sheets;
- envia e-mail quando necessário;
- registra logs de execução;
- aciona o tratamento de erros em caso de falha;
- não expõe dados sensíveis.

## Escopo dos testes

Os testes cobrem:

- execução manual;
- execução agendada;
- cenário sem alerta;
- alerta por temperatura;
- alerta por chuva;
- falha na API;
- falha no Google Sheets;
- falha no Gmail;
- validação dos logs.

## Dados base para teste

Cidade usada como exemplo:

```text
city: Recife
latitude: -8.05
longitude: -34.9
timezone: America/Recife
alert_temperature_limit: 30
alert_rain_probability_limit: 60
```

Planilha usada:

```text
portfolio_n8n_weather_monitor
```

Abas esperadas:

```text
weather_logs
execution_logs
error_logs
```

## Cenário 1 — Execução manual com sucesso

### Objetivo

Validar se o workflow executa corretamente quando iniciado manualmente no n8n.

### Pré-condições

- n8n em execução;
- credenciais Google configuradas;
- planilha criada;
- abas e cabeçalhos configurados;
- internet disponível;
- API Open-Meteo acessível.

### Passos

1. Abrir o workflow principal no n8n.
2. Clicar em **Execute Workflow**.
3. Aguardar a execução de todos os nodes.
4. Verificar o retorno do node `API_GET_WEATHER_FORECAST`.
5. Verificar os dados gerados no node `FORMAT_WEATHER_DATA`.
6. Conferir os registros no Google Sheets.

### Resultado esperado

- O workflow deve executar sem erro.
- A API deve retornar dados climáticos.
- Os dados devem ser formatados corretamente.
- Uma linha deve ser criada na aba `weather_logs`.
- Uma linha deve ser criada na aba `execution_logs`.

### Evidência recomendada

- Print do workflow executado com sucesso.
- Print da linha criada na aba `weather_logs`.
- Print da linha criada na aba `execution_logs`.

---

## Cenário 2 — Execução sem alerta

### Objetivo

Validar se o workflow registra os dados corretamente quando nenhuma regra de alerta é atingida.

### Condição de teste

```text
temperature_max <= 30
precipitation_probability <= 60
```

### Passos

1. Executar o workflow manualmente.
2. Verificar os valores retornados pela API.
3. Confirmar que o node `IF_ALERT_CONDITION` seguiu pelo caminho `false`.
4. Conferir se os dados foram registrados no Google Sheets.
5. Confirmar que nenhum e-mail de alerta foi enviado.

### Resultado esperado

- O workflow deve registrar os dados na aba `weather_logs`.
- O campo `alert_required` deve ser `false`.
- O campo `alert_reason` deve indicar ausência de alerta.
- Nenhum Gmail de alerta deve ser enviado.
- A execução deve ser registrada como sucesso.

### Evidência recomendada

- Print do IF seguindo pelo caminho `false`.
- Print da linha registrada no Google Sheets.

---

## Cenário 3 — Alerta por temperatura

### Objetivo

Validar se o workflow envia alerta quando a temperatura máxima ultrapassa o limite configurado.

### Condição de teste

```text
temperature_max > alert_temperature_limit
```

Exemplo:

```text
temperature_max: 32
alert_temperature_limit: 30
```

### Passos

1. Ajustar temporariamente o limite de temperatura para facilitar o teste, se necessário.
2. Executar o workflow manualmente.
3. Verificar o node `IF_ALERT_CONDITION`.
4. Confirmar que o caminho `true` foi seguido.
5. Verificar se o e-mail de alerta foi enviado.
6. Verificar se a execução foi registrada no Google Sheets.

### Resultado esperado

- O node IF deve seguir pelo caminho `true`.
- Um Gmail de alerta deve ser enviado.
- A linha na aba `weather_logs` deve conter `alert_required` como `true`.
- O campo `alert_reason` deve indicar alerta por temperatura.
- A execução deve ser registrada como sucesso.

### Evidência recomendada

- Print do IF no caminho `true`.
- Print do e-mail recebido.
- Print da linha registrada na aba `weather_logs`.

---

## Cenário 4 — Alerta por chuva

### Objetivo

Validar se o workflow envia alerta quando a probabilidade de chuva ultrapassa o limite configurado.

### Condição de teste

```text
precipitation_probability > alert_rain_probability_limit
```

Exemplo:

```text
precipitation_probability: 75
alert_rain_probability_limit: 60
```

### Passos

1. Ajustar temporariamente o limite de chuva para facilitar o teste, se necessário.
2. Executar o workflow manualmente.
3. Verificar o node `IF_ALERT_CONDITION`.
4. Confirmar que o caminho `true` foi seguido.
5. Verificar se o e-mail de alerta foi enviado.
6. Verificar se a execução foi registrada no Google Sheets.

### Resultado esperado

- O node IF deve seguir pelo caminho `true`.
- Um Gmail de alerta deve ser enviado.
- A linha na aba `weather_logs` deve conter `alert_required` como `true`.
- O campo `alert_reason` deve indicar alerta por chuva.
- A execução deve ser registrada como sucesso.

### Evidência recomendada

- Print do IF no caminho `true`.
- Print do e-mail recebido.
- Print da linha registrada na aba `weather_logs`.

---

## Cenário 5 — Alerta por temperatura e chuva

### Objetivo

Validar se o workflow identifica corretamente quando mais de uma regra de alerta é atingida.

### Condição de teste

```text
temperature_max > alert_temperature_limit
E
precipitation_probability > alert_rain_probability_limit
```

### Passos

1. Ajustar temporariamente os limites, se necessário.
2. Executar o workflow manualmente.
3. Verificar os dados tratados.
4. Confirmar que o IF seguiu pelo caminho `true`.
5. Verificar o conteúdo do campo `alert_reason`.
6. Confirmar o envio do Gmail.
7. Conferir o registro no Google Sheets.

### Resultado esperado

- O node IF deve seguir pelo caminho `true`.
- O e-mail deve ser enviado.
- O campo `alert_required` deve ser `true`.
- O campo `alert_reason` deve indicar as duas causas do alerta.
- A execução deve ser registrada corretamente.

### Evidência recomendada

- Print do node `FORMAT_WEATHER_DATA`.
- Print do e-mail recebido.
- Print do registro no Google Sheets.

---

## Cenário 6 — Falha na API Open-Meteo

### Objetivo

Validar se o workflow lida corretamente com erro na chamada da API externa.

### Como simular

Alterar temporariamente a URL do node `API_GET_WEATHER_FORECAST`.

Exemplo incorreto:

```text
https://api.open-meteo.com/v1/forecast-error
```

### Passos

1. Alterar temporariamente o endpoint da API para uma URL inválida.
2. Executar o workflow manualmente.
3. Verificar se a execução falha no node `API_GET_WEATHER_FORECAST`.
4. Validar se o workflow global de erros foi acionado.
5. Conferir se uma linha foi registrada na aba `error_logs`.
6. Verificar se um Gmail de erro foi enviado.

### Resultado esperado

- O node `API_GET_WEATHER_FORECAST` deve falhar.
- O erro deve ser capturado pelo workflow global de erros.
- Uma linha deve ser registrada na aba `error_logs`.
- Um e-mail de erro deve ser enviado.
- O erro deve indicar falha na chamada da API.

### Evidência recomendada

- Print do erro no n8n.
- Print da linha na aba `error_logs`.
- Print do e-mail de erro.

### Pós-teste

Restaurar a URL correta:

```text
https://api.open-meteo.com/v1/forecast
```

---

## Cenário 7 — Falha no Google Sheets

### Objetivo

Validar se o erro é capturado quando o workflow não consegue registrar dados na planilha.

### Como simular

Escolha uma das opções abaixo:

```text
usar um ID de planilha inválido
alterar temporariamente o nome da aba
remover permissão de edição da conta Google autenticada
```

### Passos

1. Simular uma falha no node `GSHEETS_APPEND_WEATHER_LOG`.
2. Executar o workflow manualmente.
3. Confirmar que o node Google Sheets retornou erro.
4. Verificar se o workflow global de erros foi acionado.
5. Conferir registro na aba `error_logs`.
6. Validar envio do Gmail de erro.

### Resultado esperado

- O node Google Sheets deve falhar.
- O erro deve indicar problema de acesso, planilha ou aba.
- A falha deve ser registrada em `error_logs`.
- Um e-mail de erro deve ser enviado.

### Evidência recomendada

- Print do erro no node Google Sheets.
- Print do log de erro.
- Print do e-mail de erro.

### Pós-teste

Restaurar o ID da planilha, nome da aba ou permissão correta.

---

## Cenário 8 — Falha no Gmail

### Objetivo

Validar comportamento quando o workflow não consegue enviar o e-mail de alerta.

### Como simular

Escolha uma das opções abaixo:

```text
usar uma credencial Gmail inválida
revogar temporariamente a permissão OAuth
usar um destinatário inválido
```

### Passos

1. Forçar uma condição de alerta.
2. Simular falha no node `GMAIL_SEND_WEATHER_ALERT`.
3. Executar o workflow manualmente.
4. Verificar se o node Gmail retornou erro.
5. Validar se o erro foi capturado pelo workflow global.
6. Conferir registro na aba `error_logs`.

### Resultado esperado

- O node Gmail deve falhar.
- O erro deve ser registrado na aba `error_logs`.
- O workflow global de erros deve ser acionado.
- A causa da falha deve ser identificável no log.

### Evidência recomendada

- Print do erro no node Gmail.
- Print do log em `error_logs`.

### Observação

Durante testes, evite fazer muitos disparos de e-mail em sequência para não gerar bloqueios ou limitações temporárias.

---

## Cenário 9 — Validação dos logs de execução

### Objetivo

Validar se toda execução bem-sucedida gera um registro operacional.

### Passos

1. Executar o workflow manualmente.
2. Acessar a aba `execution_logs`.
3. Conferir se uma nova linha foi criada.
4. Validar os campos preenchidos.

### Resultado esperado

A aba `execution_logs` deve conter:

```text
execution_id
execution_date
workflow_name
status
message
```

O status esperado é:

```text
success
```

### Evidência recomendada

- Print da aba `execution_logs`.

---

## Cenário 10 — Execução agendada

### Objetivo

Validar se o workflow executa automaticamente pelo Schedule Trigger.

### Pré-condições

- Workflow ativado;
- Schedule Trigger configurado;
- credenciais válidas;
- planilha disponível;
- internet ativa.

### Passos

1. Configurar o Schedule Trigger.
2. Ativar o workflow.
3. Aguardar a próxima execução agendada.
4. Verificar o histórico de execuções no n8n.
5. Conferir registros no Google Sheets.

### Resultado esperado

- O workflow deve executar automaticamente.
- Uma nova linha deve ser criada em `weather_logs`.
- Uma nova linha deve ser criada em `execution_logs`.
- Não deve haver erro na execução.

### Evidência recomendada

- Print do workflow ativo.
- Print do histórico de execução.
- Print dos registros no Google Sheets.

---

## Cenário 11 — Validação de dados sensíveis

### Objetivo

Garantir que nenhum dado sensível seja publicado no GitHub.

### Itens a validar

Antes de fazer commit, verificar se o repositório não contém:

```text
tokens OAuth
credenciais Google
arquivo .env
ID real de planilha sensível
e-mails pessoais ou corporativos
prints com dados sensíveis
banco local do n8n
```

### Passos

1. Revisar o arquivo JSON exportado do n8n.
2. Revisar os arquivos Markdown.
3. Revisar prints adicionados à pasta `assets`.
4. Executar `git status`.
5. Conferir os arquivos que serão commitados.
6. Substituir dados sensíveis por placeholders.

### Resultado esperado

O repositório deve conter apenas informações públicas ou placeholders.

Placeholders aceitos:

```text
SUBSTITUA_PELO_ID_DA_PLANILHA
SUBSTITUA_PELO_EMAIL_DESTINATARIO
SUBSTITUA_PELA_CIDADE
SUBSTITUA_PELA_LATITUDE
SUBSTITUA_PELA_LONGITUDE
SUBSTITUA_PELO_TIMEZONE
```

---

## Checklist geral de testes

- [ ] Workflow executa manualmente com sucesso.
- [ ] API Open-Meteo retorna dados válidos.
- [ ] Dados são formatados corretamente.
- [ ] Regra sem alerta funciona.
- [ ] Regra de alerta por temperatura funciona.
- [ ] Regra de alerta por chuva funciona.
- [ ] Regra com múltiplos alertas funciona.
- [ ] Registro em `weather_logs` funciona.
- [ ] Registro em `execution_logs` funciona.
- [ ] Gmail de alerta é enviado.
- [ ] Falha na API é capturada.
- [ ] Falha no Google Sheets é capturada.
- [ ] Falha no Gmail é capturada.
- [ ] Erros são registrados em `error_logs`.
- [ ] Workflow agendado executa corretamente.
- [ ] Dados sensíveis foram removidos antes do commit.

## Observações finais

Os testes devem ser realizados primeiro manualmente e somente depois com o workflow ativado por agendamento.

Durante os testes, recomenda-se utilizar uma conta Google e uma planilha próprias para portfólio, evitando dados reais ou sensíveis.