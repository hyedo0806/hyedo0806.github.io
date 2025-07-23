---
layout: post
title: "Secret Manager 스터디"
date: 2025-07-23 12:00:00 +0900
categories: [Secret Manager, IP Restriction, Proxy server]
---
Secret Manager 상품 중, worker의 대표적 기능 2가지에 대한 스터디입니다.

1. IP Restriction
2. Proxy Server

# 1. IP Restriction
## Spring Security
Spring Security는 자바 Spring 프레임워크 기반 애플리케이션에서 보안을 쉽게 구현할 수 있도록 돕는 모듈입니다.
### 지원 기능
- 인증(Authentication)
- 인가(Authorization)
- 공격 방어(CSRF, 세션 고정, CORS 등)
- 세밀한 URL 권한 제어, 사용자 권한(Role) 관리, 비밀번호 암호화, OAuth2, JWT 등 다양한 보안 기능

여러 개의 보안 관련 필터를 순서대로 실행하는 구조이다. 각 필터는 HTTP 요청을 가로채서 위의 기능을 처리한다. 모든 필터를 통과하면 Controller 코드가 실행된다.
#### 실행 시 동작 순서
1. 클라이언트가 HTTP 요청 전송.
2. 내장 Tomcat이 요청을 받고, Spring Security의 FilterChainProxy로 전달.
3. FilterChainProxy가 등록된 모든 보안 필터(IP 접근제어, 인증, 인가 등)를 순서대로 실행.
4. 모든 보안 검사가 통과되면 컨트롤러가 실행됨.

### 설정
#### 의존성 추가
```
implementation 'org.springframework.boot:spring-boot-starter-security'
```
#### Bean 주입하여 보안 정책 정의
```
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())              // CSRF 비활성화(테스트용)
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()          // 모든 요청은 인증 필요
            )
            .httpBasic(Customizer.withDefaults());     // HTTP Basic 인증 사용
        return http.build();
    }
}
```
#### 커스텀 필터 추가(선택 사항)
```
// HTTP Basic 인증 사용 대신
...
.addFilterBefore(ipAccessFilter, UsernamePasswordAuthenticationFilter.class); 
...
```


## IP Restriction 동작 원리

클라이언트가 worker에 요청을 보낸다,

worker가 요청을 받으면 IP 주소를 확인
- HTTP 요청에서 클라이언트 IP는 request.getRemoteAddr()로 
- 프록시 환경에서는 X-Forwarded-For 헤더를 참고

worker는 허용된 IP 목록(openbao)과 비교해서 이 IP가 허용된 IP인지 판단
- 허용된 IP면 요청을 정상 처리하도록 다음 필터 또는 컨트롤러로 넘긴다.
- 허용되지 않은 IP면 403 Forbidden 등 접근 거부 응답을 반환

## Spring Security에서 IP Restriction이 동작하는 구조
- Spring Security는 여러 필터를 순서대로 실행하는 **Filter Chain** 구조

- IP 제한도 하나의 필터 역할로 이 체인에 끼워 넣을 수 있다.
```
http.authorizeHttpRequests(auth -> auth.requestMatchers("/admin/**").hasIpAddress("192.168.0.0/24")
    .anyRequest().permitAll()
);
```
그러나 Proxy 환경도 고려해야한다. 이 경우, request.getRemoteAddr()는 실제 클라이언트 IP가 아니다.<br>
프록시 서버들이 원본 클라이언트 IP를 HTTP 헤더에 넣어 전달(ex. X-Forwarded-For)<br>
*FowardedHeaderFilter* 는 이 헤더를 읽어서 *request.getRemoteAddr()* 가 실제 클라이언트 IP를 반환하도록 래핑

```
@Component
// OncePerRequestFilter를 상속 → 한 요청마다 한 번만 실행
public class IpAccessFilter extends OncePerRequestFilter {

    private final IpAccessService ipAccessService;

    public IpAccessFilter(IpAccessService ipAccessService) {
        this.ipAccessService = ipAccessService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        String clientIp = request.getRemoteAddr();
        // request.getRemoteAddr()를 이용해 클라이언트 IP를 얻는다.
        // 프록시 환경에서는 ForwardedHeaderFilter가 먼저 실행되어 getRemoteAddr()을 실제 클라이언트 IP로 교체

        // IpAccessService에게 "이 IP가 허용되었는가?"를 물어본다.
        if (!ipAccessService.isAllowed(clientIp)) {
            response.sendError(HttpServletResponse.SC_FORBIDDEN, "Access Denied for IP: " + clientIp);
            return;
        }

        // 허용되면 filterChain.doFilter()를 호출해 요청을 컨트롤러로 전달.
        filterChain.doFilter(request, response);
    }
}
```


# 2. Proxy Server