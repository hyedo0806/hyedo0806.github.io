# AI를 활용한 테스트 코드 작성 시리즈 — 2편

## 테스트 원칙을 지키는 AI 프롬프트로 테스트 코드 작성하기

> 1편에서 다룬 프롬프트 엔지니어링 기법을 실전에 적용합니다.
> AI에게 "그냥 테스트 만들어줘"라고 하면 쓸모없는 테스트가 나옵니다.
> 좋은 테스트를 얻으려면, 좋은 테스트가 뭔지 AI에게 알려줘야 합니다.

---

### 들어가며

1편에서 우리는 AI에게 효과적으로 요청하는 프롬프트 작성법을 배웠습니다. 이번 2편에서는 그 기법을 테스트 코드 작성에 직접 적용합니다. 핵심 질문은 하나입니다: **AI가 생성한 테스트 코드가 '진짜 좋은 테스트'인지 어떻게 판단하고, 어떻게 유도할 것인가?**

AI는 컴파일되고 통과하는 테스트를 매우 빠르게 만들어냅니다. 하지만 "통과하는 테스트"와 "의미 있는 테스트"는 다릅니다. 이 글에서는 테스트 원칙을 프롬프트에 녹여서, AI가 처음부터 의미 있는 테스트를 생성하도록 유도하는 방법을 다룹니다.

---

### 1. AI가 흔히 만드는 "나쁜 테스트"의 패턴

AI에게 아무 맥락 없이 테스트를 요청하면, 다음과 같은 문제가 반복됩니다.

#### 1.1 Happy Path만 테스트하는 문제

```
프롬프트: "이 함수에 대한 테스트를 작성해줘"
```

AI는 정상 입력 → 정상 출력만 검증하는 테스트를 만듭니다. 경계값, nil 입력, 빈 슬라이스, 에러 경로는 빠집니다.

```go
// AI가 흔히 만드는 테스트 — happy path만 검증
func TestCalculateTotal(t *testing.T) {
    result := CalculateTotal([]int{10, 20, 30})
    assert.Equal(t, 60, result)  // 이것만으로는 부족하다
}
```

#### 1.2 구현을 그대로 복사하는 문제

AI가 프로덕션 코드의 로직을 테스트 안에 그대로 재현하는 경우입니다. 이런 테스트는 구현이 바뀌면 함께 깨지고, 실제 버그를 잡아내지 못합니다.

```go
// 나쁜 예: 구현 로직을 테스트에 복사
func TestCalculateDiscount(t *testing.T) {
    price := 100.0
    discount := 0.15
    // 프로덕션 코드와 동일한 계산을 테스트에서 반복
    expected := price * (1 - discount)
    assert.Equal(t, expected, CalculateDiscount(price, discount))
}
```

#### 1.3 의미 없는 단언(assertion)

`assert.NotNil(t, result)` 같은 약한 단언만 있고, 실제 값이 올바른지는 검증하지 않는 테스트입니다.

---

### 2. 테스트 원칙을 프롬프트에 녹이기

#### 2.1 FIRST 원칙을 명시적으로 요구하기

```
프롬프트:
다음 함수에 대한 테스트를 작성해줘.
테스트는 FIRST 원칙을 따라야 해:
- Fast: 외부 의존성 없이 밀리초 내에 실행
- Independent: 각 테스트가 독립적으로 실행 가능
- Repeatable: 실행할 때마다 동일한 결과
- Self-validating: assert로 명확하게 성공/실패 판단
- Timely: 경계값과 에러 케이스를 포함

[함수 코드 붙여넣기]
```

#### 2.2 테스트 시나리오를 구조화하여 요청하기

가장 효과적인 방법은 **시나리오 매트릭스**를 함께 제공하는 것입니다.

```
프롬프트:
다음 함수에 대해 아래 시나리오별로 테스트를 작성해줘.
각 테스트는 AAA(Arrange-Act-Assert) 패턴을 따라줘.

| 시나리오          | 입력                    | 기대 결과              |
|------------------|------------------------|----------------------|
| 정상 케이스        | ["a=1", "b=2"]         | 빈 문자열 (중복 없음)   |
| 중복 키 존재       | ["a=1", "b=2", "a=3"]  | "a" 반환              |
| 빈 입력           | []                     | 빈 문자열              |
| 값 없는 항목       | ["a=1", "novalue"]     | 빈 문자열              |
| 키만 같고 값 다름   | ["x=1", "x=2"]         | "x" 반환              |

함수 코드:
[코드 붙여넣기]
```

