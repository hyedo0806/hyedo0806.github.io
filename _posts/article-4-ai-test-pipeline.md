# AI를 활용한 테스트 코드 작성 시리즈 — 4편

## AI와 함께하는 Operator 통합 테스트 파이프라인

> 1편에서 프롬프트를 설계하고, 2편에서 테스트를 작성하고, 3편에서 리뷰했습니다.
> 4편에서는 개발자의 시나리오 설계부터 PR까지, AI가 동반하는 실전 워크플로우를 구축합니다.

---

### 들어가며

Kubernetes Operator는 단일 함수 테스트만으로는 부족합니다. CR이 생성될 때 StatefulSet이 만들어지는지, Security CR이 변경될 때 Broker CR의 Pod가 재시작되는지, Validation 실패 시 Status에 올바른 Condition이 기록되는지 — 이런 **CR 간의 상호작용**을 검증하는 통합 테스트가 핵심입니다.

문제는 통합 테스트의 진입장벽이 높다는 것입니다. envtest 환경 설정, Ginkgo 스펙 구조, Eventually 패턴, 리소스 정리 — 이 모든 것을 알아야 첫 테스트를 작성할 수 있습니다.

이 글에서는 **개발자가 시나리오를 말로 설명하면, AI가 검증 가능한 테스트 코드로 변환하는 5단계 파이프라인**을 제안합니다. ActiveMQ Artemis Operator의 `activemqartemis_controller.go`를 실제 예시로 사용합니다.

---

### 파이프라인 전체 흐름

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Step 1  개발자가 시나리오를 말로 작성                          │
│  ────────────────────────────────────                       │
│  "Security CR을 삭제하면 Broker의 Pod가 재시작되어야 해"         │
│                          │                                  │
│                          ▼                                  │
│  Step 2  AI가 시나리오를 정리하여 컨펌 요청                      │
│  ────────────────────────────────────                       │
│  시나리오 매트릭스 + 검증 조건 + 사전 조건 정리                   │
│                          │                                  │
│                          ▼                                  │
│  Step 3  개발자가 시나리오 수정/컨펌 → 테스트 코드 작성 요청       │
│  ────────────────────────────────────                       │
│  "3번 시나리오 빼고, 이 조건 추가해서 코드 작성해줘"               │
│                          │                                  │
│                          ▼                                  │
│  Step 4  AI가 테스트 코드 작성 + 커버리지 자동 확인               │
│  ────────────────────────────────────                       │
│  Ginkgo 스펙 생성 → go test 실행 → 커버리지 리포트              │
│                          │                                  │
│                          ▼                                  │
│  Step 5  개발자의 PR 작성                                     │
│  ────────────────────────────────────                       │
│  최종 리뷰 → 커밋 → PR 생성                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### Step 1. 개발자가 시나리오를 말로 작성

개발자가 코드를 쓰는 것이 아니라, **검증하고 싶은 동작을 자연어로 기술**합니다. 핵심은 "무엇을 확인하고 싶은가"에 집중하는 것입니다.

#### 실전 예시: ActiveMQArtemis Reconcile 시나리오

개발자가 `activemqartemis_controller.go`의 Reconcile 함수를 테스트하고 싶다고 합시다. Reconcile 함수는 다음과 같은 흐름입니다:

```go
func (r *ActiveMQArtemisReconciler) Reconcile(ctx context.Context, request ctrl.Request) (ctrl.Result, error) {
    // 1. CR 조회 (NotFound면 정상 종료, 다른 에러면 에러 반환)
    // 2. Validation 수행 (실패하면 Process 스킵, Status만 갱신)
    // 3. Process: StatefulSet, Credential, Acceptor, Console 생성/갱신
    // 4. BrokerStatus 처리 (상태 불일치면 Requeue)
    // 5. CRStatus 업데이트 (실패하면 Requeue)
    // 6. ExtraMounts가 있으면 Requeue
}
```

개발자가 작성하는 시나리오 문서:

```markdown
## 내가 확인하고 싶은 것

### 시나리오 A: 기본 Reconcile 흐름
- Broker CR을 Size=2로 생성하면, Replicas=2인 StatefulSet이 만들어져야 한다
- 생성된 StatefulSet에는 headless Service도 함께 만들어져야 한다
- CR의 Status.DeploymentPlanSize가 2로 업데이트되어야 한다

### 시나리오 B: Validation 실패 시 동작
- BrokerProperties에 중복 키가 있으면 StatefulSet이 만들어지면 안 된다
- Status.Conditions에 Valid=False, Reason=DuplicateBrokerPropertiesKey가 기록되어야 한다
- 중복 키를 수정하고 CR을 업데이트하면, 그때 StatefulSet이 만들어져야 한다

### 시나리오 C: Security CR과 Broker CR의 상호작용
- Security CR을 생성하면 기존 Broker의 Pod가 재시작(reconcile 재요청)되어야 한다
- Security CR을 삭제하면 ConfigHandler가 제거되고 Broker가 다시 reconcile되어야 한다

### 시나리오 D: CR 삭제 시 동작
- Broker CR을 삭제하면 Reconcile이 NotFound로 정상 종료해야 한다
- StatefulSet과 관련 Service가 GC에 의해 정리되어야 한다

### 시나리오 E: ExtraMounts가 있을 때
- ConfigMap을 ExtraMounts로 지정하면 Requeue가 발생해야 한다
- 지정한 ConfigMap이 존재하지 않으면 Validation 실패해야 한다
```

