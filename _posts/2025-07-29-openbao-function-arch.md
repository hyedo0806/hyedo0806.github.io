---
layout: post
title: "Function 단위 설계"
date: 2025-07-20 12:00:00 +0900
categories: [Secret Manager, OpenBao, Spring Security]
---
클라이언트가 OpenBao에 저장된 특정 시크릿을 조회하고자 할 때, 사전에 등록된 IP인지 검증한 후, 허용된 경우에만 Extension API를 통해 실제 시크릿을 반환하도록 한다.
실제 사용자가 등록하는 시크릿은 account별로 네임스페이스가 생성되고 하위헤 kv-v2/secret1, kv-v2/secret2 이런식으로 저장된다.
IP 접근제어를 위한 허용IP리스트는 iprestriction 네임스페이스 하위에서 생성 kv-v2/accountID/secret# 

```

        Client
          |
          | GET /api/secret?account={account_id}&secretPath={kv-v2/secret/db-password}
          ↓
        SecretController 
          |
        SecretService
          ├─▶ Extract Real IP (Spring Security)
          |
        OpenBaoClient 호출
          |
          ├─▶ isIpAllowed(secretPath, clientIp)  ← OpenBao 메타데이터에서 IP 확인
          |       └─ 호출: GET /v1/<namespace>/kv-v2/metadata/<path_to_secret>
          |               --header "X-Vault-Token: <your_token>"
        SecretService
          |
        SecretController
          ├─▶ [허용된 IP] → callExtension(secretPath, clientIp)
          |       └─ 호출: GET https://security-extension/secret?path=secretPath&account=account
          |
          └─▶ [비허용 IP] → 403 Forbidden 응답
  
 ```

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
