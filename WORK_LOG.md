# Online Boutique + OTel + Jaeger 작업 기록

## 클러스터 환경

| 항목 | 내용 |
|------|------|
| Kubernetes | v1.29.15 |
| 노드 | master: 192.168.64.2 / worker-1: 192.168.64.3 / worker-2: 192.168.64.5 |
| OS | Ubuntu 22.04, **arm64 (aarch64)** — UTM on Mac |
| CNI | Cilium v1.19.1 (kube-proxy 없음, kube-proxy-replacement: true) |
| 접속 대상 | Mac 브라우저에서 http://192.168.64.3:<NodePort> |

---

## 파일 구조

```
online-boutique-otel/
├── 01-namespace.yaml           # Namespace: online-boutique
├── 02-jaeger.yaml              # Jaeger all-in-one v1.57 + NodePort 32686
├── 03-otel-collector.yaml      # OTel Collector (otel/opentelemetry-collector-contrib:0.99.0)
├── 04-online-boutique.yaml     # Online Boutique (OTel 환경변수, QEMU 설정 포함)
├── 05-otel-instrumentation.yaml # Instrumentation CR (OTel Operator용)
├── original-manifests.yaml     # 원본 Google 매니페스트 (수정 전)
├── README.md
└── WORK_LOG.md                 # 이 파일
```

> `00-qemu-binfmt.yaml` 삭제됨 (2026-05-23). binfmt 등록은 worker-1 커널에 유지 중.

---

## 아키텍처 결정 사항

### 이미지 아키텍처
Online Boutique v0.10.5 이미지:
- **arm64 native (multi-arch)**: frontend, checkoutservice, productcatalogservice, shippingservice (Go)
- **amd64 전용**: emailservice, recommendationservice, currencyservice, paymentservice, cartservice, adservice (Node.js/Python/.NET/Java)

arm64 노드에서 amd64 서비스 실행은 **QEMU binfmt 에뮬레이션**으로 가능.
단, worker-1에만 QEMU가 등록되어 있음 (아래 이슈 참조).

---

## 최종 배포 상태 (2026-05-25)

### online-boutique 네임스페이스

| Pod | 상태 | 노드 | OTel 트레이스 | 비고 |
|-----|------|------|--------------|------|
| frontend | 1/1 Running | worker-2 | ✅ Jaeger | |
| checkoutservice | 1/1 Running | worker-2 | ✅ Jaeger | |
| productcatalogservice | 1/1 Running | worker-2 | ✅ Jaeger | |
| currencyservice | 1/1 Running | worker-2 | ✅ Jaeger | amd64, QEMU |
| emailservice | 1/1 Running | worker-2 | ✅ Jaeger | amd64, QEMU |
| paymentservice | 1/1 Running | worker-2 | ✅ Jaeger | amd64, QEMU |
| recommendationservice | 1/1 Running | worker-2 | ✅ Jaeger | amd64, QEMU |
| cartservice | 1/1 Running | worker-2 | ❌ | .NET gRPC h2c 문제 (아래 참조) |
| adservice | 1/1 Running | worker-2 | ❌ | OpenCensus, QEMU JVM 10분+ (아래 참조) |
| shippingservice | 1/1 Running | worker-2 | ❌ | OpenCensus, QEMU eBPF 불가 (아래 참조) |
| redis-cart | 1/1 Running | worker-2 | N/A | |
| jaeger | 1/1 Running | worker-2 | N/A | |
| loadgenerator | 1/1 Running | worker-2 | N/A | |
| opentelemetrycollector | 1/1 Running | **worker-1** | N/A | |

### Jaeger 확인 서비스 목록 (http://192.168.64.3:32686)

```
checkoutservice, currencyservice, emailservice, frontend,
paymentservice, productcatalogservice, recommendationservice
```
총 **7개 서비스** 트레이스 확인됨.

### 접근 URL
- Online Boutique: **http://192.168.64.3:30080** ✅
- Jaeger UI: **http://192.168.64.3:32686** ✅

### 설치된 추가 컴포넌트

| 컴포넌트 | 버전 | 네임스페이스 |
|----------|------|-------------|
| cert-manager | v1.17.2 | cert-manager |
| opentelemetry-operator | 0.113.1 (app: 0.151.0) | opentelemetry-operator-system |

---

## 트레이스 미확인 서비스 — 원인 및 결론

### cartservice (.NET OTel SDK) — ❌ 포기

**근본 원인**: .NET gRPC cleartext (h2c) 문제
- `COLLECTOR_SERVICE_ADDR=opentelemetrycollector:4317` 환경변수를 코드에서 읽어 OTLP endpoint를 `http://...:4317`(gRPC 포트)로 하드코딩
- `OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`로 프로토콜을 바꿔도 endpoint 포트가 여전히 4317 → HTTP/Protobuf 요청을 gRPC 전용 포트로 전송해 실패
- `COLLECTOR_SERVICE_ADDR`을 4318로 바꿔도 QEMU 환경에서 .NET Kestrel 스레드풀 starvation → health probe timeout → 재시작 반복
- **결론**: 코드 수정(endpoint 설정 방식 변경) 없이는 해결 불가

