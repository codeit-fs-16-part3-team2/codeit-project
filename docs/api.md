# 최애의 포토 API 계약

[프로젝트 범위](plan.md), [결정 기록](ADR.md), [ERD](ERD.md)를 기준으로 정리한 계약 초안이다. [예시 요청](api.http)은 판매자와 신규 구매자의 요청을 순서대로 재현한다.

## 공통 규칙

- 브라우저는 현재 Next.js 사이트의 `/api`로 요청한다. Next.js가 Render의 Express API에 전달한다. 본문의 모든 주소는 브라우저에서 보는 주소다.
- 요청 본문과 응답은 JSON을 사용한다. ID는 문자열, 시간은 UTC ISO 8601 문자열, 포인트와 수량은 정수다.
- 성공은 `{ "data": ... }`, 실패는 `{ "error": { "code": "...", "message": "..." } }` 형태다. 클라이언트는 분기할 때 `error.code`를 사용한다.
- 회원가입과 로그인에 성공하면 서버가 세션을 발급한다. 쿠키 이름은 `session`이며 값은 서버가 발급한 불투명한 세션 토큰이다. 배포에서는 `HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=86400`를 사용하고 `Domain`은 생략한다. 만료 시각은 세션 발급으로부터 24시간 뒤로 고정한다. 로그아웃은 서버의 세션을 폐기하고 쿠키를 만료시킨다.
- `POST`와 `PATCH`의 `Origin`은 환경에 설정한 웹 사이트 출처와 일치해야 한다. 배포 요청 전달 시 `Origin`, `Cookie`, `Set-Cookie`를 유지한다. 회원 정보 및 인증 응답에는 `Cache-Control: no-store`를 적용한다.
- 회원 전용 API는 서버에서 세션과 당사자 권한을 검사한다. 판매자 ID와 구매자 ID는 서버에서 확인한다. 상태 변경은 트랜잭션에서 처리한다. 구매 결과를 받지 못했을 때 클라이언트는 구매 요청을 자동 재전송하지 않는다.

## API 계약 표

아래 접근 조건의 ‘회원’은 유효한 세션을 가진 사용자를 뜻한다. 목록 쿼리와 수정 필드의 선택 규칙은 표 아래에 정리했다. 경로명과 필드명, 페이지 크기 기본값은 검토용 초안이다.

