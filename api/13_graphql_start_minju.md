# 13장. GraphQL 시작하기

- **GraphQL**은 선언적 쿼리이고 API용 조작 언어이면서 서버-사이드 런타임임. 클라이언트가 원하는 데이터를 정확하게 쿼리할 수 있도록 해줌.

## GraphQL과 REST 비교

| 구분 | REST | GraphQL |
|------|------|---------|
| **엔드포인트** | 리소스마다 별도 엔드포인트 | 단일 엔드포인트 |
| **응답 구조** | 서버가 고정된 구조로 응답 | 클라이언트가 원하는 필드만 요청 |
| **오버/언더페칭** | 발생 가능 | 방지 가능 |
| **캐싱** | 내장 HTTP 명세 사용 | Apollo/Relay 등 라이브러리 사용 |
| **버전 관리** | API 변경 시 버전 추가 필요 | 필드 추가/deprecated로 버전 없이 진화 |
| **타입 시스템** | 별도 명세(OpenAPI) 필요 | SDL로 스키마에 강력한 타입 정의 |

## GraphQL 기본 학습

| 루트 타입 | 설명 |
|-----------|------|
| **Query** | 읽기(Read) 작업. `query` 접두사 생략 시 익명 쿼리로 동작 |
| **Mutation** | 추가, 업데이트, 삭제 작업. `mutation` 키워드 생략 시 Query로 처리되어 에러 발생 |
| **Subscription** | 실시간 이벤트 스트림. 연결을 유지하며 서버 푸시를 수신. 데이터는 스트림으로 전송됨 |

> **서브스크립션을 권장하는 경우**
> 라이브 업데이트처럼 대기 시간이 짧은 경우에만 사용함. 그 외에는 **폴링**을 사용해야 함.

```graphql
# Query 예시
{ me { id username } }

# Mutation 예시
mutation { addItemInCart(productId: "abc", qty: 2) { id } }

# Subscription 예시
subscription { orderShipped(customerID: "xyz") { shipping { estDeliveryDate } } }
```

**아규먼트 종류**

| 종류 | 예시 |
|------|------|
| **필수** | `amount: Float` |
| **선택적 (디폴트 값 있음)** | `currency: String = "USD"` |
| **Non-null** | `id: ID!` — 항상 값을 제공 |
| **리스트** | `[Item]`, `[Item!]!` |

## GraphQL 스키마 설계

| 타입 | 설명 |
|------|------|
| **스칼라** | `Int`, `Float`, `String`, `Boolean`, `ID` — 커스텀/열거형(enum) 정의 가능 |
| **인터페이스** | 공통 필드를 추상화. 구현 타입에 `implements` 사용 |
| **유니온** | 공통 필드 없이 둘 이상의 구체적인 객체를 조합. `union A = B \| C` |
| **인풋** | 뮤테이션 아규먼트용 객체. `input` 키워드로 선언 |
| **프래그먼트** | 반복되는 필드 그룹을 추출해 재사용. `...fragmentName` |
| **인라인 프래그먼트** | Interface/Union 타입 필드 조회 시 사용. `... on TypeName { }` |

```graphql
# 열거형
enum OrderStatus { CREATED CONFIRMED SHIPPED DELIVERED CANCELLED }

# 인터페이스 + 인라인 프래그먼트
interface Product { id: ID!  name: String! }
type Book implements Product { id: ID!  name: String!  author: String! }

query getProducts {
  allProducts {
    id  name
    ... on Book { author }
  }
}

# 유니온 + __typename (객체 타입 구별용 내장 필드)
union SearchResult = Book | Author
search(text: "abc") { __typename ... on Book { name } ... on Author { name } }

# 인풋 타입 + 변수
input ProductInput { name: String!  price: Float! }
mutation AddProduct ($input: ProductInput) {
  addProduct(prodInput: $input) { name }
}
```

## N+1 문제 해결

- GraphQL은 각 필드에 해당 데이터를 가져오는 자체 **리졸버(resolver)** 기능을 가짐.
- 연관관계가 있는 데이터 조회 시, 상위 리졸버 1번 + 하위 리졸버 N번 실행되는 N+1 문제가 발생함.

```sql
SELECT * FROM ecomm.user;                              -- 1번: 모든 사용자 조회
SELECT * FROM ecomm.orders WHERE customer_id in (1);   -- N번: 각 사용자마다 실행
SELECT * FROM ecomm.orders WHERE customer_id in (2);
...
```

- 해결책: 모든 ID가 모이면 단일 호출로 한 번에 처리하는 **배치 처리** 방식을 사용해야 함.
- **DataLoader** (JS): <https://github.com/graphql/dataloader>
- **java-dataloader** (Java): <https://github.com/graphql-java/java-dataloader>
