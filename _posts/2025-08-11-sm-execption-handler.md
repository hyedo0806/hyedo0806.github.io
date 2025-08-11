---
layout: post
title: "Execption Handler"
date: 2025-08-11 12:00:00 +0900
categories: [Secret Manager, Execption Handler]
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

# 로그 기록 방식
### 에러 메시지 + 컨텍스트 정보
- 요청 ID (Trace ID / Span ID)
- 클라이언트 IP
- 요청한 account, secretPath
- 예외 발생 서비스명
### 민감정보 제외
- 시크릿 값, 토큰, 패스워드 등은 로그에 절대 남기지 않음
### 로그 레벨 설정
- WARN → 비정상 입력, 예상 가능한 예외
- ERROR → 시스템 장애, 미처리 예외

# 예외 처리 템플릿
[Controller / Service] → 예외 발생 → [전역 예외 처리기] → 표준 응답(JSON) 반환

## 기본 예외 클래스
```
@Getter
@AllArgsConstructor
public class ApiException extends RuntimeException {
    private final String message;
    private final int status;
    private final String errorCode; // 예: MISSING_PARAMETER, OPENBAO_SERVICE_ERROR
}
```

## 전역 예외 처리기
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

## 응답 DTO
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