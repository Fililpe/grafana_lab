# Passo 4 — Consultas LogQL no Grafana Explore

Data: 17/11/2026

Com os serviços em execução e o tráfego de testes (incluindo o modo de falhas) gerado no Passo 3, foram executadas 4 consultas no **Grafana Explore** (`http://localhost:3000/explore`) utilizando a fonte de dados **Loki**.

---

## Consulta 1 — Todos os logs coletados do arquivo da aplicação

```logql
{filename="/tmp/app-logs/app.log"}
```

- **Objetivo:** Verificar a ingestão completa dos registros gerados pelo Flask (`logs/app.log`) e coletados pelo Grafana Alloy no volume montado em `/tmp/app-logs/app.log`.
- **Resultado observado:** Retorno de todas as 209 linhas de log registradas no período (`INFO`, `WARNING` e `ERROR`), incluindo requisições HTTP (`@app.after_request`) e eventos de negócio (cadastro, login, logout e ativação do modo de falhas).

Exemplo de registros retornados:

```text
time="2026-11-17 19:21:10,412" logger="__main__" level="INFO" message="user="aluno" action="create" status="success""
time="2026-11-17 19:21:10,415" logger="__main__" level="INFO" message="path="/signup" method="POST" status="302""
time="2026-11-17 19:22:05,104" logger="__main__" level="INFO" message="Error mode toggled to on."
```

---

## Consulta 2 — Filtragem exclusiva de erros (`level="ERROR"`)

```logql
{level="ERROR"}
```

- **Objetivo:** Consultar apenas os eventos críticos da aplicação a partir do label `level`, extraído dinamicamente pelo Grafana Alloy (`stage.logfmt` + `stage.labels`).
- **Resultado observado:** Retorno de **17 registros** de erro, contemplando:
  - Tentativa de cadastro com nome de usuário já existente (`status="failed" error="username exists"`, HTTP 409).
  - Falhas aleatórias de login e logout provocadas pelo modo de falhas (`error="random failure due to error mode"` e `error="Failed to loggout."`, HTTP 500).

Exemplo de registros retornados:

```text
time="2026-11-17 19:21:32,118" logger="__main__" level="ERROR" message="user="aluno" action="signup" status="failed" error="username exists""
time="2026-11-17 19:22:18,642" logger="__main__" level="ERROR" message="user="aluno" action="login" status="error" error="random failure due to error mode""
time="2026-11-17 19:22:44,990" logger="__main__" level="ERROR" message="user="unknown" action="logout" status="error" error="Failed to loggout.""
```

---

## Consulta 3 — Auditoria de falhas de autenticação (`WARNING` + filtro textual)

```logql
{level="WARNING"} |= "invalid credentials"
```

- **Objetivo:** Combinar seletor de stream (`{level="WARNING"}`) com expressão de filtro de linha (`|= "invalid credentials"`) para identificar tentativas de login malsucedidas (senha incorreta ou usuário inexistente, HTTP 401).
- **Resultado observado:** Retorno dos **6 registros** de alerta gerados durante os testes de autenticação inválida.

Exemplo de registros retornados:

```text
time="2026-11-17 19:21:48,309" logger="__main__" level="WARNING" message="user="aluno" action="login" status="failed" error="invalid credentials""
time="2026-11-17 19:21:55,871" logger="__main__" level="WARNING" message="user="inexistente" action="login" status="failed" error="invalid credentials""
```

---

## Consulta 4 — Contagem agregada de logs por nível de severidade (Métrica LogQL)

```logql
sum by (level) (count_over_time({level=~".+"}[15m]))
```

*(Também testada com janela móvel de 5 minutos: `sum by (level) (count_over_time({level=~".+"}[5m]))`)*

- **Objetivo:** Utilizar uma consulta métrica no Loki para agregar o volume total de eventos por severidade (`level`) ao longo do tempo e visualizar a proporção entre tráfego normal e falhas.
- **Resultado observado:**

| Label `level` | Total acumulado (15m) | Descrição |
|---|---|---|
| `INFO` | 186 | Requisições HTTP normais, cadastros, logins/logouts com sucesso e toggle do modo de erro |
| `ERROR` | 17 | Cadastro duplicado (409) e erros simulados de login/logout (500) no modo de falhas |
| `WARNING` | 6 | Tentativas de login recusadas por credenciais inválidas (401) |
