---
title: Response 변환
date: 2025-11-09 16:32:11 
categories: [proxy server]
tags: [proxy server]     # TAG names should always be lowercase
---
proxy server의 주요 기능 중 하나인 **Response Transportation**에 대한 포스팅이다.

## 구조요약
Client → Proxy(Spring App) → Backend Server
- Client: 조회 요청 보냄 (예: /api/secret?name=db-password)
- Proxy(Spring): 요청을 받아 백엔드로 전달하고 응답을 다시 가공해 반환
- Backend: { "secret_value": { "key1": "value1", "key2": "value2" } } 형식으로 응답

### 목표
- Proxy는 백엔드 응답을 그대로 전달하지 않고, 클라이언트가 쓰기 쉬운 구조로 변환해야 함
- 확장성을 고려하여 status, timestamp, requestId 같은 메타데이터도 포함할 수 있도록 한다.


🔁 표준화 (Normalization)	서로 다른 백엔드들의 응답 구조를 하나의 일관된 형태로 제공 <br>
🧩 캡슐화 (Encapsulation)	클라이언트가 내부 백엔드 구조를 몰라도 되게 함<br>
🛡️ 보안 (Security)	민감정보 필터링, 마스킹, 비필요 필드 제거<br>
🔎 관찰성 (Observability)	로그/트레이스에 일관된 응답 구조를 남겨 추적 가능하게 함<br>
⚙️ 확장성 (Extensibility)	나중에 metadata, pagination, traceId 등을 쉽게 추가 가능<br>
🚨 오류 처리 일관성 (Error Handling)	모든 예외를 동일한 구조(status, message, code)로 변환<br>

### client 측 응답
```
// success
{
  "status": "success",
  "data": {
    "secrets": {
      "key1": "value1",
      "key2": "value2"
    }
  },
  "timestamp": "2025-11-09T16:30:00Z"
}

// fail
{
  "status": "error",
  "message": "Backend server unreachable",
  "code": "BACKEND_TIMEOUT",
  "timestamp": "2025-11-09T16:45:02Z"
}
```

✅ 성공 (Success)	200~299	{ "secret_value": { ... } }	요청 정상 처리<br>
⚠️ 클라이언트 오류 (Client Error)	400~499	{ "error": "Invalid parameter" }	요청이 잘못됨<br>
🔐 인증/인가 오류 (Auth Error)	401/403	{ "message": "Unauthorized" }	인증 또는 권한 문제<br>
💥 서버 오류 (Server Error)	500~599	{ "error": "Internal error" }	백엔드 내부 문제<br>

## 코드
### proxy response
```
@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class ProxyResponse<T> {
    private String status;
    private T data;
    private LocalDateTime timestamp;
}
```

### service
```
@Service
@RequiredArgsConstructor
@Slf4j
public class SecretProxyService {

    private final WebClient webClient;

    public ProxyResponse<Map<String, String>> fetchSecrets(String name) {
        // 1️⃣ 백엔드 호출
        Map<String, Object> backendResponse = webClient.get()
                .uri("/backend/secrets?name={name}", name)
                .retrieve()
                .bodyToMono(new ParameterizedTypeReference<Map<String, Object>>() {})
                .block();

        // 2️⃣ secret_value 부분 추출
        Map<String, String> secrets = (Map<String, String>) backendResponse.get("secret_value");

        // 3️⃣ 클라이언트 응답 형태로 래핑
        return ProxyResponse.<Map<String, String>>builder()
                .status("success")
                .data(secrets)
                .timestamp(LocalDateTime.now())
                .build();
    }
}
```

### controller
```
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/proxy")
public class SecretProxyController {

    private final SecretProxyService secretProxyService;

    @GetMapping("/secrets")
    public ResponseEntity<ProxyResponse<Map<String, String>>> getSecrets(
            @RequestParam String name) {
        ProxyResponse<Map<String, String>> response = secretProxyService.fetchSecrets(name);
        return ResponseEntity.ok(response);
    }
}
```

### webclient (bean 주입)
```
@Configuration
public class WebClientConfig {
    @Bean
    public WebClient webClient(WebClient.Builder builder) {
        return builder
                .baseUrl("https://backend-server.com")
                .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
                .build();
    }
}
```

