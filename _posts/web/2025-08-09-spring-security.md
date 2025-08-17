---
layout: post
title: "Spring Security"
date: 2025-08-09 12:00:00 +0900
categories: [Spring Security, FilterChain]
---
## SecurityConfig
스프링 시큐리티가 동작하는 설정 담당자<br>
@Configuration 어노테이션 붙어서 스프링 빈으로 등록

## FilterChain
```
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    // ... 설정 ...
    return http.build();
}
```
필터체인은 웹 요청이 들어올 때 실행되는 여러 필터(검증, 인증, 인가 등)의 순서와 동작 방식을 정의한다.<br>
HttpSecurity 객체를 이용해 필터 동작 방식을 조립

### CSRF 설정
```
http.csrf(csrf -> csrf.disable());
```
- CSRF 공격 방어 기능 on/off
- REST API 같은 경우 토큰 방식으로 인증하면 끄는 경우가 많다.

### 인증·인가 규칙 (Authorization)
```
http.authorizeHttpRequests(auth -> auth
    .requestMatchers("/admin/**").hasRole("ADMIN")
    .requestMatchers("/user/**").authenticated()
    .anyRequest().permitAll()
);
```
- URL 패턴별 접근 권한 설정
- 관리자 권한 필요한 URL, 로그인 필요 URL, 모두 허용 URL 등 구분

### 인증 방식 지정
- Basic Auth 사용
```
http.httpBasic(Customizer.withDefaults());  // Basic Auth 사용
```
- 로그인 폼 사용
```
http.formLogin(Customizer.withDefaults());  // 로그인 폼 사용
```

### custom Filter 등록
```
http.addFilterBefore(customFilter, UsernamePasswordAuthenticationFilter.class);
```
- 커스텀 인증, 로깅, 권한 검사 등

### 세션 관리
```
http.sessionManagement(sess -> sess
    .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
);
```
- 세션 사용 방식 설정
- JWT기반 API 면 무상태 설정을 많이 한다. 
