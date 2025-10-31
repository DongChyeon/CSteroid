---
date: 2025-09-26
user: Ywoosang
topic: "HTTP 캐시"
---

# HTTP 캐시

## 목차
1. [용어 정리](#용어-정리)
   - [HTTP 리소스 (Resource)](#http-리소스-resource)
   - [Cache-Control 헤더](#cache-control-헤더)
   - [ETag와 Last-Modified](#etag와-last-modified)
   - [CDN (Content Delivery Network)](#cdn-content-delivery-network)
2. [HTTP 캐시의 기본 원리](#http-캐시의-기본-원리)
   - [캐시의 생명 주기](#캐시의-생명-주기)
   - [캐시 유효 기간: max-age](#캐시-유효-기간-max-age)
   - [캐시 재검증 (Revalidation)](#캐시-재검증-revalidation)
3. [Cache-Control 헤더 값들](#cache-control-헤더-값들)
   - [no-cache와 no-store](#no-cache와-no-store)
   - [public과 private](#public과-private)
   - [s-maxage](#s-maxage)
4. [캐시 계층 구조](#캐시-계층-구조)
   - [브라우저 캐시](#브라우저-캐시)
   - [CDN 캐시](#cdn-캐시)
   - [CDN Invalidation](#cdn-invalidation)
5. [실제 적용 사례](#실제-적용-사례)
   - [HTML 파일 캐시 전략](#html-파일-캐시-전략)
   - [JS, CSS 파일 캐시 전략](#js-css-파일-캐시-전략)

## 용어 정리

### HTTP 리소스 (Resource)
HTTP에서 리소스란 웹 브라우저가 HTTP 요청으로 가져올 수 있는 모든 종류의 파일을 말함
- 대표적인 리소스: HTML, CSS, JS, 이미지, 비디오 파일 등
- 웹 성능 최적화의 핵심 대상이 되는 데이터 단위

### Cache-Control 헤더
HTTP 응답 헤더로, 캐시의 동작 방식을 제어하는 지시어를 포함함
- 브라우저와 중간 서버(CDN)가 리소스를 어떻게 캐시할지 결정
- 웹 성능과 비용 최적화의 핵심 설정

### ETag와 Last-Modified
캐시 재검증에 사용되는 HTTP 헤더들
- **ETag**: 리소스의 고유 식별자 (해시값 등)
- **Last-Modified**: 리소스가 마지막으로 수정된 시간
- 서버에서 리소스 변경 여부를 확인하는 데 사용

### CDN (Content Delivery Network)
지리적으로 분산된 서버 네트워크로, 사용자에게 가까운 위치에서 콘텐츠를 제공함
- 원본 서버의 부하를 줄이고 응답 속도를 향상
- 중간 캐시 역할을 수행하여 네트워크 트래픽 감소

## HTTP 캐시의 기본 원리

웹 성능을 최대한으로 높이기 위해 HTTP 캐시를 사용함
캐시를 잘못 관리했을 때, 원하는 시점에 캐시가 사라지지 않을 수 있으며 필요 이상으로 HTTP 요청이 발생하기도 할 수 있음
HTTP 캐시를 효율적으로 관리하려면 Cache-Control 헤더에 적절한 값을 설정해야 함

### 캐시의 생명 주기

웹 브라우저가 서버에서 지금까지 요청한 적이 없는 리소스를 가져오려고 할 때, 서버와 브라우저는 완전한 HTTP 요청/응답을 주고받음
- HTTP 요청도 완전하고, 응답도 완전함
- 이후 HTTP 응답에 포함된 Cache-Control 헤더에 따라 받은 리소스의 생명 주기가 결정됨

### 캐시 유효 기간: max-age

서버의 Cache-Control 헤더의 값으로 `max-age=<seconds>` 값을 지정하면, 이 리소스의 캐시가 유효한 시간은 `<seconds>` 초가 됨

#### 캐시의 유효 기간이 지나기 전
한 번 받아온 리소스의 유효 기간이 지나기 전이라면, 브라우저는 서버에 요청을 보내지 않고 디스크 또는 메모리에서만 캐시를 읽어와 계속 사용함

예시: `max-age=31536000` (1년) 설정 시
- JavaScript 파일이 1년 동안 캐시됨
- 개발자 도구에서 "(from memory cache)" 표시 확인 가능
- **중요**: 한번 브라우저에 캐시가 저장되면 만료될 때까지 캐시는 계속 브라우저에 남아 있게 됨
- CDN Invalidation을 포함한 서버의 어떤 작업이 있어도 브라우저의 유효한 캐시를 지우기는 어려움

**Note**: Cache-Control max-age 값 대신 Expires 헤더로 캐시 만료 시간을 정확히 지정할 수도 있음




### 캐시 재검증 (Revalidation)

캐시의 유효 기간이 지나면 캐시가 완전히 사라지게 되는것인가? => 그렇지는 않음
대신 브라우저는 서버에 조건부 요청(Conditional request)을 통해 캐시가 유효한지 재검증(Revalidation)을 수행함

#### 재검증 과정
1. **조건부 요청 헤더 전송**
   - `If-None-Match`: 캐시된 리소스의 ETag 값과 현재 서버 리소스의 ETag 값이 같은지 확인
   - `If-Modified-Since`: 캐시된 리소스의 Last-Modified 값 이후에 서버 리소스가 수정되었는지 확인

2. **서버 응답**
   - **캐시가 유효한 경우**: `304 Not Modified` 응답 (HTTP 본문 없음, 매우 빠름)
   - **캐시가 무효한 경우**: `200 OK` 응답 (새로운 리소스와 함께)

예시: 59.1KB 리소스의 캐시 검증을 위해 324바이트만의 네트워크 송수신

#### max-age=0 주의사항
정의대로라면 `max-age=0` 값이 Cache-Control 헤더로 설정되었을 때, 매번 리소스를 요청할 때마다 서버에 재검증 요청을 보내야 함
하지만 일부 모바일 브라우저의 경우 웹 브라우저를 껐다 켜기 전까지 리소스가 만료되지 않도록 하는 경우가 있음
- 네트워크 요청을 아끼고 사용자에게 빠른 웹 경험을 제공하기 위함
- 이 경우에는 웹 브라우저를 껐다 켜거나, `no-store` 값을 사용해야 함

## Cache-Control 헤더 값들

### no-cache와 no-store

이름은 비슷하지만 두 값의 동작은 매우 다름

#### no-cache
- 대부분의 브라우저에서 `max-age=0`과 동일한 뜻을 가짐
- 캐시는 저장하지만 사용하려고 할 때마다 서버에 재검증 요청을 보내야 함

#### no-store
- 캐시를 절대로 해서는 안 되는 리소스일 때 사용
- 캐시를 만들어서 저장조차 하지 말라는 가장 강력한 Cache-Control 값
- `no-store`를 사용하면 브라우저는 어떤 경우에도 캐시 저장소에 해당 리소스를 저장하지 않음

### public과 private

CDN과 같은 중간 서버가 특정 리소스를 캐시할 수 있는지 여부를 지정하기 위해 Cache-Control 헤더 값으로 `public` 또는 `private`을 추가할 수 있음

- **public**: 공유 캐시(CDN/프록시)와 브라우저 모두 저장 가능
- **private**: 사용자 브라우저만 캐시를 저장할 수 있음을 나타냄

기존과 max-age 값과 조합하려면 `Cache-Control: public, max-age=86400`과 같이 콤마로 연결할 수 있음

### s-maxage

중간 서버(CDN)에서만 적용되는 max-age 값을 설정하기 위해 `s-maxage` 값을 사용할 수 있음

예시: `Cache-Control: s-maxage=31536000, max-age=0`
- CDN에서는 1년동안 캐시되지만 브라우저에서는 매번 재검증 요청을 보내도록 설정

## 캐시 계층 구조

### 브라우저 캐시
- 사용자의 브라우저에 저장되는 캐시
- 메모리 캐시와 디스크 캐시로 구분
- 가장 빠른 접근 속도를 제공

### CDN 캐시
- 지리적으로 분산된 중간 서버의 캐시
- 원본 서버의 부하를 줄이고 응답 속도 향상
- 여러 CDN 계층이 존재할 수 있음

### CDN Invalidation

일반적으로 캐시를 없애기 위해서 "CDN Invalidation"을 수행한다고 이야기함
- CDN Invalidation은 CDN에 저장되어 있는 캐시를 삭제한다는 뜻
- 브라우저의 캐시는 다른 곳에 위치하기 때문에 CDN 캐시를 삭제한다고 해서 브라우저 캐시가 삭제되지는 않음

경우에 따라 중간 서버나 CDN이 여러 개 있는 경우도 발생하는데, 이 경우 전체 캐시를 날리려면 중간 서버 각각에 대해서 캐시를 삭제해야 함

이렇게 한번 저장된 캐시는 지우기 어렵기 때문에 Cache-Control의 max-age 값은 신중히 설정하여야 함

## 실제 적용 사례

- https://toss.tech/article/smart-web-service-cache

### HTML 파일 캐시 전략

일반적으로 HTML 리소스는 새로 배포가 이루어질 때마다 값이 바뀔 수 있음
때문에 브라우저는 항상 HTML 파일을 불러올 때 새로운 배포가 있는지 확인해야 함

**토스의 HTML 파일 캐시 전략**:
- `Cache-Control: max-age=0, s-maxage=31536000`
- 브라우저는 HTML 파일을 가져올 때마다 서버에 재검증 요청을 보냄
- CDN은 계속해서 HTML 파일에 대한 캐시를 가지고 있도록 함
- 배포가 이루어질 때마다 CDN Invalidation을 발생시켜 CDN이 서버로부터 새로운 HTML 파일들을 받아오도록 설정

### JS, CSS 파일 캐시 전략

JavaScript나 CSS 파일은 프론트엔드 웹 서비스를 빌드할 때마다 새로 생김
토스 프론트엔드 챕터는 임의의 버전 번호를 URL 앞부분에 붙여서 빌드 결과물마다 고유한 URL을 가지도록 설정함

**고유 URL 관리의 장점**:
- 같은 URL에 대해 내용이 바뀔 수 있는 경우는 없음
- 내용이 바뀔 여지가 없으므로 리소스의 캐시가 만료될 일도 없음

**토스의 JS, CSS 파일 캐시 전략**:
- `Cache-Control: max-age=31536000` (최대치)
- 새로 배포가 일어나지 않는 한, 브라우저는 캐시에 저장된 JavaScript 파일을 계속 사용

## 요약

캐시 설정을 섬세히 제어함으로써 사용자는 더 빠르게 HTTP 리소스를 로드할 수 있고, 개발자는 트래픽 비용을 절감할 수 있음

Cache-Control과 ETag 헤더를 리소스의 성격에 따라 잘 설정하는 것만으로 캐시를 정확하게 설정할 수 있음

### 핵심 포인트
1. **리소스 특성에 따른 캐시 전략**: HTML은 재검증, JS/CSS는 장기 캐시
2. **캐시 계층 이해**: 브라우저 캐시와 CDN 캐시의 차이점
3. **재검증 메커니즘**: ETag와 Last-Modified를 활용한 효율적인 캐시 검증
4. **CDN Invalidation**: 서버 측 캐시 관리의 중요성
