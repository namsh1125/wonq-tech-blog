---
slug: aws-eks-instance-spec-decision
title: AWS EKS 인스턴스 스펙 선택기
authors: polar
tags: [ AWS, EKS, Kubernetes, GCP, migration, instance-sizing, cost-optimization ]
---

안녕하세요, 원큐 오더 PL 남승현입니다.

이전 [인프라 기술 스택 선택기](../2025-05-23/infrastructure-stack-selection.md) 포스트에서 Kubernetes와 Linkerd를 도입한 이야기를 나눴는데요,
오늘은 GCP에서 POC를 진행한 후, AWS EKS로 마이그레이션하면서 **EKS 인스턴스 스펙을 결정**한 경험을 공유하려고 해요.

## 1. 현재 GCP 리소스 현황 파악

비용 효율적으로 AWS 리소스를 활용하기 위해, GCP에서 현재 운영 중인 서비스들의 리소스 사용량을 측정하고 분석했어요.

### 1-1. Kubernetes 클러스터 상태 측정

#### 전체 네임스페이스의 Pod 상태

```text
❯ kubectl get pods --all-namespaces -o wide

NAMESPACE                    NAME                                                     READY   STATUS    RESTARTS      AGE   IP            NODE                                          NOMINATED NODE   READINESS GATES
app-card-server              app-card-server-85f6f567b4-2dpfx                         2/2     Running   0             11h   10.64.1.4     gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
app-card-server              app-card-server-85f6f567b4-r29ck                         2/2     Running   0             11h   10.64.1.3     gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
app-card-server              app-card-server-85f6f567b4-t5kpf                         2/2     Running   0             11h   10.64.0.21    gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
argocd                       argocd-application-controller-0                          1/1     Running   0             11h   10.64.0.16    gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
argocd                       argocd-applicationset-controller-777d5b5dc7-vm6vc        1/1     Running   0             11h   10.64.0.4     gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
argocd                       argocd-dex-server-7d8fcd845-7gjwm                        1/1     Running   0             11h   10.64.4.4     gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
argocd                       argocd-notifications-controller-655df7c996-x9nzl         1/1     Running   0             11h   10.64.0.3     gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
argocd                       argocd-redis-574484f6db-g9zbd                            1/1     Running   0             11h   10.64.4.5     gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
argocd                       argocd-repo-server-57449f957c-swfwz                      1/1     Running   0             11h   10.64.0.5     gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
argocd                       argocd-server-7dd4c8cf5f-lgs4g                           1/1     Running   0             11h   10.64.4.6     gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
bank-server                  bank-server-75d568f9fb-66pct                             2/2     Running   0             11h   10.64.1.6     gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
bank-server                  bank-server-75d568f9fb-6wdtg                             2/2     Running   0             11h   10.64.4.7     gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
bank-server                  bank-server-75d568f9fb-mn2rp                             2/2     Running   0             11h   10.64.1.5     gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
card-server                  card-server-595c89869f-29kkl                             2/2     Running   0             11h   10.64.4.8     gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
card-server                  card-server-595c89869f-gb6tk                             2/2     Running   0             11h   10.64.1.8     gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
card-server                  card-server-595c89869f-xgxfd                             2/2     Running   0             11h   10.64.0.6     gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
cert-manager                 cert-manager-7d67448f59-r5jb9                            1/1     Running   0             11h   10.64.4.9     gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
cert-manager                 cert-manager-cainjector-666b8b6b66-sqtld                 1/1     Running   0             11h   10.64.0.7     gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
cert-manager                 cert-manager-webhook-78cb4cf989-cwdmv                    1/1     Running   0             11h   10.64.0.8     gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
gke-managed-cim              kube-state-metrics-0                                     2/2     Running   0             11h   10.64.4.15    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
gmp-system                   collector-4wm8b                                          2/2     Running   0             11h   10.64.4.3     gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
gmp-system                   collector-5v2hl                                          2/2     Running   0             11h   10.64.0.2     gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
gmp-system                   collector-c86tg                                          2/2     Running   0             11h   10.64.1.2     gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
gmp-system                   gmp-operator-658db64bf5-lgs44                            1/1     Running   0             11h   10.64.1.7     gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
ingress-nginx                ingress-nginx-controller-7989649c5c-tmb9j                1/1     Running   0             11h   10.64.1.9     gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
kube-system                  event-exporter-gke-bb5b454cd-dhzjz                       2/2     Running   0             11h   10.64.1.10    gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
kube-system                  fluentbit-gke-555st                                      3/3     Running   0             11h   10.178.0.13   gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
kube-system                  fluentbit-gke-gs6nf                                      3/3     Running   0             11h   10.178.0.12   gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
kube-system                  fluentbit-gke-ncbbn                                      3/3     Running   0             11h   10.178.0.14   gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
kube-system                  gke-metrics-agent-85txc                                  2/2     Running   0             11h   10.178.0.14   gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
kube-system                  gke-metrics-agent-gqxjh                                  2/2     Running   0             11h   10.178.0.12   gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
kube-system                  gke-metrics-agent-mzrsk                                  2/2     Running   0             11h   10.178.0.13   gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
kube-system                  konnectivity-agent-7fc6dbc6b-2vwmc                       2/2     Running   0             11h   10.64.1.11    gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
kube-system                  konnectivity-agent-7fc6dbc6b-cdqrt                       2/2     Running   0             11h   10.64.4.2     gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
kube-system                  konnectivity-agent-7fc6dbc6b-lk8c9                       2/2     Running   0             11h   10.64.0.9     gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
kube-system                  konnectivity-agent-autoscaler-5745d65c48-2bjd6           1/1     Running   0             11h   10.64.4.11    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
kube-system                  kube-dns-667895bc57-2ngtt                                4/4     Running   0             11h   10.64.4.13    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
kube-system                  kube-dns-667895bc57-6nv7x                                4/4     Running   0             11h   10.64.0.10    gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
kube-system                  kube-dns-autoscaler-576f8b96b7-bv5pt                     1/1     Running   0             11h   10.64.4.12    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
kube-system                  kube-proxy-gke-wonq-cluster-default-pool-4f3d2c53-7ox0   1/1     Running   0             11h   10.178.0.12   gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
kube-system                  kube-proxy-gke-wonq-cluster-default-pool-4f3d2c53-k97z   1/1     Running   0             11h   10.178.0.14   gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
kube-system                  kube-proxy-gke-wonq-cluster-default-pool-4f3d2c53-mk98   1/1     Running   0             11h   10.178.0.13   gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
kube-system                  l7-default-backend-654548fd95-7lcfm                      1/1     Running   0             11h   10.64.4.14    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
kube-system                  metrics-server-v1.32.2-7f79cc8f49-wv7lp                  1/1     Running   0             11h   10.64.1.12    gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
kube-system                  pdcsi-node-84t9t                                         2/2     Running   0             11h   10.178.0.14   gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
kube-system                  pdcsi-node-c2cw7                                         2/2     Running   0             11h   10.178.0.13   gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
kube-system                  pdcsi-node-kpcvc                                         2/2     Running   0             11h   10.178.0.12   gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
linkerd-viz                  metrics-api-6bdfcb79f5-t5wjc                             2/2     Running   0             11h   10.64.0.11    gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
linkerd-viz                  tap-85d56788dc-s9pbq                                     2/2     Running   0             11h   10.64.0.12    gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
linkerd-viz                  tap-injector-67d7659686-vg5xq                            2/2     Running   0             11h   10.64.4.16    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
linkerd-viz                  web-54dd5bd7d6-bsxw6                                     2/2     Running   0             11h   10.64.0.13    gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
linkerd                      linkerd-destination-79fc456664-j7dz4                     4/4     Running   0             11h   10.64.4.17    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
linkerd                      linkerd-identity-59b685cd68-wp797                        2/2     Running   0             11h   10.64.4.18    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
linkerd                      linkerd-proxy-injector-5568b9df4b-jnl68                  2/2     Running   0             11h   10.64.0.14    gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
monitoring                   alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0             11h   10.64.0.22    gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
monitoring                   prometheus-grafana-77bcfb9bdb-9vk9g                      3/3     Running   0             11h   10.64.4.19    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
monitoring                   prometheus-kube-prometheus-operator-6bbd85b5fd-8k4sm     1/1     Running   0             11h   10.64.4.21    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
monitoring                   prometheus-kube-state-metrics-7457555cf7-8sxvv           1/1     Running   0             11h   10.64.4.20    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
monitoring                   prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0             11h   10.64.0.23    gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
monitoring                   prometheus-prometheus-node-exporter-48k8l                1/1     Running   0             11h   10.178.0.12   gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
monitoring                   prometheus-prometheus-node-exporter-d5f4d                1/1     Running   0             11h   10.178.0.14   gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
monitoring                   prometheus-prometheus-node-exporter-fs75v                1/1     Running   0             11h   10.178.0.13   gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
pg-client                    pg-client-cbb8489bb-bslkq                                2/2     Running   0             11h   10.64.1.13    gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
pg-client                    pg-client-cbb8489bb-jgffj                                2/2     Running   0             11h   10.64.4.22    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
pg-client                    pg-client-cbb8489bb-t2hbj                                1/1     Running   0             11h   10.64.0.15    gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
pg-server                    pg-server-c5bc8cbd4-28zfj                                2/2     Running   2 (11h ago)   11h   10.64.4.26    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
pg-server                    pg-server-c5bc8cbd4-bx5b7                                2/2     Running   0             11h   10.64.1.15    gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
pg-server                    pg-server-c5bc8cbd4-j6lfx                                1/1     Running   0             11h   10.64.0.17    gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
wonq-order-merchant-client   wonq-order-merchant-client-5b6885f747-7qz7j              2/2     Running   0             11h   10.64.4.23    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
wonq-order-merchant-client   wonq-order-merchant-client-5b6885f747-bgzzn              1/1     Running   0             11h   10.64.0.18    gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
wonq-order-merchant-client   wonq-order-merchant-client-5b6885f747-fh4zd              2/2     Running   0             11h   10.64.1.17    gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
wonq-order-server            wonq-order-server-59bccf89dd-5vgpc                       2/2     Running   0             11h   10.64.1.18    gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
wonq-order-server            wonq-order-server-59bccf89dd-9dqmn                       2/2     Running   0             11h   10.64.4.24    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
wonq-order-server            wonq-order-server-59bccf89dd-lrqdz                       1/1     Running   0             11h   10.64.0.19    gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
wonq-order-user-client       wonq-order-user-client-8498d5cf4f-77jkv                  2/2     Running   0             11h   10.64.1.19    gke-wonq-cluster-default-pool-4f3d2c53-k97z   <none>           <none>
wonq-order-user-client       wonq-order-user-client-8498d5cf4f-prwrs                  2/2     Running   0             11h   10.64.4.25    gke-wonq-cluster-default-pool-4f3d2c53-7ox0   <none>           <none>
wonq-order-user-client       wonq-order-user-client-8498d5cf4f-w56gp                  1/1     Running   0             11h   10.64.0.20    gke-wonq-cluster-default-pool-4f3d2c53-mk98   <none>           <none>
```

