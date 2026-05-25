# OpenTelemetry 트레이싱 실습 — OTel Operator + Collector + Jaeger

> Istio 없이, 앱 코드 수정 없이, Kubernetes에서 분산 트레이싱 구축하기

---

## 1. 왜 이 스택인가 — 접근 방식 비교

분산 트레이싱을 구현하는 방법은 크게 세 가지다.

| 방식 | 동작 원리 | 코드 수정 | 언어 제약 | 특징 |
|------|-----------|-----------|-----------|------|
| **Istio (Service Mesh)** | 사이드카 프록시(Envoy)가 네트워크 계층에서 트레이스 수집 | 없음 | 없음 | L7 트래픽만 캡처. 앱 내부 스팬(DB 쿼리, 함수 단위) 불가 |
| **OTel SDK (수동)** | 앱 코드에 직접 SDK 임포트 → 스팬 생성 | 많음 | 언어별 SDK 필요 | 가장 정밀한 계측. 원하는 모든 것을 스팬으로 만들 수 있음 |
| **OTel Operator (자동)** | K8s Admission Webhook이 Pod 시작 시 에이전트를 자동 주입 | 없음 | Java, .NET, Python, Node.js, Go(제한) | SDK 계측 수준에 근접. 코드 수정 불필요 |

### 이번 실습에서 선택한 방식

**OTel SDK (기존 코드) + OTel Collector + Jaeger**

Online Boutique는 이미 각 서비스에 OTel SDK가 내장되어 있었음.
추가 코드 수정 없이 환경변수만으로 OTLP exporter를 활성화하는 구조.
OTel Operator는 SDK가 없는 서비스(Java OpenCensus 등)를 보완하기 위해 추가 설치.

---

## 2. 핵심 개념

### OpenTelemetry란

CNCF(Cloud Native Computing Foundation) 프로젝트.
관측 가능성(Observability)을 위한 **표준 규격 + SDK + 에이전트** 모음.
벤더 중립적 — 데이터를 수집하는 방식을 표준화하고, 어디에 보낼지는 나중에 결정.

세 가지 신호(Signal)를 다룬다:
- **Traces** — 요청이 여러 서비스를 거치는 흐름
- **Metrics** — 숫자로 표현되는 시스템 상태
- **Logs** — 이벤트 기록

이번 실습은 **Traces** 만 다룬다.

---

### Trace / Span 개념

```
사용자 요청 → frontend → checkoutservice → paymentservice → emailservice
                    ↓
              productcatalogservice
```

- **Trace**: 하나의 요청이 시스템을 통과하는 전체 여정 (위 그림 전체)
- **Span**: 각 서비스에서 처리한 작업 단위 (네모 하나하나)
- **Context Propagation**: 서비스 간 호출 시 Trace ID를 HTTP 헤더로 전달해 연결

```
# 전파되는 헤더 예시 (W3C TraceContext)
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^ ^^
             version  trace-id (128bit)           span-id (64bit)  flags
```

---

### OTLP (OpenTelemetry Protocol)

OTel에서 정의한 텔레메트리 전송 프로토콜.

| 프로토콜 | 포트 | 특징 |
|----------|------|------|
| OTLP/gRPC | 4317 | 기본값. HTTP/2 기반. 고성능 |
| OTLP/HTTP (Protobuf) | 4318 | HTTP/1.1도 지원. 방화벽 친화적 |

---

### OTel Collector

수집 파이프라인 역할. 앱이 직접 Jaeger에 보내지 않고 Collector를 경유한다.

```
[앱] --OTLP--> [OTel Collector] --OTLP--> [Jaeger]
                    │
                    ├─ Receivers  : 수신 (OTLP gRPC :4317, OTLP HTTP :4318)
                    ├─ Processors : 가공 (batch, filter, sampling 등)
                    └─ Exporters  : 전송 (Jaeger, Prometheus, stdout 등)
```

**왜 Collector를 쓰는가?**
- 앱과 백엔드를 디커플링 → Jaeger를 다른 것으로 교체해도 앱 코드 변경 불필요
- 배치 처리, 샘플링, 필터링을 한 곳에서 관리
- 여러 백엔드로 동시 전송 가능

이번 실습 Collector 설정:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 1024

exporters:
  otlp:
    endpoint: jaeger:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
