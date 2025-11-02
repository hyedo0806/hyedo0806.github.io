---
title: annotation
date: 2025-10-19 23:59:59
categories: [Spring]
tags: [TAG]     # TAG names should always be lowercase
---

# @Entity
이 클래스가 DB 테이블과 매핑되는 JPA 엔티티 임을 나타낸다.<br>
JPA가 이 클래스는 보고 DB 테이블을 생성하거나 매핑할 수 있게 한다. <br>

### JPA
Java Persistence API으로 자바 객체를 데이터베이스의 테이블에 자동으로 매핑하는 기술<br>
즉, 
```
Task task = new Task("할 일 추가");
```
처럼 자바 객체를 만들면 JPA가 자동으로 이 객체를 SQL으로 변환해서 
```
INSERT INTO task (contents) VALUES ('할 일 추가');
```
를 대신 실행한다.<br>
JPA의 역할은 Object-Relational Mapping 기술이다. 객체(Object)와 관계형 데이터베이스(Relational Database)의 데이터를 서로 변환해주는 기술이다. 

# @NoArgsConstructor
기본 생성자를 자동으로 만들어준다.<br>
JPA는 내부적으로 리플렉션을 사용하여 객체를 생성하므로 엔티티 클래스에 기본 생성자가 반드시 있어야 한다. <br>
DTO에서 Json 역직렬화 시에도 필요할 수 있다.<br>

### 직렬화(Serialization)와 역직렬화(Deserialization)
객체(Object) 를 다른 형태(주로 데이터 스트림) 로 변환하고,
그걸 다시 객체로 복원하는 과정

**직렬화(Serialization)**<br>
객체를 바이트(byte) 형태로 변환하는 과정<br>
자바 객체는 메모리 안에서만 존재하므로, 네트워크로 보내거나(API 응답) 파일로 저장(캐싱, 로그 등)하려면 전송 가능한 형태로 바꿔야 한다. <br>
**역직렬화(Deserialization)**<br>
바이트 데이터를 다시 객체로 복원하는 과정



# @SuperBuilder
Lombok의 @Builder를 상속 구조에서도 사용할 수 있게 확장.(@Builder는 상속된 필드를 다루지 못함)

---

```
@Data
@SuperBuilder
@NoArgsConstructor
public class TaskDto {
    private Long id;
    private String contents;
}
```
```
@Data
@SuperBuilder
@NoArgsConstructor
public class BaseDto {
    private LocalDateTime createdDate;
}

@Data
@SuperBuilder
@NoArgsConstructor
public class TaskDto extends BaseDto {
    private String contents;
}

TaskDto dto = TaskDto.builder()
    .createdDate(LocalDateTime.now())
    .contents("테스트")
    .build();
```

# @GeneratedValue(strategy = GenerationType.IDENTITY)
엔티티의 기본 키(id)를 데이터베이스가 자동 생성하도록 지정하는 설정