#### 각 네임스페이스별 리소스 사용량

```text
❯ kubectl top pods --all-namespaces

NAMESPACE                    NAME                                                     CPU(cores)   MEMORY(bytes)
app-card-server              app-card-server-85f6f567b4-2dpfx                         2m           183Mi
app-card-server              app-card-server-85f6f567b4-r29ck                         2m           185Mi
app-card-server              app-card-server-85f6f567b4-t5kpf                         2m           184Mi
argocd                       argocd-application-controller-0                          5m           171Mi
argocd                       argocd-applicationset-controller-777d5b5dc7-vm6vc        1m           22Mi
argocd                       argocd-dex-server-7d8fcd845-7gjwm                        1m           22Mi
argocd                       argocd-notifications-controller-655df7c996-x9nzl         1m           19Mi
argocd                       argocd-redis-574484f6db-g9zbd                            2m           3Mi
argocd                       argocd-repo-server-57449f957c-swfwz                      1m           35Mi
argocd                       argocd-server-7dd4c8cf5f-lgs4g                           1m           42Mi
bank-server                  bank-server-75d568f9fb-66pct                             1m           6Mi
bank-server                  bank-server-75d568f9fb-6wdtg                             1m           6Mi
bank-server                  bank-server-75d568f9fb-mn2rp                             1m           6Mi
card-server                  card-server-595c89869f-29kkl                             2m           212Mi
card-server                  card-server-595c89869f-gb6tk                             2m           210Mi
card-server                  card-server-595c89869f-xgxfd                             2m           211Mi
cert-manager                 cert-manager-7d67448f59-r5jb9                            1m           23Mi
cert-manager                 cert-manager-cainjector-666b8b6b66-sqtld                 2m           41Mi
cert-manager                 cert-manager-webhook-78cb4cf989-cwdmv                    1m           20Mi
gke-managed-cim              kube-state-metrics-0                                     1m           47Mi
gmp-system                   collector-4wm8b                                          9m           103Mi
gmp-system                   collector-5v2hl                                          9m           79Mi
gmp-system                   collector-c86tg                                          9m           74Mi
gmp-system                   gmp-operator-658db64bf5-lgs44                            3m           19Mi
ingress-nginx                ingress-nginx-controller-7989649c5c-tmb9j                4m           50Mi
kube-system                  event-exporter-gke-bb5b454cd-dhzjz                       1m           22Mi
kube-system                  fluentbit-gke-555st                                      9m           44Mi
kube-system                  fluentbit-gke-gs6nf                                      9m           42Mi
kube-system                  fluentbit-gke-ncbbn                                      8m           42Mi
kube-system                  gke-metrics-agent-85txc                                  3m           52Mi
kube-system                  gke-metrics-agent-gqxjh                                  4m           49Mi
kube-system                  gke-metrics-agent-mzrsk                                  6m           52Mi
kube-system                  konnectivity-agent-7fc6dbc6b-2vwmc                       1m           22Mi
kube-system                  konnectivity-agent-7fc6dbc6b-cdqrt                       1m           22Mi
kube-system                  konnectivity-agent-7fc6dbc6b-lk8c9                       2m           23Mi
kube-system                  konnectivity-agent-autoscaler-5745d65c48-2bjd6           1m           7Mi
kube-system                  kube-dns-667895bc57-2ngtt                                4m           41Mi
kube-system                  kube-dns-667895bc57-6nv7x                                5m           43Mi
kube-system                  kube-dns-autoscaler-576f8b96b7-bv5pt                     1m           7Mi
kube-system                  kube-proxy-gke-wonq-cluster-default-pool-4f3d2c53-7ox0   1m           51Mi
kube-system                  kube-proxy-gke-wonq-cluster-default-pool-4f3d2c53-k97z   1m           47Mi
kube-system                  kube-proxy-gke-wonq-cluster-default-pool-4f3d2c53-mk98   2m           45Mi
kube-system                  l7-default-backend-654548fd95-7lcfm                      1m           3Mi
kube-system                  metrics-server-v1.32.2-7f79cc8f49-wv7lp                  4m           22Mi
kube-system                  pdcsi-node-84t9t                                         4m           11Mi
kube-system                  pdcsi-node-c2cw7                                         4m           12Mi
kube-system                  pdcsi-node-kpcvc                                         6m           12Mi
linkerd                      linkerd-destination-79fc456664-j7dz4                     2m           44Mi
linkerd                      linkerd-identity-59b685cd68-wp797                        1m           13Mi
linkerd                      linkerd-proxy-injector-5568b9df4b-jnl68                  1m           20Mi
linkerd-viz                  metrics-api-6bdfcb79f5-t5wjc                             1m           25Mi
linkerd-viz                  tap-85d56788dc-s9pbq                                     2m           26Mi
linkerd-viz                  tap-injector-67d7659686-vg5xq                            1m           14Mi
linkerd-viz                  web-54dd5bd7d6-bsxw6                                     1m           16Mi
monitoring                   alertmanager-prometheus-kube-prometheus-alertmanager-0   1m           29Mi
monitoring                   prometheus-grafana-77bcfb9bdb-9vk9g                      12m          330Mi
monitoring                   prometheus-kube-prometheus-operator-6bbd85b5fd-8k4sm     1m           24Mi
monitoring                   prometheus-kube-state-metrics-7457555cf7-8sxvv           5m           17Mi
monitoring                   prometheus-prometheus-kube-prometheus-prometheus-0       51m          422Mi
monitoring                   prometheus-prometheus-node-exporter-48k8l                3m           10Mi
monitoring                   prometheus-prometheus-node-exporter-d5f4d                2m           9Mi
monitoring                   prometheus-prometheus-node-exporter-fs75v                4m           11Mi
pg-client                    pg-client-cbb8489bb-bslkq                                1m           42Mi
pg-client                    pg-client-cbb8489bb-jgffj                                1m           41Mi
pg-client                    pg-client-cbb8489bb-t2hbj                                1m           38Mi
pg-server                    pg-server-c5bc8cbd4-28zfj                                2m           200Mi
pg-server                    pg-server-c5bc8cbd4-bx5b7                                2m           200Mi
pg-server                    pg-server-c5bc8cbd4-j6lfx                                2m           192Mi
wonq-order-merchant-client   wonq-order-merchant-client-5b6885f747-7qz7j              1m           41Mi
wonq-order-merchant-client   wonq-order-merchant-client-5b6885f747-bgzzn              1m           36Mi
wonq-order-merchant-client   wonq-order-merchant-client-5b6885f747-fh4zd              1m           44Mi
wonq-order-server            wonq-order-server-59bccf89dd-5vgpc                       2m           291Mi
wonq-order-server            wonq-order-server-59bccf89dd-9dqmn                       2m           297Mi
wonq-order-server            wonq-order-server-59bccf89dd-lrqdz                       2m           275Mi
wonq-order-user-client       wonq-order-user-client-8498d5cf4f-77jkv                  1m           50Mi
wonq-order-user-client       wonq-order-user-client-8498d5cf4f-prwrs                  1m           37Mi
wonq-order-user-client       wonq-order-user-client-8498d5cf4f-w56gp                  1m           37Mi
```

