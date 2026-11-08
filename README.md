# security-pipelines

GitHub Actions reutilizáveis de segurança. Este repo guarda só os workflows: quem roda é o repo que chama, então log e relatório ficam com a visibilidade dele.

| Workflow | O que faz | Ferramentas |
|---|---|---|
| `scan-codigo.yml` | segredos no histórico, código, dependências e imagens Docker | gitleaks, semgrep, pip-audit, npm audit, trivy |
| `pentest-web.yml` | pentest dinâmico com ou sem login | ZAP, nuclei |
| `corrige-dependencias.yml` | sobe as libs com CVE e abre PR | uv, npm audit fix, gh |

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
      imagens: |
        . docker/Dockerfile.api
        web web/Dockerfile
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

Para corrigir sozinho quando o scan achar CVE no main:

```yaml
  corrige:
    needs: scan
    if: failure() && github.ref == 'refs/heads/main' && github.event_name != 'pull_request'
    permissions:
      contents: write
      pull-requests: write
      actions: write
    uses: LastHasagi/security-pipelines/.github/workflows/corrige-dependencias.yml@<sha>
    with:
      dir_node: web
      workflows_ci: ci.yml seguranca.yml
```

Ligue em Settings > Actions "Allow GitHub Actions to create and approve pull requests", e ponha `workflow_dispatch` nos workflows de `workflows_ci`: PR aberto pelo `GITHUB_TOKEN` não dispara CI sozinho. Só versão compatível sobe; o que pedir major fica no aviso do scan.

O plano do ZAP é do repo que chama, porque as rotas de cada sistema são diferentes. Ele lê a URL em `${ALVO}` e precisa gerar `relatorios/zap.json`.

`bloquear: false` levanta o passivo sem travar o deploy. Depois de limpar os achados, troque para `true`.

## Antes de rodar o pentest

- Só contra sistema seu ou com autorização por escrito.
- Chame a partir de **repo privado**. Em repo público, log e artefato ficam abertos, e isso publica as falhas do alvo.
- Use uma conta de teste sem acesso a dado real.
- Tire do plano toda rota que acione terceiros, gaste cota paga ou derrube a sessão.
