# 환율 설계서

## 개요

이 설계서는 지출에 적용할 개인 저장 환율을 고르는 쿼리, 결제일 기준 API 환율의 기준일과 지출을 저장한 뒤 적용 환율을 보존하는 방식을 다룬다.
기본 환율을 고르는 조건과 환전 날짜의 입력 기준은 [환율 정책](../policy/exchange-rate.md)을, 지출의 환율 저장 컬럼은 [지출 설계서](expense.md#저장-데이터)을 따른다.

## 저장 데이터

| 이름 | 타입 | 제약 | 뜻 |
| --- | --- | --- | --- |
| `manual_exchange_rate.trip_id` | `BIGINT` | NOT NULL, FK `trip.id` | 환율을 기록한 여행 |
| `manual_exchange_rate.created_by_trip_member_id` | `BIGINT` | NOT NULL, FK `trip_member.id` | 환율을 등록한 참여자 |
| `manual_exchange_rate.currency` | `CHAR(3)` | NOT NULL | 환전한 외화의 통화 코드 |
| `manual_exchange_rate.rate` | `DECIMAL(24,12)` | NOT NULL | 외화 1단위의 원화 금액 |
| `manual_exchange_rate.exchange_date` | `DATE` | NOT NULL | 회원이 입력한 환전 당시의 현지 날짜 |
| `manual_exchange_rate.exchange_method` | `exchange_method` | NOT NULL | 환전 수단. `CASH`와 `TRAVEL_CARD` |
| `manual_exchange_rate.created_at` | `TIMESTAMPTZ` | NOT NULL | 최초 등록 시각 |

- 서버는 환전 시각, 시간대, 현재 위치와 국가코드를 저장하지 않는다.

## 처리 흐름

- 서버는 회원이 입력한 현지 결제 날짜 `paidDate`를 기준일로 제공 기관에 API 환율을 요청한다.
- 서버는 제공 기관이 반환한 실제 기준일을 요청 날짜로 덮어쓰지 않는다.
- 최종 부담액을 계산하는 환율의 통화는 지출의 결제 통화와 같다.
- 지출에 적용한 환율의 저장은 [지출 설계서](expense.md#지출-저장)를 따른다.
- 개인 저장 환율을 수정하거나 삭제해도 서버는 기존 지출과 정산의 환율을 바꾸지 않는다.

> 논의 필요
>
> - 서버가 기본 환율을 결정함 vs 앱이 기본 환율을 결정함. [FX-04 지출 기본 환율 결정](../api-spec/exchange-rate.md#fx-04-지출-기본-환율-결정-조건부)
> - 직접 수정한 환율의 자동 저장 시점, 저장할 환전 날짜와 수단, 일반 카드나 타인 결제의 자동 저장 여부: [지출 환율 정책](../policy/exchange-rate.md#지출-환율)의 논의 필요를 따른다.
> - 지출 저장을 재시도할 때 자동 저장한 환율 기록의 중복 처리

## 쿼리

`:requester_trip_member_id`는 인증된 요청자의 여행 참여 정보에서 구한다.
`:payment_currency`는 회원이 지출에 선택한 결제 통화다.
쿼리는 최초 등록 시각인 `created_at`만 정렬 기준으로 쓴다.
수정 시각 `updated_at`과 환전 날짜 `exchange_date`는 순서에 영향을 주지 않는다.

다음 쿼리는 요청자가 같은 여행에서 결제 통화로 등록한 개인 저장 환율 중 최초 등록 시각이 가장 늦은 기록 하나를 조회한다.

```sql
-- 결제 통화의 최근 개인 저장 환율
SELECT id, currency, rate, exchange_date, exchange_method, created_at
FROM manual_exchange_rate
WHERE trip_id = :trip_id
  AND created_by_trip_member_id = :requester_trip_member_id
  AND currency = :payment_currency
ORDER BY created_at DESC
LIMIT 1;
```

다음은 개인 저장 환율 기록 3개로 쿼리 결과를 보여 주는 가상 예시다.

```text
기록 A: USD, 10:00 등록, 14:00 수정
기록 B: USD, 11:00 등록
기록 C: JPY, 12:00 등록

1. 회원이 본인 현금 지출의 결제 통화로 USD를 고른다.
   결과: 기록 B를 적용한다. 기록 A를 나중에 수정했어도 최초 등록이 더 늦은 기록은 B다.

2. 회원이 결제 통화를 JPY로 바꾼다.
   결과: 기록 C를 적용한다.

3. 회원이 결제 통화를 다시 USD로 바꾼다.
   결과: 기록 B를 적용한다.
```

> 논의 필요
>
> - 환전 수단까지 같은 기록만 고를지 여부: [지출 환율 정책](../policy/exchange-rate.md#지출-환율)의 논의 필요를 따른다.
> - `created_at`이 같은 기록의 정렬 보조 기준

## 실패 처리

- 결제 통화의 개인 저장 환율이 없으면 조회 쿼리는 다른 통화의 기록을 반환하지 않는다.
- 요청 날짜와 그 이전에 고시된 환율이 없으면 서버는 조회 불가 오류를 반환한다.
- 제공 기관이 장애나 타임아웃으로 응답하지 않으면 서버는 `503 EXCHANGE_RATE_PROVIDER_UNAVAILABLE`을 반환한다.
- 서버는 제공 기관의 장애를 고시값이 없는 상태로 판단하지 않는다.
- API 환율 조회에 실패하면 서버는 임의 환율을 반환하지 않는다.

## 검증

- 같은 여행에 결제 통화와 다른 통화의 기록만 있으면 조회 쿼리는 행을 반환하지 않는다.
- 같은 여행의 다른 참여자만 결제 통화의 기록을 등록했으면 조회 쿼리는 행을 반환하지 않는다.
- 결제 통화의 기록이 여러 개면 조회 쿼리는 `created_at`이 가장 늦은 기록을 반환한다.
- 먼저 등록한 기록을 나중에 수정해도 조회 쿼리는 나중에 등록한 기록을 반환한다.
- 개인 저장 환율을 수정하거나 삭제해도 서버는 기존 지출의 `exchange_rate`를 바꾸지 않는다.
