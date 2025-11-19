---
title: opentelemetry가 수집한 로그를 필터링 거쳐서 redis로 전송하기
date: 2025-11-19 22:19:00 
categories: [opentelemetry, redis, lma]
tags: [lma]     # TAG names should always be lowercase
---
# 배경
k8s 환경에서 구동중인 apigateway pod는 opentelemetry를 사용해서 파일로그 형태로 pvc에 로그를 저장한다. 저장된 로그는 opensearch로 전송된다. <br>
pvc에 저장하는 로그를 redis로 전송하고 애플리케이션에서 redis에 저장된 로그를 1분 단위로 로그를 파싱 후 db에 저장한다. 따라서 실시간으로 report 기능에 필요한 메트릭 데이터를 제공한다. 

# 설계

## 흐름
Pod 로그 → OTel Collector → (로그 버퍼) → 파싱 애플리케이션(1분 배치) → DB

## otel pipeline
### filelog receiver

### filter processor
```
processors:
  filter:
    logs:
      include:
        match_type: regexp
        expressions:
          - 'resource.attributes["k8s.pod.name"].matches(".*{{필터링 키워드}}.*")'
          - 'body.contains("{{로그에 포함된 키워드}}")'
```

### (버퍼) exporter
``` 
exporters:
  redis:
    endpoint: "redis-master.redis.svc.cluster.local:6379"
    stream: "logs"
    tls:
      insecure: true
    password: "${REDIS_PASSWORD}"
    max_retries: 3
    queue_size: 10000
    write_timeout: 5s
```

## kafka 
### 대안
1. SQS
2. rabbitMQ
3. kafka

## 애플리케이션

# 고민해야할 것 
만약 redis pod가 죽는다면?(복구 전략)<br>
--> Redis Streams는 ephemeral(일시적) 버퍼 역할이고, 원본 로그는 항상 PV(또는 node 로그)에서 남아 있기 때문에 재복구 가능.<br>
정확한 시간대만 복구하려면 "마지막으로 처리된 로그 시간"을 기록해두면 된다.(redis와 독립적으로) <br>
--> 가장 좋은 방식은 Redis 장애 시 fallback 복구 애플리케이션을 자동화하는 구조<br>

```
last_processed_log_time ~ now()
```

# 개선점
기존에는 opensearch에서 1시간마다 쿼리문을 통해 저장된 로그를 가져와서 파싱한다. 이 주기가 1분으로 변경되는 경우 cluster 부하 증가가 예상됨. 또한 1분단위로 주기가 짧아지는 경우, opensearch의 동기화 문제도 있다.(정확한 시점에 로그가 쌓여있을지 모름)