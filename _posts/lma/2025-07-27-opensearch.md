---
layout: post
title: "opensearch 설치와 개념"
date: 2025-07-27 19:00:00 +0900
categories: [opensearch, lma]
---
# Opensearch란?
대량의 데이터를 저장하고, 빠르게 검색·분석할 수 있는 시스템
- 문서(document) 형태의 데이터를 **인덱스**라는 단위로 저장.
- JSON 기반 API로 데이터를 추가, 수정, 삭제, 조회(필터링, 집계 등) 가능.
- 클러스터를 구성하면 데이터를 여러 노드와 **샤드**로 분산 
- **OpenSearch Dashboards**를 통해 데이터 검색, 분석, 시각화 대시보드를 구축 가능.

### 동작 과정

로그 수집기(opentelemetry-collector 등) → OpenSearch 인덱스 → 쿼리 & 시각화 

## 인덱스
데이터를 저장하고 검색하기 위한 기본 단위

1. 데이터 구분과 관리<br>
로그, 문서, 이벤트 등 다양한 데이터를 유형별로 나누어 저장한다.
예를 들어, web-logs, app-logs, security-logs처럼 데이터 종류별로 인덱스를 만들어 관리하면
검색이나 삭제, 권한 관리가 쉽다.

2. 성능 최적화<br>
인덱스는 내부적으로 여러 샤드(shard)로 나뉘어 분산 저장된다.
인덱스를 잘 나누면 대규모 데이터를 병렬로 처리할 수 있어 검색 속도가 빨라진다.

3. 수명 주기 관리 (Index Lifecycle Management)<br>
특정 기간 이후 오래된 인덱스는 삭제하거나 다른 저장소로 옮길 수 있다.
예를 들어, 3개월 지난 로그 인덱스는 자동으로 삭제하는 정책을 설정할 수 있다.

4. 검색 쿼리의 대상 지정<br>
쿼리할 때 특정 인덱스를 지정해야 원하는 데이터만 빠르게 찾을 수 있다.
예: GET /web-logs-2025.07.27/_search → 해당 날짜 웹 로그만 검색


## 샤드
인덱스를 여러 조각으로 나누어 저장하는 단위<br>

대규모 데이터를 빠르게 검색하기 위해 데이터를 분산 저장한다. 하나의 인덱스가 너무 크면 서버 하나에 다 넣을 수 없으니, 여러 조각으로 나누어 여러 서버에 분산 저장한다. 각 샤드는 독립적인 Lucene 검색 엔진 인스턴스로, 데이터를 병렬로 저장하고 검색할 수 있어 성능이 향상된다.

### Primary shard
- 원본 데이터를 저장하는 샤드.
- 인덱스를 만들 때 프라이머리 샤드 개수를 지정 (기본 1개, 보통 1~5개).

### Replica shard
- 프라이머리 샤드의 복제본.
- 장애가 발생했을 때 데이터 유실을 방지하고, 검색 요청 처리 속도를 높이는 역할.

### 동작 원리 예시
logs-2025.07.27 인덱스에 100GB 데이터가 저장된다고 가정.<br>
#### 프라이머리 샤드를 5개로 설정
각 샤드는 약 20GB씩 데이터를 저장. → 클러스터에 5개 노드가 있다면 샤드가 노드별로 분산 저장됨. → 검색 시 5개 샤드가 동시에 병렬로 작업해 응답 속도가 빨라짐.

### 샤드 개수 설정 시 고려할 점
- 너무 적으면 한 샤드가 너무 커져 성능 저하 발생.
- 너무 많으면 관리 오버헤드 증가 및 메모리 낭비.
- 일반적으로 인덱스의 예상 크기와 클러스터 노드 수에 따라 결정.


# Opensearch 설치 방법
로컬환경에서 바이너리 파일을 다운받아 직접 설치하거나 Docker로 실행할 수 있다. 
가상머신에 설치할 수도 있고(단일 노드 클러스터) k8s 에 다중 노드로 클러스터를 구성할 수도 있다. <br>
이 문서에서는 k8s에 설치하는 방법을 정리했다.

```
helm repo add opensearch https://opensearch-project.github.io/helm-charts/
helm repo update
helm install my-opensearch opensearch/opensearch --version 2.9.0 -f values.yaml
```
helm install할 때, 초기 설정해야할 것들이 있다,
```
// values.yaml
replicas: 1

clusterName: "opensearch-cluster"

opensearchJavaOpts: "-Xms512m -Xmx512m"

config:
  opensearch.yml: |
    cluster.name: "opensearch-cluster"
    network.host: 0.0.0.0
    discovery.type: single-node

# (필수!!) 기본 admin 계정 설정
extraEnvs:
  - name: OPENSEARCH_INITIAL_ADMIN_PASSWORD
    value: "MySecurePassword123!"
```
로컬환경에 설치한 vm에서 띄웠기때문에 LB가 없다. port-forwarding으로 호스트 PC에서 접속해야한다.

```
kubectl patch svc opensearch-cluster-master -n lam -p ‘{“spec”:{“type”:”NodePort”}}’
kubectl port-forward svc/opensearch-cluster-master -n lm 9200:9200
```
```
// Host PC
https://<노드 IP>:<port>
```