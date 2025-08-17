---
layout: post
title: "Execption Handler"
date: 2025-08-11 12:00:00 +0900
categories: [Secret Manager, Execption Handler, Logging]
---
# 예외 종류

### Err1. 허용되지 않은 IP(401)
worker --> client
```
// 허용되지 않은 IP 예외
 if (!allowedIps.contains(clientIp)) {
    throw new ApiException(
            "Access denied for IP: " + clientIp,
            403,
            "IP_NOT_ALLOWED"
    );
}
```
### Err2. 존재하지 않는 account/secret(404)
worker --> client
```
List<String> allowedIps;
try {
    allowedIps = openBaoClient.getAllowedIpList(account);
} catch (IOException e) {
    throw new ApiException(
            "Failed to retrieve allowed IP list from OpenBao metadata",
            502,
            "OPENBAO_METADATA_ERROR"
    );
}
```
### Err3. Openbao 장애(502)
worker --> worker
```
try {
  if (!openBaoClient.isServiceHealthy()) {
      throw new ApiException(
              "OpenBao service unavailable",
              502,
              "OPENBAO_SERVICE_UNAVAILABLE"
      );
  }
} catch (TimeoutException e) {
    throw new ApiException(
      "OpenBao service timeout",
      502,
      "OPENBAO_TIMEOUT"
    );
}
```
### Err4. 파라미터 누락(400)
worker --> client
```
if (account == null || account.isBlank() || secretPath == null || secretPath.isBlank()) {
    throw new ApiException(
      "Required parameter is missing: account or secretPath",
      400,
      "MISSING_PARAMETER"
    );
}
```
### Err5. Extension 통신에러(502)
extension --> worekr
```
try {
    return businessServiceClient.callSecretRetrieval(account, secretPath);
} catch (Exception e) {
  throw new ApiException(
    "Failed to call business service: " + e.getMessage(),
    502,
    "BUSINESS_SERVICE_ERROR"
  );
}
```

## 예외 처리 템플릿
[Controller / Service] → 예외 발생 → [전역 예외 처리기] → 표준 응답(JSON) 반환

### 기본 예외 클래스
```
@Getter
@AllArgsConstructor
public class ApiException extends RuntimeException {
    private final String message;
    private final int status;
    private final String errorCode; // 예: MISSING_PARAMETER, OPENBAO_SERVICE_ERROR
}
```

### 전역 예외 처리기
```
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ApiException.class)
    public ResponseEntity<ErrorResponse> handleApiException(ApiException ex, WebRequest request) {

        ErrorResponse errorResponse = ErrorResponse.builder()
                .timestamp(Instant.now())
                .status(ex.getStatus())
                .error(ex.getErrorCode())
                .message(ex.getMessage())
                .traceId(getTraceId()) // OTEL traceId
                .build();

        return ResponseEntity.status(ex.getStatus()).body(errorResponse);
    }

    // 처리되지 않은 예외
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpectedException(Exception ex) {
        ErrorResponse errorResponse = ErrorResponse.builder()
                .timestamp(Instant.now())
                .status(500)
                .error("INTERNAL_SERVER_ERROR")
                .message("Unexpected error occurred: " + ex.getMessage())
                .traceId(getTraceId())
                .build();

        return ResponseEntity.status(500).body(errorResponse);
    }

    private String getTraceId() {
        return io.opentelemetry.api.trace.Span.current().getSpanContext().getTraceId();
    }
}
```

### 응답 DTO
```
@Getter
@Builder
@AllArgsConstructor
public class ErrorResponse {
    private Instant timestamp;
    private int status;
    private String error;
    private String message;
    private String traceId; // OpenTelemetry Trace ID
}
```