```

---

### Jaeger

분산 트레이스 저장 및 시각화 백엔드.
CNCF 졸업 프로젝트. Uber에서 개발.

- `jaeger-all-in-one`: 수집기 + 저장소 + UI를 단일 바이너리로 제공 (개발/실습용)
- 프로덕션에서는 Cassandra 또는 Elasticsearch 스토리지 분리
- **주의**: all-in-one은 in-memory 저장 → 재시작 시 모든 트레이스 사라짐

---

### OTel Operator

Kubernetes Operator 패턴으로 동작.
핵심 기능 두 가지:

1. **OpenTelemetryCollector CR**: Collector 배포를 K8s 리소스로 선언
2. **Instrumentation CR**: Pod에 자동으로 언어별 에이전트를 주입

**Auto-Instrumentation 주입 원리:**

```
Pod 생성 요청
    ↓
API Server → Admission Webhook (OTel Operator)
    ↓
annotation 확인: instrumentation.opentelemetry.io/inject-java: "true"
    ↓
initContainer 추가: javaagent.jar 복사
env 주입: JAVA_TOOL_OPTIONS=-javaagent:/path/javaagent.jar
          OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4317
    ↓
앱 컨테이너 시작 → JVM이 agent 자동 로드 → 트레이스 자동 생성
```

코드 변경 없이 Deployment에 annotation 한 줄로 계측 가능.

OTel Operator는 cert-manager에 의존함 (Admission Webhook TLS 인증서 자동 발급).

```bash
# 설치 순서
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true --version v1.17.2

helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
  --namespace opentelemetry-operator-system --create-namespace \
  --set admissionWebhooks.certManager.enabled=true \
  --set "manager.extraArgs={--enable-go-instrumentation}"
```

---

## 3. 전체 아키텍처

```
┌─────────────────────────────────────────────────────┐
│                  online-boutique namespace            │
│                                                       │
│  [frontend] ──→ [checkoutservice] ──→ [paymentservice]│
│      │                │                      │       │
│      ↓                ↓                      ↓       │
│  [productcatalog] [currencyservice]  [emailservice]  │
│                                                       │
│  모든 서비스 ──OTLP──→ [OTel Collector]               │
│                       :4317(gRPC) :4318(HTTP)         │
│                              │                        │
│                              ↓ OTLP                   │
│                          [Jaeger]                     │
│                       :4317 (수신)                    │
│                       :16686 (UI)                     │
└─────────────────────────────────────────────────────┘

NodePort 외부 접근:
  192.168.64.3:30080 → Online Boutique UI
  192.168.64.3:32686 → Jaeger UI
```

---

## 4. 트레이싱 활성화 방법

### 환경변수 방식 (SDK 탑재 서비스)

Online Boutique 서비스들은 `ENABLE_TRACING=1`이 있어야 OTel SDK가 초기화됨.
`OTEL_SERVICE_NAME`이 없으면 서비스명이 `unknown_service:{바이너리명}`으로 표시됨.

```yaml
env:
  - name: ENABLE_TRACING
    value: "1"
  - name: OTEL_SERVICE_NAME
    value: "frontend"
  - name: COLLECTOR_SERVICE_ADDR
    value: "opentelemetrycollector:4317"
```

코드 수정 없이 환경변수만 추가하면 트레이스 전송 시작.

---

### 서비스별 트레이싱 현황

| 서비스 | 언어 | 라이브러리 | Jaeger | 비고 |
|--------|------|-----------|--------|------|
| frontend | Go | OTel SDK | ✅ | |
| checkoutservice | Go | OTel SDK | ✅ | |
| productcatalogservice | Go | OTel SDK | ✅ | 바이너리명 `server` → OTEL_SERVICE_NAME 필수 |
| currencyservice | Node.js | OTel SDK | ✅ | |
| emailservice | Python | OTel SDK | ✅ | |
| paymentservice | Node.js | OTel SDK | ✅ | |
| recommendationservice | Python | OTel SDK | ✅ | |
| cartservice | .NET | OTel SDK | ❌ | gRPC cleartext 문제 (아래 참조) |
| shippingservice | Go | OpenCensus | ❌ | OTLP 미지원 (아래 참조) |

---

## 5. 트러블슈팅 — 겪은 문제들

### OTEL_SERVICE_NAME 없으면 이름이 이상하게 나온다

Jaeger에 `unknown_service:server`가 표시됨.
Go OTel SDK는 `OTEL_SERVICE_NAME`이 없으면 바이너리 실행 파일명을 서비스 이름으로 사용.
`productcatalogservice`의 바이너리명이 `server`였기 때문.

**Fix**: `OTEL_SERVICE_NAME=productcatalogservice` 추가.

---

### cartservice .NET gRPC cleartext 문제

.NET gRPC 클라이언트는 기본적으로 **TLS가 없는 HTTP/2 (h2c) 를 거부**함.
OTel Collector는 TLS 없이 gRPC 포트 4317을 열고 있어서 연결 실패.

HTTP/Protobuf(포트 4318)로 전환을 시도했으나, 앱 코드가 `COLLECTOR_SERVICE_ADDR`
환경변수를 읽어서 endpoint를 직접 하드코딩하기 때문에 `OTEL_EXPORTER_OTLP_ENDPOINT`
env var가 무시됨.

```
OTEL_EXPORTER_OTLP_ENDPOINT 설정
    ↓
