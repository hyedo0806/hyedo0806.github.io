---
layout: post
title: "Function 단위 설계"
date: 2025-07-20 12:00:00 +0900
categories: [Secret Manager, OpenBao, Architecture]
---
controller →　service → 외부 application 구조의 아키텍처

## Controller
```
@RequestMapping("/api/secret")
public class SecretController {

    private final SecretAccessService secretAccessService;

    public SecretController(SecretAccessService secretAccessService) {
        this.secretAccessService = secretAccessService;
    }

    @GetMapping("/{secretPath}")
    public ResponseEntity<?> getSecret(
            @PathVariable("secretPath") String secretPath,
            @RequestParam("account") String account,
            Authentication authentication) {

        return secretAccessService.handleRequest(secretPath, account, authentication);
    }
}
```
## Service
```
@Service
public class SecretAccessService {

    private final OpenBaoClient openBaoClient;
    private final IpValidator ipValidator;
    private final BusinessServiceClient businessServiceClient;

    public SecretAccessService(OpenBaoClient openBaoClient,
                               IpValidator ipValidator,
                               BusinessServiceClient businessServiceClient) {
        this.openBaoClient = openBaoClient;
        this.ipValidator = ipValidator;
        this.businessServiceClient = businessServiceClient;
    }

    public ResponseEntity<?> handleRequest(String secretPath, Authentication authentication) {
        // 1. Spring Security에서 클라이언트 IP 획득
        String clientIp = ((WebAuthenticationDetails) authentication.getDetails()).getRemoteAddress();

        // 2. OpenBao에서 허용된 IP 목록 조회
        List<String> allowedIpList = openBaoClient.fetchAllowedIpList(secretPath);

        // 3. IP 검증
        if (!ipValidator.isIpAllowed(clientIp, allowedIpList)) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN)
                                 .body("Access denied: IP not allowed");
        }

        // 4. 허용된 경우 Business Service 호출
        return businessServiceClient.callBusinessService(secretPath, clientIp);
    }
}
```
## Application 
```
@Component
public class OpenBaoClient {

    private final RestTemplate restTemplate;

    public OpenBaoClient(RestTemplateBuilder builder) {
        this.restTemplate = builder.build();
    }

    public List<String> fetchAllowedIpList(String secretPath) {
        // OpenBao API 호출 (메타데이터에서 allowed_ips 필드 가져오기)
        String url = "https://openbao.example.com/v1/secret/metadata/" + secretPath;
        // 여기서는 예시로 고정 리스트 반환
        return List.of("192.168.0.10", "10.0.0.0/24");
    }
}

@Component
public class BusinessServiceClient {

    private final RestTemplate restTemplate;

    public BusinessServiceClient(RestTemplateBuilder builder) {
        this.restTemplate = builder.build();
    }

    public ResponseEntity<?> callBusinessService(String secretPath, String account, String clientIp) {
        HttpHeaders headers = new HttpHeaders();
        headers.set("X-Client-IP", clientIp);
        headers.setBearerAuth("SERVICE_TOKEN"); // 외부 서비스 인증 토큰 (필요시)

        HttpEntity<Void> entity = new HttpEntity<>(headers);

        // 쿼리 파라미터로 path, account 포함
        UriComponentsBuilder uriBuilder = UriComponentsBuilder
                .fromHttpUrl("https://secrutiryextension/secret")
                .queryParam("path", secretPath)
                .queryParam("account", account);

        String url = uriBuilder.toUriString();

        return restTemplate.exchange(url, HttpMethod.GET, entity, String.class);
    }
}
```
## Util
```
@Component
public class IpValidator {

    public boolean isIpAllowed(String clientIp, List<String> allowedIpList) {
        // 단순 검증: CIDR 매칭 로직 추가 가능
        return allowedIpList.contains(clientIp);
    }
}
```