**포인트:** 코드도 프레임워크 지식도 필요 없습니다. "무엇이 어떻게 되어야 하는가"만 적습니다.

---

### Step 2. AI가 시나리오를 정리하여 컨펌 요청

개발자의 시나리오를 AI에게 전달합니다. 이때 프로덕션 코드도 함께 제공합니다.

#### AI에게 보내는 프롬프트

```
역할: Kubernetes Operator 테스트 설계자

아래는 개발자가 자연어로 작성한 테스트 시나리오와,
테스트 대상인 ActiveMQArtemisReconciler의 코드입니다.

다음 작업을 수행해줘:

1. 각 시나리오를 정형화된 테스트 스펙으로 정리
   - 사전 조건 (Given)
   - 실행 동작 (When)
   - 검증 조건 (Then)
   - 정리 작업 (Cleanup)

2. 개발자가 놓친 시나리오가 있으면 추가 제안
   (코드의 분기를 분석하여 커버되지 않는 경로 식별)

3. 각 시나리오의 테스트 유형 분류
   - Unit (fake client) / Integration (envtest) / E2E (real cluster)

4. 시나리오 간 의존 관계가 있으면 표시

출력 형식: 개발자가 컨펌할 수 있는 체크리스트

---
[개발자의 시나리오 문서]
---
[activemqartemis_controller.go 코드]
---
[activemqartemissecurity_controller.go 코드]
```

#### AI가 정리한 시나리오 스펙

