---
layout: post
title: "Exception"
date: 2025-08-09 12:00:00 +0900
categories: [exception]
---
GlobalExceptionHandler는 애플리케이션 전역에서 발생하는 예외를 한 곳에서 수집·표준화해서 처리하는 컴포넌트입니다. 장점은 다음과 같습니다.

컨트롤러마다 중복된 예외 처리 코드 제거

API 응답 포맷을 일관되게 유지 (클라이언트 파싱/처리 쉬워짐)

로깅 · 모니터링 · 트레이싱을 중앙에서 제어 가능

입력 검증(validation), 권한/인증, DB 충돌 등 다양한 에러를 HTTP 상태 코드에 맞춰 처리

Spring에서는 보통 @ControllerAdvice 또는 @RestControllerAdvice로 구현합니다. (@RestControllerAdvice는 @ControllerAdvice + @ResponseBody 의미)

동작 원리 (간단)

컨트롤러(또는 서비스)에서 예외 발생 → 예외가 핸들러로 전파

@ExceptionHandler가 매칭되는 예외를 잡아서 ResponseEntity나 객체를 반환

클라이언트는 일관된 JSON 에러 응답을 받음

기본 템플릿 (실무용 예제)

다음 예제는 ResponseEntityExceptionHandler를 상속해 Spring이 던지는 MVC 관련 예외도 커스터마이징하고, 커스텀 BusinessException 등도 처리하는 전형적인 전역 핸들러입니다.

```
// ErrorResponse.java
import java.time.LocalDateTime;
import java.util.List;

public class ErrorResponse {
    private String code;            // 예: VALIDATION_FAILED, NOT_FOUND, INTERNAL_ERROR
    private String message;         // 사용자 친화적 메시지
    private int status;             // HTTP status code
    private String path;            // 요청 URI
    private LocalDateTime timestamp;
    private List<ApiFieldError> errors; // 상세 필드 에러 (옵션)

    // 생성자, getter, setter 생략 (또는 Lombok 사용)
}

// ApiFieldError.java
public class ApiFieldError {
    private String field;
    private Object rejectedValue;
    private String reason;
    // 생성자/getter/setter
}
// BusinessException.java (커스텀 비즈니스 예외)
import org.springframework.http.HttpStatus;

public class BusinessException extends RuntimeException {
    private final String code;
    private final HttpStatus status;

    public BusinessException(String code, String message, HttpStatus status) {
        super(message);
        this.code = code;
        this.status = status;
    }

    public String getCode() { return code; }
    public HttpStatus getStatus() { return status; }
}

// GlobalExceptionHandler.java
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.http.*;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.context.request.ServletWebRequest;
import org.springframework.web.context.request.WebRequest;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;

import javax.validation.ConstraintViolationException;
import java.time.LocalDateTime;
import java.util.List;
import java.util.stream.Collectors;

@RestControllerAdvice
@Order(Ordered.HIGHEST_PRECEDENCE)
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {
    private final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // 1) 커스텀 비즈니스 예외 처리
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException ex, HttpServletRequest request) {
        ErrorResponse body = new ErrorResponse(
                ex.getCode(),
                ex.getMessage(),
                ex.getStatus().value(),
                request.getRequestURI(),
                LocalDateTime.now(),
                null
        );
        return ResponseEntity.status(ex.getStatus()).body(body);
    }

    // 2) 입력 검증 (Bean Validation) 핸들링
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpHeaders headers,
            HttpStatus status, WebRequest request) {

        List<ApiFieldError> fieldErrors = ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> new ApiFieldError(fe.getField(), fe.getRejectedValue(), fe.getDefaultMessage()))
                .collect(Collectors.toList());

        ErrorResponse body = new ErrorResponse(
                "VALIDATION_FAILED",
                "입력 값이 유효하지 않습니다.",
                HttpStatus.BAD_REQUEST.value(),
                ((ServletWebRequest) request).getRequest().getRequestURI(),
                LocalDateTime.now(),
                fieldErrors
        );
        return ResponseEntity.badRequest().body(body);
    }

    // 3) @Validated(메소드 레벨)에서 발생하는 제약 위반 처리
    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ErrorResponse> handleConstraintViolation(ConstraintViolationException ex, HttpServletRequest request) {
        List<ApiFieldError> errors = ex.getConstraintViolations().stream()
                .map(cv -> new ApiFieldError(cv.getPropertyPath().toString(), cv.getInvalidValue(), cv.getMessage()))
                .collect(Collectors.toList());

        ErrorResponse body = new ErrorResponse("VALIDATION_FAILED", "입력 값이 유효하지 않습니다.",
                HttpStatus.BAD_REQUEST.value(), request.getRequestURI(), LocalDateTime.now(), errors);
        return ResponseEntity.badRequest().body(body);
    }

    // 4) DB 무결성 위반 (예: unique 제약) -> 409 Conflict 권장
    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ErrorResponse> handleDataIntegrity(DataIntegrityViolationException ex, HttpServletRequest request) {
        logger.warn("Data integrity violation: {}", ex.getMessage());
        ErrorResponse body = new ErrorResponse("CONFLICT", "데이터 무결성 위배", HttpStatus.CONFLICT.value(), request.getRequestURI(), LocalDateTime.now(), null);
        return ResponseEntity.status(HttpStatus.CONFLICT).body(body);
    }

    // 5) 엔티티 미발견
    @ExceptionHandler(javax.persistence.EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(javax.persistence.EntityNotFoundException ex, HttpServletRequest request) {
        ErrorResponse body = new ErrorResponse("NOT_FOUND", ex.getMessage(), HttpStatus.NOT_FOUND.value(), request.getRequestURI(), LocalDateTime.now(), null);
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(body);
    }

    // 6) 권한 거부
    @ExceptionHandler(org.springframework.security.access.AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(org.springframework.security.access.AccessDeniedException ex, HttpServletRequest request) {
        ErrorResponse body = new ErrorResponse("FORBIDDEN", "권한이 없습니다.", HttpStatus.FORBIDDEN.value(), request.getRequestURI(), LocalDateTime.now(), null);
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(body);
    }

    // 7) 그 외 예외 (최종 핸들러)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAll(Exception ex, HttpServletRequest request) {
        logger.error("Unhandled exception occurred", ex);
        ErrorResponse body = new ErrorResponse("INTERNAL_SERVER_ERROR", "서버 오류가 발생했습니다.", HttpStatus.INTERNAL_SERVER_ERROR.value(), request.getRequestURI(), LocalDateTime.now(), null);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(body);
    }
}

```
추가 실무 팁 & 권장 사항