#### 현재 노드 리소스 사용량

```text
❯ kubectl top nodes

NAME                                          CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
gke-wonq-cluster-default-pool-4f3d2c53-7ox0   219m         11%    3512Mi          58%
gke-wonq-cluster-default-pool-4f3d2c53-k97z   167m         8%     2870Mi          47%
gke-wonq-cluster-default-pool-4f3d2c53-mk98   274m         14%    3489Mi          57%
```

:::info kubectl top nodes vs kubectl top pods 차이
`kubectl top pods --all-namespaces` 총합(CPU 253m, Memory 5,475Mi)과 `kubectl top nodes` 총합(CPU 660m, Memory 9,871Mi)이 다른
이유는 **시스템 오버헤드** 때문이에요.

운영체제 커널, 컨테이너 런타임(Docker/containerd), Kubernetes 시스템 컴포넌트들이 추가로 리소스를 사용하기 때문에 실제 노드에서는 Pod 사용량보다 훨씬 많은 리소스가 필요해요.
:::

### 1-2. 실제 리소스 사용량 분석

#### 비즈니스 애플리케이션별 리소스 사용량

| 네임스페이스                     | Pod 수   | CPU 사용량 | 메모리 사용량     | 비고        |
|----------------------------|---------|---------|-------------|-----------|
| app-card-server            | 3개      | 6m      | 552Mi       | 앱카드 서버    |
| bank-server                | 3개      | 3m      | 18Mi        | 은행 서버     |
| card-server                | 3개      | 6m      | 633Mi       | 카드 서버     |
| pg-client                  | 3개      | 3m      | 121Mi       | PG 클라이언트  |
| pg-server                  | 3개      | 6m      | 592Mi       | PG 서버     |
| wonq-order-merchant-client | 3개      | 3m      | 121Mi       | 가맹점 클라이언트 |
| wonq-order-server          | 3개      | 6m      | 863Mi       | 주문 서버     |
| wonq-order-user-client     | 3개      | 3m      | 124Mi       | 사용자 클라이언트 |
| **합계**                     | **24개** | **36m** | **3,024Mi** |           |