```markdown
## 테스트 시나리오 스펙 (AI 정리본)

개발자님의 시나리오를 분석하고 정형화했습니다.
코드 분기 분석 결과 2개의 누락 시나리오를 추가 제안합니다.
각 항목을 확인 후 ✅ 또는 수정 요청해주세요.

---

### A-1. 기본 CR 생성 → StatefulSet 생성 [Integration]

| 항목 | 내용 |
|------|------|
| **Given** | 빈 네임스페이스 |
| **When** | `ActiveMQArtemis` CR 생성 (Size=2) |
| **Then** | ① StatefulSet 생성, Replicas=2 |
|          | ② Headless Service 생성 (`{name}-hdls-svc`) |
|          | ③ Ping Service 생성 (`{name}-ping-svc`) |
|          | ④ CR Status.DeploymentPlanSize = 2 |
| **Cleanup** | CR 삭제, StatefulSet/Service GC 대기 |
| **비동기** | Eventually (timeout: 30s, interval: 500ms) |

### A-2. CR Size 변경 → StatefulSet 반영 [Integration]

| 항목 | 내용 |
|------|------|
| **Given** | Size=1인 ActiveMQArtemis CR이 배포된 상태 |
| **When** | CR의 Size를 1 → 3으로 업데이트 |
| **Then** | ① StatefulSet Replicas가 3으로 변경 |
|          | ② CR Status.DeploymentPlanSize = 3 |
| **Cleanup** | CR 삭제 |

---

### B-1. BrokerProperties 중복 키 → Validation 실패 [Integration]

| 항목 | 내용 |
|------|------|
| **Given** | 빈 네임스페이스 |
| **When** | BrokerProperties에 `["min=X", "min=Y"]`로 CR 생성 |
| **Then** | ① StatefulSet이 생성되지 않음 (Consistently 사용) |
|          | ② Status.Conditions: Valid=False |
|          | ③ Reason = `ValidConditionFailedDuplicateBrokerPropertiesKey` |
|          | ④ Message에 "min" 포함 |
| **Cleanup** | CR 삭제 |

### B-2. Validation 실패 → 수정 후 복구 [Integration]

| 항목 | 내용 |
|------|------|
| **Given** | B-1 상태 (Validation 실패한 CR) |
| **When** | BrokerProperties를 `["min=X", "max=Y"]`로 수정 |
| **Then** | ① StatefulSet 생성됨 |
|          | ② Status.Conditions: Valid=True |
| **Cleanup** | CR 삭제 |

---

### C-1. Security CR 생성 → Broker 재reconcile [Integration]

| 항목 | 내용 |
|------|------|
| **Given** | ActiveMQArtemis CR이 배포된 상태 |
| **When** | ActiveMQArtemisSecurity CR 생성 |
| **Then** | ① `AddBrokerConfigHandler` 호출됨 |
|          | ② Broker의 StatefulSet 재업데이트 (resourceVersion 변경) |
| **Cleanup** | Security CR 삭제 → Broker CR 삭제 |

### C-2. Security CR 삭제 → Handler 제거 [Integration]

| 항목 | 내용 |
|------|------|
| **Given** | Broker CR + Security CR이 배포된 상태 |
| **When** | Security CR 삭제 |
| **Then** | ① `RemoveBrokerConfigHandler` 호출됨 |
|          | ② `namespaceToConfigHandler`에서 제거 |
|          | ③ Broker의 StatefulSet 재업데이트 |
| **Cleanup** | Broker CR 삭제 |

---

### D-1. CR 삭제 → NotFound 정상 종료 [Unit]

| 항목 | 내용 |
|------|------|
| **Given** | Fake Client: Get → NotFound 반환 |
| **When** | Reconcile 호출 |
| **Then** | ① err == nil |
|          | ② result.Requeue == false |
| **Cleanup** | 없음 |

### D-2. CR 삭제 → 하위 리소스 GC [Integration]

| 항목 | 내용 |
|------|------|
| **Given** | StatefulSet, Service가 생성된 상태 |
| **When** | Broker CR 삭제 |
| **Then** | ① StatefulSet NotFound (OwnerReference에 의한 GC) |
|          | ② Service NotFound |
| **Cleanup** | 없음 |

---

### E-1. ExtraMounts ConfigMap 존재 → Requeue [Integration]

| 항목 | 내용 |
|------|------|
| **Given** | ConfigMap `my-config` 생성됨 |
| **When** | ExtraMounts.ConfigMaps에 `my-config` 지정하여 CR 생성 |
| **Then** | ① Reconcile 결과 RequeueAfter > 0 |
|          | ② StatefulSet은 정상 생성 |
| **Cleanup** | CR 삭제, ConfigMap 삭제 |

### E-2. ExtraMounts ConfigMap 미존재 → Validation 실패 [Integration]

| 항목 | 내용 |
|------|------|
| **Given** | ConfigMap `nonexistent-config`가 존재하지 않음 |
| **When** | ExtraMounts.ConfigMaps에 `nonexistent-config` 지정하여 CR 생성 |
| **Then** | ① StatefulSet 미생성 |
|          | ② Status.Conditions: Valid=False |
|          | ③ Reason = `ValidConditionMissingResourcesReason` |
| **Cleanup** | CR 삭제 |

---

### 🆕 AI 추가 제안 시나리오

### F-1. API 서버 에러 → 에러 전파 [Unit]
> 코드 분기: controller.go:163 `reqLogger.Error(err, "unable to retrieve")`

| 항목 | 내용 |
|------|------|
| **Given** | Fake Client: Get → InternalError 반환 |
| **When** | Reconcile 호출 |
| **Then** | ① err != nil (InternalError 전파) |
|          | ② result.Requeue == false |

### F-2. Status 업데이트 실패 → Requeue [Unit/Integration]
> 코드 분기: controller.go:184 `crStatusUpdateErr != nil`

| 항목 | 내용 |
|------|------|
| **Given** | CR은 존재하지만, Status 서브리소스 업데이트가 실패 |
| **When** | Reconcile 호출 |
| **Then** | ① requeueRequest = true |
|          | ② result.RequeueAfter > 0 |

### F-3. Acceptor 포트 중복 → Validation 실패 [Unit]
> 코드 분기: controller.go:322 `validateAcceptorPorts`

| 항목 | 내용 |
|------|------|
| **Given** | 두 Acceptor가 동일한 Port(5672) 사용 |
| **When** | validate 호출 |
| **Then** | ① valid = false |
|          | ② Reason = `ValidConditionFailedDuplicateAcceptorPort` |

---

### 시나리오 요약

| ID | 시나리오 | 유형 | 개발자 제안 | AI 추가 |
|----|---------|------|:---------:|:------:|
| A-1 | 기본 CR → StatefulSet | Integration | ✅ | |
| A-2 | Size 변경 반영 | Integration | ✅ | |
| B-1 | BrokerProperties 중복 | Integration | ✅ | |
| B-2 | Validation 실패 후 복구 | Integration | ✅ | |
| C-1 | Security CR → Broker 재reconcile | Integration | ✅ | |
| C-2 | Security CR 삭제 → Handler 제거 | Integration | ✅ | |
| D-1 | CR NotFound 정상 종료 | Unit | ✅ | |
| D-2 | CR 삭제 → GC | Integration | ✅ | |
| E-1 | ExtraMounts → Requeue | Integration | ✅ | |
| E-2 | ExtraMounts 미존재 → 실패 | Integration | ✅ | |
| F-1 | API 에러 전파 | Unit | | 🆕 |
| F-2 | Status 업데이트 실패 Requeue | Unit | | 🆕 |
| F-3 | Acceptor 포트 중복 | Unit | | 🆕 |

**확인 요청**: 위 13개 시나리오 중 수정/제거할 항목이 있으면 알려주세요.
```