| 기능 | 메서드와 주소 | 접근 | 요청 본문·쿼리 | 성공 응답의 `data` |
| --- | --- | --- | --- | --- |
| 가입 및 자동 로그인 | `POST /api/auth/signup` | 비로그인 | `email, nickname, password, passwordConfirmation` | `201` 회원. 가입 보너스 10P 지급, 세션 쿠키 설정 |
| 로그인 | `POST /api/auth/login` | 비로그인 | `email, password` | `200` 회원, 세션 쿠키 설정 |
| 로그아웃 | `POST /api/auth/logout` | 회원 | 본문 없음 | `200 { loggedOut: true }`, 세션 폐기 및 쿠키 만료 |
| 내 정보 | `GET /api/me` | 회원 | 없음 | `200` 회원. 현재 포인트 포함 |
| 판매 목록 | `GET /api/listings` | 회원 | `q, filterBy, filterValue, sort, cursor, limit` | `200 { items: 판매 요약[], nextCursor }` |
| 판매 상세 | `GET /api/listings/:listingId` | 회원. 취소된 판매글은 등록자만 열람 | 없음 | `200` 판매 상세 |
| 판매 등록 | `POST /api/listings` | 판매 가능한 카드를 보유한 회원 | 필수 `inventoryId, quantity, unitPrice`. 선택 `wantedGrade, wantedGenre, wantedDescription` | `201` 판매 상세 |
| 판매 수정 | `PATCH /api/listings/:listingId` | 등록자 | `unitPrice, wantedGrade, wantedGenre, wantedDescription` 중 하나 이상 | `200` 수정된 판매 상세 |
| 판매 취소 | `POST /api/listings/:listingId/cancel` | 등록자 | 본문 없음 | `200` 취소된 판매 상세 |
| 구매 | `POST /api/listings/:listingId/purchases` | 등록자가 아닌 회원 | `quantity, expectedUnitPrice` | `201` 구매 결과 |
| 교환 제안 조회 | `GET /api/listings/:listingId/exchange-offers` | 회원. 취소된 판매글은 등록자만 열람 | 없음 | `200 { items: 교환 제안[] }`. 등록자는 받은 제안, 다른 회원은 본인이 보낸 제안만 조회 |
| 교환 제안 | `POST /api/listings/:listingId/exchange-offers` | 등록자가 아닌 회원 | `offeredInventoryId` | `201` 교환 제안. 판매 대상과 종류가 다른 카드 한 장이 대기 수량에 반영됨 |
| 교환 승인 | `POST /api/exchange-offers/:offerId/approve` | 대상 판매글 등록자 | 본문 없음 | `200` 승인된 교환 제안. 양쪽 카드 한 장씩 이전 |
| 교환 거절 | `POST /api/exchange-offers/:offerId/reject` | 대상 판매글 등록자 | 본문 없음 | `200` 거절된 교환 제안 |
| 교환 제안 취소 | `POST /api/exchange-offers/:offerId/cancel` | 제안자 | 본문 없음 | `200` 취소된 교환 제안 |
| 마이 갤러리 | `GET /api/me/inventory` | 회원 | `q, grade, genre, sort, page, pageSize` | `200 { items: 갤러리 항목[], page, pageSize, totalItems }` |
| 나의 판매 포토카드 | `GET /api/me/sales` | 회원 | `grade, genre, method, soldOut, canceled, page, pageSize` | `200 { items: 내 판매 항목[], page, pageSize, totalItems }` |
| 내 알림 | `GET /api/me/notifications` | 회원 | `page, pageSize` | `200 { items: 알림[], page, pageSize, totalItems }`. 최신순 |
| 랜덤 상자 상태 | `GET /api/me/point-box` | 회원 | 없음 | `200 { eligible, nextEligibleAt }` |
| 랜덤 상자 열기 | `POST /api/me/point-box/open` | 회원 | 본문 없음 | `200 { awardedPoints, pointBalance, nextEligibleAt }` |

### 입력과 목록