#### 인프라 서비스 리소스 사용량

| 네임스페이스                    | Pod 수   | CPU 사용량  | 메모리 사용량     | 비고                            |
|---------------------------|---------|----------|-------------|-------------------------------|
| **argocd**                | 7개      | 12m      | 314Mi       | GitOps 기반 애플리케이션 배포           |
| **cert-manager**          | 3개      | 4m       | 84Mi        | SSL/TLS 인증서 자동 관리             |
| **gke-managed-cim**       | 1개      | 1m       | 47Mi        | GKE 클러스터 정보 수집 (GCP 전용)       |
| **gmp-system**            | 4개      | 30m      | 275Mi       | GCP 전용 메트릭 (AWS 마이그레이션 시 제거)  |
| **ingress-nginx**         | 1개      | 4m       | 50Mi        | HTTP/HTTPS 인그레스 컨트롤러          |
| **kube-system**           | 22개     | 78m      | 671Mi       | Kubernetes 핵심 시스템 컴포넌트        |
| **linkerd + linkerd-viz** | 7개      | 9m       | 158Mi       | 서비스 메시 및 트래픽 관리               |
| **monitoring**            | 8개      | 79m      | 852Mi       | Prometheus + Grafana 모니터링 시스템 |
| **합계**                    | **53개** | **217m** | **2,451Mi** |                               |