이렇게 하면 AI는 각 시나리오를 독립된 테스트 함수로 만들며, 누락 없이 커버합니다.

#### 2.3 네거티브 테스트를 명시적으로 요청하기

```
프롬프트:
이 Reconcile 함수에 대한 테스트를 작성해줘.
반드시 다음 에러 시나리오를 포함해야 해:
1. CR이 존재하지 않을 때 (NotFound) → err == nil, Requeue == false
2. API 서버 연결 실패 시 (InternalError) → err 전파
3. Status 업데이트 실패 시 → Requeue == true
4. 유효성 검증 실패 시 → Status.Condition에 에러 기록

각 에러 시나리오는 별도의 테스트 함수로 분리해줘.
```

---

### 3. 프롬프트 템플릿: 실전 레시피

#### 3.1 Unit 테스트 프롬프트 템플릿

```
역할: 너는 Go 테스트 전문가야.

컨텍스트:
- 프레임워크: testing + testify/assert
- 패턴: AAA (Arrange-Act-Assert)
- 파일명 규칙: {대상}_unit_test.go

요청:
아래 함수에 대한 Unit 테스트를 작성해줘.

조건:
1. 정상 케이스, 경계값 케이스, 에러 케이스를 각각 별도 함수로 작성
2. 테스트 함수명은 Test{함수명}_{시나리오} 형식
3. 매직 넘버 금지 — 기대값은 변수나 상수로 정의
4. 외부 의존성(DB, API, 파일 I/O)이 있으면 mock/fake 사용
5. 각 테스트에 // 주석으로 "무엇을 검증하는지" 한 줄 설명

함수 코드:
```

#### 3.2 Integration 테스트 프롬프트 템플릿

```
역할: 너는 Kubernetes controller-runtime 테스트 전문가야.

컨텍스트:
- 프레임워크: Ginkgo v2 + Gomega + envtest
- 패턴: Describe/Context/It (BDD)
- 비동기 검증: Eventually/Consistently 사용 필수
- timeout: 30초, interval: 500ms

요청:
아래 Reconciler에 대한 Integration 테스트를 작성해줘.

조건:
1. BeforeEach에서 테스트용 CR 생성, AfterEach에서 삭제
2. Eventually로 비동기 상태 변화 검증 (Reconcile은 비동기)
3. 하나의 It 블록에 하나의 동작만 검증
4. Context로 시나리오를 논리적으로 그룹핑
5. 에러 경로도 포함 (CR 삭제 후 Reconcile, 잘못된 Spec 등)

Reconciler 코드:
```

#### 3.3 "이 테스트를 개선해줘" 프롬프트

이미 있는 테스트의 품질을 높일 때 사용합니다.

```
아래 테스트 코드를 리뷰하고 개선해줘.

체크리스트:
- [ ] 경계값 테스트가 있는가?
- [ ] 에러 경로를 테스트하는가?
- [ ] 테스트 이름이 동작을 설명하는가?
- [ ] 하나의 테스트가 하나의 동작만 검증하는가?
- [ ] time.Sleep 대신 Eventually를 사용하는가?
- [ ] 하드코딩된 값이 상수로 정의되어 있는가?
- [ ] Mock/Fake가 과도하지 않은가?

부족한 부분이 있다면 개선된 버전을 작성해줘.
누락된 시나리오가 있다면 추가해줘.

기존 테스트 코드:
```

---

### 4. 실전 예시: Operator Validation 테스트 생성

실제로 프롬프트를 적용하여 AI에게 테스트를 요청하는 과정을 보겠습니다.

#### 4.1 대상 코드

```go
func DuplicateKeyIn(keyValues []string) string {
    keysMap := map[string]string{}
    for _, keyAndValue := range keyValues {
        if key, _, found := strings.Cut(keyAndValue, "="); found {
            _, duplicate := keysMap[key]
            if !duplicate {
                keysMap[key] = key
            } else {
                return key
            }
        }
    }
    return ""
}
```

#### 4.2 프롬프트

```
역할: Go 테스트 전문가

아래 DuplicateKeyIn 함수에 대한 Unit 테스트를 작성해줘.

시나리오 매트릭스:
| 시나리오              | 입력                          | 기대 결과 |
|----------------------|------------------------------|----------|
| 중복 없음             | ["a=1", "b=2", "c=3"]       | ""       |
| 첫 번째 키가 중복      | ["a=1", "a=2"]              | "a"      |
| 마지막 키가 중복       | ["a=1", "b=2", "b=3"]       | "b"      |
| 빈 슬라이스           | []                           | ""       |
| = 없는 항목           | ["noequals"]                 | ""       |
| 빈 문자열 키          | ["=value1", "=value2"]       | ""       |
| 공백 포함 키          | [" a =1", " a =2"]          | " a "    |

조건:
- testing + testify/assert 사용
- 테스트명: TestDuplicateKeyIn_{시나리오}
- 각 테스트에 검증 목적 주석 추가
```

