# Troubleshooting

## Visão geral

Este documento reúne erros comuns, causas prováveis e formas de correção para o workflow **Monitor Diário de Clima com n8n, Open-Meteo, Google Sheets e Gmail**.

O objetivo é facilitar a manutenção do processo e ajudar na identificação rápida de problemas durante testes ou execução do workflow.

## Como usar este guia

Quando ocorrer um erro:

1. Identifique em qual node o erro aconteceu.
2. Leia a mensagem exibida pelo n8n.
3. Compare com os cenários deste documento.
4. Aplique a correção sugerida.
5. Execute novamente o workflow manualmente.
6. Registre o erro, se necessário, na aba `error_logs`.

## Erro 1 — API Open-Meteo não retorna dados

### Node provável

```text
API_GET_WEATHER_FORECAST
```

### Sintomas

- O node HTTP Request retorna erro.
- O workflow para na etapa de consulta da API.
- Não há dados disponíveis para formatação.
- Os campos `daily`, `temperature_2m_max` ou `precipitation_probability_max` não aparecem no retorno.

### Causas prováveis

- URL da API incorreta.
- Parâmetros de latitude ou longitude inválidos.
- Parâmetro `daily` preenchido incorretamente.
- Falha temporária de internet.
- Indisponibilidade temporária da API.

### Como validar

Verifique se o endpoint base está correto:

```text
https://api.open-meteo.com/v1/forecast
```

Teste a chamada no navegador:

```text
https://api.open-meteo.com/v1/forecast?latitude=-8.05&longitude=-34.9&daily=temperature_2m_max,temperature_2m_min,precipitation_probability_max,windspeed_10m_max&timezone=America%2FRecife
```

Confirme se a resposta contém uma estrutura parecida com:

```text
daily
daily_units
latitude
longitude
timezone
```

### Correção

- Corrigir a URL no node `API_GET_WEATHER_FORECAST`.
- Conferir os parâmetros `latitude`, `longitude`, `daily` e `timezone`.
- Validar se os dados vindos do node `SET_CITY_COORDINATES` estão corretos.
- Executar novamente o workflow manualmente.

### Prevenção

- Não editar manualmente a URL da API sem testar.
- Manter os parâmetros separados no HTTP Request.
- Documentar qualquer alteração na chamada da API.

---

## Erro 2 — Campo climático retorna vazio ou `undefined`

### Node provável

```text
FORMAT_WEATHER_DATA
```

### Sintomas

- Campos como `temperature_max`, `temperature_min` ou `precipitation_probability` ficam vazios.
- O n8n mostra erro de expressão.
- A expressão retorna `undefined`.
- O Google Sheets recebe valores vazios.

### Causas prováveis

- A estrutura retornada pela API mudou.
- O nome do campo foi escrito incorretamente.
- O índice `[0]` está sendo usado em um array vazio.
- O node anterior não retornou dados válidos.

### Como validar

No output do node `API_GET_WEATHER_FORECAST`, confirme se existem campos como:

```text
daily.temperature_2m_max
daily.temperature_2m_min
daily.precipitation_probability_max
daily.windspeed_10m_max
```

Confira se as expressões estão seguindo este padrão:

```text
{{$json.daily.temperature_2m_max[0]}}
{{$json.daily.temperature_2m_min[0]}}
{{$json.daily.precipitation_probability_max[0]}}
{{$json.daily.windspeed_10m_max[0]}}
```

### Correção

- Corrigir o nome do campo na expressão.
- Verificar se o node anterior realmente retornou dados.
- Reexecutar apenas os nodes anteriores para validar o payload.
- Conferir se o parâmetro `daily` da API está solicitando todos os campos necessários.

### Prevenção

- Sempre verificar o output real da API antes de escrever expressões.
- Evitar copiar nomes de campos sem conferir a resposta JSON.
- Documentar os campos esperados no `setup.md`.

---

## Erro 3 — IF não segue o caminho esperado

### Node provável

```text
IF_ALERT_CONDITION
```

### Sintomas

- O workflow não envia alerta mesmo quando deveria.
- O workflow envia alerta quando não deveria.
- O caminho `true` ou `false` parece incorreto.
- A regra de temperatura ou chuva não funciona.

### Causas prováveis

- Tipo de dado incorreto, por exemplo número vindo como texto.
- Limite de alerta configurado errado.
- Campo usado na condição está vazio.
- Operador incorreto no IF.
- Condição configurada com `AND` quando deveria ser `OR`.

### Como validar

Confira os valores recebidos pelo IF:

```text
temperature_max
precipitation_probability
alert_temperature_limit
alert_rain_probability_limit
```

Exemplo esperado:

```text
temperature_max: 32
alert_temperature_limit: 30
precipitation_probability: 75
alert_rain_probability_limit: 60
```

A lógica esperada é:

```text
temperature_max > alert_temperature_limit
OU
precipitation_probability > alert_rain_probability_limit
```

### Correção

- Ajustar o operador para `maior que`.
- Conferir se as condições estão usando lógica `OR`.
- Validar se os campos comparados são numéricos.
- Conferir os limites definidos no node `SET_CITY_COORDINATES`.

