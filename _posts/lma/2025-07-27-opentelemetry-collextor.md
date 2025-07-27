---
layout: post
title: "opentelemetry collector 설치와 개념"
date: 2025-07-27 19:00:00 +0900
categories: [opentelemetry, lma]
---
# Opentelemetry(otel) 란?
애플리케이션, 인프라, 클라우드 서비스에서 생성되는 로그, 메트릭, 트레이스를 수집하고 다양한 백엔드(예: OpenSearch, Jaeger, Prometheus 등)로 전달하는 범용 수집기

## 구조
- **Receiver** : 로그를 어떤 방식으로 수집할지 정의 (otlp, filelog, fluentforward, syslog 등).
- **Processor** : 수집된 로그를 필터링, 변환, 집계.
- **Exporter** : 처리된 데이터를 OpenSearch로 전송 (opensearchexporter 사용).
### 예시 설정(otel-collector-config.yaml)
k8s 컨테이너 로그를 수집해서 otel-logs 인덱스에 저장한다. 
#### **receivers** : 로그를 어디서 가져올지 정의
filelog 리시버는 파일 기반 로그를 수집한다.<br>
/var/log/containers 경로에서 모든 log파일을 수집<br>
다른 옵션으로는 otlp, fluentfoward, syslog 등이 있다.<br>

```
receivers:
  filelog:
    include: [ /var/log/containers/*.log ]
```
***
#### **processors** : 수집한 로그를 어떻게 가공할지 정의
batch 프로세서는 로그를 묶어서 전송하여 성능을 최적화한다.<br>
다른 예시
- attributes: 특정 필드 추가, 제거, 수정.
- filter: 특정 조건에 맞는 로그만 통과.
- memory_limiter: Collector 메모리 사용량 제어.

```
processors:
  batch:
```
***
#### **exporters** : 로그를 어디로 보낼지 정의
opensearch 익스포터는 수집된 로그를 OpenSearch 인덱스에 전송<br>
주요 옵션
- endpoints: OpenSearch 클러스터의 엔드포인트 URL
- username, password: OpenSearch 인증 정보
- index: 로그가 저장될 인덱스 이름 (otel-logs)

```
exporters:
  opensearch:
    endpoints: [ "http://opensearch-cluster-master:9200" ]
    username: "admin"
    password: "<비밀번호>"
    index: "otel-logs"
```
***
#### **service** 
전체적인 파이프라인을 정의하고 있다. <br>
receiver → processor → exporter 
```
service:
  pipelines:
    logs:
      receivers: [filelog]
      processors: [batch]
      exporters: [opensearch]
```
***

<br>

# 설치방법
Kubernetes에 DaemonSet 또는 Deployment 형태로 설치할 수 있다.
```
helm upgrade otel-collector open-telemetry/opentelemetry-collector -f opentelemetry-values.yaml -n lma
```

### config 파일 
```
mode: daemonset

exporters:
  debug: {}
  opensearch/otel:
    http:
      auth:
        authenticator: basicauth/otel
      endpoint: http://opensearch-cluster-master.lma.svc.cluster.local:9200
      tls:
        insecure_skip_verify: true
    logs_index: otel-logs
extensions:
  basicauth/otel:
    client_auth:
      password: Zkfltmak0425@
      username: admin
  health_check:
    endpoint: ${env:MY_POD_IP}:13133
processors:
  batch: {}
  memory_limiter:
    check_interval: 5s
    limit_percentage: 80
    spike_limit_percentage: 25
receivers:
  filelog:
    include:
    - /var/log/containers/*.log
  jaeger:
    protocols:
      grpc:
        endpoint: ${env:MY_POD_IP}:14250
      thrift_compact:
        endpoint: ${env:MY_POD_IP}:6831
      thrift_http:
        endpoint: ${env:MY_POD_IP}:14268
  otlp:
    protocols:
      grpc:
        endpoint: ${env:MY_POD_IP}:4317
      http:
        endpoint: ${env:MY_POD_IP}:4318
  prometheus:
    config:
      scrape_configs:
      - job_name: opentelemetry-collector
        scrape_interval: 10s
        static_configs:
        - targets:
          - ${env:MY_POD_IP}:8888
  zipkin:
    endpoint: ${env:MY_POD_IP}:9411
service:
  extensions:
  - health_check
  - basicauth/otel
  pipelines:
    logs:
      exporters:
      - opensearch/otel
      processors:
      - batch
      receivers:
      - filelog
    metrics:
      exporters:
      - debug
      processors:
      - memory_limiter
      - batch
      receivers:
      - otlp
      - prometheus
    traces:
      exporters:
      - debug
      processors:
      - memory_limiter
      - batch
      receivers:
      - otlp
      - jaeger
      - zipkin
  telemetry:
    metrics:
      readers:
      - pull:
          exporter:
            prometheus:
              host: ${env:MY_POD_IP}
              port: 8888
```
기본값에 변경할 부분만 덮어쓰기하려면 -f values.yaml 을 사용하면 된다.