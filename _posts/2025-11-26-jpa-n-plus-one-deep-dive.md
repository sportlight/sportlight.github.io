---
title: JPA N+1 문제, 원인 분석부터 QueryDSL 활용까지 완벽 정리
date: 2025-11-26 12:00:00 +0900
categories: [Backend, JPA]
tags: [optimization, querydsl, database, performance]
math: true
---

## 1. 배경 (Background)

사내 커머스 시스템의 '주문 내역 조회 API' 성능 튜닝 과정에서 발생한 이슈를 기술합니다.
트래픽이 급증하는 시간대에 DB CPU 사용률이 비정상적으로 치솟는 현상이 발견되었으며, 분석 결과 `Order`(주문) 엔티티를 조회할 때 연관된 `Member`(회원)와 `Delivery`(배송) 정보를 참조하는 과정에서 성능 병목이 확인되었습니다.

## 2. 현상 및 문제 인식 (Problem Definition)

### 2.1. N+1 문제의 정의
단순히 쿼리가 느린 것이 아니라, 실행되는 **쿼리의 횟수**가 기하급수적으로 늘어나는 현상입니다.
하이버네이트 로그를 분석한 결과, 상위 엔티티(`Order`)를 100건 조회했을 뿐인데, 연관 엔티티를 조회하기 위한 추가 쿼리가 200회 이상 실행되었습니다.

> **수식 표현:** $Query Count = 1 (Root) + N (Member) + N (Delivery)$

### 2.2. 흔한 오해: 즉시 로딩(EAGER) vs 지연 로딩(LAZY)
흔히 "지연 로딩으로 설정하면 N+1 문제가 해결된다"고 오해하지만, 이는 사실이 아닙니다. 두 방식 모두 N+1 문제가 발생하며, 발생 시점만 다를 뿐입니다.

* **즉시 로딩 (FetchType.EAGER):** `findAll()` 호출 시점에 즉시 N개의 추가 쿼리가 발생 (예측 불가능한 쿼리 폭탄).
* **지연 로딩 (FetchType.LAZY):** 조회 시점엔 프록시(Proxy)만 가져오지만, 로직에서 객체를 사용하는 시점에 쿼리 발생 (지연된 폭탄).

결국 글로벌 로딩 전략(`LAZY`)만으로는 근본적인 해결이 불가능하며, 별도의 **Fetch 전략**이 필요합니다.

## 3. 기술적 심층 분석 (Deep Dive)

단순한 해결책 적용에 앞서, 기술적 제약 사항과 부작용(Side Effect)을 분석했습니다.

### 3.1. 카테시안 곱 (Cartesian Product)
가장 일반적인 해결책인 `Fetch Join`은 SQL의 `Inner Join`을 사용합니다.
하지만 `1:N` (일대다) 관계를 조인할 경우, DB 결과 집합(Result Set)의 Row 수가 'N'개만큼 증폭되는 **데이터 뻥튀기 현상**이 발생합니다. JPA는 이를 애플리케이션 메모리에서 중복을 제거(Distinct)해야 하므로 성능 비용이 듭니다.

### 3.2. 페이징 처리 불가 (In-Memory Paging)
`OneToMany` 관계를 Fetch Join 하면서 페이징(`Pageable`)을 시도하면, 하이버네이트는 경고 로그를 남기고 **모든 데이터를 메모리로 로딩한 뒤 페이징**을 수행합니다. 이는 대용량 데이터 환경에서 `OutOfMemoryError`를 유발하는 치명적인 원인이 됩니다.

## 4. 해결 전략 비교 (Solution Comparison)

프로젝트 환경에 적합한 최적의 해를 찾기 위해 4가지 방식을 비교했습니다.

### 전략 A: Fetch Join (JPQL)
JPQL을 직접 작성하여 연관 엔티티를 한 번의 쿼리로 조회합니다.

```java
// distinct를 사용하여 애플리케이션 레벨의 중복 엔티티 제거
@Query("select distinct o from Order o join fetch o.member join fetch o.delivery")
List<Order> findAllWithDetails();
```

* **장점:** 가장 직관적이며 쿼리 1방으로 해결 가능.
* **단점:** `FetchType`을 하드코딩해야 하며, 페이징 쿼리가 불가능함.

### 전략 B: @EntityGraph
JPQL 작성 없이 어노테이션으로 Fetch Join을 수행합니다.

```java
@EntityGraph(attributePaths = {"member", "delivery"})
List<Order> findAll();
```

* **장점:** 코드가 간결함.
* **단점:** `Left Outer Join`이 강제됨 (Fetch Join은 Inner Join 가능).

### 전략 C: Batch Size (In-Query)
`IN` 절을 사용하여 설정한 사이즈만큼 데이터를 미리 로딩합니다. N+1 문제를 $1 + (N/batch\_size)$ 문제로 완화합니다.

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 1000
```

* **장점:** **페이징 처리가 가능**하며, 컬렉션 조인 시 데이터 증폭 문제가 없음.
* **단점:** 쿼리가 완전히 1방은 아님.

### 전략 D: QueryDSL
컴파일 시점의 타입 안정성을 보장하며, 동적 쿼리와 Fetch Join을 유연하게 구현합니다.

```java
return queryFactory
    .selectFrom(order)
    .join(order.member, member).fetchJoin()
    .join(order.delivery, delivery).fetchJoin()
    .fetch();
```

## 5. 최종 적용 및 최적화 (Optimization Strategy)

본 프로젝트는 **대량 데이터의 페이징 처리**가 필수 요구사항이었기에, 다음과 같은 이원화된 전략을 수립하여 적용했습니다.

### 5.1. To-One 관계 (ManyToOne, OneToOne)
데이터 증폭 이슈가 없으므로 **Fetch Join (QueryDSL)**을 적극 사용하여 쿼리 수를 1회로 단축시켰습니다.

### 5.2. To-Many 관계 (OneToMany)
페이징 이슈와 데이터 중복(Cartesian Product)을 방지하기 위해 Fetch Join을 사용하지 않았습니다. 대신 **`default_batch_fetch_size: 1000`** 옵션을 적용하여 성능을 방어했습니다.

### 5.3. 조회 전용 로직 (DTO Projection)
엔티티의 변경이 필요 없는 단순 조회 화면에서는 엔티티를 거치지 않고 **DTO로 직접 조회**하여 영속성 컨텍스트 관리 비용을 제거했습니다.

```java
// QueryDSL Projections 활용
List<OrderDto> result = queryFactory
    .select(Projections.constructor(OrderDto.class, 
        order.id, member.name, delivery.address))
    .from(order)
    .join(order.member, member)
    .join(order.delivery, delivery)
    .fetch();
```

## 6. 결론 (Conclusion)

N+1 문제는 JPA를 사용하는 한 피할 수 없는 과제입니다. 이번 튜닝을 통해 무조건적인 Fetch Join 사용이 정답은 아님을 확인했습니다.

* **실시간 단건/소량 조회:** Fetch Join (QueryDSL) 활용
* **대량 데이터/페이징 조회:** Batch Size 활용
* **통계/단순 조회:** DTO Projection 또는 Native SQL 활용

데이터의 관계(Cardinality)와 비즈니스 요구사항(Paging 여부)에 따라 적절한 도구를 선택하는 것이 엔지니어링의 핵심입니다.
