# 📊 deploy-monitoring — ToggleMaster Observability Stack

> **Fase 4 do projeto ToggleMaster** — Stack completa de observabilidade (Metrics + Logs + Traces + Alerting + Self-Healing) deployada via **GitOps** com ArgoCD no padrão **App-of-Apps**.

[![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-EF7B4D?logo=argo&logoColor=white)](https://argoproj.github.io/argo-cd/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-EKS-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Prometheus](https://img.shields.io/badge/Prometheus-83.4.2-E6522C?logo=prometheus&logoColor=white)](https://prometheus.io/)
[![Grafana](https://img.shields.io/badge/Grafana-Dashboards-F46800?logo=grafana&logoColor=white)](https://grafana.com/)
[![Loki](https://img.shields.io/badge/Loki-2.9.10-F46800?logo=grafana&logoColor=white)](https://grafana.com/oss/loki/)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-0.146.1-425CC7?logo=opentelemetry&logoColor=white)](https://opentelemetry.io/)
[![New Relic](https://img.shields.io/badge/New_Relic-APM-008C99?logo=newrelic&logoColor=white)](https://newrelic.com/)

---

## 🎯 Princípio

> **"Se não está no código, não existe."**
>
> Todo o deploy é feito por **um único `kubectl apply`**. Nenhum clique manual na UI do ArgoCD. Qualquer mudança é feita via `git push` e o ArgoCD sincroniza sozinho.

---

## 🏗️ Arquitetura

```
                        ┌───────────────────────────────────────────────┐
                        │          ToggleMaster Microservices           │
                        │  auth │ flag │ targeting │ evaluation │ analytics
                        └─────────────────────┬─────────────────────────┘
                                              │ OTLP (4317/4318)
                                              ▼
                         ┌──────────────────────────────────────┐
                         │     OpenTelemetry Collector          │
                         │     (receivers / processors)         │
                         └──┬──────────────┬──────────────┬─────┘
                            │ metrics      │ logs         │ traces
                            ▼              ▼              ▼
              ┌─────────────────┐  ┌──────────────┐  ┌──────────────┐
              │   Prometheus    │  │     Loki     │  │  New Relic   │
              │   (interno)     │  │   (S3 bkt)   │  │    (SaaS)    │
              └────────┬────────┘  └──────┬───────┘  └──────────────┘
                       │                  │
                       └──────┬───────────┘
                              ▼
                       ┌────────────┐
                       │  Grafana   │  ←  Datasources (Prometheus + Loki)
                       └────────────┘

                       ┌────────────────┐    ┌─────────────────┐
                       │ Alertmanager   │───▶│   PagerDuty     │  (incidentes críticos)
                       │  (rules)       │───▶│    Discord      │  (notificações)
                       │                │───▶│ Self-Healing    │  (rollout restart)
                       └────────────────┘    └─────────────────┘
```

### Pipelines do OTel Collector

| Sinal | Receiver | Backend interno | Backend externo |
|---|---|---|---|
| **Metrics** | OTLP | Prometheus (`prometheusremotewrite`) | New Relic |
| **Logs** | OTLP | Loki (`otlphttp/loki`) | New Relic |
| **Traces** | OTLP | — | New Relic *(único backend)* |

---

## 📦 Stack

| Componente | Versão | Função |
|---|---|---|
| **kube-prometheus-stack** | 83.4.2 | Prometheus + Grafana + Alertmanager + node-exporter + kube-state-metrics |
| **loki-stack** | 2.10.3 | Loki (storage S3) + Promtail (DaemonSet) |
| **opentelemetry-collector** | 0.146.1 (`contrib`) | Hub central de observabilidade |
| **External Secrets Operator** | — | Sincroniza secrets do AWS Secrets Manager |
| **Self-Healing Webhook** | custom (Python) | Recebe webhook do Alertmanager e roda `kubectl rollout restart` |
| **PagerDuty** | — | Incident management |
| **Discord** | — | Notificações de alertas e self-healing |
| **New Relic** | — | APM externo (traces + correlação de sinais) |

---

## 📁 Estrutura do repositório

```
deploy-monitoring-gitops/
│
├── monitoring-app-of-apps.yaml          # ★ ÚNICO kubectl apply (cria tudo)
│
├── apps/                                # Apps do togglemaster-root
│   ├── 00-monitoring.yaml               #   sync-wave -10 (sobe primeiro)
│   ├── 10-auth-service.yaml             #   sync-wave 10 (microsserviços)
│   ├── 10-flag-service.yaml
│   ├── 10-targeting-service.yaml
│   ├── 10-evaluation-service.yaml
│   └── 10-analytics-service.yaml
│
├── argocd-apps/                         # 4 Applications filhas do monitoring
│   ├── 01-kube-prometheus-stack.yaml    # Helm chart upstream
│   ├── 02-loki-stack.yaml               # Helm chart + S3 backend
│   ├── 03-otel-collector.yaml           # Helm chart contrib + 3 pipelines
│   └── 04-monitoring-manifests.yaml     # Aponta para manifests/
│
└── manifests/                           # Manifests customizados
    ├── prometheus/
    │   └── service-monitors.yaml        # ServiceMonitors dos 5 microsserviços
    ├── grafana/
    │   └── dashboard-configmap.yaml     # Dashboard ToggleMaster
    ├── alerting/
    │   ├── prometheus-rules.yaml        # 4 alert rules
    │   └── alertmanager-config.yaml     # Roteamento PagerDuty + Discord + healing
    ├── self-healing/
    │   ├── rbac.yaml                    # ServiceAccount + ClusterRole
    │   └── webhook-receiver.yaml        # Pod Python com kubectl
    └── external-secrets/
        ├── secretstore.yaml             # AWS Secrets Manager
        └── externalsecrets.yaml         # Renderiza secrets do cluster
```

---

## 🚀 Deploy

### Pré-requisitos (uma vez)

```bash
# 1) Bucket S3 para o Loki
aws s3 mb s3://togglemaster-loki-$(aws sts get-caller-identity --query Account --output text) \
  --region us-east-1

# 2) Conferir se o ArgoCD está rodando
kubectl get pods -n argocd

# 3) Conferir se o External Secrets Operator está instalado
kubectl get pods -n external-secrets
```

> ⚠️ Se sua conta AWS tiver um ID diferente de `789754462323`, edite a linha `s3: s3://us-east-1/togglemaster-loki-789754462323` em `argocd-apps/02-loki-stack.yaml`.

### Secrets necessários no AWS Secrets Manager

Crie o segredo `togglemaster/monitoring` com as seguintes chaves:

```json
{
  "DISCORD_WEBHOOK_URL": "https://discord.com/api/webhooks/...",
  "PAGERDUTY_SERVICE_KEY": "...",
  "GRAFANA_ADMIN_USER": "admin",
  "GRAFANA_ADMIN_PASSWORD": "...",
  "NEW_RELIC_API_KEY": "..."
}
```

### Deploy completo

```bash
kubectl apply -f monitoring-app-of-apps.yaml
```

**É só isso.** O ArgoCD vai:

1. Criar a Application `togglemaster-root` (raiz)
2. Descobrir os YAMLs em `apps/` e criar a Application `monitoring-app-of-apps` (sync-wave `-10`)
3. Esta, por sua vez, descobre os 4 YAMLs em `argocd-apps/` e cria:
   - `kube-prometheus-stack` → Prometheus + Grafana + Alertmanager
   - `loki-stack` → Loki + Promtail
   - `otel-collector` → OpenTelemetry Collector
   - `monitoring-manifests` → dashboards, alertas, self-healing, RBAC
4. Só depois sobem os microsserviços (sync-wave `10`)
5. Sincroniza tudo em ~5 minutos

### Acompanhar o sync

```bash
# Todas as Applications
kubectl get application -n argocd

# Pods de monitoring
kubectl get pods -n monitoring -w
```

Todas devem ficar `Synced` + `Healthy`.

---

## 🌐 URLs de acesso

Os componentes são expostos via Ingress NGINX no host `toggle.pt`:

| Serviço | URL | Credenciais |
|---|---|---|
| Grafana | `http://toggle.pt/grafana` | `admin-user` / `admin-password` (Secret `grafana-admin-credentials`) |
| Prometheus | `http://toggle.pt/prometheus` | — |
| Alertmanager | `http://toggle.pt/alertmanager` | — |

> Se estiver rodando local, adicione no `/etc/hosts`: `<EKS-LB-IP>  toggle.pt`

---

## 🔔 Alertas configurados

| Alerta | Threshold | Severidade | Self-Healing |
|---|---|---|---|
| `HighErrorRate5xx` | 5xx > 5% por 2 min | critical | ✅ rollout restart |
| `PodCrashLooping` | restarts > 3 em 15 min | warning | — |
| `HighLatencyP95` | P95 > 2s por 3 min | warning | — |
| `HighMemoryUsage` | mem > 85% do limit | warning | — |

### Fluxo de alerta

```
Prometheus  ──(rule fires)──▶  Alertmanager
                                    │
                  ┌─────────────────┼─────────────────┐
                  ▼                 ▼                 ▼
              PagerDuty         Discord       Self-Healing webhook
            (severity=          (notif)         (kubectl rollout
             critical)                           restart deployment)
```

---

## 🔄 Como fazer mudanças (GitOps workflow)

```bash
# Exemplo: adicionar uma nova regra de alerta
vim manifests/alerting/prometheus-rules.yaml
git add .
git commit -m "feat: adiciona alerta HighPodRestarts"
git push origin main
```

O ArgoCD detecta a mudança em ~1 minuto e aplica automaticamente. **Nada de `kubectl apply` manual.**

---


## ♻️ Reset total

```bash
# Deleta a Application pai (e por causa do finalizer, todas as filhas caem)
kubectl delete application monitoring-app-of-apps -n argocd

# Aguarda 1 min e recria
kubectl apply -f monitoring-app-of-apps.yaml
```

---

## 📚 Referências

- [ArgoCD App-of-Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [kube-prometheus-stack Helm Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Loki S3 storage config](https://grafana.com/docs/loki/latest/configure/storage/)
- [OpenTelemetry Collector — exporters](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter)
- [External Secrets Operator](https://external-secrets.io/)

---

## 📄 Licença

Projeto acadêmico — ToggleMaster Fase 4.