### Prevenção

- Testar separadamente alerta por temperatura e alerta por chuva.
- Manter os limites documentados.
- Evitar valores fixos diretamente no IF quando possível.

---

## Erro 4 — Google Sheets não adiciona linha

### Node provável

```text
GSHEETS_APPEND_WEATHER_LOG
```

ou

```text
LOG_EXECUTION_SUCCESS
```

### Sintomas

- O node Google Sheets retorna erro.
- Nenhuma linha é criada na planilha.
- A execução falha ao tentar registrar dados.
- O n8n informa que a aba ou planilha não foi encontrada.

### Causas prováveis

- ID da planilha incorreto.
- Nome da aba incorreto.
- Cabeçalhos diferentes do esperado.
- Conta Google sem permissão de edição.
- Credencial Google expirada ou inválida.
- Operação errada no node Google Sheets.

### Como validar

Confirme se a planilha possui as abas:

```text
weather_logs
execution_logs
error_logs
```

Confirme se a aba `weather_logs` possui os cabeçalhos:

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

Confirme se a aba `execution_logs` possui os cabeçalhos:

```text
execution_id
execution_date
workflow_name
status
message
```

### Correção

- Corrigir o ID da planilha.
- Corrigir o nome da aba.
- Ajustar os cabeçalhos.
- Reautenticar a credencial Google Sheets.
- Garantir permissão de edição para a conta usada no n8n.
- Testar o node Google Sheets isoladamente.

### Prevenção

- Não alterar nomes das abas depois da configuração.
- Proteger a primeira linha da planilha.
- Usar placeholders no GitHub e valores reais apenas no n8n.

---

## Erro 5 — Gmail não envia alerta

### Node provável

```text
GMAIL_SEND_WEATHER_ALERT
```

### Sintomas

- O node Gmail retorna erro.
- O workflow falha ao enviar e-mail.
- Nenhum e-mail chega ao destinatário.
- O e-mail vai para spam.
- O alerta não é enviado mesmo com IF verdadeiro.

### Causas prováveis

- Credencial Gmail inválida ou expirada.
- Destinatário incorreto.
- Permissão OAuth insuficiente.
- Conta Google bloqueando envio.
- Condição do IF não direcionou para o node Gmail.
- Limite de envio temporário.

### Como validar

- Confirme se o IF seguiu pelo caminho `true`.
- Confira se o campo destinatário está correto.
- Teste o node Gmail isoladamente.
- Verifique se a credencial Gmail está ativa.
- Confira a caixa de spam.

### Correção

- Reautenticar a credencial Gmail.
- Ajustar o destinatário.
- Testar envio para o próprio e-mail.
- Reduzir disparos repetidos durante testes.
- Conferir se o node Gmail está conectado ao caminho `true` do IF.

### Prevenção

- Usar destinatário de teste.
- Evitar loops ou múltiplos disparos.
- Não publicar e-mails reais no GitHub.

---

## Erro 6 — Workflow global de erros não dispara

### Node provável

```text
ERROR_GLOBAL_TRIGGER
```

ou configuração do workflow principal.

### Sintomas

- O workflow principal falha.
- Nenhum registro é criado em `error_logs`.
- Nenhum e-mail de erro é enviado.
- O Error Trigger não é acionado.

### Causas prováveis

- Workflow global de erros não está ativado.
- Workflow principal não está configurado para usar o Error Workflow.
- Erro foi tratado internamente e não chegou ao Error Trigger.
- Workflow de erro também falhou.
- Credenciais do workflow de erro estão inválidas.

### Como validar

- Abra as configurações do workflow principal.
- Verifique se o Error Workflow está selecionado.
- Confirme se o workflow global está ativo.
- Force uma falha no workflow principal, por exemplo alterando temporariamente a URL da API.
- Confira se o workflow de erro executou.

### Correção

- Ativar o workflow global de erros.
- Configurar o workflow principal para usar esse Error Workflow.
- Testar o erro com uma falha simples e controlada.
- Verificar credenciais Google Sheets e Gmail no workflow de erro.
- Garantir que o workflow de erro não dependa de dados inexistentes.

### Prevenção

- Testar o Error Trigger antes de considerar o projeto finalizado.
- Manter o workflow de erro simples.
- Evitar lógica complexa no tratamento de erro.

---

## Erro 7 — Dados aparecem em colunas erradas no Google Sheets

### Node provável

```text
GSHEETS_APPEND_WEATHER_LOG
```

### Sintomas

- Dados são gravados, mas em colunas incorretas.
- Campos aparecem deslocados.
- Algumas colunas ficam vazias.
- O valor de uma coluna aparece em outra.

### Causas prováveis

- Cabeçalhos da planilha alterados.
- Ordem de colunas diferente da esperada.
- Mapeamento incorreto no node Google Sheets.
- Campo do JSON com nome diferente do cabeçalho.

### Como validar

Compare os nomes dos campos do output do n8n com os cabeçalhos da planilha.

Exemplo esperado:

```text
temperature_max → temperature_max
precipitation_probability → precipitation_probability
alert_required → alert_required
```

### Correção