---

### Step 3. 개발자가 시나리오 수정/컨펌 후 테스트 코드 작성 요청

개발자는 AI가 정리한 시나리오를 검토합니다. 이 단계가 **가장 중요**합니다. 개발자의 도메인 지식으로 AI의 제안을 걸러내는 과정입니다.

#### 개발자의 피드백 예시

```markdown
시나리오 리뷰 완료. 피드백:

1. A-1, A-2: OK
2. B-1, B-2: OK. B-2는 B-1에 이어서 하나의 It 블록에서 진행해도 좋겠다
3. C-1: OK. 단, "AddBrokerConfigHandler 호출됨"은 내부 구현이라
   StatefulSet의 annotation이나 resourceVersion 변경으로 검증하자
4. C-2: OK
5. D-1: OK
6. D-2: OK. 단, PVC는 GC 대상이 아니므로 PVC가 남아있는지도 확인
7. E-1, E-2: OK
8. F-1: OK 추가하자
9. F-2: 이건 빼자. Status 서브리소스 실패를 envtest에서 시뮬레이션하기 어려움
10. F-3: OK 추가하자

총 12개 시나리오로 확정.
Unit 테스트 먼저 작성하고, Integration 테스트 작성해줘.
```

#### 개발자의 코드 작성 요청 프롬프트

```
컨펌한 12개 시나리오에 대해 테스트 코드를 작성해줘.

프로젝트 컨텍스트:
- 언어: Go 1.21
- Unit 테스트: testing + testify/assert
- Integration 테스트: Ginkgo v2 + Gomega + envtest
- 기존 suite_test.go 사용 (BeforeSuite에서 envtest 환경 구성됨)
- k8sClient, ctx, testEnv는 전역 변수로 사용 가능
- BeforeEachSpec(), AfterEachSpec() 헬퍼 사용

규칙:
1. Unit 테스트 파일: activemqartemis_controller_pipeline_unit_test.go
2. Integration 테스트 파일: activemqartemis_controller_pipeline_test.go
3. 각 시나리오 ID를 Label로 사용 (예: Label("A-1"))
4. Eventually timeout=30s, interval=500ms
5. Consistently duration=10s (미생성 검증 시)
6. 리소스 이름은 "pipeline-test-" 접두사 사용
7. AfterEach에서 반드시 CR 삭제

대상 코드:
[activemqartemis_controller.go 전체]
[activemqartemissecurity_controller.go Reconcile 함수]
```

---

### Step 4. AI가 테스트 코드 작성 + 커버리지 자동 확인

AI가 생성하는 코드의 핵심 부분을 보여드립니다.

#### Unit 테스트 (D-1, F-1, F-3)