OTel SDK options 초기화 시 env var 읽음
    ↓
코드에서 COLLECTOR_SERVICE_ADDR 읽어 endpoint 덮어쓰기 (hardcode)
    ↓
결국 포트 4317 (gRPC 전용) 으로 HTTP/Protobuf 전송 → 실패
```

**결론**: 앱 코드 수정 없이 해결 불가.

---

### OpenCensus vs OpenTelemetry

| | OpenCensus | OpenTelemetry |
|--|------------|---------------|
| 상태 | 구버전, 유지보수 중단 예정 | 현재 표준 |
| 프로토콜 | 자체 포맷 | OTLP |
| OTLP 지원 | ❌ | ✅ |

shippingservice가 OpenCensus를 사용하기 때문에 OTel Collector의 OTLP 수신 포트로 전송 불가.
해결하려면: Collector에 OpenCensus receiver 추가 또는 앱 코드를 OTel SDK로 마이그레이션.

---

## 6. 의사결정 포인트

### Collector를 앱과 같은 namespace에 배치

`opentelemetrycollector.online-boutique.svc` — 같은 namespace 내 DNS로 접근.
별도 namespace에 두면 모든 서비스의 endpoint를 FQDN으로 바꿔야 하고,
NetworkPolicy가 있으면 cross-namespace 트래픽을 별도로 허용해야 함.

### Jaeger all-in-one 선택

실습 목적이므로 in-memory 스토리지로 충분.
컴포넌트 분리(collector, query, storage) 없이 단일 Pod으로 운영 가능.

### OTel Operator 설치 (결과적으로 미활용)

SDK가 없는 서비스에 auto-instrumentation을 주입하기 위해 설치.
Java adservice에 적용했으나 QEMU 환경에서 JVM + agent 초기화가 10분 이상 걸려 실용성이 없었음.
네이티브 환경이라면 annotation 한 줄로 바로 동작함.

---

## 7. 이 스택으로 얻은 것

```
사용자가 checkout 버튼을 누른다
    │
    ▼ Jaeger UI에서 확인 가능한 것들:
    ├─ frontend → checkoutservice 호출: 몇 ms?
    ├─ checkoutservice → productcatalogservice 호출: 몇 ms?
    ├─ checkoutservice → currencyservice 호출: 몇 ms?
    ├─ checkoutservice → paymentservice 호출: 몇 ms?
    └─ checkoutservice → emailservice 호출: 몇 ms?
```

- **병목 서비스 파악**: 어느 서비스에서 시간이 많이 걸리는지 한눈에
- **오류 전파 추적**: 어느 서비스에서 에러가 발생해서 요청이 실패했는지
- **코드 수정 없이**: 환경변수만으로 구현 (SDK 이미 탑재된 서비스 기준)

---

## 8. 실습 환경 스펙

| 항목 | 내용 |
|------|------|
| Kubernetes | v1.29.15 |
| 노드 구성 | master 1 + worker 2 (UTM 가상머신, arm64) |
| CNI | Cilium v1.19.1 (kube-proxy 없음) |
| 앱 | Online Boutique v0.10.5 (Google microservices demo) |
| OTel Collector | otel/opentelemetry-collector-contrib:0.99.0 |
| Jaeger | jaegertracing/all-in-one:1.57 |
| OTel Operator | 0.151.0 (cert-manager v1.17.2 의존) |