#### 전체 리소스 사용량 요약

| 항목            | 값                 |
|---------------|-------------------|
| **전체 Pod 수**  | 76개               |
| **총 CPU 사용량** | 253m (0.253 vCPU) |
| **총 메모리 사용량** | 5,475Mi (5.74 GB) |

#### 시스템 오버헤드

| 구분           | CPU             | 메모리              |
|--------------|-----------------|------------------|
| **시스템 오버헤드** | 407m (0.4 vCPU) | 4,396Mi (4.61GB) |

이 데이터를 바탕으로 AWS EKS 인스턴스 사양을 결정할 수 있어요.

## AWS EKS 클러스터 설계 원칙

AWS EKS로 마이그레이션하기 전에 [AWS EKS Best Practices Guide](https://docs.aws.amazon.com/eks/latest/best-practices/introduction.html)를 꼼꼼히 검토했어요.

[AWS EKS Best Practices Guide](https://docs.aws.amazon.com/eks/latest/best-practices/introduction.html)에 따르면 다음과 같은 핵심 원칙들이 있어요:

### 1. 클러스터 구조화: 고가용성 및 확장성

- **다중 노드 구성**: 단일 노드 대신 여러 노드를 사용하는 것이 장애 조치(fault tolerance)와 확장성 측면에서 권장돼요. 
  Kubernetes의 설계 철학상, 여러 노드를 배포하면 리소스 활용도가 높아지고, 노드 장애 시에도 애플리케이션이 지속적으로 실행될 수 있어요.

- **다중 가용 영역(AZ) 배포**: EKS 클러스터를 여러 가용 영역에 걸쳐 배포하면 고가용성을 확보할 수 있어요.

- **관리형 노드 그룹**: EKS는 관리형 노드 그룹을 통해 EC2 노드의 프로비저닝과 생명 주기를 자동화해요.
  이는 노드 업데이트와 스케일링을 간소화하며, 클러스터 오토스케일러와 통합하여 필요에 따라 노드 수를 동적으로 조정할 수 있어요.

### 2. 비용 최적화

- **오토스케일링**: 필요에 따라 노드 수를 동적으로 조정하여 불필요한 비용을 줄일 수 있어요.

- **적절한 인스턴스 크기**: 너무 작은 인스턴스보다는 적당한 크기가 효율적이에요.

이러한 베스트 프랙티스를 기반으로 저희는 **다중 노드 구성을 채택**하기로 결정했어요.

## AWS EKS 인스턴스 사양 결정 과정

### 1. 필요 리소스 계산 (시스템 오버헤드 포함)

3개 노드에서 발생하는 시스템 오버헤드를 기반으로 노드당 평균 오버헤드를 계산하고, 이를 바탕으로 전체 필요량을 계산해 보았어요.

| 구분              | CPU                | 메모리                      |
|-----------------|--------------------|--------------------------|
| **노드당 평균 오버헤드** | 136m (407m ÷ 3)    | 1,465Mi (4,396Mi ÷ 3)    |
| **Pod 사용량**     | 253m               | 5,475Mi                  |
| **전체 필요량**      | 253m + (136m × N개) | 5,475Mi + (1,465Mi × N개) |

이를 통해 N개의 노드에서 필요한 CPU와 메모리를 계산할 수 있었죠.

| 노드 수   | 필요 CPU                           | 필요 메모리                                            |
|--------|----------------------------------|---------------------------------------------------|
| **2개** | (253m + 272m) × 1.2 = **630m**   | (5,475Mi + 2,930Mi) × 1.2 = **10,086Mi (10.2GB)** |
| **3개** | (253m + 408m) × 1.2 = **793m**   | (5,475Mi + 4,395Mi) × 1.2 = **11,844Mi (12.4GB)** |
| **4개** | (253m + 544m) × 1.2 = **956m**   | (5,475Mi + 5,860Mi) × 1.2 = **13,602Mi (14.3GB)** |
| **5개** | (253m + 680m) × 1.2 = **1,120m** | (5,475Mi + 7,325Mi) × 1.2 = **15,360Mi (16.0GB)** |

### 2. 운영 가능한 인스턴스 유형 선택

위 계산을 바탕으로 실제 운영 가능한 인스턴스 유형들만 비교 분석했어요.

:::tip 검토 기준
메모리가 크게 부족한 옵션들은 운영 리스크가 높으므로 제외하고, **여유롭거나 약간 부족한 수준**의 현실적인 옵션들만 검토했습니다.
:::

#### 옵션 1: t3.medium × 3개 - 최종 선택

- **제공 리소스**: 6 vCPU, 12GB RAM (총합)
- **필요 리소스**: 793m CPU, 12.4GB 메모리
- **리소스 충족도**: CPU 여유, Memory 약간 부족
- **시간당 비용**: $0.00624 × 3 = $0.01872
- **평가**: 가장 비용 효율적이며 3개 노드로 적절한 고가용성 확보, 메모리 최적화로 운영 가능

#### 옵션 2: t3.medium × 4개

- **제공 리소스**: 8 vCPU, 16GB RAM (총합)
- **필요 리소스**: 956m CPU, 14.3GB 메모리
- **리소스 충족도**: CPU 여유, Memory 여유
- **시간당 비용**: $0.00624 × 4 = $0.02496
- **평가**: 리소스 여유도가 높지만, 비용이 **옵션 1보다 33% 비쌈**. 메모리 여유는 있지만, 3개 노드로도 충분히 고가용성 확보 가능

#### 옵션 3: t3.large × 2개

- **제공 리소스**: 4 vCPU, 16GB RAM (총합)
- **필요 리소스**: 630m CPU, 10.6GB 메모리
- **리소스 충족도**: CPU 여유, Memory 여유
- **시간당 비용**: $0.01248 × 2 = $0.02496
- **평가**: 리소스 여유도가 높지만, 동일 가격에 비해 **옵션 2보다 CPU가 부족**하고, 2개 노드로 인한 **가용성이 떨어짐**.

## 최종 결정: t3.medium × 3개 다중 노드 구성

여러 요소를 종합적으로 고려해 **t3.medium × 3개 다중 노드 구성**을 선택했어요. 해당 구성을 선택한 이유는 다음과 같아요:

1. 먼저 **비용 효율성**이 가장 중요한 요소였어요. 시간당 $0.01872로 **가장 저렴한 비용**을 실현할 수 있어요.

2. 메모리 부족분(0.4GB)에 대해서는 **EKS 오토스케일링을 신뢰**하기로 했어요.
   현재 저희가 계산한 리소스는 **이미 1.2배 여유분이 포함**된 것이고, **EKS Cluster Autoscaler가 부하 증가 시 자동으로 노드를 추가**해줄 거예요.

3. **고가용성** 측면에서도 3개 노드면 충분한 **분산 효과와 장애 복원력을 확보**할 수 있어요.
   1개 노드에 장애가 발생해도 나머지 2개 노드로 서비스를 지속할 수 있고, 과도한 노드 수보다는 실용적인 구성이라고 생각해요.

## 마무리

이렇게 AWS EKS 클러스터 인스턴스 스펙을 결정하는 과정을 공유해 보았어요.
GCP에서 POC를 진행하면서, 리소스 사용량을 면밀히 분석하고, AWS의 다양한 인스턴스 옵션을 비교하여 최적의 선택을 할 수 있었죠.

다음 포스트에서는 AWS 인프라 구성을 어떻게 진행했는지 공유해 볼게요.

AWS EKS 마이그레이션이나 인스턴스 스펙 결정과 관련해 궁금한 점이 있으시다면 언제든지 문의해주세요!

## 참고 자료

### AWS 공식 문서

- [Amazon EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html)
- [EC2 Instance Types](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-types.html)
- [AWS EKS Best Practices Guide](https://docs.aws.amazon.com/eks/latest/best-practices/introduction.html)
- [AWS EKS Pricing](https://aws.amazon.com/eks/pricing/)