```go
// activemqartemis_controller_pipeline_unit_test.go
package controllers

import (
    "context"
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
    brokerv1beta1 "github.com/artemiscloud/activemq-artemis-operator/api/v1beta1"
    apierrors "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/api/meta"
    "k8s.io/apimachinery/pkg/runtime/schema"
    "k8s.io/apimachinery/pkg/types"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client/fake"
    "sigs.k8s.io/controller-runtime/pkg/client/interceptor"
    "github.com/go-logr/logr"
    "sigs.k8s.io/controller-runtime/pkg/log"
    "errors"
)

// ============================================================
// D-1: CR이 존재하지 않을 때 Reconcile은 에러 없이 종료해야 한다
// ============================================================
func TestReconcile_CRNotFound_ReturnsNilError(t *testing.T) {
    // Given: Get 호출 시 NotFound를 반환하는 Fake Client
    interceptorFuncs := interceptor.Funcs{
        Get: func(ctx context.Context, client client.WithWatch,
            key client.ObjectKey, obj client.Object,
            opts ...client.GetOption) error {
            return apierrors.NewNotFound(
                schema.GroupResource{Group: "broker.amq.io", Resource: "activemqartemises"},
                key.Name,
            )
        },
    }
    fakeClient := fake.NewClientBuilder().
        WithInterceptorFuncs(interceptorFuncs).Build()

    reconciler := &ActiveMQArtemisReconciler{
        Client: fakeClient,
        log:    logr.New(log.NullLogSink{}),
    }

    // When: Reconcile 호출
    result, err := reconciler.Reconcile(context.TODO(), ctrl.Request{
        NamespacedName: types.NamespacedName{
            Namespace: "test-ns",
            Name:      "deleted-broker",
        },
    })

    // Then: 에러 없이 종료, Requeue 없음
    assert.Nil(t, err, "NotFound는 에러가 아닌 정상 삭제 상황")
    assert.False(t, result.Requeue)
    assert.Equal(t, time.Duration(0), result.RequeueAfter)
}

// ============================================================
// F-1: API 서버 에러 시 에러가 그대로 전파되어야 한다
// ============================================================
func TestReconcile_APIServerError_PropagatesError(t *testing.T) {
    // Given: Get 호출 시 InternalError를 반환하는 Fake Client
    internalErr := apierrors.NewInternalError(errors.New("etcd timeout"))

    interceptorFuncs := interceptor.Funcs{
        Get: func(ctx context.Context, client client.WithWatch,
            key client.ObjectKey, obj client.Object,
            opts ...client.GetOption) error {
            return internalErr
        },
    }
    fakeClient := fake.NewClientBuilder().
        WithInterceptorFuncs(interceptorFuncs).Build()

    reconciler := &ActiveMQArtemisReconciler{
        Client: fakeClient,
        log:    logr.New(log.NullLogSink{}),
    }

    // When: Reconcile 호출
    result, err := reconciler.Reconcile(context.TODO(), ctrl.Request{
        NamespacedName: types.NamespacedName{
            Namespace: "test-ns",
            Name:      "error-broker",
        },
    })

    // Then: InternalError가 그대로 전파됨
    assert.Equal(t, internalErr, err, "API 에러는 호출자에게 전파되어야 한다")
    assert.False(t, result.Requeue)
}

// ============================================================
// F-3: 동일 포트를 사용하는 두 Acceptor → Validation 실패
// ============================================================
func TestValidate_DuplicateAcceptorPort_ReturnsFalse(t *testing.T) {
    // Given: 두 Acceptor가 포트 5672를 공유하는 CR
    duplicatePort := int32(5672)
    cr := &brokerv1beta1.ActiveMQArtemis{
        ObjectMeta: metav1.ObjectMeta{Name: "dup-port-broker", Namespace: "test"},
        Spec: brokerv1beta1.ActiveMQArtemisSpec{
            Acceptors: []brokerv1beta1.AcceptorType{
                {Name: "amqp", Port: duplicatePort},
                {Name: "core", Port: duplicatePort}, // 의도적 중복
            },
        },
    }

    namer := MakeNamers(cr)
    r := NewActiveMQArtemisReconciler(&NillCluster{}, ctrl.Log, false)
    ri := NewActiveMQArtemisReconcilerImpl(cr, r)

    // When: validate 호출
    valid, retry := ri.validate(cr, k8sClient, *namer)

    // Then: Validation 실패, 중복 포트 Reason 기록
    assert.False(t, valid, "중복 포트가 있으면 유효하지 않아야 한다")
    assert.False(t, retry)

    condition := meta.FindStatusCondition(
        cr.Status.Conditions, brokerv1beta1.ValidConditionType)
    assert.NotNil(t, condition)
    assert.Equal(t, metav1.ConditionFalse, condition.Status)
    assert.Equal(t,
        brokerv1beta1.ValidConditionFailedDuplicateAcceptorPort,
        condition.Reason,
    )
}
```

#### Integration 테스트 (A-1, B-1/B-2, C-1, D-2, E-2)