#### 4.3 AI 생성 결과 (예상)

```go
func TestDuplicateKeyIn_NoDuplicates(t *testing.T) {
    // 모든 키가 고유할 때 빈 문자열을 반환해야 한다
    result := DuplicateKeyIn([]string{"a=1", "b=2", "c=3"})
    assert.Equal(t, "", result)
}

func TestDuplicateKeyIn_FirstKeyDuplicated(t *testing.T) {
    // 첫 번째 키가 중복될 때 해당 키를 반환해야 한다
    result := DuplicateKeyIn([]string{"a=1", "a=2"})
    assert.Equal(t, "a", result)
}

func TestDuplicateKeyIn_EmptySlice(t *testing.T) {
    // 빈 슬라이스 입력 시 빈 문자열을 반환해야 한다
    result := DuplicateKeyIn([]string{})
    assert.Equal(t, "", result)
}

func TestDuplicateKeyIn_NoEqualsSign(t *testing.T) {
    // = 기호가 없는 항목은 무시해야 한다
    result := DuplicateKeyIn([]string{"noequals"})
    assert.Equal(t, "", result)
}

// ... 나머지 시나리오
```

#### 4.4 AI 결과 검증 포인트

AI가 생성한 테스트를 받았을 때, 다음을 확인합니다.

1. **시나리오 매트릭스의 모든 행이 커버되었는가?** → 누락된 시나리오가 있다면 추가 요청
2. **assertion이 구체적인가?** → `assert.NotEmpty` 대신 `assert.Equal`로 정확한 값 비교
3. **엣지 케이스를 AI가 추가로 발견했는가?** → AI가 제안하는 추가 시나리오는 검토 가치가 있음
4. **테스트가 실제로 컴파일되고 통과하는가?** → 반드시 실행 확인

---

### 5. 언어별 프롬프트 조정 포인트

| 언어 | 프레임워크 | 프롬프트에 명시할 것 |
|------|-----------|-------------------|
| Go | testing + testify | Table-driven tests, `t.Run` 서브테스트 패턴 |
| Java | JUnit 5 + Mockito | `@ParameterizedTest`, `@DisplayName` 사용 |
| Python | pytest | fixture, parametrize, conftest 패턴 |
| TypeScript | Jest / Vitest | `describe`/`it` 구조, mock 모듈 패턴 |
| Rust | built-in #[test] | `#[should_panic]`, Result 반환 테스트 |

각 언어의 관용적 테스트 패턴을 프롬프트에 명시하면, AI가 해당 생태계에 맞는 자연스러운 테스트를 생성합니다.

---

### 6. 체크리스트: AI 생성 테스트를 머지하기 전에

AI가 만든 테스트를 코드베이스에 반영하기 전, 최종 검토 체크리스트입니다.

- [ ] **테스트가 실패할 수 있는가?** — 프로덕션 코드에 버그를 일부러 넣었을 때 빨간불이 켜지는가?
- [ ] **테스트 이름만 읽어도 무엇을 검증하는지 알 수 있는가?**
- [ ] **하나의 테스트에 하나의 assertion 주제만 있는가?**
- [ ] **경계값과 에러 경로가 포함되어 있는가?**
- [ ] **time.Sleep 같은 비결정적 대기가 없는가?**
- [ ] **프로덕션 코드를 그대로 복사한 로직이 없는가?**
- [ ] **테스트가 다른 테스트의 실행 순서에 의존하지 않는가?**

---

### 마무리

AI는 테스트 코드의 "양"을 빠르게 늘려줍니다. 하지만 "질"은 프롬프트의 품질에 달려 있습니다. 테스트 원칙을 프롬프트에 명시적으로 녹이면, AI는 처음부터 의미 있는 테스트를 생성합니다.

핵심 정리:
- 시나리오 매트릭스를 제공하면 누락이 줄어든다
- FIRST 원칙과 AAA 패턴을 프롬프트에 명시하면 구조가 잡힌다
- AI 생성 결과는 반드시 "실패 가능성"을 검증해야 한다
- 프롬프트 템플릿을 팀 내 공유하면 일관된 품질을 유지할 수 있다

> 다음 3편에서는 AI를 활용하여 테스트 코드 PR을 리뷰하는 방법을 다룹니다.
