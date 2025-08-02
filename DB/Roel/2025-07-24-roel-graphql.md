---
date: 2025-07-24
user: Roel4990
topic: GraphQL과 데이터 소비의 변화
---

# GraphQL과 데이터 소비의 변화

## 1. 데이터 저장 방식의 출발점: RDBMS와 NoSQL

### 1.1 RDBMS (관계형 데이터베이스)
- 데이터를 표 형태(테이블)로 구조화
- 정해진 스키마(열 구조)를 따름
- SQL을 통해 데이터 조작
- 트랜잭션(ACID)을 기본적으로 지원
- 예: MySQL, PostgreSQL, Oracle

### 1.2 NoSQL
- 비정형/반정형 데이터에 적합
- 유연한 스키마 구조 (document, key-value, columnar, graph 등)
- 수평 확장에 유리
- 예: MongoDB, Redis, Cassandra

> 💡 참고: RDBMS/NoSQL은 데이터를 "어떻게 저장하고 관리할까"에 초점

---

## 2. 데이터 소비의 방식: REST API

### 2.1 REST 개요
- 각 리소스는 URL 단위로 관리됨 (`/users`, `/users/1/posts`)
- HTTP 메서드 기반 (GET, POST, PUT, DELETE)
- 응답 데이터는 서버가 정함 (고정된 형태)

### 2.2 REST의 한계

| 문제 | 설명 | 예시 |
|------|------|------|
| Over-fetching | 필요 이상 데이터가 전송됨 | 사용자 이름만 필요해도 주소, 이메일 등 전체 전송 |
| Under-fetching | 여러 번 요청해야 완전한 데이터 확보 | 사용자 → 게시글 → 댓글을 위해 다수 API 호출 필요 |
| 버전 관리 어려움 | 응답 형태 변경 시 `/v1`, `/v2` 등 분기 필요 | 프론트엔드와 백엔드 간 조율이 복잡해짐 |

---

## 3. GraphQL의 등장 배경

- **Facebook**이 모바일에서 REST API의 한계를 해결하기 위해 2012년 내부 개발 → 2015년 공개
- **문제의식**: "필요한 데이터만, 하나의 요청으로, 원하는 구조로 받고 싶다"
- **해결 전략**: 쿼리 언어로 API 요청을 선언적으로 구성

---

## 4. GraphQL의 핵심 개념

| 요소 | 설명 |
|------|------|
| **Query** | 필요한 필드만 선택해서 요청 |
| **Mutation** | 데이터를 수정, 추가, 삭제하는 요청 |
| **Schema** | 데이터 타입 및 관계를 명확히 선언 |
| **Resolver** | 요청 필드를 실제 데이터로 매핑하는 함수 |
| **단일 Endpoint** | 모든 요청은 `/graphql` 하나로 처리 |

```graphql
query {
  user(id: 1) {
    name
    posts {
      title
      comments {
        text
      }
    }
  }
}
```
> 💡 위 쿼리는 REST라면 3~4번 요청이 필요하지만, GraphQL에서는 한 번에 가져올 수 있음

---

## 5. REST vs GraphQL 비교

| 항목 | REST | GraphQL |
|------|------|---------|
| 요청 수 | 여러 endpoint | 단일 endpoint |
| 응답 구조 | 고정형 | 선택형 |
| 데이터 과다/부족 | 빈번함 | 없음 |
| API 버전 관리 | 명시적 버전 필요 | 스키마 진화 방식 |
| 클라이언트 유연성 | 제한적 | 높음 |

---

## 6. GraphQL 사용 시 주의점

### 6.1 장점
- 클라이언트가 원하는 데이터만 가져옴 (Over/Under-fetching 해결)
- 프론트엔드 주도 개발 가능 (프론트 요구사항에 맞춘 쿼리)
- 직관적인 API 구조

### 6.2 단점
- 서버 구현 복잡도 ↑ (Resolver 관리)
- 쿼리 복잡성에 따른 보안/성능 이슈 (e.g., Depth Limit)
- 캐싱 어려움 (기존 REST 기반 CDN 캐시 불가)

---

## 7. GraphQL이 잘 맞는 경우

| 상황 | 설명 |
|------|------|
| 다양한 클라이언트 대응 | 모바일/웹/태블릿 등 각기 다른 요청 구조 필요 |
| 빠르게 변하는 UI | 화면 요소에 따라 유연한 쿼리 필요 |
| 마이크로서비스 통합 | 여러 서비스의 데이터를 한 번에 조회 |

---

## 8. 사용 예시 (Apollo Server 기반)

```javascript
const typeDefs = gql\`
  type User {
    id: ID!
    name: String!
    posts: [Post!]!
  }

  type Post {
    id: ID!
    title: String!
  }

  type Query {
    user(id: ID!): User
  }
\`;

const resolvers = {
  Query: {
    user: (_, { id }) => db.getUserById(id),
  },
  User: {
    posts: (parent) => db.getPostsByUser(parent.id),
  }
};
```

---

## 9. 결론

GraphQL은 단순히 REST를 대체하는 기술이 아니라,  
“**클라이언트와 백엔드 간의 데이터 요청 계약을 재정의**”한 접근이다.

- 저장 방식(RDB/NoSQL)은 GraphQL과는 별개
- GraphQL은 "어떻게 데이터를 줄 것인가"에 대한 선언적 해결책
- REST와 GraphQL은 상호 보완적이며, 상황에 따라 하이브리드도 가능

---

GraphQL에서 가장 인상 깊었던 점은 프론트엔드에서 필요한 데이터만 정확히 요청할 수 있다는 점이다.
기회가 된다면 한번 써보고 싶은 기술이다.