```go
// activemqartemis_controller_pipeline_test.go
package controllers

import (
    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    brokerv1beta1 "github.com/artemiscloud/activemq-artemis-operator/api/v1beta1"
    "github.com/artemiscloud/activemq-artemis-operator/pkg/utils/common"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/api/meta"
    "k8s.io/apimachinery/pkg/types"
    "time"
)

const (
    pipelineTimeout  = 30 * time.Second
    pipelineInterval = 500 * time.Millisecond
    consistDuration  = 10 * time.Second
)

var _ = Describe("Reconcile Pipeline Tests", func() {

    BeforeEach(func() {
        BeforeEachSpec()
    })

    AfterEach(func() {
        AfterEachSpec()
    })

    // ================================================================
    // A-1: 기본 CR 생성 → StatefulSet + Service 생성
    // ================================================================
    Context("기본 Reconcile 흐름", Label("A-1"), func() {
        It("Size=2 CR 생성 시 StatefulSet과 Service가 만들어져야 한다", func() {

            brokerName := "pipeline-test-basic"
            cr := &brokerv1beta1.ActiveMQArtemis{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      brokerName,
                    Namespace: defaultNamespace,
                },
                Spec: brokerv1beta1.ActiveMQArtemisSpec{
                    DeploymentPlan: brokerv1beta1.DeploymentPlanType{
                        Size:  common.Int32ToPtr(2),
                        Image: "placeholder",
                    },
                },
            }

            // When: CR 생성
            Expect(k8sClient.Create(ctx, cr)).Should(Succeed())

            // Then: StatefulSet 생성, Replicas=2
            ssKey := types.NamespacedName{
                Name:      brokerName + "-ss",
                Namespace: defaultNamespace,
            }
            Eventually(func(g Gomega) {
                ss := &appsv1.StatefulSet{}
                g.Expect(k8sClient.Get(ctx, ssKey, ss)).To(Succeed())
                g.Expect(*ss.Spec.Replicas).To(Equal(int32(2)))
            }, pipelineTimeout, pipelineInterval).Should(Succeed())

            // Then: Headless Service 생성
            hdlsSvcKey := types.NamespacedName{
                Name:      brokerName + "-hdls-svc",
                Namespace: defaultNamespace,
            }
            Eventually(func(g Gomega) {
                svc := &corev1.Service{}
                g.Expect(k8sClient.Get(ctx, hdlsSvcKey, svc)).To(Succeed())
                g.Expect(svc.Spec.ClusterIP).To(Equal("None"))
            }, pipelineTimeout, pipelineInterval).Should(Succeed())

            // Cleanup
            Expect(k8sClient.Delete(ctx, cr)).Should(Succeed())
        })
    })

    // ================================================================
    // B-1 + B-2: Validation 실패 → 수정 후 복구
    // ================================================================
    Context("BrokerProperties Validation", Label("B-1", "B-2"), func() {
        It("중복 키 → 실패 → 수정 → StatefulSet 생성", func() {

            brokerName := "pipeline-test-validation"

            // B-1: 중복 키가 있는 CR 생성
            cr := &brokerv1beta1.ActiveMQArtemis{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      brokerName,
                    Namespace: defaultNamespace,
                },
                Spec: brokerv1beta1.ActiveMQArtemisSpec{
                    DeploymentPlan: brokerv1beta1.DeploymentPlanType{
                        Size:  common.Int32ToPtr(1),
                        Image: "placeholder",
                    },
                    BrokerProperties: []string{"min=X", "min=Y"}, // 중복!
                },
            }
            Expect(k8sClient.Create(ctx, cr)).Should(Succeed())

            // Then: Valid=False 상태
            crKey := types.NamespacedName{Name: brokerName, Namespace: defaultNamespace}
            Eventually(func(g Gomega) {
                current := &brokerv1beta1.ActiveMQArtemis{}
                g.Expect(k8sClient.Get(ctx, crKey, current)).To(Succeed())
                condition := meta.FindStatusCondition(
                    current.Status.Conditions,
                    brokerv1beta1.ValidConditionType,
                )
                g.Expect(condition).NotTo(BeNil())
                g.Expect(condition.Status).To(Equal(metav1.ConditionFalse))
                g.Expect(condition.Reason).To(Equal(
                    brokerv1beta1.ValidConditionFailedDuplicateBrokerPropertiesKey,
                ))
            }, pipelineTimeout, pipelineInterval).Should(Succeed())

            // Then: StatefulSet이 만들어지면 안 됨
            ssKey := types.NamespacedName{
                Name:      brokerName + "-ss",
                Namespace: defaultNamespace,
            }
            Consistently(func(g Gomega) {
                ss := &appsv1.StatefulSet{}
                err := k8sClient.Get(ctx, ssKey, ss)
                g.Expect(errors.IsNotFound(err)).To(BeTrue())
            }, consistDuration, pipelineInterval).Should(Succeed())

            // B-2: 중복 키 수정
            current := &brokerv1beta1.ActiveMQArtemis{}
            Expect(k8sClient.Get(ctx, crKey, current)).To(Succeed())
            current.Spec.BrokerProperties = []string{"min=X", "max=Y"} // 수정!
            Expect(k8sClient.Update(ctx, current)).To(Succeed())

            // Then: StatefulSet 생성됨
            Eventually(func(g Gomega) {
                ss := &appsv1.StatefulSet{}
                g.Expect(k8sClient.Get(ctx, ssKey, ss)).To(Succeed())
            }, pipelineTimeout, pipelineInterval).Should(Succeed())

            // Then: Valid=True로 복구
            Eventually(func(g Gomega) {
                updated := &brokerv1beta1.ActiveMQArtemis{}
                g.Expect(k8sClient.Get(ctx, crKey, updated)).To(Succeed())
                g.Expect(meta.IsStatusConditionTrue(
                    updated.Status.Conditions,
                    brokerv1beta1.ValidConditionType,
                )).To(BeTrue())
            }, pipelineTimeout, pipelineInterval).Should(Succeed())

            // Cleanup
            Expect(k8sClient.Delete(ctx, cr)).Should(Succeed())
        })
    })

    // ================================================================
    // D-2: CR 삭제 → OwnerReference에 의한 하위 리소스 GC
    // ================================================================
    Context("CR 삭제 시 리소스 정리", Label("D-2"), func() {
        It("Broker CR 삭제 후 StatefulSet은 GC되고 PVC는 남아야 한다", func() {

            brokerName := "pipeline-test-gc"
            cr := &brokerv1beta1.ActiveMQArtemis{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      brokerName,
                    Namespace: defaultNamespace,
                },
                Spec: brokerv1beta1.ActiveMQArtemisSpec{
                    DeploymentPlan: brokerv1beta1.DeploymentPlanType{
                        Size:  common.Int32ToPtr(1),
                        Image: "placeholder",
                    },
                },
            }
            Expect(k8sClient.Create(ctx, cr)).Should(Succeed())

            // StatefulSet 생성 대기
            ssKey := types.NamespacedName{
                Name:      brokerName + "-ss",
                Namespace: defaultNamespace,
            }
            Eventually(func(g Gomega) {
                ss := &appsv1.StatefulSet{}
                g.Expect(k8sClient.Get(ctx, ssKey, ss)).To(Succeed())
            }, pipelineTimeout, pipelineInterval).Should(Succeed())

            // When: CR 삭제
            Expect(k8sClient.Delete(ctx, cr)).Should(Succeed())

            // Then: StatefulSet이 GC에 의해 삭제됨
            Eventually(func(g Gomega) {
                ss := &appsv1.StatefulSet{}
                err := k8sClient.Get(ctx, ssKey, ss)
                g.Expect(errors.IsNotFound(err)).To(BeTrue())
            }, pipelineTimeout, pipelineInterval).Should(Succeed())
        })
    })

    // ================================================================
    // E-2: 존재하지 않는 ExtraMounts ConfigMap → Validation 실패
    // ================================================================
    Context("ExtraMounts Validation", Label("E-2"), func() {
        It("미존재 ConfigMap 참조 시 Valid=False", func() {

            brokerName := "pipeline-test-extramounts"
            cr := &brokerv1beta1.ActiveMQArtemis{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      brokerName,
                    Namespace: defaultNamespace,
                },
                Spec: brokerv1beta1.ActiveMQArtemisSpec{
                    DeploymentPlan: brokerv1beta1.DeploymentPlanType{
                        Size:  common.Int32ToPtr(1),
                        Image: "placeholder",
                        ExtraMounts: brokerv1beta1.ExtraMountsType{
                            ConfigMaps: []string{"nonexistent-config"},
                        },
                    },
                },
            }
            Expect(k8sClient.Create(ctx, cr)).Should(Succeed())

            crKey := types.NamespacedName{Name: brokerName, Namespace: defaultNamespace}
            Eventually(func(g Gomega) {
                current := &brokerv1beta1.ActiveMQArtemis{}
                g.Expect(k8sClient.Get(ctx, crKey, current)).To(Succeed())
                condition := meta.FindStatusCondition(
                    current.Status.Conditions,
                    brokerv1beta1.ValidConditionType,
                )
                g.Expect(condition).NotTo(BeNil())
                g.Expect(condition.Status).To(Equal(metav1.ConditionFalse))
                g.Expect(condition.Reason).To(Equal(
                    brokerv1beta1.ValidConditionMissingResourcesReason,
                ))
            }, pipelineTimeout, pipelineInterval).Should(Succeed())

            Expect(k8sClient.Delete(ctx, cr)).Should(Succeed())
        })
    })
})
```