- 회원가입 닉네임은 2~12자다. 비밀번호는 8~32자이며 공백 문자를 허용하지 않는다. 영문·숫자·특수문자 조합은 강제하지 않으며, 비밀번호와 비밀번호 확인 값은 일치해야 한다. 조건을 위반하면 `400 VALIDATION_ERROR`를 반환한다.
- 카드 등급은 `COMMON`, `RARE`, `SUPER RARE`, `LEGENDARY`다. 장르 허용 목록은 초기 시드 데이터 작성 때 확정한다. 시드 카드, 목록의 등급·장르 필터, 입력된 교환 희망 등급·장르에 같은 허용 목록을 적용한다.
- 가입에 실패하면 화면에 이메일과 닉네임을 유지하고 비밀번호와 비밀번호 확인을 비운다. 구매에 실패하면 선택 수량을 유지한다. 가격·재고가 바뀌었으면 판매 상세를 다시 조회해 최신 값을 보여주고, 현재 조건에서 구매할 수 있는 수량을 사용자가 다시 확인한 뒤 재요청한다.
- 등록·구매 수량은 양의 정수다. 장당 가격과 `expectedUnitPrice`는 0 이상의 정수다. 구매 요청의 `expectedUnitPrice`는 화면에서 확인한 장당 가격이다. 실제 가격과 다르면 사용자가 갱신된 가격을 확인한 뒤 다시 요청한다.
- 판매 등록의 입력 검증 실패(`400 VALIDATION_ERROR`)는 등록 화면에서 입력을 유지하고 수정한다. 입력 검증 이후 등록 처리 실패의 재시도는 실패 화면에서 시작하며, 등록 화면에서 직전 입력을 복원한다. 인증·접근 실패는 공통 접근 규칙을 따른다.
- 교환 희망 등급·장르·설명은 각각 선택이다. 판매 등록에서 미입력한 필드는 생략하거나 `null`로 보낼 수 있으며, 상세 응답은 이를 `null`로 반환한다. 화면에서 빈 입력은 미입력으로 처리한다.
- 판매 수정은 보낸 필드만 변경한다. 교환 희망 필드를 생략하면 기존 값을 유지하고, `null`을 보내면 해당 조건을 지운다. 남은 판매 수량은 구매와 교환 승인에서만 감소한다. 판매 취소나 마지막 판매 수량 소진은 같은 거래 안에서 남은 교환 제안을 자동 종료한다. 판매를 취소하면 `canceledAt`을 기록하고, 자동 종료된 대기 제안의 제안자에게 판매글당 `LISTING_CANCELED` 알림을 한 건씩 저장한다. 같은 회원의 대기 제안이 여러 건이어도 취소 알림은 한 건이다.
- 마켓플레이스의 쿼리는 모두 선택이다. `q`는 카드 이름 검색이며, `filterBy`는 `grade, genre, description, soldOut` 중 하나다. `filterBy`와 `filterValue`는 함께 제공한다. 등급·장르·설명은 판매 대상 `PhotoCard`의 `grade`, `genre`, `description`에 적용한다. 설명 필터는 부분 일치, 매진 여부 값은 `true` 또는 `false`다.
- 마켓플레이스의 `sort`는 `newest, oldest, priceAsc, priceDesc`이며 판매 등록 시각과 장당 가격 기준으로 정렬한다.
- 마이 갤러리의 `sort`는 `newest, oldest`다. 회원·카드 종류별 마지막 획득 시각(`Inventory.lastAcquiredAt`)을 내림차순 또는 오름차순으로 적용하고 동률은 `inventoryId` 오름차순으로 정렬한다. 두 목록의 기본값은 `newest`다.
- 마켓플레이스 첫 묶음은 `cursor` 없이 요청한다. 이후 응답의 불투명한 `nextCursor`를 `cursor`에 넣는다. 마지막 묶음의 `nextCursor`는 `null`이다. 검색·필터·정렬이 바뀌면 첫 묶음부터 요청한다. 동률은 ID를 보조 정렬 키로 사용한다. 정렬 기준 값이 바뀌는 동안 조회하면 같은 목록을 유지한다는 보장은 없으므로 필요하면 첫 묶음부터 갱신한다.
- 개인 목록 쿼리는 모두 선택이다. `page`는 1부터 시작한다. `limit` 또는 `pageSize` 기본값은 20, 허용 범위는 1~50이다. 빈 페이지도 `200`과 빈 `items`를 반환한다.
- 교환 제안은 제안에 사용하는 보유 행의 `photoCardId`와 판매 대상의 `photoCardId`가 달라야 한다. 같은 종류의 카드는 `SAME_CARD_EXCHANGE_NOT_ALLOWED`로 거절한다.
- 내 판매 목록은 본인의 판매 등록과 본인이 보낸 대기 중인 교환 제안을 조회한다. `method`는 `SALE, EXCHANGE`다. 기본 조회에서 선택 쿼리 `soldOut=true`는 `SOLD_OUT`, `soldOut=false`는 `ACTIVE`와 `PENDING`을 반환하며, 생략하면 세 상태를 모두 반환한다. `status`는 응답에 표시하는 상태다. 같은 카드라도 판매 중 항목과 교환 대기 제안은 별도로 반환한다. `canceled=true`는 본인이 취소한 판매(`CANCELED`)만 반환한다. `canceled`를 생략하거나 `false`로 보내면 취소된 판매는 제외한다. 취소 필터의 화면 요청에서는 `soldOut`을 생략하며, 두 필터를 함께 보낸 경우 모두 충족하는 항목을 반환한다. 판매 취소로 자동 종료된 제안은 제안자가 알림에서 확인한다. 그 밖의 종료된 교환 제안은 열람 가능한 판매 상세의 제안 조회에서 확인한다.