**에러 코드(Error Code)**를 사용하세요. 클라이언트가 메시지 텍스트가 아닌 코드로 분기 처리하기 쉬워집니다.

로깅 레벨을 상황에 맞게 나누세요. (Validation → INFO/WARN, 예상하지 못한 예외 → ERROR)

민감 정보 노출 금지: ex.getMessage()가 민감한 DB 쿼리나 내부 정보를 포함할 수 있으니 응답에 그대로 노출하지 마세요. (로그에는 남겨도 괜찮음)

추적 id(Trace ID): 요청 헤더에 X-Request-ID나 로깅 MDC에 traceId를 넣어 응답에도 포함하면 문제 추적이 쉬워집니다.

스코핑: 전체 애플리케이션이 아닌 특정 패키지/컨트롤러에만 적용하려면 @ControllerAdvice(basePackages = "com.example.api") 처럼 범위 지정 가능.

ResponseEntityExceptionHandler 확장: Spring 내부 예외(예: HttpMessageNotReadableException, MissingServletRequestParameterException)를 오버라이드해 일관된 형식으로 응답하세요.

RFC7807(Problem Details): 표준 포맷을 원하면 ProblemDetail(Spring 6+) 또는 직접 RFC7807 형식으로 구현할 수 있습니다.

테스트: MockMvc로 예외 발생 시 HTTP 상태, body 구조를 단위 테스트로 검증하세요.

예외 분류 예시 (권장)

Client 오류(4xx)

VALIDATION_FAILED (400)

NOT_FOUND (404)

FORBIDDEN (403)

UNAUTHORIZED (401)

CONFLICT (409)

Server 오류(5xx)

INTERNAL_SERVER_ERROR (500)