**현재 설정**: `COLLECTOR_SERVICE_ADDR=opentelemetrycollector:4317` (원본 유지), OTLP env 추가 없음

---

### adservice (Java OpenCensus) — ❌ 포기

**근본 원인**: OpenCensus 라이브러리 + QEMU 환경
- adservice는 **OpenCensus** 라이브러리 사용 → OTLP exporter 없음 → OTel Collector에 직접 전송 불가
- OTel Operator Java auto-instrumentation으로 우회 시도: `JAVA_TOOL_OPTIONS=-javaagent:/otel-auto-instrumentation-java-server/javaagent.jar` 주입
- QEMU x86_64에서 JVM + Java agent 초기화가 **10분 이상** 소요 (probe initialDelaySeconds: 600 필요)
- 메모리 OOMKill (512Mi → 768Mi로 증가 필요)
- **결론**: 시간 대비 효용 낮음. Java agent annotation 제거 완료 (2026-05-25)

---

### shippingservice (Go OpenCensus) — ❌ 불가

**근본 원인**: OpenCensus + QEMU 이중 제한

1. **OpenCensus 라이브러리**: OTLP 미지원. OTel SDK로 마이그레이션하지 않으면 OTLP 전송 불가
2. **Go eBPF 불가**: shippingservice 이미지가 amd64 전용 → QEMU로 실행 중
   ```
   PID 7: exe="" cmd=/usr/libexec/qemu-binfmt/x86_64-binfmt-P /src/shippingservice /src/shippingservice
   ```
   - `/proc/{pid}/exe` readlink가 빈 값 반환 (컨테이너 procfs 제한)
   - eBPF uprobe가 QEMU 에뮬레이션 프로세스에 attach 불가
- **결론**: 소스코드 수정 (OpenCensus → OTel SDK 마이그레이션) 없이는 근본 해결 불가

---

## 2026-05-23 작업 내용

### 문제 1: NodePort가 동작하지 않음

**증상**
- `curl http://192.168.64.3:30080` → `Connection refused`
- `cilium bpf lb list`에 NodePort 항목 없음

**원인**
```
kube-proxy-replacement: false  ← eBPF NodePort 규칙 미생성
kube-proxy: 미설치             ← NodePort 처리 불가
```

**수정**
```bash
kubectl patch configmap cilium-config -n kube-system \
  --type merge -p '{"data":{"kube-proxy-replacement":"true"}}'
kubectl delete pod -n kube-system cilium-2tfq8 cilium-rx64h
```

**결과**: NodePort 30080, 32686 정상 작동.

---

### 문제 2: amd64 서비스 미배포 및 OOMKilled

**원인**: QEMU 에뮬레이션 오버헤드로 메모리 부족, probe 타이밍 부족

**수정 요약**:

| 서비스 | 주요 변경 |
|--------|-----------|
| currencyservice | initialDelaySeconds: 60, failureThreshold: 10, memory 256Mi |
| emailservice | initialDelaySeconds: 60, periodSeconds: 10, memory 256Mi |
| paymentservice | initialDelaySeconds: 60, failureThreshold: 10, memory 256Mi |
| cartservice | readOnlyRootFilesystem: false, /tmp emptyDir, DOTNET_GCConserveMemory=9, memory 256Mi |
| recommendationservice | initialDelaySeconds: 60, periodSeconds: 10, failureThreshold: 10 |
| adservice | initialDelaySeconds: 180, periodSeconds: 30, failureThreshold: 10, memory 512Mi |

---

### 문제 3: loadgenerator CrashLoopBackOff

**원인 1**: initContainer가 최대 12회 재시도 후 종료 → 무한 루프로 변경

**원인 2**: amd64 loadgenerator 이미지의 psutil이 QEMU /proc/1/stat EROFS 오류
- **수정**: arm64 네이티브 `locustio/locust:2.43.4`로 교체

**원인 3**: locustfile에서 Luhn 체크섬 미적용 카드번호 사용 → paymentservice 422
- **수정**: Visa 테스트 카드 목록으로 교체

---

### 문제 4: frontend 롤링 업데이트 데드락

**원인**: `kubectl patch`로 추가된 `hostPort: 30080`이 YAML에 없어서 롤링 업데이트 시 포트 충돌

**수정**:
```bash
kubectl patch deployment frontend -n online-boutique --type=json \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/ports",
        "value": [{"containerPort": 8080, "protocol": "TCP"}]}]'
```

---

### 문제 5: 서비스들이 Jaeger에 트레이스 미전송