## 응답 필드

| 이름 | `data` 또는 목록 항목의 필드 |
| --- | --- |
| 회원 | `id, email, nickname, pointBalance` |
| 카드 | `id, name, grade, genre, description, imageUrl, issuedQuantity` |
| 판매 요약 | `id, card(카드), sellerNickname, unitPrice, remainingQuantity, status, createdAt`. 상태는 `ACTIVE, SOLD_OUT, CANCELED`이며 마켓플레이스 목록에서 취소된 판매글은 제외한다. |
| 판매 상세 | 판매 요약 필드와 `wantedGrade, wantedGenre, wantedDescription, isOwner, canceledAt`. 희망 필드 미입력 값과 취소되지 않은 판매글의 `canceledAt`은 각각 `null` |
| 구매 결과 | `id, listingId, quantity, unitPrice, totalPrice, buyerPointBalance, listingRemainingQuantity, createdAt`. 0P 구매도 영수증을 반환한다. |
| 교환 제안 | `id, listingId, proposerNickname, offeredCard(카드), status, createdAt, resolvedAt`. 상태는 `PENDING, APPROVED, REJECTED, CANCELED_BY_PROPOSER, CLOSED_LISTING_CANCELED, CLOSED_SOLD_OUT`이다. `resolvedAt`은 대기 중일 때 `null`이다. |
| 갤러리 항목 | `inventoryId, card(카드), visibleQuantity, pendingExchangeQuantity, availableQuantity`. `visibleQuantity`에는 판매 등록 수량을 제외하고 교환 대기 카드를 포함한다. `visibleQuantity > 0`인 항목만 반환하며, `totalItems`도 이 조건을 적용한 수다. |
| 내 판매 항목 | `method, status, listingId, offerId, card(카드), quantity, canceledAt`. 취소 판매의 `quantity`는 취소 시점의 잔여 수량이고, 다른 상태의 `canceledAt`은 `null`이다. 판매 항목의 `offerId`는 `null`, 교환 제안 항목은 제안별 한 행이며, 대상 판매글 ID·제안 ID와 수량 `1`을 담는다. 같은 종류 카드의 대기 제안도 합치지 않는다. |
| 알림 | `id, listingId, type, message, createdAt, readAt`. API의 `type`에는 DB의 `Notification.kind`를 매핑한다. 판매자의 `SALE_COMPLETED`는 0P 거래를 포함한 구매 완료에 발송한다. 교환 승인 시 제안자에게 `EXCHANGE_APPROVED`를 발송한다. 구매 또는 교환으로 마지막 수량이 소진되면 판매자에게 `SOLD_OUT`을 발송한다. `LISTING_CANCELED`는 판매 취소로 대기 제안이 자동 종료된 회원에게 보낸다. 화면은 해당 알림에 취소된 판매 상세로 이동하는 링크를 제공하지 않는다. 값은 `EXCHANGE_OFFER_RECEIVED, EXCHANGE_APPROVED, EXCHANGE_REJECTED, PURCHASE_COMPLETED, SALE_COMPLETED, SOLD_OUT, LISTING_CANCELED`이다. |
| 상자 상태·적립 | `eligible`은 현재 열 수 있는지 나타낸다. 상태 조회에서 열 수 있으면 `nextEligibleAt`은 `null`이다. 성공한 열기 응답의 `nextEligibleAt`은 연 시각부터 한 시간 뒤이며, 보상은 1P~5P다. 확률은 [ERD의 보상표](ERD.md#랜덤-상자-보상-확률)를 따른다. |

알림 화면은 응답의 `createdAt`부터 경과한 시간을 계산한다. 1분 미만은 ‘방금 전’, 1분 이상 한 시간 미만은 경과한 분을 내림해 ‘N분 전’으로 표시하고, 한 시간 이상은 [브리프](brief.md)의 경과 시간 기준을 적용한다.

두 장 구매에 성공한 예시다.

```json
{
  "data": {
    "id": "purchase-1",
    "listingId": "listing-1",
    "quantity": 2,
    "unitPrice": 4,
    "totalPrice": 8,
    "buyerPointBalance": 2,
    "listingRemainingQuantity": 1,
    "createdAt": "2026-12-04T03:00:00Z"
  }
}
```

`409`와 함께 포인트 부족을 알리는 예시다.

```json
{
  "error": {
    "code": "INSUFFICIENT_POINTS",
    "message": "구매에 필요한 포인트가 부족합니다."
  }
}
```

## 실패 목록

| HTTP | `error.code` | 발생 조건 |
| --- | --- | --- |
| 400 | `VALIDATION_ERROR` | JSON, 입력 형식, 필수 값, 페이지·커서·필터·수량·가격이 계약과 다름 |
| 400 | `SAME_CARD_EXCHANGE_NOT_ALLOWED` | 판매 대상과 같은 `photoCardId`의 카드로 교환을 제안함 |
| 401 | `AUTH_REQUIRED` | 회원 전용 요청에 세션이 없거나 세션이 만료됨 |
| 401 | `INVALID_CREDENTIALS` | 이메일 또는 비밀번호를 확인할 수 없음 |
| 403 | `ORIGIN_NOT_ALLOWED` | 상태 변경 요청의 `Origin`이 없거나 허용한 웹 출처와 다름 |
| 403 | `FORBIDDEN` | 타인 판매 수정·취소, 타인 제안 처리, 자기 판매 구매·교환 제안 등 권한 위반 |
| 404 | `LISTING_NOT_FOUND`, `EXCHANGE_OFFER_NOT_FOUND`, `INVENTORY_NOT_FOUND` | 요청한 판매글이나 제안이 없거나, 요청 회원이 선택할 수 있는 카드 보유 기록이 없음. 등록자가 아닌 회원이 취소된 판매글의 상세 또는 교환 제안을 조회하면 `LISTING_NOT_FOUND` |
| 409 | `EMAIL_IN_USE`, `NICKNAME_IN_USE` | 가입 이메일 또는 닉네임이 이미 사용 중 |
| 409 | `ALREADY_AUTHENTICATED` | 유효한 세션에서 가입·로그인 API를 직접 요청함. 화면의 `/signup`·`/login` 직접 접속은 마켓플레이스로 이동 |
| 409 | `LISTING_UNAVAILABLE` | 취소 또는 품절된 판매글에 구매·제안·수정·취소를 요청함 |
| 409 | `PRICE_CHANGED` | 현재 판매 가격이 구매 화면에서 확인한 가격과 다름 |
| 409 | `INSUFFICIENT_STOCK` | 요청 수량이 판매 잔여 수량보다 많음 |
| 409 | `INSUFFICIENT_AVAILABLE_QUANTITY` | 판매 등록이나 교환 제안에 쓸 수 있는 보유 수량이 부족함 |
| 409 | `INSUFFICIENT_POINTS` | 구매자 포인트가 결제액보다 적음 |
| 409 | `EXCHANGE_ALREADY_RESOLVED` | 대기 상태를 벗어난 제안에 승인·거절·취소 요청 |
| 409 | `POINT_BOX_COOLDOWN` | 마지막 상자를 연 뒤 한 시간이 지나지 않음 |
| 415 | `UNSUPPORTED_MEDIA_TYPE` | JSON 입력을 요구하는 요청의 `Content-Type`이 JSON이 아님 |
| 500 | `INTERNAL_ERROR` | 서버의 예기치 않은 오류. 상태 변경이 실패하면 거래 결과를 저장하지 않음 |

## 팀 확인이 필요한 항목

- 장르 허용 목록은 초기 시드 데이터 작성 시 확정한다. 목록을 확정하면 시드, 필터, 교환 희망 장르 검증에 반영한다.
- 읽음 처리·안 읽은 개수, 내 판매 키워드 검색, 추가 알림은 범위 단계에서 정한 선택 기능이다. 채택할 때 관련 필드와 경로를 추가한다.