#### 커버리지 자동 확인

AI가 테스트 코드를 작성한 직후, 커버리지를 자동으로 측정하고 리포트합니다.

```bash
# AI가 자동 실행하는 커버리지 스크립트
KUBEBUILDER_ASSETS="$(./bin/setup-envtest use 1.26.0 --bin-dir ./bin -p path)" \
go test ./controllers/ \
  -run "TestReconcile_CRNotFound|TestReconcile_APIServer|TestValidate_Duplicate|Pipeline" \
  -v -coverprofile=pipeline-cover.out -p 1

# 커버리지 결과 출력
go tool cover -func=pipeline-cover.out | grep "activemqartemis_controller.go"
```

AI가 보고하는 커버리지 리포트:

```markdown
## 커버리지 리포트

### activemqartemis_controller.go 커버리지 변화

| 함수 | 변경 전 | 변경 후 | 변화 |
|------|--------|--------|------|
| Reconcile | 65.2% | 82.4% | +17.2% |
| validate | 71.0% | 88.5% | +17.5% |
| validateAcceptorPorts | 0% | 100% | +100% |
| validateExtraMounts | 45.3% | 78.9% | +33.6% |
| DuplicateKeyIn | 100% | 100% | - |
| hasExtraMounts | 50.0% | 100% | +50.0% |
| UpdateCRStatus | 40.0% | 40.0% | - |
| **전체** | **58.3%** | **78.6%** | **+20.3%** |

### 여전히 커버되지 않는 분기
- `UpdateCRStatus`: Status 서브리소스 업데이트 실패 경로 (F-2, 제외 결정됨)
- `validateSSLEnabledSecrets`: SSL Secret 관련 검증 (이번 스코프 외)
- `SetupWithManager`: Manager 설정 (테스트 불필요)
```

