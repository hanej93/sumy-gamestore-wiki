# API 개요

store가 제공하는 REST API의 규모와 공통 규칙을 정리했습니다. 프론트는 게이트웨이(`:8080`)를 거쳐 호출하고, OpenAPI 스펙에서 TypeScript 타입을 생성해 씁니다.

## 규모

컨트롤러의 매핑 애노테이션을 직접 센 값입니다.

| 구분 | 엔드포인트(메서드+경로) | 고유 경로 |
|---|---|---|
| 전체(store) | 64 | 52 |
| 사용자용 | 33 | 26 |
| 관리자용(`/api/admin/**`) | 31 | 26 |

- 결제 웹훅 1개(`POST /api/payments/portone/webhook`)는 PG 서버가 부르는 경로라 OpenAPI 스펙에서 뺐습니다. 그래서 스펙에는 엔드포인트 63개, 경로 51개가 나옵니다.
- 소셜 로그인 경로(`/oauth2/**`, `/login/oauth2/**`)는 Spring Security가 처리하므로 위 수에 넣지 않았습니다.
- 게이트웨이와 알림 서비스에는 업무 API가 없습니다(Actuator만 엽니다).

## 도메인별 경로 묶음

| 도메인 | 기본 경로 | 사용자 | 관리자 | 문서 |
|---|---|---|---|---|
| 인증 | `/api/auth` | 3 | - | [회원·인증](domains/member.md) |
| 회원 | `/api/members`, `/api/admin/members` | 2 | 5 | [회원·인증](domains/member.md) |
| 카탈로그 | `/api/games`, `/api/categories`, `/api/admin/catalog` | 3 | 9 | [카탈로그](domains/catalog.md) |
| 업로드 | `/api/uploads` | 1 | - | [카탈로그](domains/catalog.md) |
| 주문·결제 | `/api/orders`, `/api/payments/portone` | 6 | - | [주문·결제](domains/order-payment.md) |
| 라이브러리 | `/api/library` | 2 | - | [라이브러리·위시리스트](domains/library-wishlist.md) |
| 위시리스트 | `/api/wishlist` | 4 | - | [라이브러리·위시리스트](domains/library-wishlist.md) |
| 리뷰 | `/api/games/{id}/reviews`, `/api/reviews`, `/api/admin/reviews` | 7 | 4 | [리뷰·신고](domains/review-report.md) |
| 문의·소식 | `/api/inquiries`, `/api/news`, `/api/admin/support` | 5 | 10 | [문의·공지](domains/inquiry-news.md) |
| 매출 | `/api/admin/sales` | - | 3 | [관리자 기능](domains/admin.md) |
| 합계 | | 33 | 31 | |

## 공통 규칙

### 인증

- **세션 쿠키(`SUMYSESSION`) 방식입니다.** 세션은 Spring Session으로 Redis에 저장해, store를 여러 대로 띄워도 로그인이 유지됩니다. 기본 만료는 30분입니다.
- 쿠키는 `HttpOnly`, `Secure`(로컬 제외), `SameSite=Lax`입니다.
- **기본값은 닫힘입니다.** 공개 경로(카탈로그·카테고리·리뷰 목록·소식 조회, 가입, 로그인, 로그아웃, 웹훅)만 열거하고 나머지는 인증을 요구합니다. 관리자 경로는 `ADMIN` 권한이 필요합니다.
- **회원 식별자는 항상 세션에서 가져옵니다.** 요청 본문이나 쿼리로 받지 않습니다.
- 비로그인은 401, 권한 부족은 403입니다. 로그인 페이지로 리다이렉트하지 않습니다. 로그인 실패만 예외적으로 400입니다(401은 "세션 없음"으로 이미 쓰이고 있어서, D19).

### 오류 응답: ProblemDetail

모든 4xx·5xx는 RFC 9457 ProblemDetail 형식입니다.

```json
{ "type": "urn:sumy:problem:birth-date-required", "title": "...", "status": 409,
  "detail": "생년월일을 등록해야 구매할 수 있는 게임이 있습니다: ..." }
```

검증 실패(400)에는 `errors: [{ "field": "size", "message": "..." }]`가 더 붙습니다.

| 상태 | 언제 |
|---|---|
| 400 | 요청 형식·검증 실패. `errors[]`에 필드별 메시지 |
| 401 / 403 | 비로그인 / 권한 부족 |
| 404 | 없는 리소스. **남의 주문·문의·리뷰도 404**(403이면 존재 여부가 샘) |
| 409 | 상태 규칙 위반(이미 보유, 판매 중 아님, 이미 처리된 신고 등) |
| 503 | PG 타임아웃·5xx처럼 결과를 알 수 없는 외부 실패. 상태는 바뀌지 않았으니 다시 시도 |
| 500 | 예상하지 못한 예외. 내부 사정은 응답에 싣지 않고 로그로만 |

