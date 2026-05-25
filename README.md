# Online Boutique × OpenTelemetry × Jaeger

Google의 마이크로서비스 데모 앱 [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo)에 OpenTelemetry + Jaeger를 연동해 **코드 수정 없이 분산 트레이싱**을 구성한 Kubernetes 실습 프로젝트.

> Istio 없이, SDK 직접 수정 없이, 환경변수만으로 7개 서비스 트레이스 확보

---

## Architecture

```
[서비스들]  ──OTLP(gRPC:4317)──▶  [OTel Collector]  ──OTLP──▶  [Jaeger]
```

**Traced services** (7/9): `frontend` · `checkoutservice` · `productcatalogservice` · `currencyservice` · `emailservice` · `paymentservice` · `recommendationservice`

---

## Stack

| | |
|---|---|
| Kubernetes | v1.29.15 |
| CNI | Cilium v1.19.1 — kube-proxy 없음, eBPF NodePort |
| Node | arm64 × 3 (Mac UTM 가상머신) |
| App | Online Boutique v0.10.5 |
| Collector | opentelemetry-collector-contrib:0.99.0 |
| Tracing Backend | Jaeger all-in-one:1.57 |
| OTel Operator | 0.151.0 + cert-manager v1.17.2 |

amd64 전용 이미지(Node.js / Python / .NET / Java)는 QEMU binfmt 에뮬레이션으로 arm64 노드에서 실행.

---

## Quick Start

```bash
kubectl apply -f 01-namespace.yaml
kubectl apply -f 02-jaeger.yaml
kubectl apply -f 03-otel-collector.yaml
kubectl apply -f 04-online-boutique.yaml
```

| URL | 용도 |
|-----|------|
| `http://<node-ip>:30080` | Online Boutique UI |
| `http://<node-ip>:32686` | Jaeger UI |

---

## Docs

- [`OTEL_STUDY.md`](./OTEL_STUDY.md) — OTel / Collector / Jaeger 개념 정리 및 접근 방식 비교
- [`WORK_LOG.md`](./WORK_LOG.md) — 트러블슈팅 전체 기록
