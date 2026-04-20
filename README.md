# deploy-monitoring — ToggleMaster Fase 4

Stack de observabilidade completa (Prometheus + Grafana + Loki + OpenTelemetry + New Relic + PagerDuty + Discord + Self-Healing) deployada via **GitOps** com ArgoCD no padrão **App-of-Apps**.

## Princípio

> **"Se não está no código, não existe."**
>
> Todo o deploy é feito por **um único `kubectl apply`**. Nenhum clique manual na UI do ArgoCD. Qualquer mudança é feita via `git push` e o ArgoCD sincroniza sozinho.

---

## Como fazer o deploy (fluxo completo)

### 1) Pré-requisitos (uma vez)

```bash
# 1.1 Bucket S3 para o Loki (AWS Academy não permite PVC com EBS)
aws s3 mb s3://togglemaster-loki-$(aws sts get-caller-identity --query Account --output text) \
  --region us-east-1

# 1.2 Conferir se o ArgoCD está rodando
kubectl get pods -n argocd
```

> Se sua conta AWS tiver um ID diferente de `637423306132`, edite a linha `s3: s3://us-east-1/togglemaster-loki-637423306132` em `argocd-apps/02-loki-stack.yaml`.

### 2) Deploy de toda a stack de observabilidade

```bash
kubectl apply -f monitoring-app-of-apps.yaml
```

**É só isso.** O ArgoCD vai:

1. Criar a Application `monitoring-app-of-apps` (pai)
2. Descobrir os 4 YAMLs em `argocd-apps/` e criar automaticamente:
   - `kube-prometheus-stack` → Prometheus + Grafana + Alertmanager
   - `loki-stack` → Loki + Promtail
   - `otel-collector` → OpenTelemetry Collector
   - `monitoring-manifests` → dashboards, alertas, self-healing, RBAC
3. Sincronizar tudo em ~5 minutos

### 3) Acompanhar

```bash
# Ver todas as Applications
kubectl get application -n argocd

# Ver todos os pods do monitoring
kubectl get pods -n monitoring -w
```

Todas devem ficar `Synced` + `Healthy`.

---

## Estrutura do repositório

```
deploy-monitoring-gitops/
├── monitoring-app-of-apps.yaml          # ÚNICO kubectl apply (cria tudo)
│
├── argocd-apps/                         # Applications filhas (ArgoCD descobre sozinho)
│   ├── 01-kube-prometheus-stack.yaml    # Helm chart (v83.4.2)
│   ├── 02-loki-stack.yaml               # Helm chart (v2.10.2) + S3
│   ├── 03-otel-collector.yaml           # Helm chart (v0.146.1) + contrib
│   └── 04-monitoring-manifests.yaml     # Aponta para manifests/
│
└── manifests/                           # Manifests customizados (aplicados pela 04)
    ├── namespace/
    │   └── namespace.yaml
    ├── prometheus/
    │   └── service-monitors.yaml        # ServiceMonitors dos 5 microsserviços
    ├── grafana/
    │   └── dashboard-configmap.yaml     # Dashboard ToggleMaster
    ├── alerting/
    │   ├── prometheus-rules.yaml        # 4 alert rules
    │   └── alertmanager-config.yaml     # PagerDuty + Discord + self-healing
    └── self-healing/
        ├── rbac.yaml                    # ServiceAccount + ClusterRole
        └── webhook-receiver.yaml        # Pod que executa kubectl rollout restart
```

---

## Mudanças em relação ao guia original (PDF)

| Item | Guia original | Aplicado aqui | Motivo |
|---|---|---|---|
| `kube-prometheus-stack` versão | 65.1.0 | **83.4.2** | Mais recente estável |
| `kube-prometheus-stack` sync | client-side | **ServerSideApply=true** | CRDs > 262 KB não cabem em anotação |
| `prometheusOperator.admissionWebhooks` | enabled | **disabled** | TLS bad certificate em lab |
| `opentelemetry-collector` versão | 0.97.1 | **0.146.1** | Mais recente |
| `opentelemetry-collector` imagem | default (k8s) | **contrib** | `k8s` não tem `prometheusremotewrite` |
| Exporter de logs no OTel | `loki` | **`otlphttp/loki`** | Exporter `loki` foi removido do contrib |
| Self-healing image | `bitnami/kubectl:1.29` | **`alpine/k8s:1.29.2`** | bitnami descontinuou a tag 1.29 |
| Self-healing `initialDelaySeconds` | 5 | **60** | Pod precisa de tempo para subir |
| Incident management | OpsGenie | **PagerDuty** | OpsGenie descontinuado pela Atlassian |
| Loki storage | emptyDir | **S3** | Recomendação do professor |

---

## Como fazer mudanças

Qualquer alteração em qualquer arquivo do repo é GitOps-friendly:

```bash
# Exemplo: adicionar uma nova regra de alerta
vim manifests/alerting/prometheus-rules.yaml
git add .
git commit -m "feat: adiciona alerta HighPodRestarts"
git push origin main
```

O ArgoCD detecta a mudança em ~1 minuto e aplica automaticamente. Nada de `kubectl apply` manual.

---

## Troubleshooting rápido

| Sintoma | Causa provável | Solução |
|---|---|---|
| Application `OutOfSync` | Placeholder não substituído no YAML | Edite, `git push`, ArgoCD sincroniza |
| `metadata.annotations: Too long` | kube-prometheus-stack sem Server-Side Apply | Já está em `syncOptions` |
| StatefulSet do Prometheus não aparece | Webhook TLS do operator | Já desabilitado no values |
| OTel pod em CrashLoopBackOff | Imagem errada ou exporter inexistente | Já usa `contrib` + `otlphttp/loki` |
| Self-healing em CrashLoop | Imagem sem python ou sem kubectl | Já usa `alpine/k8s:1.29.2` |
| Grafana `MountVolume failed grafana-dashboards` | Ordem de criação (ConfigMap vem da Application 4) | Aguarde 1-2 min, self-heal resolve |
| Credenciais AWS Academy expiraram | Sessão ~4h | `aws eks update-kubeconfig --name togglemaster-cluster --region us-east-1` |

---

## Recriar do zero (reset total)

```bash
# Deleta a Application pai (e por causa do finalizer, todas as filhas caem)
kubectl delete application monitoring-app-of-apps -n argocd

# Aguarda 1 min e recria
kubectl apply -f monitoring-app-of-apps.yaml
```
