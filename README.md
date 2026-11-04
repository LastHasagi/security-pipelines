# security-pipelines

GitHub Actions reutilizáveis de segurança. Este repo guarda só os workflows: quem roda é o repo que chama, então log e relatório ficam com a visibilidade dele.

| Workflow | O que faz | Ferramentas |
|---|---|---|
| `scan-codigo.yml` | segredos no histórico, código e dependências | gitleaks, semgrep, pip-audit, npm audit |
| `pentest-web.yml` | pentest dinâmico com ou sem login | ZAP, nuclei |

## Uso

Fixe sempre um commit, nunca `@main`.

```yaml
# .github/workflows/seguranca.yml
on: [push]
permissions:
  contents: read
jobs:
  scan:
    uses: LastHasagi/security-pipelines/.github/workflows/scan-codigo.yml@<sha>
    with:
      dir_node: web
      bloquear: false
```

```yaml
# .github/workflows/pentest.yml
on: workflow_dispatch
permissions:
  contents: read
jobs:
  pentest:
    uses: LastHasagi/security-pipelines/.github/workflows/pentest-web.yml@<sha>
    with:
      alvo: https://exemplo.com
      plano_zap: .github/pentest/zap.yaml
      login_caminho: /api/login
      sair_caminho: /api/logout
      cookie_sessao: sessao
    secrets:
      login_json: ${{ secrets.PENTEST_LOGIN_JSON }}
```

O plano do ZAP é do repo que chama, porque as rotas de cada sistema são diferentes. Ele lê a URL em `${ALVO}` e precisa gerar `relatorios/zap.json`.

`bloquear: false` levanta o passivo sem travar o deploy. Depois de limpar os achados, troque para `true`.

## Antes de rodar o pentest

- Só contra sistema seu ou com autorização por escrito.
- Chame a partir de **repo privado**. Em repo público, log e artefato ficam abertos, e isso publica as falhas do alvo.
- Use uma conta de teste sem acesso a dado real.
- Tire do plano toda rota que acione terceiros, gaste cota paga ou derrube a sessão.