---

### Step 5. 개발자의 PR 작성

개발자는 AI가 생성한 테스트를 최종 리뷰하고 PR을 생성합니다.

#### 최종 체크리스트

```markdown
## PR 전 최종 확인

- [x] 모든 테스트 통과: `go test ./controllers/ -v -p 1`
- [x] 커버리지 78.6% (기준 75% 이상 충족)
- [x] AI 생성 코드 리뷰 완료
  - [x] Eventually/Consistently timeout 적정성 확인
  - [x] Cleanup 누락 없음
  - [x] 테스트 이름이 동작을 설명함
  - [x] 하드코딩된 값 없음
- [x] 기존 테스트와 충돌 없음
- [x] Label로 개별 실행 가능 확인
```

#### PR 설명

```markdown
## Add Reconcile Pipeline Tests

### 배경
ActiveMQArtemisReconciler의 핵심 경로에 대한 통합 테스트가 부족하여,
CR 간 상호작용과 Validation 실패/복구 시나리오를 커버하는 테스트를 추가합니다.

### 변경 내용
- `activemqartemis_controller_pipeline_unit_test.go`: Unit 테스트 3건
  - D-1: CR NotFound 정상 종료
  - F-1: API 서버 에러 전파
  - F-3: Acceptor 포트 중복 Validation
- `activemqartemis_controller_pipeline_test.go`: Integration 테스트 5건
  - A-1: 기본 CR → StatefulSet/Service 생성
  - B-1/B-2: Validation 실패 → 수정 후 복구
  - D-2: CR 삭제 → OwnerReference GC
  - E-2: ExtraMounts 미존재 → Validation 실패

### 커버리지
- activemqartemis_controller.go: 58.3% → 78.6% (+20.3%)

### 테스트 방법
```bash
# Unit 테스트만
go test ./controllers/ -run "TestReconcile_|TestValidate_Duplicate" -v

# Integration 테스트만
make test TEST_EXTRA_ARGS="-ginkgo.label-filter='A-1 || B-1 || B-2 || D-2 || E-2'"

# 전체
make test
```
```

---

### 파이프라인을 팀에 정착시키기

#### 단계별 정착 전략

```
Month 1: 리더가 먼저 시범
  - 리더가 이 파이프라인으로 3개 PR 작성
  - 팀에 경험 공유, 시행착오 문서화

Month 2: 팀원 1-2명 시범
  - 새 기능 개발 시 이 파이프라인 적용
  - Step 1의 시나리오 문서를 팀 위키에 축적

Month 3: 팀 전체 도입
  - PR 템플릿에 시나리오 문서 섹션 추가
  - 시나리오 리뷰를 코드 리뷰의 일부로 포함
```

#### 이 파이프라인이 효과적인 이유

| 기존 방식 | 이 파이프라인 |
|----------|------------|
| 개발자가 코드와 테스트를 모두 작성 | 개발자는 시나리오 설계에 집중 |
| 테스트 구현에 대한 진입장벽 높음 | 자연어로 시나리오만 기술하면 됨 |
| 누락 시나리오를 리뷰에서 발견 | AI가 Step 2에서 코드 분기 기반 제안 |
| 커버리지는 나중에 확인 | Step 4에서 자동으로 측정/리포트 |
| 시나리오 설계와 구현이 뒤섞임 | 5단계로 명확히 분리 |

---

### 전체 시리즈 정리

```
1편: 프롬프트 엔지니어링 기초
     → AI에게 효과적으로 요청하는 법

2편: 테스트 원칙 + 프롬프트 = 좋은 테스트
     → FIRST, AAA 원칙을 프롬프트에 녹여 의미 있는 테스트 생성

3편: AI로 테스트 PR 리뷰하기
     → 분기 매핑, 안티패턴 탐지, 누락 시나리오 식별

4편: AI와 함께하는 통합 테스트 파이프라인
     → 시나리오 설계 → 정형화 → 컨펌 → 코드 생성 → PR
```

이 시리즈의 핵심 메시지는 일관됩니다: **AI는 "테스트를 대신 짜주는 도구"가 아니라 "개발자의 테스트 설계를 실행 가능한 코드로 변환하는 동료"입니다.** 시나리오 설계는 개발자의 도메인 지식이 필요하고, 코드 변환은 AI가 빠르게 처리하고, 최종 판단은 다시 개발자가 합니다.

이 역할 분담이 맞아떨어질 때, 테스트의 양과 질이 동시에 올라갑니다.
