---
layout: post
title: "Composite Key"
date: 2025-08-09 12:00:00 +0900
categories: [Spring Security, Composite Key]
---
1. @EmbeddedId와 @Embeddable (객체지향에 더 가까운 방법)
복합키 필드들을 하나의 별도 클래스(@Embeddable)로 묶어서 다룹니다.
```
@Embeddable
public class MetricId implements Serializable {
    private String api;
    private String stage;
    // ...
}

@Entity
public class Metric {
    @EmbeddedId
    private MetricId id;
}
```
2. @IdClass (RDB에 더 가까운 방법)