## 디렉토리 구조
```
src
 └── main
     ├── java
     │    └── com.example.worker
     │         ├── WorkerApplication.java           # Spring Boot 메인 클래스
     │
     │         ├── config                           # 설정 관련
     │         │    ├── OpenTelemetryConfig.java    # OpenTelemetry 자동 설정
     │         │    ├── RestTemplateConfig.java     # RestTemplate Bean, 타임아웃 등
     │         │    └── SecurityConfig.java         # Spring Security 설정 (IP 추출 포함)
     │
     │         ├── controller
     │         │    └── SecretController.java       # /api/secrets 엔드포인트
     │
     │         ├── service
     │         │    ├── SecretAccessService.java    # 메인 비즈니스 로직
     │         │    ├── IpValidationService.java    # IP 허용 여부 검증 로직
     │         │    └── BusinessServiceClient.java  # 외부 비즈니스 서버 호출
     │
     │         ├── client
     │         │    └── OpenBaoClient.java          # OpenBao API 호출 (메타데이터 조회 등)
     │
     │         ├── exception
     │         │    ├── ApiException.java           # 커스텀 예외
     │         │    ├── ErrorCode.java              # 에러 코드 Enum
     │         │    └── GlobalExceptionHandler.java # @ControllerAdvice
     │
     │         ├── model
     │         │    ├── dto
     │         │    │    ├── SecretRequest.java     # account, secretPath, clientIp 등
     │         │    │    └── SecretResponse.java    # 조회 결과 DTO
     │         │    └── OpenBaoMetadata.java        # 허용 IP 리스트 등 메타데이터 모델
     │
     │         └── util
     │              └── IpUtils.java                # IP 파싱, IPv4/IPv6 변환 등
     │
     └── resources
          ├── application.yml                       # 환경설정
          └── logback-spring.xml                    # 로깅 설정 (OpenTelemetry와 연동)
```

# 로그 
- 예외 5개 --> Error 레벨
```
{
  "level": "ERROR",
  "time": "2025-08-12T10:32:15Z",
  "traceId": "abc123",
  "clientIp": "192.168.0.5",
  "account": "teamA",
  "secretPath": "kv-v2/secret1",
  "status": 502,
  "message": "Failed to connect to OpenBao",
  "exception": "java.net.ConnectException: Connection refused"
}
```
- client가 secret 조회 요청 --> Info 레벨
```
{
  "level": "INFO",
  "time": "2025-08-12T10:36:21Z",
  "traceId": "abc125",
  "clientIp": "192.168.0.5",
  "account": "teamA",
  "secretPath": "kv-v2/secret1",
  "message": "Secret retrieval successful",
  "durationMs": 128
}
```

## 로그 수집 + 저장 흐름
```
Spring Boot App (Logback)
    ↓ 로그 기록 (텍스트 or JSON)
OpenTelemetry Java Agent (Optional) → MDC에 trace_id 삽입
    ↓ 로그 출력 (주로 JSON 포맷 권장)
    ↓
OpenTelemetry Collector (로그 수집기)
    ↓ 로그 파싱, 처리
    ↓
OpenSearch (로그 저장 및 검색)
```

## 로그 종류
### 액세스 로그(Access Log)
- 클라이언트 요청 정보 기록 (요청 URL, HTTP 메서드, IP, 응답 코드, 응답 시간 등)
- 사용량 분석, 보안 감사, 트래픽 패턴 분석에 활용
### 비즈니스 로그(Business Log)
- 중요한 비즈니스 이벤트 기록 (예: 시크릿 조회 성공/실패, IP 허용 여부 확인 결과)
- 애플리케이션 동작 분석 및 감사 기록 목적
### 예외 로그(Error Log)
- 에러 및 예외 발생 시 상세 정보 기록 (스택 트레이스, 에러 코드, 메시지)
- 문제 분석 및 대응에 핵심
### 디버그 로그(Debug Log)
- 개발 및 문제 해결 시 내부 상태나 변수값 추적용
- 운영 환경에서는 일반적으로 비활성화

## 로깅 프레임워크
- **SLF4J + Logback (Spring Boot 기본)**
- Log4j2
- Spring Boot Starter Logging

