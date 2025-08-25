---
layout: post
title: "Spring Acutator와 비교한 Opentelemetry"
date: 2025-08-25 12:00:00 +0900
categories: [Spring, Logging, Actuator]
---
# Spring Boot Actuator
- Spring Boot에서 기본 제공

- JVM + Spring 애플리케이션 상태 모니터링에 최적화


### 단점

- 분산 추적(Distributed Tracing) 불가

- 로그 수집과 연동하려면 추가 설정 필요(/actuator/logger 에서 로그를 보여주는 것이 아님)

## 분산추적(Distributed Tracing)
마이크로서비스 환경에서 요청(Request)이 여러 서비스/서버를 거칠 때 전체 흐름을 추적하는 것으로 각 서비스에서 어떤 업이 얼마나 걸렸는지, 어떤 서비스에서 지연이 발생했는지를 한눈에 보여준다.

예시
- 클라이언트가 서비스A → 서비스B → 서비스C → DB 요청

- 전통적인 로그만 보면 “서비스A에서 DB 쿼리 완료”만 보일 수 있음 → 전체 흐름 파악 어려움

- 분산 추적을 하면 각 서비스에서 발생한 span(작업 단위)을 연결하여 “전체 요청 경로 + 소요 시간”을 시각화 가능
```
Client → A (50ms) → B (100ms) → C (200ms)
```
- 여기서 B가 병목인 것을 바로 확인 가능

### Spring Boot Actuator는 왜 분산 추적을 못하는가?
- Actuator는 단일 애플리케이션의 JVM/스프링 메트릭 중심

- HTTP 요청, DB 호출, 메모리 상태, GC 시간 등 서비스 내부 상태만 수집

- 다른 서비스까지 이어지는 요청 흐름을 자동으로 연결하지 못함

## OpenTelemetry가 분산 추적을 하는 방법
### 핵심 개념

Trace: 하나의 요청(Request)을 추적한 전체 단위

Span: Trace 안에서 작업 단위 (예: HTTP 요청, DB 쿼리, 메시지 처리)

Attribute: secret ID, client IP, 접근 허용 여부

Context Propagation: 요청이 다른 서비스로 갈 때 Trace ID와 Span ID를 함께 전달
→ 다음 서비스가 이어서 추적 가능

### 동작 예시
```
Client
   │
   │ GET /secret?account=abc&secretPath=xyz
   ▼
Worker (Spring Boot)
   │
   │─ Span: "request-receive" (root)
   │
   │─ IpAccessFilter
   │     └─ Span: "ip-access-check"
   │         └─ Worker → OpenBaoClient: fetch allowed IP list
   │
   │─ TokenFilter
   │     └─ Span: "token-auth"
   │
   │─ SecretAccessService
   │     └─ Span: "secret-request"
   │         └─ Worker → Extension: fetch secret
   │             └─ Span: "extension-secret-fetch"
   │
   │─ SecretAccessService returns secret
   │
   ▼
Client receives secret
```

#### span 구성 예시
Parent-Child 관계
```
request-receive (root)
├─ ip-access-check
├─ (token-auth)
└─ secret-request
    └─ extension-secret-fetch
```
request-receive : Client 요청을 Worker가 처음 받는 시점

ip-access-check : Worker에서 OpenBao 메타데이터 조회 후 IP 허용 여부 확인
  - client.ip: 요청 보낸 클라이언트 IP
  - allowed: true/false
    
(token-auth : Bearer token 인증)
  - auth.status: success/failure
  - token.present: true/false

secret-request : Worker 내부에서 Extension 호출 준비 및 요청
  - accountId, secretPath

extension-secret-fetch : Worker → Extension API 호출, 실제 Secret 조회 수행
  - status.code: 200, 400, 502 등
  - duration.ms: 응답 시간
  - error.message: 실패 시 에러 메시지