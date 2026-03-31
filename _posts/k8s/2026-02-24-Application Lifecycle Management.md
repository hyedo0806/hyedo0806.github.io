---
title: Application Lifecycle Management
date: 2025-10-19 23:59:59
categories: [Spring Security]
tags: [security]     # TAG names should always be lowercase
---
## Deployment Strategy
- rollingUpdateStrategy: 25% max unavailable, 25% max surge
  - max unavailable : 업데이트 중 동시에 "사용 불가능" 상태가 될 수 있는 최대 Pod 수
  - max surge :   업데이트 중 일시적으로 추가 생성할 수 있는 Pod 수
- StrategyType: 
  - recreate --> rollingUpdateStrategy 필요 XX
  - rolling update(default)


```
// replicaset으ㄹ 하나씩 revert
kubectl rollout undo deployment {}
```

```
kubectl apply -f manifest.yml
kubectl set image deployment/{} {containter_name}={image}
```

```
kubectl rollout status deployment/{}
```