## WebClient bean 설정을 하는 이유?
**1. 일관된 설정 유지**<br>
WebClient는 호출할 때마다 baseUrl, header, timeout 등을 설정할 수 있지만,
여러 서비스 클래스에서 반복하면 관리가 어려워집니다.

이렇게 등록된 bean은 다음과 같이 주입된다. 
이때 Spring은 자동으로 @Configuration에서 등록한 WebClient Bean을 찾아서 주입합니다.
즉, new WebClient() 없이 바로 사용 가능.
```
@Service
@RequiredArgsConstructor
public class SecretProxyService {
    private final WebClient webClient; // ✅ Bean으로 주입됨
}
```

**2. 성능 효율성 (커넥션 풀 재사용)**

WebClient는 내부적으로 Connection Pool (TCP 커넥션 재활용) 을 사용합니다.
→ 한 번 생성된 WebClient 인스턴스를 재사용하면,
매번 새 TCP 연결을 열지 않아도 돼서 성능이 훨씬 좋아집니다.

만약 매번 WebClient.create()를 호출하면,
요청마다 새로운 커넥션이 만들어지고 성능 저하 + 리소스 낭비가 발생합니다.

**3. 추가 기능 확장성 (Filter, Interceptor, Logging 등)**

Bean으로 등록하면 아래처럼 공통 필터를 추가할 수도 있습니다:
```
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder
            .baseUrl("https://backend-server.com")
            .filter(logRequest()) // ✅ 요청 로깅 필터
            .filter(authHeaderInjector()) // ✅ 인증 헤더 주입
            .build();
}
```

## CommonApiResponse
```
package com.example.common.dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.Builder;
import lombok.Data;

import java.time.LocalDateTime;

/**
 * 애플리케이션 전역 공통 응답 포맷
 */
@Data
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL) // null 필드는 직렬화 제외
public class CommonApiResponse<T> {

    private String status;        // success / error
    private String code;          // 예외 코드, 비즈니스 코드
    private String message;       // 상태 또는 에러 메시지
    private T data;               // 응답 데이터 (제네릭)
    private LocalDateTime timestamp; // 응답 생성 시각

    // ✅ 성공 응답 생성자
    public static <T> CommonApiResponse<T> success(T data) {
        return CommonApiResponse.<T>builder()
                .status("success")
                .data(data)
                .timestamp(LocalDateTime.now())
                .build();
    }

    // ✅ 성공 + 메시지 포함
    public static <T> CommonApiResponse<T> success(String message, T data) {
        return CommonApiResponse.<T>builder()
                .status("success")
                .message(message)
                .data(data)
                .timestamp(LocalDateTime.now())
                .build();
    }

    // ✅ 실패 응답 생성자
    public static <T> CommonApiResponse<T> error(String code, String message) {
        return CommonApiResponse.<T>builder()
                .status("error")
                .code(code)
                .message(message)
                .timestamp(LocalDateTime.now())
                .build();
    }
}
```
### controller
```
@RestController
@RequestMapping("/api/proxy")
@RequiredArgsConstructor
public class SecretProxyController {

    private final SecretProxyService secretProxyService;

    @GetMapping("/secrets")
    public ResponseEntity<CommonApiResponse<Map<String, String>>> getSecrets(
            @RequestParam String name) {

        try {
            Map<String, String> secrets = secretProxyService.fetchSecrets(name);

            return ResponseEntity.ok(
                    CommonApiResponse.success("Secret fetched successfully", secrets)
            );

        } catch (UnauthorizedException e) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                    .body(CommonApiResponse.error("UNAUTHORIZED", e.getMessage()));

        } catch (Exception e) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body(CommonApiResponse.error("SERVER_ERROR", "Unexpected error occurred"));
        }
    }
}
```
### service
```
@Service
@RequiredArgsConstructor
@Slf4j
public class SecretProxyService {

    private final WebClient webClient;

    public Map<String, String> fetchSecrets(String name) {
        Map<String, Object> backendResponse = webClient.get()
                .uri("/backend/secrets?name={name}", name)
                .retrieve()
                .bodyToMono(new ParameterizedTypeReference<Map<String, Object>>() {})
                .block();

        if (backendResponse == null || !backendResponse.containsKey("secret_value")) {
            throw new RuntimeException("Invalid backend response");
        }

        return (Map<String, String>) backendResponse.get("secret_value");
    }
}
```