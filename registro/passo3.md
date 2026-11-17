# Passo 3 — Teste de execução dos serviços

Data: 17/11/2026

## Subida

```
docker compose up -d --build
docker compose ps
```

| Container | Status | Porta |
|---|---|---|
| carnivorous-garden | Up | 5005 |
| grafana_lab-alloy-1 | Up | 12345 |
| grafana_lab-loki-1 | Up | 3100 |
| grafana_lab-grafana-1 | Up | 3000 |

> Obs.: se a porta 3000 já estiver em uso por outro container, pare-o antes (`docker stop <container>`).

## Verificações

- App: http://localhost:5005 → HTTP 200
- Grafana: http://localhost:3000 → saudável (v13.3.1), acesso anônimo como Admin
- Datasource Loki (provisionado pelo entrypoint): Connections > Data sources > Loki > Save & test →
  "Data source successfully connected."

## Tráfego gerado na app

1. Cadastro do usuário `aluno` / `senha123` (e cadastro repetido → 409, log ERROR)
2. Login certo → redirect para o dashboard
3. Login errado (senha errada e usuário inexistente) → 401, log WARNING
4. Modo de falhas ligado (`/toggle_error_mode`) + 10 logins e 6 logouts → ~50% retornando 500, log ERROR

## Resultado no Loki (últimos 15 min)

Consulta: `sum by (level) (count_over_time({level=~".+"}[15m]))`

| level | quantidade |
|---|---|
| ERROR | 17 |
| WARNING | 6 |
| INFO | 186 |

Exemplo de linha de ERROR:

```
time="2026-11-17 19:22:44,990" logger="__main__" level="ERROR" message="user="unknown" action="logout" status="error" error="Failed to loggout.""
```

O label `level` é extraído pelo Alloy (`stage.logfmt` + `stage.labels`), então dá para filtrar por nível no Grafana.

## Sugestões de consultas para o passo 4

- `{level="ERROR"}`
- `{level="WARNING"} |= "invalid credentials"`
- `sum by (level) (count_over_time({level=~".+"}[5m]))`