- JSON 형식으로 로그 남기기 권장
  - OpenTelemetry Collector가 로그를 쉽게 파싱하고 필드별 색인 가능
  - Logback에서 logstash-logback-encoder 라이브러리를 사용해 JSON 로그 출력
- 로그 패턴에 trace_id 포함
  - OpenTelemetry Java Agent가 MDC에 자동으로 trace_id를 넣어줌
  - Logback 패턴 또는 JSON 필드에 trace_id 포함해서 추적 연동 가능

## 로그 포맷과 표준화
### JSON 포맷 권장
메타데이터(트레이스ID, 타임스탬프, 서비스명 등) 포함
### 로그 레벨 분리
| 레벨        | 의미                | 용도 및 예시                       |   
| --------- | ----------------- | ----------------------------- |
| **ERROR** | 치명적인 에러           | 시스템 장애, 처리 불가한 예외, DB 연결 실패 등 |
| **WARN**  | 경고, 주의 필요         | 성능 저하, 비정상 요청, 재시도 가능한 문제 등   |
| **INFO**  | 정상 동작 로그          | 서비스 시작/종료, 주요 비즈니스 이벤트 기록     |
| **DEBUG** | 개발/디버깅용 상세 로그     | 변수 상태, 외부 호출 결과, 흐름 추적용       |
| **TRACE** | 가장 상세한 로그 (거의 안씀) | 함수 진입/종료, 상세 내부 상태            |

### 로그 레벨별 포맷
#### ERROR
목적 : 장애 원인 파악, 신속 대응
로그포맷 특징 : 상세한 스택 트레이스 + 에러 코드 포함
예시 포함 항목 : 타임스탬프, 로그레벨, 클래스, 메시지, 에러코드, 스택트레이스, TraceId
#### WARN
목적 : 주의 필요한 상황 감지
로그포맷 특징 : 	요약된 경고 메시지 + 주요 컨텍스트 포함
예시 : 타임스탬프, 로그레벨, 경고 메시지, 관련 정보, TraceId
```
%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - traceId=%X{trace_id} - %msg%n%ex

```
#### INFO
목적 : 	정상 이벤트 기록
로그포맷 특징 : 간결하고 일관된 메시지 중심
예시 : 타임스탬프, 로그레벨, 이벤트명, 사용자ID, IP, TraceId
```
%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - traceId=%X{trace_id} - event=%msg%n

```

### 개발 환경용 application-dev.yaml
```
logging:
  level:
    ROOT: DEBUG                           # 전체 디버깅 로그 활성화, 개발 시 상세 분석 용이
    com.example.worker.controller: DEBUG # 요청/응답 상세 추적
    com.example.worker.service: DEBUG    # 비즈니스 로직 흐름 상세 기록
    com.example.worker.client: DEBUG     # 외부 API 요청/응답 상세 기록
    com.example.worker.exception: ERROR  # 예외는 ERROR 이상으로 제한
    com.example.worker.config: INFO      # 설정 로딩 정보
    com.example.worker.model: DEBUG      # DTO 변환 상세 추적 가능
    com.example.worker.util: DEBUG       # 유틸 함수 내부 동작 추적
```

### 운영 환경용 application-prod.yaml
```
logging:
  level:
    ROOT: INFO                           # 전체적으로 INFO 이상 로그만 기록 (운영 부하 감소)
    com.example.worker.controller: INFO # API 호출 기본 로그 (성공/실패 이벤트 중심)
    com.example.worker.service: INFO    # 주요 비즈니스 이벤트만 기록 (상세 디버그 제외)
    com.example.worker.client: WARN     # 외부 API 통신 실패/경고만 기록
    com.example.worker.exception: ERROR # 예외 발생 시 심각도에 맞게 기록
    com.example.worker.config: INFO     # 설정 로딩 및 초기화 상태
    com.example.worker.model: INFO      # DTO 생성 등 최소 정보
    com.example.worker.util: WARN       # 유틸 내부 오류나 주의 사항만 기록
```