디렉토리 구조
```
src
 └── main
     └── java
         └── com
             └── example
                 └── project
                     ├── ProjectApplication.java
                     │
                     ├── controller
                     │    └── UserController.java
                     │
                     ├── service
                     │    └── UserService.java
                     │
                     ├── repository
                     │    └── UserRepository.java
                     │
                     ├── domain
                     │    └── User.java
                     │
                     ├── exception
                     │    ├── GlobalExceptionHandler.java   // 전역 예외 처리기
                     │    ├── BusinessException.java        // 커스텀 비즈니스 예외
                     │    ├── ErrorResponse.java            // 에러 응답 DTO
                     │    ├── ApiFieldError.java            // Validation 필드 에러 DTO
                     │    ├── ErrorCode.java                // 에러 코드 enum
                     │    └── NotFoundException.java        // 엔티티 미발견 예외 등 개별 커스텀 예외
                     │
                     └── security
                          └── SecurityConfig.java
```
각 파일 역할 정리

GlobalExceptionHandler.java

@RestControllerAdvice

프로젝트 전역의 예외 처리

@ExceptionHandler 메소드들 정의 (Validation, DB 오류, BusinessException 등)

BusinessException.java

RuntimeException 상속

비즈니스 로직에서 의도적으로 던지는 예외 (예: "이미 존재하는 사용자")

ErrorCode + 메시지 포함

ErrorResponse.java

클라이언트에 내려줄 공통 에러 응답 객체

code, message, status, timestamp, path, errors(필드 오류 목록) 포함

ApiFieldError.java

Validation 오류 발생 시, 잘못된 필드 정보 담는 DTO

```
public enum ErrorCode {
    VALIDATION_FAILED(400, "VALIDATION_FAILED", "입력 값이 유효하지 않습니다."),
    NOT_FOUND(404, "NOT_FOUND", "리소스를 찾을 수 없습니다."),
    CONFLICT(409, "CONFLICT", "데이터 충돌이 발생했습니다."),
    INTERNAL_ERROR(500, "INTERNAL_SERVER_ERROR", "서버 오류가 발생했습니다.");
    
    private final int status;
    private final String code;
    private final String message;

    ErrorCode(int status, String code, String message) {
        this.status = status;
        this.code = code;
        this.message = message;
    }
    // getter
}

public class NotFoundException extends BusinessException {
    public NotFoundException(String message) {
        super(ErrorCode.NOT_FOUND.getCode(), message, HttpStatus.NOT_FOUND);
    }
}

```
Controller → Service → Repository 실행 중 예외 발생

예외가 던져짐 (BusinessException, NotFoundException, ValidationException 등)

GlobalExceptionHandler가 잡아서 → ErrorResponse 객체 변환

클라이언트는 항상 동일한 JSON 구조로 에러 응답 받음

```
ErrorCode.java
package com.example.project.exception;

import org.springframework.http.HttpStatus;

public enum ErrorCode {
    VALIDATION_FAILED(HttpStatus.BAD_REQUEST, "VALIDATION_FAILED", "입력 값이 유효하지 않습니다."),
    NOT_FOUND(HttpStatus.NOT_FOUND, "NOT_FOUND", "리소스를 찾을 수 없습니다."),
    CONFLICT(HttpStatus.CONFLICT, "CONFLICT", "데이터 충돌이 발생했습니다."),
    FORBIDDEN(HttpStatus.FORBIDDEN, "FORBIDDEN", "접근 권한이 없습니다."),
    INTERNAL_ERROR(HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_SERVER_ERROR", "서버 오류가 발생했습니다.");

    private final HttpStatus status;
    private final String code;
    private final String message;

    ErrorCode(HttpStatus status, String code, String message) {
        this.status = status;
        this.code = code;
        this.message = message;
    }

    public HttpStatus getStatus() { return status; }
    public String getCode() { return code; }
    public String getMessage() { return message; }
}
```

```
ErrorResponse.java
package com.example.project.exception;

import java.time.LocalDateTime;
import java.util.List;

public class ErrorResponse {
    private String code;
    private String message;
    private int status;
    private String path;
    private LocalDateTime timestamp;
    private List<ApiFieldError> errors;

    public ErrorResponse(String code, String message, int status, String path, List<ApiFieldError> errors) {
        this.code = code;
        this.message = message;
        this.status = status;
        this.path = path;
        this.timestamp = LocalDateTime.now();
        this.errors = errors;
    }

    // Getter
    public String getCode() { return code; }
    public String getMessage() { return message; }
    public int getStatus() { return status; }
    public String getPath() { return path; }
    public LocalDateTime getTimestamp() { return timestamp; }
    public List<ApiFieldError> getErrors() { return errors; }
}

```

