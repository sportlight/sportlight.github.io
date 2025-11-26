---
title: JPA N+1 문제, 발생 원인부터 QueryDSL까지 완벽 정리
date: 2024-05-20 12:00:00 +0900
categories: [Backend, Performance]
tags: [jpa, querydsl, optimization, database]
---

## 배경

사내 커머스 시스템의 '주문 내역 조회 API'를 모니터링하던 중, 트래픽이 몰리는 시간대에 DB CPU 사용률이 급증하는 현상을 발견했습니다.
로직을 분석해 보니 `Order`(주문) 엔티티를 조회할 때 연관된 `Member`(회원)와 `Delivery`(배송) 정보를 참조하는 과정에서 성능 병목이 발생하고 있었습니다.

## 문제 인식

단순히 쿼리가 느린 것이 아니라, 쿼리의 **개수**가 비정상적으로 많았습니다.
`Order`를 100건 조회(`SELECT 1회`)했을 뿐인데, 이후 연관된 데이터를 가져오기 위해 200회 이상의 추가 쿼리가 발생하는 **N+1 문제**였습니다.

흔히 "지연 로딩(LAZY)으로 설정하면 해결된다"고 오해하지만, N+1 문제는 **즉시 로딩(EAGER)**과 **지연 로딩(LAZY)** 모두에서 발생합니다.

* **즉시 로딩(EAGER):** `findAll()` 호출 시점에 즉시 N개의 추가 쿼리가 발생 (예측 불가능한 쿼리 폭탄).
* **지연 로딩(LAZY):** 조회 시점엔 프록시만 가져오지만, 루프를 돌며 객체를 사용할 때 쿼리 발생 (지연된 폭탄).

결국 로딩 전략 수정만으로는 근본적인 해결이 불가능함을 확인했습니다.

## 기술적 심층 분석

해결책을 적용하기 전, 각 기술의 동작 원리와 사이드 이펙트를 분석했습니다.

### 1. Fetch Join의 딜레마 (카테시안 곱)
가장 일반적인 해결책인 `Fetch Join`은 SQL의 `Inner Join`을 사용하여 데이터를 한 번에 가져옵니다. 하지만 `1:N` 관계를 조인할 경우, DB 결과 집합(Result Set)의 Row 수가 'N'개만큼 증폭되는 **카테시안 곱(Cartesian Product)** 현상이 발생합니다.

이로 인해 애플리케이션에서 **중복 엔티티**가 생성될 수 있으며, JPA는 이를 메모리에서 걸러내야 하는 오버헤드가 발생합니다.

### 2. MultipleBagFetchException
두 개 이상의 컬렉션(`List`)을 동시에 Fetch Join 하려고 하면, 하이버네이트는 "어떤 자식 데이터가 어떤 부모에 매핑되는지" 혼란을 겪으며 에러를 뱉습니다. 이는 데이터 정합성을 보장할 수 없기 때문입니다.

## 해결 및 적용 (솔루션 비교)

프로젝트 상황에 맞춰 4가지 해결 전략을 비교 검토했습니다.

### 전략 1: Fetch Join (가장 대중적인 방법)
JPQL을 직접 작성하여 연관 엔티티를 한 번에 조회합니다.

```java
// distinct를 사용하여 애플리케이션 레벨의 중복 제거
@Query("select distinct o from Order o join fetch o.member join fetch o.delivery")
List<Order> findAllWithMemberDelivery();
