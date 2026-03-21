# Kubernetes Operator 테스트 코드 작성 가이드

> ActiveMQ Artemis Operator 프로젝트를 기반으로 한 실습 세미나 자료

---

## 목차

1. [테스트 코드란 무엇인가](#1-테스트-코드란-무엇인가)
2. [좋은 테스트 코드 작성하기](#2-좋은-테스트-코드-작성하기)
3. [K8s Operator 테스트 코드의 종류](#3-k8s-operator-테스트-코드의-종류)
4. [테스트 환경 설정](#4-테스트-환경-설정)
5. [테스트 커버리지 측정](#5-테스트-커버리지-측정)
6. [실습 과제](#6-실습-과제)

---

## 1. 테스트 코드란 무엇인가

### 1.1 정의

테스트 코드는 작성한 프로덕션 코드가 의도한 대로 동작하는지 자동으로 검증하는 코드입니다. 수동으로 매번 확인하는 대신, 코드 변경 시 기존 기능이 깨지지 않았는지 빠르게 확인할 수 있습니다.

### 1.2 왜 Operator에 테스트가 중요한가

Kubernetes Operator는 클러스터의 상태를 관리하는 컨트롤러입니다. 잘못된 Reconcile 로직은 리소스 누수, 무한 재시도, 클러스터 장애로 이어질 수 있습니다. 따라서 Operator는 다른 어떤 소프트웨어보다 테스트가 중요합니다.

- Reconcile 루프가 올바른 상태로 수렴하는지 검증
- CRD 변경 시 기존 리소스에 대한 회귀(regression) 방지
- Edge case (리소스 미존재, 권한 부족, 네트워크 오류) 대응 확인
- 멱등성(idempotency) 보장: 같은 Reconcile을 여러 번 실행해도 동일한 결과

### 1.3 테스트 피라미드

```
        ╱╲
       ╱  ╲        E2E 테스트 (실제 클러스터)
      ╱────╲       - 느림, 비용 높음
     ╱      ╲      - 전체 플로우 검증
    ╱────────╲
   ╱          ╲    Integration 테스트 (envtest)
  ╱────────────╲   - 중간 속도
 ╱              ╲  - API Server와의 상호작용 검증
╱────────────────╲
╲                ╱ Unit 테스트
 ╲──────────────╱  - 빠름, 의존성 없음
                   - 개별 함수/로직 검증
```

---

## 2. 좋은 테스트 코드 작성하기

### 2.1 테스트의 FIRST 원칙

| 원칙 | 설명 | Operator 예시 |
|------|------|---------------|
| **F**ast | 빠르게 실행되어야 함 | Unit 테스트는 외부 의존성 없이 실행 |
| **I**ndependent | 테스트 간 독립적 | 각 테스트에서 CR을 새로 생성 |
| **R**epeatable | 동일 결과 보장 | 네임스페이스 격리, 테스트 후 정리 |
| **S**elf-validating | 결과를 자동 판단 | assert/Expect로 명확한 검증 |
| **T**imely | 적시에 작성 | 기능 구현과 함께 테스트 작성 |

### 2.2 Arrange-Act-Assert (AAA) 패턴

모든 테스트는 세 단계로 구성합니다.

```go
func TestValidateBrokerPropsDuplicate(t *testing.T) {

    // ===== Arrange: 테스트 데이터 준비 =====
    cr := &brokerv1beta1.ActiveMQArtemis{
        Spec: brokerv1beta1.ActiveMQArtemisSpec{
            BrokerProperties: []string{
                "min=X",
                "min=y",  // 의도적 중복 키
            },
        },
    }
    namer := MakeNamers(cr)
    r := NewActiveMQArtemisReconciler(&NillCluster{}, ctrl.Log, false)
    ri := NewActiveMQArtemisReconcilerImpl(cr, r)

    // ===== Act: 테스트 대상 함수 실행 =====
    valid, retry := ri.validate(cr, k8sClient, *namer)

    // ===== Assert: 결과 검증 =====
    assert.False(t, valid)
    assert.False(t, retry)
    condition := meta.FindStatusCondition(cr.Status.Conditions, brokerv1beta1.ValidConditionType)
    assert.Equal(t, condition.Reason, brokerv1beta1.ValidConditionFailedDuplicateBrokerPropertiesKey)
}
```

### 2.3 테스트 네이밍 컨벤션

좋은 테스트 이름은 그 자체로 문서 역할을 합니다.

```go
// Go 표준 테스트: Test + 대상함수 + 시나리오
func TestValidate(t *testing.T)                              // 기본 케이스
func TestValidateBrokerPropsDuplicate(t *testing.T)          // 특정 시나리오
func TestDeleteAddressWithNotFoundError(t *testing.T)        // 에러 케이스
func TestDeleteAddressWithInternalError(t *testing.T)        // 다른 에러 케이스

// Ginkgo BDD 스타일: Describe + Context + It
var _ = Describe("artemis controller", func() {
    Context("tls secret reuse", Label("tls-secret-reuse"), func() {
        It("console and acceptor share one secret", func() {
            // ...
        })
    })
})
```

### 2.4 테스트에서 피해야 할 것

- **매직 넘버**: `assert.Equal(t, 3, count)` 대신 상수 정의 사용
- **테스트 간 상태 공유**: 전역 변수에 의존하지 말 것
- **과도한 Mocking**: 꼭 필요한 부분만 Mock 처리
- **Sleep 기반 동기화**: `time.Sleep` 대신 `Eventually`/`Consistently` 사용
- **하나의 테스트에 여러 시나리오**: 테스트 하나당 하나의 동작 검증

---

## 3. K8s Operator 테스트 코드의 종류

### 3.1 Unit 테스트

외부 의존성 없이 순수 로직만 검증합니다. K8s API Server가 필요하지 않습니다.

**대상**: Validation 로직, 유틸리티 함수, 상태 비교 함수, 에러 핸들링

**프레임워크**: Go 표준 `testing` + `testify/assert`

#### 예시: Validation 로직 Unit 테스트

프로젝트의 `activemqartemis_controller_unit_test.go` 참고:

```go
package controllers

import (
    "testing"
    brokerv1beta1 "github.com/artemiscloud/activemq-artemis-operator/api/v1beta1"
    "github.com/stretchr/testify/assert"
    "k8s.io/apimachinery/pkg/api/meta"
)

func TestValidateReservedLabels(t *testing.T) {
    // Reserved label을 사용하면 validation 실패해야 함
    cr := &brokerv1beta1.ActiveMQArtemis{
        Spec: brokerv1beta1.ActiveMQArtemisSpec{
            ResourceTemplates: []brokerv1beta1.ResourceTemplate{
                {
                    Labels: map[string]string{selectors.LabelAppKey: "myAppKey"},
                },
            },
        },
    }

    namer := MakeNamers(cr)
    r := NewActiveMQArtemisReconciler(&NillCluster{}, ctrl.Log, false)
    ri := NewActiveMQArtemisReconcilerImpl(cr, r)

    valid, retry := ri.validate(cr, k8sClient, *namer)

    assert.False(t, valid)
    assert.False(t, retry)
    assert.True(t, meta.IsStatusConditionFalse(
        cr.Status.Conditions, brokerv1beta1.ValidConditionType))
}
```

#### 예시: Fake Client를 활용한 Reconcile Unit 테스트

프로젝트의 `activemqartemisaddress_controller_unit_test.go` 참고:

```go
func TestDeleteAddressWithNotFoundError(t *testing.T) {
    // Fake Client 설정 - Get 호출 시 항상 NotFound 반환
    interceptorFuncs := interceptor.Funcs{
        Get: func(ctx context.Context, client client.WithWatch,
            key client.ObjectKey, obj client.Object,
            opts ...client.GetOption) error {
            return apierrors.NewNotFound(schema.GroupResource{}, "")
        },
    }
    fakeClient := fake.NewClientBuilder().
        WithInterceptorFuncs(interceptorFuncs).Build()

    // Reconciler 생성 (Fake Client 주입)
    r := NewActiveMQArtemisAddressReconciler(
        fakeClient, nil, logr.New(log.NullLogSink{}))

    // Reconcile 실행
    result, err := r.Reconcile(context.TODO(),
        ctrl.Request{NamespacedName: types.NamespacedName{
            Namespace: "test-namespace", Name: "test-name"}})

    // NotFound는 에러가 아님 (정상 삭제 상황)
    assert.Nil(t, err)
    assert.False(t, result.Requeue)
    assert.Equal(t, time.Duration(0), result.RequeueAfter)
}

func TestDeleteAddressWithInternalError(t *testing.T) {
    // Internal 에러 시 에러 반환 검증
    internalError := apierrors.NewInternalError(errors.New("internal-error"))

    interceptorFuncs := interceptor.Funcs{
        Get: func(ctx context.Context, client client.WithWatch,
            key client.ObjectKey, obj client.Object,
            opts ...client.GetOption) error {
            return internalError
        },
    }
    fakeClient := fake.NewClientBuilder().
        WithInterceptorFuncs(interceptorFuncs).Build()

    r := NewActiveMQArtemisAddressReconciler(
        fakeClient, nil, logr.New(log.NullLogSink{}))

    result, err := r.Reconcile(context.TODO(),
        ctrl.Request{NamespacedName: types.NamespacedName{
            Namespace: "test-namespace", Name: "test-name"}})

    assert.Equal(t, internalError, err)  // 에러가 전파되어야 함
    assert.False(t, result.Requeue)
}
```

**핵심 포인트:**
- `fake.NewClientBuilder()`: K8s API 호출을 Fake로 대체
- `interceptor.Funcs`: Get/List/Create 등 특정 동작을 커스터마이즈
- `logr.New(log.NullLogSink{})`: 테스트 시 로그 무시

### 3.2 Integration 테스트 (envtest 기반)

`envtest`는 실제 etcd + API Server를 로컬에 띄워서 CRD, RBAC, Webhook 등을 검증합니다. 실제 클러스터 없이도 Reconcile 루프 전체를 테스트할 수 있습니다.

**대상**: Reconcile 전체 흐름, CR 생성/수정/삭제 시 리소스 생성 검증, Status 업데이트, 소유권(OwnerReference) 검증

**프레임워크**: Ginkgo + Gomega + envtest

#### envtest 동작 구조

```
┌──────────────────────────────────────────┐
│           테스트 프로세스                   │
│                                          │
│  ┌──────────┐    ┌───────────────────┐   │
│  │ Ginkgo   │───▶│  Controller       │   │
│  │ Test     │    │  Manager          │   │
│  │ Specs    │    │  ├─ Reconciler    │   │
│  └──────────┘    │  ├─ Scheme        │   │
│       │          │  └─ Client        │   │
│       ▼          └────────┬──────────┘   │
│  ┌──────────┐             │              │
│  │ k8sClient│─────────────┤              │
│  └──────────┘             ▼              │
│              ┌────────────────────┐      │
│              │  envtest           │      │
│              │  ├─ etcd           │      │
│              │  └─ kube-apiserver │      │
│              └────────────────────┘      │
└──────────────────────────────────────────┘
```

#### Suite 설정 (suite_test.go)

프로젝트의 `controllers/suite_test.go`를 참고한 핵심 구조:

```go
package controllers

import (
    "path/filepath"
    "testing"
    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    "sigs.k8s.io/controller-runtime/pkg/envtest"
)

var (
    k8sClient  client.Client
    testEnv    *envtest.Environment
    ctx        context.Context
    cancel     context.CancelFunc
)

func TestAPIs(t *testing.T) {
    RegisterFailHandler(Fail)
    RunSpecs(t, "Controller Suite")
}

var _ = BeforeSuite(func() {
    ctx, cancel = context.WithCancel(context.TODO())

    // 1) envtest 환경 구성
    testEnv = &envtest.Environment{
        CRDDirectoryPaths:     []string{
            filepath.Join("..", "config", "crd", "bases"),
        },
        ErrorIfCRDPathMissing: true,
    }

    // 2) API Server 시작
    restConfig, err := testEnv.Start()
    Expect(err).NotTo(HaveOccurred())

    // 3) Scheme 등록 (모든 API 버전)
    err = brokerv1beta1.AddToScheme(scheme.Scheme)
    Expect(err).NotTo(HaveOccurred())

    // 4) Client 생성
    k8sClient, err = client.New(restConfig, client.Options{
        Scheme: scheme.Scheme,
    })
    Expect(err).NotTo(HaveOccurred())

    // 5) Controller Manager 시작
    k8Manager, err := ctrl.NewManager(restConfig, ctrl.Options{
        Scheme: scheme.Scheme,
    })
    Expect(err).ToNot(HaveOccurred())

    // 6) Reconciler 등록
    brokerReconciler := NewActiveMQArtemisReconciler(
        k8Manager, ctrl.Log, false)
    err = brokerReconciler.SetupWithManager(k8Manager)
    Expect(err).ToNot(HaveOccurred())

    // 7) Manager를 goroutine에서 시작
    go func() {
        defer GinkgoRecover()
        err = k8Manager.Start(ctx)
        Expect(err).ToNot(HaveOccurred())
    }()
})

var _ = AfterSuite(func() {
    cancel()
    err := testEnv.Stop()
    Expect(err).NotTo(HaveOccurred())
})
```

#### Integration 테스트 스펙 작성

```go
var _ = Describe("artemis controller", func() {

    BeforeEach(func() {
        // 각 테스트 전 초기화
    })

    AfterEach(func() {
        // 리소스 정리
    })

    Context("CR 생성 시", func() {
        It("StatefulSet이 생성되어야 한다", func() {
            // 1) CR 생성
            cr := &brokerv1beta1.ActiveMQArtemis{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      "test-broker",
                    Namespace: "test",
                },
                Spec: brokerv1beta1.ActiveMQArtemisSpec{
                    DeploymentPlan: brokerv1beta1.DeploymentPlanType{
                        Size: common.Int32ToPtr(1),
                    },
                },
            }
            Expect(k8sClient.Create(ctx, cr)).Should(Succeed())

            // 2) Eventually로 비동기 결과 확인
            ssKey := types.NamespacedName{
                Name:      cr.Name + "-ss",
                Namespace: "test",
            }
            createdSS := &appsv1.StatefulSet{}

            Eventually(func(g Gomega) {
                g.Expect(k8sClient.Get(ctx, ssKey, createdSS)).
                    Should(Succeed())
                g.Expect(*createdSS.Spec.Replicas).
                    Should(Equal(int32(1)))
            }, timeout, interval).Should(Succeed())

            // 3) 정리
            Expect(k8sClient.Delete(ctx, cr)).Should(Succeed())
        })
    })
})
```

**핵심 패턴:**
- `Eventually`: 비동기 조건이 만족될 때까지 반복 확인 (Reconcile은 비동기)
- `Consistently`: 일정 시간 동안 조건이 유지되는지 확인
- `timeout`, `interval`: 전역 상수로 정의하여 일관성 유지

```go
const (
    timeout  = time.Second * 30
    interval = time.Millisecond * 500
)
```

### 3.3 E2E 테스트 (실제 클러스터 기반)

실제 K8s 클러스터(minikube, kind, OpenShift)에서 Operator를 배포하고 전체 플로우를 검증합니다.

**대상**: 실제 Pod 생성 및 Health Check, Ingress/Route 연결, TLS 설정, Scale Up/Down, 장애 복구

**프레임워크**: 동일 (Ginkgo + Gomega), 환경 변수로 모드 전환

#### 환경 변수에 따른 테스트 모드 전환

이 프로젝트는 환경 변수로 테스트 모드를 전환하는 패턴을 사용합니다:

```go
// suite_test.go의 BeforeSuite에서
if os.Getenv("DEPLOY_OPERATOR") == "true" {
    // E2E 모드: 실제 클러스터에 Operator 배포
    setUpRealOperator()
} else {
    // Integration 모드: envtest 사용
    setUpEnvTest()
}
```

#### Label을 통한 테스트 분류

```go
// envtest에서만 실행 (Label 없음 또는 !do)
Context("basic validation", func() {
    It("validates broker properties", func() { /* ... */ })
})

// 실제 클러스터에서만 실행 (Label: "do")
Context("deployed operator test", Label("do"), func() {
    It("creates actual pods", func() {
        if os.Getenv("USE_EXISTING_CLUSTER") != "true" {
            Skip("requires real cluster")
        }
        // ...
    })
})

// 느린 테스트 (Label: "slow") - CI에서 제외 가능
Context("HA failover", Label("do", "slow"), func() {
    It("survives node failure", func() { /* ... */ })
})
```

### 3.4 테스트 종류 비교표

| 구분 | Unit 테스트 | Integration 테스트 | E2E 테스트 |
|------|------------|-------------------|-----------|
| **속도** | 매우 빠름 (ms) | 보통 (초~분) | 느림 (분~십분) |
| **API Server** | 불필요 | envtest (로컬) | 실제 클러스터 |
| **프레임워크** | testing + testify | Ginkgo + envtest | Ginkgo + 실제 클러스터 |
| **파일 접미사** | `_unit_test.go` | `_test.go` | `_deploy_operator_test.go` |
| **Makefile 타겟** | `make test` | `make test` | `make test-mk-do` |
| **주요 검증 대상** | 순수 로직 | Reconcile 흐름 | 전체 시스템 |
| **Fake/Mock** | fake.Client | envtest API Server | 실제 K8s |

---

## 4. 테스트 환경 설정

### 4.1 사전 요구 사항

```bash
# Go 설치 (go.mod에 정의된 버전과 일치)
go version  # go 1.21 이상

# Ginkgo CLI 설치
go install github.com/onsi/ginkgo/v2/ginkgo@latest
```

### 4.2 프로젝트 의존성 (go.mod)

```
# 테스트 프레임워크
github.com/onsi/ginkgo/v2 v2.13.0    # BDD 스타일 테스트 프레임워크
github.com/onsi/gomega v1.28.1        # Matcher/Assertion 라이브러리
github.com/stretchr/testify v1.8.4    # 추가 Assertion 도구

# K8s 테스트 인프라
sigs.k8s.io/controller-runtime v0.16.3       # envtest 포함
controller-runtime/tools/setup-envtest       # envtest 바이너리 관리
```

### 4.3 Makefile 테스트 타겟

```makefile
# ============================================================
# 1) Unit + Integration 테스트 (envtest, 로컬 실행)
# ============================================================
make test
# 내부 동작:
#   KUBEBUILDER_ASSETS="..." RECONCILE_RESYNC_PERIOD=5s \
#   go test ./... -p 1

# Verbose 모드
make test-v

# ============================================================
# 2) E2E 테스트 - 로컬 Operator (minikube)
# ============================================================
make test-mk
# 내부 동작:
#   USE_EXISTING_CLUSTER=true ENABLE_WEBHOOKS=false \
#   go test ./... -p 1 -test.timeout=120m \
#   -ginkgo.label-filter='!do'

# ============================================================
# 3) E2E 테스트 - 배포된 Operator (minikube)
# ============================================================
make test-mk-do
# 내부 동작:
#   DEPLOY_OPERATOR=true USE_EXISTING_CLUSTER=true \
#   go test ./... -p 1 -test.timeout=60m \
#   -ginkgo.label-filter='do'

# ============================================================
# 4) 빠른 CI 테스트 (느린 테스트 제외)
# ============================================================
make test-mk-do-fast
#   -ginkgo.label-filter='do && !slow' -test.timeout=30m
```

### 4.4 envtest 바이너리 설정

```bash
# setup-envtest 도구로 K8s 바이너리 다운로드
make envtest

# 설치된 바이너리 경로 확인
$(ENVTEST) use 1.26.0 --bin-dir ./bin -p path
# 출력: ./bin/k8s/1.26.0-linux-amd64

# 환경 변수 설정 (IDE에서 직접 실행 시)
export KUBEBUILDER_ASSETS="./bin/k8s/1.26.0-linux-amd64"
```

### 4.5 주요 환경 변수

| 변수 | 값 | 설명 |
|------|------|------|
| `KUBEBUILDER_ASSETS` | 바이너리 경로 | envtest 바이너리 위치 |
| `USE_EXISTING_CLUSTER` | `true`/`false` | 실제 클러스터 사용 여부 |
| `DEPLOY_OPERATOR` | `true`/`false` | Operator를 클러스터에 배포할지 여부 |
| `ENABLE_WEBHOOKS` | `true`/`false` | Webhook 테스트 활성화 |
| `RECONCILE_RESYNC_PERIOD` | `5s` | Reconcile 재동기화 주기 |
| `OPERATOR_NAMESPACE` | `test` | Operator 네임스페이스 |
| `TEST_VERBOSE` | `true`/`false` | 상세 로그 출력 |

### 4.6 IDE 설정 (GoLand / VS Code)

#### VS Code settings.json

```json
{
  "go.testEnvVars": {
    "KUBEBUILDER_ASSETS": "${workspaceFolder}/bin/k8s/1.26.0-linux-amd64"
  },
  "go.testTimeout": "600s",
  "go.testFlags": ["-v", "-count=1"]
}
```

#### GoLand Run Configuration

- Working Directory: `<project-root>/controllers`
- Environment: `KUBEBUILDER_ASSETS=<project-root>/bin/k8s/1.26.0-linux-amd64`

---

## 5. 테스트 커버리지 측정

### 5.1 커버리지 생성

```bash
# 기본 커버리지 리포트 생성
go test ./... -coverprofile=cover.out -p 1

# envtest 포함 커버리지
make test TEST_EXTRA_ARGS="-coverprofile=cover.out"

# minikube 테스트 커버리지 (자동 생성: cover-mk.out)
make test-mk
```

### 5.2 커버리지 확인

```bash
# 터미널에서 요약 확인
go tool cover -func=cover.out

# 출력 예시:
# controllers/activemqartemis_controller.go:56:    GetBrokerConfigHandler    75.0%
# controllers/activemqartemis_controller.go:150:   Reconcile                 82.3%
# controllers/activemqartemis_controller.go:204:   validate                  91.0%
# total:                                           (statements)              78.5%

# HTML 리포트 생성 (브라우저에서 확인)
go tool cover -html=cover.out -o coverage_report.html
```

### 5.3 커버리지 해석

HTML 리포트에서 색상으로 확인:
- **초록색**: 테스트에 의해 실행된 코드
- **빨간색**: 실행되지 않은 코드 (커버리지 갭)
- **회색**: 실행 불가능한 코드 (import, 타입 선언 등)

### 5.4 커버리지 가이드라인

| 영역 | 권장 커버리지 | 이유 |
|------|-------------|------|
| Reconcile 메인 로직 | 80% 이상 | 핵심 비즈니스 로직 |
| Validation 함수 | 90% 이상 | 경계값/에러 케이스 중요 |
| 유틸리티 함수 | 85% 이상 | 재사용되므로 신뢰성 중요 |
| 에러 핸들링 | 70% 이상 | 에러 경로도 테스트 |
| 전체 프로젝트 | 75% 이상 | 일반적인 건강한 수준 |

### 5.5 패키지별 커버리지

```bash
# 특정 패키지만 측정
go test -coverprofile=cover.out ./controllers/...
go test -coverprofile=cover.out ./pkg/utils/...

# 모든 패키지의 커버리지 요약
go test ./... -coverprofile=cover.out -p 1 && \
go tool cover -func=cover.out | grep total
```

---

## 6. 실습 과제

### 실습 1: Unit 테스트 작성 (기초)

**목표**: `activemqartemis_controller.go`의 순수 함수에 대한 Unit 테스트 작성

**대상 함수**: `DuplicateKeyIn`, `hasExtraMounts`, `EqualCRStatus`

```go
// 파일: controllers/my_unit_test.go
package controllers

import (
    "testing"
    "github.com/stretchr/testify/assert"
    brokerv1beta1 "github.com/artemiscloud/activemq-artemis-operator/api/v1beta1"
)

// 과제 1-1: DuplicateKeyIn 테스트
func TestDuplicateKeyIn_NoDuplicates(t *testing.T) {
    result := DuplicateKeyIn([]string{"a=1", "b=2", "c=3"})
    assert.Equal(t, "", result) // 중복 없으면 빈 문자열
}

func TestDuplicateKeyIn_WithDuplicate(t *testing.T) {
    // TODO: "a=1", "b=2", "a=3" 입력 시 "a" 반환 검증
}

func TestDuplicateKeyIn_EmptyInput(t *testing.T) {
    // TODO: 빈 슬라이스 입력 시 동작 검증
}

// 과제 1-2: hasExtraMounts 테스트
func TestHasExtraMounts_NilCR(t *testing.T) {
    // TODO: nil CR 입력 시 false 반환 검증
}

func TestHasExtraMounts_WithConfigMaps(t *testing.T) {
    // TODO: ConfigMaps가 있을 때 true 반환 검증
}

func TestHasExtraMounts_WithSecrets(t *testing.T) {
    // TODO: Secrets가 있을 때 true 반환 검증
}

func TestHasExtraMounts_Empty(t *testing.T) {
    // TODO: ExtraMounts가 비어있을 때 false 반환 검증
}
```

**실행**:
```bash
go test -v -run "TestDuplicateKeyIn|TestHasExtraMounts" ./controllers/
```

### 실습 2: Fake Client를 활용한 Reconcile 테스트 (중급)

**목표**: Fake Client와 interceptor를 사용하여 Reconcile 동작 검증

```go
// 파일: controllers/my_reconcile_unit_test.go
package controllers

import (
    "context"
    "testing"
    "github.com/stretchr/testify/assert"
    "sigs.k8s.io/controller-runtime/pkg/client/fake"
    "sigs.k8s.io/controller-runtime/pkg/client/interceptor"
    ctrl "sigs.k8s.io/controller-runtime"
)

// 과제 2-1: CR이 존재하지 않을 때 Reconcile 동작 검증
func TestReconcile_CRNotFound(t *testing.T) {
    // TODO:
    // 1) interceptor.Funcs의 Get에서 NotFound 에러 반환
    // 2) fake.NewClientBuilder()로 Fake Client 생성
    // 3) Reconcile 호출
    // 4) err가 nil이고 Requeue가 false인지 검증
}

// 과제 2-2: CR이 존재할 때 정상 Reconcile 검증
func TestReconcile_CRExists(t *testing.T) {
    // TODO:
    // 1) Get에서 유효한 ActiveMQArtemis CR 반환
    // 2) Reconcile 호출
    // 3) 결과 검증
}
```

### 실습 3: envtest Integration 테스트 (고급)

**목표**: envtest를 사용한 전체 Reconcile 흐름 테스트

```go
// 파일: controllers/my_integration_test.go
package controllers

import (
    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    brokerv1beta1 "github.com/artemiscloud/activemq-artemis-operator/api/v1beta1"
    "github.com/artemiscloud/activemq-artemis-operator/pkg/utils/common"
    appsv1 "k8s.io/api/apps/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"
)

var _ = Describe("Reconcile Integration", func() {

    BeforeEach(func() {
        BeforeEachSpec()
    })

    AfterEach(func() {
        AfterEachSpec()
    })

    // 과제 3-1: CR 생성 시 StatefulSet 생성 확인
    Context("CR 생성", func() {
        It("기본 CR 생성 시 StatefulSet이 만들어져야 한다", func() {
            // TODO:
            // 1) ActiveMQArtemis CR 생성
            // 2) Eventually로 StatefulSet 존재 확인
            // 3) Replicas 수 검증
            // 4) CR 삭제 후 정리
        })
    })

    // 과제 3-2: CR 업데이트 시 StatefulSet 반영 확인
    Context("CR 업데이트", func() {
        It("Size 변경 시 StatefulSet Replicas가 업데이트되어야 한다", func() {
            // TODO:
            // 1) Size=1로 CR 생성
            // 2) StatefulSet 생성 확인
            // 3) CR의 Size를 2로 업데이트
            // 4) Eventually로 StatefulSet Replicas=2 확인
            // 5) 정리
        })
    })

    // 과제 3-3: Validation 실패 시 Status Condition 확인
    Context("Validation", func() {
        It("중복 BrokerProperties가 있으면 Valid=False가 되어야 한다", func() {
            // TODO:
            // 1) 중복 키가 있는 BrokerProperties로 CR 생성
            // 2) Eventually로 Status.Conditions에서
            //    ValidConditionType이 False인지 확인
            // 3) Reason이 ValidConditionFailedDuplicateBrokerPropertiesKey인지 확인
            // 4) 정리
        })
    })
})
```

**실행**:
```bash
# envtest 바이너리 준비
make envtest

# Integration 테스트 실행
KUBEBUILDER_ASSETS="$(./bin/setup-envtest use 1.26.0 --bin-dir ./bin -p path)" \
go test -v -run "Reconcile Integration" ./controllers/
```

### 실습 4: 커버리지 분석 (분석)

```bash
# 과제 4-1: 현재 프로젝트 커버리지 측정
make test TEST_EXTRA_ARGS="-coverprofile=cover.out"
go tool cover -func=cover.out | head -30

# 과제 4-2: HTML 리포트 생성 후 커버리지 갭 분석
go tool cover -html=cover.out -o coverage_report.html

# 과제 4-3: 특정 파일의 커버리지 확인
go tool cover -func=cover.out | grep "activemqartemis_controller.go"

# 과제 4-4: 커버되지 않은 코드 경로 식별 후 테스트 추가
# coverage_report.html에서 빨간색 부분을 확인하고
# 해당 코드 경로를 커버하는 테스트 작성
```

---

## 부록: 프로젝트 테스트 파일 구조

```
activemq-artemis-operator/
├── controllers/
│   ├── suite_test.go                              # 테스트 Suite 설정 (envtest)
│   ├── activemqartemis_controller.go              # 메인 컨트롤러 (테스트 대상)
│   ├── activemqartemis_controller_unit_test.go     # Unit 테스트 (testify)
│   ├── activemqartemis_controller_test.go          # Integration 테스트 (Ginkgo)
│   ├── activemqartemis_controller2_test.go         # 추가 Integration 테스트
│   ├── activemqartemis_controller_deploy_operator_test.go  # E2E 테스트
│   ├── activemqartemis_reconciler.go              # Reconciler 구현
│   ├── activemqartemis_reconciler_test.go         # Reconciler 테스트
│   └── ...
├── pkg/
│   ├── utils/common/common_test.go                # 유틸리티 Unit 테스트
│   ├── resources/statefulsets/statefulset_test.go  # StatefulSet 헬퍼 테스트
│   └── ...
├── api/v1beta1/
│   └── webhook_suite_test.go                      # Webhook 테스트
├── Makefile                                       # 테스트 실행 타겟
└── go.mod                                         # 테스트 의존성
```

---

## 부록: 자주 쓰는 Gomega Matcher

```go
// 기본
Expect(err).NotTo(HaveOccurred())
Expect(result).To(BeNil())
Expect(count).To(Equal(3))
Expect(name).To(BeEmpty())

// 비동기 (Reconcile 결과 대기)
Eventually(func(g Gomega) {
    obj := &appsv1.StatefulSet{}
    g.Expect(k8sClient.Get(ctx, key, obj)).To(Succeed())
    g.Expect(*obj.Spec.Replicas).To(Equal(int32(2)))
}, timeout, interval).Should(Succeed())

// 조건이 유지되는지 확인 (의도적으로 변하지 않아야 할 때)
Consistently(func(g Gomega) {
    obj := &appsv1.StatefulSet{}
    g.Expect(k8sClient.Get(ctx, key, obj)).To(Succeed())
    g.Expect(*obj.Spec.Replicas).To(Equal(int32(1)))
}, duration, interval).Should(Succeed())

// 컬렉션
Expect(items).To(HaveLen(3))
Expect(items).To(ContainElement("value"))
Expect(labels).To(HaveKey("app"))
Expect(labels).To(HaveKeyWithValue("app", "broker"))
```

---

## 부록: 트러블슈팅

| 문제 | 원인 | 해결 |
|------|------|------|
| `KUBEBUILDER_ASSETS not set` | envtest 바이너리 미설정 | `make envtest` 후 환경변수 설정 |
| `timeout waiting for` | Reconcile이 완료되지 않음 | timeout 증가 또는 Reconcile 로직 확인 |
| `resource already exists` | 이전 테스트 정리 안됨 | AfterEach에서 리소스 삭제 추가 |
| `no kind registered` | Scheme에 API 미등록 | `AddToScheme` 호출 확인 |
| `connection refused` | API Server 미시작 | BeforeSuite에서 testEnv.Start() 확인 |
| `CRD not found` | CRD 경로 오류 | `CRDDirectoryPaths` 경로 확인 |
