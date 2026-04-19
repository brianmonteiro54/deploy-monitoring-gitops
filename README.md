# deploy-monitoring — ToggleMaster Fase 4

## Como funciona

Este repositorio segue o mesmo padrao dos outros repos de deploy (deploy-auth-service, deploy-flag-service, etc.).


## Estrutura

```
deploy-monitoring/
├── argocd-apps/           # 3 Applications que apontam para Helm charts
│   ├── 01-kube-prometheus-stack.yaml
│   ├── 02-loki-stack.yaml
│   └── 03-otel-collector.yaml
└── manifests/             # Manifests customizados (1 Application aponta aqui)
    ├── namespace/
    ├── prometheus/
    ├── grafana/
    ├── alerting/
    └── self-healing/
```
