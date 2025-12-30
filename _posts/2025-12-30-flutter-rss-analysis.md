---
title: "[Flutter] 뉴스 앱 개발기 - REST API 시대에 왜 다시 RSS인가? 기술적 분석과 전망"
date: 2025-12-30 10:00:00 +0900
categories: [Mobile-App, Architecture]
tags: [flutter, rss, xml, system-design, tech-trend, data-aggregation]
mermaid: true
---

## 1. 서론: 화려한 API 시대, 왜 '고대 유물' RSS인가?

최근 Flutter로 뉴스 애그리게이션(Aggregation) 앱을 개발하며 데이터 소싱 방식을 고민하던 중, `NewsAPI`나 `Mediastack` 같은 최신 REST API 대신 **RSS(Really Simple Syndication)**를 선택했다.

혹자는 RSS를 "2000년대의 유물"이라 부르지만, 엔지니어링 관점에서 RSS는 여전히 **가장 비용 효율적이고, 탈중앙화된 데이터 배포 프로토콜**이다. 본 포스팅에서는 최신 앱 개발 환경에서 RSS가 갖는 기술적 가치와 장단점, 그리고 향후 전망을 심층 분석한다.

## 2. 기술적 심층 분석 (Technical Deep Dive)

### 2.1. RSS의 아키텍처: Pull vs Push
현대적 뉴스 서비스가 FCM(Firebase Cloud Messaging) 등을 이용한 **Push** 방식이라면, RSS는 전형적인 **Pull** 방식의 폴링(Polling) 모델이다.

* **구조:** XML(Extensible Markup Language) 기반의 표준 규격 (RSS 2.0, Atom).
* **동작:** 클라이언트가 정해진 주기로 서버에 `GET` 요청을 보내고, 변경 사항(새로운 `<item>`)을 스스로 감지해야 한다.

```mermaid
sequenceDiagram
    participant User as Flutter App
    participant Isolate as Background Isolate
    participant Server as Media Server (RSS)

    loop Polling Cycle (e.g., 15min)
        User->>Server: HTTP GET /rss.xml
        Server-->>User: Return XML Body
        
        Note right of User: UI 스레드 블로킹 방지
        User->>Isolate: Spawn & Send XML string
        Isolate->>Isolate: XML Parsing & Regex (Image Extraction)
        Isolate-->>User: Return List<NewsItem>
        User->>User: Update UI State
    end