```
ApiFieldError.java
package com.example.project.exception;

public class ApiFieldError {
    private String field;
    private Object rejectedValue;
    private String reason;

    public ApiFieldError(String field, Object rejectedValue, String reason) {
        this.field = field;
        this.rejectedValue = rejectedValue;
        this.reason = reason;
    }

    public String getField() { return field; }
    public Object getRejectedValue() { return rejectedValue; }
    public String getReason() { return reason; }
}

```
```
BusinessException.java
package com.example.project.exception;

public class BusinessException extends RuntimeException {
    private final ErrorCode errorCode;

    public BusinessException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }

    public BusinessException(ErrorCode errorCode, String detailMessage) {
        super(detailMessage);
        this.errorCode = errorCode;
    }

    public ErrorCode getErrorCode() { return errorCode; }
}
```

```
NotFoundException.java
package com.example.project.exception;

public class NotFoundException extends BusinessException {
    public NotFoundException(String message) {
        super(ErrorCode.NOT_FOUND, message);
    }
}

```
```
GlobalExceptionHandler.java
package com.example.project.exception;

import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.*;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.stream.Collectors;

@RestControllerAdvice
public class GlobalExceptionHandler {
    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // 1) 비즈니스 예외
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException ex, HttpServletRequest request) {
        ErrorCode code = ex.getErrorCode();
        ErrorResponse response = new ErrorResponse(
                code.getCode(),
                ex.getMessage(),
                code.getStatus().value(),
                request.getRequestURI(),
                null
        );
        return ResponseEntity.status(code.getStatus()).body(response);
    }

    // 2) Validation 에러
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex, HttpServletRequest request) {
        List<ApiFieldError> errors = ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> new ApiFieldError(fe.getField(), fe.getRejectedValue(), fe.getDefaultMessage()))
                .collect(Collectors.toList());

        ErrorResponse response = new ErrorResponse(
                ErrorCode.VALIDATION_FAILED.getCode(),
                ErrorCode.VALIDATION_FAILED.getMessage(),
                ErrorCode.VALIDATION_FAILED.getStatus().value(),
                request.getRequestURI(),
                errors
        );
        return ResponseEntity.badRequest().body(response);
    }

    // 3) 그 외 모든 예외
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception ex, HttpServletRequest request) {
        log.error("Unexpected error", ex);

        ErrorResponse response = new ErrorResponse(
                ErrorCode.INTERNAL_ERROR.getCode(),
                ErrorCode.INTERNAL_ERROR.getMessage(),
                ErrorCode.INTERNAL_ERROR.getStatus().value(),
                request.getRequestURI(),
                null
        );
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
    }
}
package com.example.project.exception;

import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.*;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.stream.Collectors;

@RestControllerAdvice
public class GlobalExceptionHandler {
    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // 1) 비즈니스 예외
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException ex, HttpServletRequest request) {
        ErrorCode code = ex.getErrorCode();
        ErrorResponse response = new ErrorResponse(
                code.getCode(),
                ex.getMessage(),
                code.getStatus().value(),
                request.getRequestURI(),
                null
        );
        return ResponseEntity.status(code.getStatus()).body(response);
    }

    // 2) Validation 에러
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex, HttpServletRequest request) {
        List<ApiFieldError> errors = ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> new ApiFieldError(fe.getField(), fe.getRejectedValue(), fe.getDefaultMessage()))
                .collect(Collectors.toList());

        ErrorResponse response = new ErrorResponse(
                ErrorCode.VALIDATION_FAILED.getCode(),
                ErrorCode.VALIDATION_FAILED.getMessage(),
                ErrorCode.VALIDATION_FAILED.getStatus().value(),
                request.getRequestURI(),
                errors
        );
        return ResponseEntity.badRequest().body(response);
    }

    // 3) 그 외 모든 예외
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception ex, HttpServletRequest request) {
        log.error("Unexpected error", ex);

        ErrorResponse response = new ErrorResponse(
                ErrorCode.INTERNAL_ERROR.getCode(),
                ErrorCode.INTERNAL_ERROR.getMessage(),
                ErrorCode.INTERNAL_ERROR.getStatus().value(),
                request.getRequestURI(),
                null
        );
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
    }
}

```



