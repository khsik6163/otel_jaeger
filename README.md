# Online Boutique + OpenTelemetry + Jaeger on arm64 / Cilium

> **환경**: Kubernetes v1.29.15 · Cilium v1.19.1 · arm64 (UTM on Mac) · kube-proxy 없음

---

## 목차

1. [아키텍처 개요](#아키텍처-개요)
2. [왜 NodePort가 안 됐나 — Cilium kube-proxy-replacement 완전 해설](#왜-nodeport가-안-됐나)
   - [일반 쿠버네티스의 NodePort: iptables 방식](#일반-쿠버네티스의-nodeport-iptables-방식)
   - [Cilium eBPF 방식과 kube-proxy-replacement](#cilium-ebpf-방식과-kube-proxy-replacement)
   - [이 클러스터의 문제: false + 노 kube-proxy](#이-클러스터의-문제-false--노-kube-proxy)
   - [수정 방법](#수정-방법)
3. [arm64에서 amd64 이미지 실행 — QEMU binfmt](#arm64에서-amd64-이미지-실행--qemu-binfmt)
4. [loadgenerator 수정 내역](#loadgenerator-수정-내역)
5. [설치 순서](#설치-순서)
6. [현재 상태 및 접근 URL](#현재-상태-및-접근-url)
7. [트러블슈팅 명령어 모음](#트러블슈팅-명령어-모음)
8. [알려진 이슈](#알려진-이슈)

---

## 아키텍처 개요

```
Mac 브라우저
    │
    │  http://192.168.64.3:30080  (NodePort)
    │  http://192.168.64.3:32686  (Jaeger)
    ▼
┌─────────────────────────────────────────────┐
│  k8s-worker-1  (192.168.64.3, arm64)        │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │  online-boutique namespace          │    │
│  │                                     │    │
│  │  [frontend]──────────────────────┐  │    │
│  │  [checkoutservice]               │  │    │
│  │  [productcatalogservice]    OTLP  │  │    │
│  │  [shippingservice]          gRPC  │  │    │
│  │  [currencyservice]*    ─────────▶ │  │    │
│  │  [paymentservice]*          4317  │  │    │
│  │  [emailservice]*                  │  │    │
│  │  [recommendationservice]*  [OTel Collector]
│  │  [cartservice]*             │    │  │    │
│  │  [adservice]*               │ OTLP gRPC  │
│  │  [redis-cart]               ▼    │  │    │
│  │  [loadgenerator]       [Jaeger]  │  │    │
│  │                         :32686   │  │    │
│  └─────────────────────────────────-┘  │    │
│                                        │    │
│  CNI: Cilium v1.19.1 (eBPF)           │    │
│  * = amd64 이미지, QEMU 에뮬레이션    │    │
└────────────────────────────────────────┘    │
                                              │
┌─────────────────────────────────────────────┐
│  k8s-worker-2  (192.168.64.4, arm64)        │
│  Cilium: CrashLoopBackOff (앱 pod 없음)     │
└─────────────────────────────────────────────┘
```

---

## 왜 NodePort가 안 됐나

### 일반 쿠버네티스의 NodePort: iptables 방식

표준 Kubernetes에서 NodePort 트래픽 처리는 **kube-proxy**가 담당한다.
kube-proxy는 노드마다 실행되며, iptables(또는 ipvs) 규칙을 직접 조작해 외부 포트 → Pod IP:Port 변환(DNAT)을 처리한다.

```
외부 패킷 (dst: NodeIP:30080)
        │
        ▼
  [iptables PREROUTING chain]
        │
        │  KUBE-NODEPORTS chain (kube-proxy가 생성)
        │    -p tcp --dport 30080 -j DNAT --to-destination 10.0.1.11:8080
        │
        ▼
  [라우팅] → Pod (10.0.1.11:8080)
```

kube-proxy가 없으면 이 iptables 규칙이 존재하지 않아 NodePort 포트로 들어오는 패킷이 처리되지 않고 `Connection refused`가 발생한다.

---

### Cilium eBPF 방식과 kube-proxy-replacement

Cilium은 iptables 대신 **eBPF 프로그램**을 커널에 삽입해 네트워킹을 처리한다. iptables보다 훨씬 빠르며(O(1) 룩업), 커널 내부에서 직접 패킷을 처리한다.

NodePort 처리도 Cilium이 eBPF로 대체할 수 있다. `kube-proxy-replacement` 설정이 이를 제어한다.

```
kube-proxy-replacement: true 일 때:
                                                   
외부 패킷 (dst: NodeIP:30080)
        │
        ▼
  [XDP / tc eBPF hook]  ← Cilium이 커널에 삽입한 eBPF 프로그램
        │
        │  cilium_lb4_services 맵 룩업 (O(1))
        │    key: 192.168.64.3:30080
        │    val: 10.0.1.11:8080
        │
        ▼ (DNAT, 커널 내부에서 직접 처리)
  Pod (10.0.1.11:8080)
```

Cilium eBPF 맵 확인:
```bash
kubectl exec -n kube-system <cilium-pod> -- cilium bpf lb list | grep NodePort
# 출력 예시:
# 192.168.64.3:30080/TCP (0)     0.0.0.0:0 (55) (0) [NodePort]
# 192.168.64.3:30080/TCP (1)     10.0.1.11:8080/TCP (55) (1)
```

---

### 이 클러스터의 문제: false + 노 kube-proxy

이 클러스터는 처음부터 **kube-proxy가 설치되지 않았다**. 하지만 Cilium의 `kube-proxy-replacement`는 `false`로 설정되어 있었다.

```
kube-proxy         : ❌ 미설치 (iptables 규칙 없음)
kube-proxy-replacement: false  → Cilium도 NodePort eBPF 규칙 생성 안 함

결과: NodePort를 처리하는 컴포넌트가 아무것도 없음
      → curl http://192.168.64.3:30080 → Connection refused (HTTP 000)
```

`kube-proxy-replacement`의 두 모드 차이:

| 설정 | 동작 |
|------|------|
| `false` | Cilium은 Pod 간 네트워킹(L3/L4)만 담당. Service/NodePort 처리는 kube-proxy에 위임 |
| `true` | Cilium이 Service, NodePort, LoadBalancer, ExternalIP를 모두 eBPF로 직접 처리. kube-proxy 불필요 |

> **핵심**: kube-proxy도 없고, Cilium도 NodePort를 처리하지 않으면 → NodePort는 아무도 처리 안 함.

---

### 수정 방법

```bash
# 1. Cilium ConfigMap 패치 (kube-proxy-replacement: true)
kubectl patch configmap cilium-config -n kube-system \
  --type merge \
  -p '{"data":{"kube-proxy-replacement":"true"}}'

# 2. Cilium pod 재시작 (ConfigMap 변경은 자동 반영 안 됨)
kubectl delete pod -n kube-system \
  $(kubectl get pod -n kube-system -l k8s-app=cilium -o name | grep -v worker-2)

# 3. 재시작 완료 확인
kubectl get pods -n kube-system -l k8s-app=cilium

# 4. NodePort eBPF 맵 확인
kubectl exec -n kube-system <cilium-pod-on-worker-1> -- \
  cilium bpf lb list | grep "30080\|32686"
```

재시작 후 Cilium은 자동으로 Kubernetes API에서 Service 목록을 읽어 NodePort용 eBPF 맵을 생성한다.

---

## arm64에서 amd64 이미지 실행 — QEMU binfmt

### 왜 필요한가

Online Boutique v0.10.5 이미지 중 일부는 **amd64 전용**으로만 빌드되어 있다.

| 서비스 | 언어 | 아키텍처 |
|--------|------|----------|
| frontend, checkoutservice, productcatalogservice, shippingservice | Go | **multi-arch** (arm64 네이티브) |
| currencyservice, paymentservice | Node.js | **amd64 전용** |
| emailservice, recommendationservice | Python | **amd64 전용** |
| cartservice | .NET | **amd64 전용** |
| adservice | Java | **amd64 전용** |

arm64 노드에서 amd64 바이너리를 실행하려면 **Linux binfmt_misc + QEMU user-mode 에뮬레이션**이 필요하다.

### binfmt_misc 동작 원리

```
arm64 커널이 x86_64 ELF 바이너리 실행 요청을 받음
        │
        ▼
  /proc/sys/fs/binfmt_misc/ 에 등록된 인터프리터 확인
  (magic: ELF x86_64 헤더 매칭)
        │
        ▼
  /usr/libexec/qemu-binfmt/x86_64-binfmt-P 호출
        │
        ▼
  QEMU user-mode가 x86_64 명령어를 arm64 명령어로 실시간 변환 (JIT)
        │
        ▼
  arm64 커널에서 실행
```

등록된 인터프리터 확인:
```bash
kubectl debug node/k8s-worker-1 --image=busybox:1.36 -- \
  sh -c "cat /host/proc/sys/fs/binfmt_misc/qemu-x86_64"
# interpreter /usr/libexec/qemu-binfmt/x86_64-binfmt-P
# flags: PF
# P: Preserve argv[0] (원본 바이너리 경로 유지)
# F: Fix-binary (인터프리터를 메모리에 미리 올려 빠른 로딩)
```

### QEMU 에뮬레이션 트레이드오프

| 항목 | 설명 |
|------|------|
| 실행 속도 | native 대비 2~5배 느림 (JIT 변환 오버헤드) |
| 메모리 | QEMU 에뮬레이터 자체가 추가 메모리 소비 (100~200MB) |
| 호환성 | 대부분 동작하나 일부 syscall 에뮬레이션 불완전 |
| Cold start | JVM, V8(Node.js), Python 인터프리터 기동이 느려짐 |

이 때문에 QEMU 환경의 서비스들에는 아래 설정을 적용했다:

```yaml
livenessProbe:
  initialDelaySeconds: 60   # native 기본값(10~30)보다 길게
  failureThreshold: 10      # 재시작 전 허용 실패 횟수 증가
resources:
  limits:
    memory: 256Mi           # QEMU 오버헤드 포함 (기본 128Mi에서 증가)
```

.NET cartservice 추가 설정:
```yaml
env:
- name: DOTNET_GCConserveMemory
  value: '9'               # GC 적극적 메모리 절약 모드 (0~9)
securityContext:
  readOnlyRootFilesystem: false  # .NET 런타임이 임시 파일 필요
volumeMounts:
- name: tmp-dir
  mountPath: /tmp          # 쓰기 가능한 /tmp 제공
```

### QEMU binfmt 등록 영속성 주의

binfmt_misc 등록은 **커널 메모리에 존재**하므로 노드 재부팅 시 초기화된다.
재부팅 후에는 QEMU DaemonSet을 다시 적용해야 한다:

```bash
# 현재 클러스터에는 DaemonSet이 삭제됨 (binfmt 등록은 worker-1 커널에 유지 중)
# worker-1 재부팅 시 아래 명령 실행 필요
kubectl apply -f 00-qemu-binfmt.yaml
```

---

## loadgenerator 수정 내역

Google 원본 loadgenerator 이미지(`google-samples/microservices-demo/loadgenerator:v0.10.5`)는 **amd64 전용**이다.

### 문제 1: psutil EROFS (QEMU 에뮬레이션 오류)

```
OSError: [Errno 30] Read-only file system: '/proc/1/stat'
```

locust가 CPU/메모리 모니터링을 위해 `psutil.Process()` 초기화 시 `/proc/1/stat` 파일 읽기 시도.
QEMU x86_64 에뮬레이션 환경에서 `/proc` 접근이 EROFS(errno 30)로 실패.

**해결**: arm64 네이티브 이미지로 교체

```yaml
# 기존
image: us-central1-docker.pkg.dev/google-samples/microservices-demo/loadgenerator:v0.10.5
# (amd64 전용 → QEMU 에뮬레이션 → psutil 오류)

# 변경
image: locustio/locust:2.43.4
# (multi-arch, arm64 네이티브 → psutil 정상 동작)
```

locustfile.py는 원본 컨테이너에서 추출해 ConfigMap으로 마운트:
```bash
# locustfile 위치: /loadgen/locustfile.py
kubectl exec <loadgenerator-pod> -- cat /loadgen/locustfile.py
```

### 문제 2: checkout 422 (Luhn 체크섬 실패)

원본 locustfile은 `faker.credit_card_number(card_type="visa")`로 Luhn 알고리즘 유효 카드번호를 생성.
faker 라이브러리 없이 직접 구현한 임의 숫자는 Luhn 검증을 통과하지 못해 paymentservice에서 422 반환.

```python
# 잘못된 구현 (Luhn 검증 실패)
f"4{''.join([str(random.randint(0,9)) for _ in range(15)])}"

# 수정: 알려진 Luhn 유효 Visa 테스트 카드 사용
_visa_test_cards = ['4111111111111111', '4242424242424242', ...]
random.choice(_visa_test_cards)
```

---

## 설치 순서

```bash
# 1. Namespace
kubectl apply -f 01-namespace.yaml

# 2. Jaeger (트레이스 백엔드 + UI)
kubectl apply -f 02-jaeger.yaml

# 3. OTel Collector (트레이스 수집 → Jaeger 전달)
kubectl apply -f 03-otel-collector.yaml

# 4. Online Boutique (모든 서비스 + loadgenerator)
kubectl apply -f 04-online-boutique.yaml

# 상태 확인
kubectl get pods -n online-boutique
```

**주의**: arm64 클러스터에서 최초 배포 시 amd64 서비스들의 Pod Ready까지 2~5분 소요.
(QEMU 에뮬레이션 하에서 JVM/V8/Python 인터프리터 기동 시간)

---

## 현재 상태 및 접근 URL

### Pod 상태

| Pod | 이미지 아키텍처 | 상태 |
|-----|----------------|------|
| frontend | arm64 (Go) | 1/1 Running |
| checkoutservice | arm64 (Go) | 1/1 Running |
| productcatalogservice | arm64 (Go) | 1/1 Running |
| shippingservice | arm64 (Go) | 1/1 Running |
| redis-cart | arm64 | 1/1 Running |
| currencyservice | amd64/QEMU (Node.js) | 1/1 Running |
| paymentservice | amd64/QEMU (Node.js) | 1/1 Running |
| emailservice | amd64/QEMU (Python) | 1/1 Running |
| recommendationservice | amd64/QEMU (Python) | 1/1 Running |
| cartservice | amd64/QEMU (.NET) | 1/1 Running |
| adservice | amd64/QEMU (Java) | 1/1 Running |
| loadgenerator | arm64 (locustio/locust) | 1/1 Running |
| jaeger | arm64 | 1/1 Running |
| opentelemetrycollector | arm64 | 1/1 Running |

### 접근 URL

| 서비스 | URL |
|--------|-----|
| Online Boutique | **http://192.168.64.3:30080** |
| Jaeger UI | **http://192.168.64.3:32686** |

### Jaeger 트레이스 조회

1. `http://192.168.64.3:32686` 접속
2. **Service** 드롭다운에서 서비스 선택 (`frontend`, `checkoutservice` 등)
3. **Find Traces** → 트레이스 클릭 → Span 타임라인 확인

---

## 트러블슈팅 명령어 모음

```bash
# 전체 Pod 상태
kubectl get pods -n online-boutique -o wide

# Cilium 상태
kubectl get pods -n kube-system -l 'k8s-app in (cilium,cilium-envoy)' -o wide

# Cilium NodePort eBPF 맵 확인
kubectl exec -n kube-system $(kubectl get pod -n kube-system -l k8s-app=cilium \
  --field-selector=spec.nodeName=k8s-worker-1 -o name | head -1) \
  -- cilium bpf lb list | grep NodePort

# NodePort 연결 테스트
nc -z -w 3 192.168.64.3 30080 && echo OPEN || echo CLOSED
curl -s -o /dev/null -w "%{http_code}" http://192.168.64.3:30080

# QEMU binfmt 등록 확인 (worker-1)
kubectl debug node/k8s-worker-1 --image=busybox:1.36 -- \
  sh -c "ls /host/proc/sys/fs/binfmt_misc/ | grep qemu"

# OOMKilled 확인
kubectl get pod <pod-name> -n online-boutique -o json | python3 -c \
  "import sys,json; cs=json.load(sys.stdin)['status']['containerStatuses'][0]; \
   t=cs.get('lastState',{}).get('terminated',{}); \
   print('exitCode:', t.get('exitCode'), 'reason:', t.get('reason'))"

# worker-2 Cilium 임시 복구
kubectl delete pod -n kube-system \
  $(kubectl get pod -n kube-system -l k8s-app=cilium \
    --field-selector=spec.nodeName=k8s-worker-2 -o name | xargs -I{} basename {}) \
  $(kubectl get pod -n kube-system -l k8s-app=cilium-envoy \
    --field-selector=spec.nodeName=k8s-worker-2 -o name | xargs -I{} basename {})

# loadgenerator 트래픽 확인
kubectl logs -n online-boutique -l app=loadgenerator --tail=20
```

---

## 알려진 이슈

### worker-2 Cilium CrashLoopBackOff (지속 중)

**현상**: cilium-agent와 cilium-envoy가 순환 크래시

```
cilium-agent 시작 → xds.sock 생성 → cilium-envoy 연결
  → cilium-agent liveness probe 실패 (/healthz 미응답)
  → kubelet SIGTERM 전송 → cilium-agent 정상 종료 (xds.sock 삭제)
  → cilium-envoy: xds.sock 사라짐 → 크래시
  → 반복
```

**영향**: worker-2에 `node.cilium.io/agent-not-ready:NoSchedule` taint 자동 설정됨 → 앱 Pod는 자동으로 worker-1에만 스케줄링

**임시 복구** (위 명령어 참조): 두 Pod 동시 삭제 후 약 2~3분 정상 운전, 이후 재발

**근본 원인**: 미확인. cilium-agent `/healthz`가 왜 실패하는지 추가 조사 필요.

### QEMU binfmt 재부팅 시 초기화

worker-1 재부팅 시 amd64 서비스 6개 실행 불가 → `kubectl apply -f 00-qemu-binfmt.yaml` 실행 필요.

### adservice DeadlineExceeded

QEMU 하에서 Java(JVM) 응답이 느려 gRPC 데드라인 초과 발생. 프론트엔드에서 "failed to retrieve ads" 경고 로그 출력. 기능에는 영향 없음 (광고 미표시로 그래이스풀 처리됨).

---

## 파일 구조

```
online-boutique-otel/
├── 01-namespace.yaml          # Namespace: online-boutique
├── 02-jaeger.yaml             # Jaeger all-in-one + NodePort 32686
├── 03-otel-collector.yaml     # OTel Collector (수집 → Jaeger 전달)
├── 04-online-boutique.yaml    # Online Boutique + locustfile ConfigMap
│                              #   (OTel 환경변수, QEMU 설정, arm64 loadgenerator)
├── original-manifests.yaml    # 원본 Google 매니페스트 (수정 전)
├── WORK_LOG.md                # 문제 분석 및 수정 상세 기록
└── README.md                  # 이 문서
```