**원인**: `ENABLE_TRACING=1` 환경변수 누락 + `OTEL_SERVICE_NAME` 미설정

**수정**: 모든 서비스에 두 환경변수 추가. productcatalogservice 바이너리명 `server` → `OTEL_SERVICE_NAME=productcatalogservice`로 올바른 이름 지정.

**결과**: 7개 서비스 Jaeger 정상 표시.

---

## 2026-05-24 작업 내용

### worker-2 Cilium 완전 수리

**근본 원인**: 클러스터 설치 시 worker-2에만 `br_netfilter` + `ip_forward` 영속 설정 누락. 재부팅마다 초기화되어 Cilium networking 실패.

**수리 과정 요약**:

| 단계 | 조치 |
|------|------|
| 1 | worker-2에서 stale tc BPF 프로그램 제거 (`tc qdisc del dev enp0s1 clsact` 등) |
| 2 | `clean-cilium-bpf-state: true` ConfigMap 제거 (무한 루프 차단) |
| 3 | Cilium pod 재시작 |
| 4 | 재발 → VM 재설치 결정 |

**VM 재설치 절차**: 01-namespace~04-online-boutique.yaml 적용 + br_netfilter 영속 설정 포함 재설치.

**결과**: worker-2 IP 변경 (192.168.64.4 → 192.168.64.5), Cilium 1/1 Running 안정 유지.

### pod 분산 배포

`kubectl rollout restart deployment -n online-boutique` 후 두 노드에 분산:
- worker-1: opentelemetrycollector 1개
- worker-2: 나머지 13개

### nodeSelector 정리

| 파일 | 제거된 nodeSelector |
|------|-------------------|
| 02-jaeger.yaml | `kubernetes.io/hostname: k8s-worker-1` + live hostPort:32686 |
| 03-otel-collector.yaml | `kubernetes.io/hostname: k8s-worker-1` |

### OTel Operator 설치

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true --version v1.17.2

helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
  --namespace opentelemetry-operator-system --create-namespace \
  --set "manager.collectorImage.repository=otel/opentelemetry-collector-k8s" \
  --set admissionWebhooks.certManager.enabled=true \
  --set "manager.extraArgs={--enable-go-instrumentation}"
```

adservice/cartservice/shippingservice 트레이스 확보를 목표로 설치했으나, 세 서비스 모두 QEMU 환경 제한 + 라이브러리 제약으로 포기. 설치 상태는 유지 (추후 활용 가능).

---

## 알려진 이슈 및 운영 주의사항

### [주의] worker-1 재부팅 시 QEMU binfmt 초기화

worker-1이 재부팅되면 amd64 서비스 6개가 모두 실행 불가 상태가 됨.
재부팅 후:
```bash
kubectl apply -f online-boutique-otel/00-qemu-binfmt.yaml
# initContainer 완료 후 (약 2-3분) amd64 서비스 자동 복구
```

### [중요] worker-2 Cilium crash 1순위 체크

worker-2 재부팅 후 Cilium이 비정상이면:
```bash
# worker-2 SSH에서
lsmod | grep br_netfilter        # 출력 없으면 미탑재
cat /proc/sys/net/ipv4/ip_forward  # 0이면 미설정
```
미설정 시:
```bash
modprobe br_netfilter
echo "br_netfilter" > /etc/modules-load.d/k8s.conf
echo "net.bridge.bridge-nf-call-iptables = 1" > /etc/sysctl.d/k8s.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" >> /etc/sysctl.d/k8s.conf
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.d/k8s.conf
sysctl --system
kubectl delete pod -n kube-system <cilium-pod> <cilium-envoy-pod>
```

---

## 주요 명령어 모음

```bash
# 클러스터 상태
kubectl get nodes -o wide
kubectl get pods -n online-boutique -o wide
kubectl get pods -n kube-system -l 'k8s-app in (cilium,cilium-envoy)' -o wide

# Jaeger 서비스 확인
curl -s http://192.168.64.3:32686/api/services | python3 -m json.tool

# OTel Operator 상태
kubectl get pods -n opentelemetry-operator-system
kubectl get instrumentation -n online-boutique
kubectl get pods -n cert-manager

# NodePort 접근 확인
nc -z -w 3 192.168.64.3 30080 && echo OPEN
nc -z -w 3 192.168.64.3 32686 && echo OPEN

# QEMU binfmt 등록 확인 (worker-1)
kubectl debug node/k8s-worker-1 --image=busybox:1.36 -- \
  sh -c "ls /host/proc/sys/fs/binfmt_misc/ | grep qemu"

# OOMKilled 여부 확인
kubectl get pod <pod> -o json | python3 -c \
  "import sys,json; cs=json.load(sys.stdin)['status']['containerStatuses'][0]; \
   t=cs.get('lastState',{}).get('terminated',{}); \
   print('exitCode:', t.get('exitCode'), 'reason:', t.get('reason'))"
```