- Ajustar o mapeamento no node Google Sheets.
- Corrigir os cabeçalhos da planilha.
- Padronizar nomes dos campos no node `FORMAT_WEATHER_DATA`.
- Testar o append com uma única execução.

### Prevenção

- Não renomear cabeçalhos manualmente.
- Manter padrão snake_case.
- Documentar qualquer nova coluna.

---

## Erro 8 — Execução duplicada

### Node provável

```text
SCHEDULE_DAILY_WEATHER_CHECK
```

ou configuração do workflow.

### Sintomas

- Mais de uma linha é criada para o mesmo dia.
- O Gmail é enviado mais de uma vez.
- Existem execuções duplicadas no histórico do n8n.

### Causas prováveis

- Workflow executado manualmente e também por agendamento.
- Mais de um workflow ativo com a mesma função.
- Schedule configurado com frequência maior que a esperada.
- Retry manual feito durante testes.

### Como validar

- Verifique o histórico de execuções no n8n.
- Confira se há mais de um workflow parecido ativo.
- Confira a configuração do Schedule Trigger.
- Compare os horários das linhas no Google Sheets.

### Correção

- Desativar workflows duplicados.
- Ajustar o Schedule Trigger para execução diária.
- Durante testes, deixar o workflow desativado e executar manualmente.
- Registrar um `execution_id` para rastrear cada execução.

### Prevenção

- Ativar o agendamento apenas depois dos testes.
- Usar nomes claros para workflows.
- Documentar o horário de execução esperado.

---

## Erro 9 — Erro de timezone

### Node provável

```text
SET_CITY_COORDINATES
```

ou

```text
API_GET_WEATHER_FORECAST
```

### Sintomas

- Data da previsão parece incorreta.
- Registro fica com horário diferente do esperado.
- Execução aparece em outro dia.
- Dados climáticos não correspondem ao período esperado.

### Causas prováveis

- Timezone configurado incorretamente.
- Timezone não enviado para a API.
- n8n usando timezone diferente do esperado.
- Schedule Trigger configurado em outro fuso.

### Como validar

Confirme se o timezone usado é:

```text
America/Recife
```

Valide também o Schedule Trigger:

```text
Frequência: diária
Horário: 08:00
Timezone: America/Recife
```

### Correção

- Corrigir o campo `timezone` no node de configuração.
- Conferir o timezone do Schedule Trigger.
- Validar o timezone enviado à API.
- Conferir a data registrada no Google Sheets.

### Prevenção

- Usar timezone explícito.
- Evitar depender do timezone padrão do ambiente.
- Documentar o timezone no setup.

---

## Erro 10 — JSON exportado contém dados sensíveis

### Arquivo provável

```text
workflows/main-weather-monitor.json
```

ou

```text
workflows/global-error-handler.json
```

### Sintomas

- O JSON exportado contém ID real de planilha.
- O JSON contém e-mail pessoal.
- O JSON contém credenciais ou tokens.
- O arquivo contém dados que não deveriam ser públicos.

### Causas prováveis

- Exportação direta do n8n sem revisão.
- Valores reais configurados diretamente nos nodes.
- Prints ou exemplos com dados reais.
- Falta de placeholders.

### Como validar

Antes do commit, procure no VS Code por termos como:

```text
@gmail.com
spreadsheet
token
credential
client_secret
access_token
refresh_token
```

Também revise os arquivos:

```text
README.md
docs/
workflows/
assets/
```

### Correção

- Substituir valores sensíveis por placeholders.
- Remover tokens ou credenciais.
- Refazer prints ocultando dados sensíveis.
- Garantir que `.env` e banco local do n8n estejam no `.gitignore`.

### Prevenção

- Revisar arquivos antes de cada commit.
- Usar placeholders no repositório público.
- Nunca versionar `.env`, banco local do n8n ou credenciais.

---

## Checklist rápido de diagnóstico

Quando algo falhar, verificar:

- [ ] O n8n está em execução?
- [ ] O workflow correto foi executado?
- [ ] O node com erro foi identificado?
- [ ] A API Open-Meteo respondeu corretamente?
- [ ] Os campos esperados existem no JSON?
- [ ] As expressões estão apontando para os campos corretos?
- [ ] O IF está usando a lógica correta?
- [ ] A planilha possui as abas corretas?
- [ ] Os cabeçalhos estão corretos?
- [ ] As credenciais Google estão válidas?
- [ ] O Gmail está conectado ao caminho correto?
- [ ] O Error Workflow está configurado e ativo?
- [ ] Não há dados sensíveis no repositório?

## Boas práticas de troubleshooting

- Testar um node por vez.
- Conferir sempre o output real do node anterior.
- Usar nomes claros nos nodes.
- Registrar erros na aba `error_logs`.
- Evitar alterar várias coisas ao mesmo tempo.
- Após corrigir, executar novamente manualmente.
- Documentar erros recorrentes neste arquivo.

## Registro de aprendizados

Durante o desenvolvimento, novos erros podem ser adicionados a este documento.

Formato recomendado:

```text
Erro:
Node provável:
Sintoma:
Causa raiz:
Correção aplicada:
Como prevenir:
```