- 프론트가 분기해야 하는 409에는 `type`(`urn:sumy:problem:<code>`)을 붙입니다. 프론트가 `detail` 문자열을 비교하지 않게 하기 위해서입니다. 그 밖의 409는 `type`이 기본값입니다.
- 모든 응답에 `X-Trace-Id` 헤더가 붙습니다. 이 값으로 게이트웨이부터 알림 서비스까지 로그를 한 번에 찾습니다.
- 사용자에게 보이는 문구는 메시지 파일에 두고, 로케일은 한국어로 고정했습니다(D16).

### 목록: 공통 페이지 봉투

```json
{ "items": [ ... ], "totalCount": 42, "page": 0, "size": 20, "hasNext": true }
```

- 목록 API는 `PageResponse<T>` 하나를 씁니다(리뷰 목록만 평균 별점을 더한 별도 응답). 배열을 최상위로 돌려주지 않습니다. 나중에 메타데이터를 붙일 자리가 없어지기 때문입니다.
- `page`는 0부터, `size`는 1~100입니다. 범위를 벗어나면 조용히 보정하지 않고 400입니다(D33).

### 요청 객체 바인딩

- **쿼리 파라미터를 `@RequestParam`으로 낱개로 받지 않습니다.** 요청 객체(`*Request`)로 묶고 `@ParameterObject @ModelAttribute @Valid`로 바인딩합니다. 검증 규칙은 요청 객체의 jakarta validation 애노테이션에 모읍니다. 요청 형태가 곧 OpenAPI 문서가 되고, `@ParameterObject` 덕분에 스펙에 파라미터가 하나씩 펼쳐집니다.
- 응답은 `*Response` DTO로만 내보냅니다. 도메인 객체나 JPA 엔티티를 그대로 내보내지 않습니다(아키텍처 테스트가 엔티티 유출을 막습니다).
- 값 하나만 돌려주는 응답도 맵이 아니라 이름 있는 DTO입니다(D18).

!!! warning "아직 맞추지 못한 곳"
    리뷰 목록, 위시리스트, 라이브러리 목록 3곳은 아직 `page`·`size`를 `@RequestParam`으로 받고, 범위 밖 값을 400 대신 보정합니다. 정리 대상으로 남겨 두었습니다.

### 값 표기

- 금액은 원 단위 정수입니다. 소수점이 없습니다.
- 시각은 ISO-8601 UTC 문자열입니다.
- `null`이 올 수 있는 응답 필드는 스펙에 nullable로 표시합니다. 표시가 빠져 화면에 1970년이 찍힌 문제를 겪고 바로잡았습니다([트러블슈팅 4](../troubleshooting/04-spec-vs-response.md)).

### 멱등과 재시도

| 요청 | 다시 보내면 |
|---|---|
| `POST /api/orders/{id}/confirm` | 한 번만 확정. 현재 상태를 돌려줌 |
| `POST /api/orders/{id}/cancel` | 이미 취소됐으면 200(D34) |
| `PUT /api/members/me/birth-date` | 같은 날짜면 200 |
| `PUT /api/reviews/{id}/like` | 한 번만 반영 |
| `PUT /api/wishlist/games/{id}` | 이미 담김이면 409(프론트는 담긴 상태로 처리) |
| PG 웹훅 | 같은 웹훅은 한 번만 처리 |

## 스펙 품질을 지키는 장치

- springdoc으로 OpenAPI 3.1 스펙을 만들고, 프론트 저장소로 내보내는 스크립트가 있습니다.
- 오류 응답 스키마는 컨트롤러마다 적지 않고 한 곳에서 모두 ProblemDetail로 덮어씁니다(D17). 적지 않으면 springdoc이 성공 응답 스키마를 복사해 넣기 때문입니다.
- 스모크가 매번 스펙을 받아 검사합니다. 응답에 자유형 맵이 없는지, 선택 필드에 nullable 표시가 있는지, 오류 응답이 정해진 형식인지 봅니다. 관리자 경로 전체의 401·403도 같은 스펙에서 뽑아 검사합니다([관리자 기능](domains/admin.md)).

## 관련 문서
- [기능 지도](domains/index.md)
- [아키텍처](architecture.md)
- [데이터 모델](data-model.md)
- [트러블슈팅 4: API 스펙과 실제 응답의 차이](../troubleshooting/04-spec-vs-response.md)
