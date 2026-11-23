# 환율 API 스펙 초안 (1차 MVP)

## 목차

- [문서 범위](#문서-범위)
- [API 목록](#api-목록)
- [개인 환율 API](#개인-환율-api)
  - [FX-01 내 환율 목록](#fx-01-내-환율-목록)
  - [FX-02 개인 환율 등록](#fx-02-개인-환율-등록)
- [환율 조회와 선택 API](#환율-조회와-선택-api)
  - [FX-03 결제일 기준 환율 조회](#fx-03-결제일-기준-환율-조회)
  - [FX-04 지출 기본 환율 결정: 조건부](#fx-04-지출-기본-환율-결정-조건부)
- [개인 기록 변경 API 후보](#개인-기록-변경-api-후보)
- [공통 모델](#공통-모델)
  - [PersonalRate](#personalrate)
  - [PersonalRateInput](#personalrateinput)
  - [ReferenceRate](#referencerate)
  - [RateResolutionInput](#rateresolutioninput)
- [오류 코드](#오류-코드)

## 문서 범위

개인 환전 기록과 외부 API 환율을 구분한다. 외부 조회값 자체는 개인 기록으로 자동 저장하지 않는다. 직접 수정한 환율은 지출에 적용한다. 본인 현금 또는 트래블 카드 지출의 수동 수정값은 개인 기록으로 자동 저장한다. 일반 카드 또는 타인 결제의 수동 입력값을 자동 저장할지는 논의 필요다. 여행방별, 결제 통화별 직전 입력값을 다음 지출에 노출한다. 자동 저장 시점과 일자, 수단 매핑, 기존 기본값 선택 규칙과의 우선순위는 [환율 정책](../policy/exchange-rate.md#지출-환율)의 미결 사항을 따른다.

[목록](README.md) | [공통 API 스펙](common.md) | [정책](../policy/exchange-rate.md)

이 문서는 API 스펙 검토용 초안이다.
API 식별자는 화면 정의서 식별자와 별개다.
경로, 요청과 응답 필드, 오류 코드를 팀에서 확정하지 않은 API는 Method와 Path 아래에 `논의 필요`로 표시한다.

## API 목록

| 기능 | Method | Path | 인증 |
| --- | --- | --- | --- |
| [내 환율 목록](#fx-01-내-환율-목록) | `GET` | `/api/v1/trips/{tripId}/exchange-rates` | 여행 참여자 (본인 기록) |
| [개인 환율 등록](#fx-02-개인-환율-등록) | `POST` | `/api/v1/trips/{tripId}/exchange-rates` | 여행 참여자 |
| [결제일 기준 환율 조회](#fx-03-결제일-기준-환율-조회) | `GET` | `/api/v1/reference-exchange-rates` | 로그인한 가입 완료 회원 |
| [지출 기본 환율 결정: 조건부](#fx-04-지출-기본-환율-결정-조건부) | `POST` | `/api/v1/trips/{tripId}/exchange-rate-resolutions` | 여행 참여자 (여행 확정 후) |

## 개인 환율 API

### FX-01 내 환율 목록

`GET /api/v1/trips/{tripId}/exchange-rates`

> 논의 필요
>
> 경로, 요청, 응답과 오류 코드의 확정 여부

#### 인증

로그인한 여행 참여자는 본인이 등록한 환율만 조회할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 여행에 저장한 본인의 환전 기록을 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

##### Query Parameter

선택 필터를 생략하면 그 조건을 적용하지 않는다.

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `page` | Integer | 선택 | 0 | 0 이상. 다음 페이지는 현재 page+1. |
| `size` | Integer | 선택 | 20 | 1~100, 범위 밖이면 400. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 Array&lt;[PersonalRate](#personalrate)&gt;이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | Array&lt;[PersonalRate](#personalrate)&gt; | 아니요 | 항상 | 이 API의 처리 결과. |

응답 최상위 `page`는 [Page](common.md#page)를 따른다.
페이지 적용 자체와 개별 기본 정렬은 검토 대상이다.

```json
{
  "data": [
    {
      "exchangeRateId": "rate-example-a",
      "currency": "JPY",
      "rate": "9.12",
      "exchangeMethod": "CASH",
      "exchangeDate": "2026-11-05",
      "createdAt": "2026-11-05T03:00:00Z"
    }
  ],
  "page": {
    "page": 0,
    "size": 20,
    "totalElements": 1,
    "hasNext": false
  }
}
```

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

서버는 현재 로그인한 회원이 등록한 기록만 반환한다.
회원은 다른 회원의 환율을 조회할 수 없다.

> 논의 필요
>
> - 통화와 환전 수단 필터, 정렬
> - 최근 기록의 기준: 등록 시각 vs 수정 시각

### FX-02 개인 환율 등록

`POST /api/v1/trips/{tripId}/exchange-rates`

> 논의 필요
>
> 경로, 요청, 응답과 오류 코드의 확정 여부

#### 인증

로그인한 회원 중 그 여행에 참여하고 여행방에 접근할 수 있는 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

본인의 환전 환율과 환전 수단, 날짜를 기록한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

##### Body

Content-Type: `application/json`

요청 필드: [PersonalRateInput](#personalrateinput).

```json
{
  "currency": "JPY",
  "rate": "9.12",
  "exchangeMethod": "CASH",
  "exchangeDate": "2026-11-05"
}
```

#### 성공 응답

HTTP `201 Created`를 반환한다.

공통 응답의 `data`는 [PersonalRate](#personalrate)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [PersonalRate](#personalrate) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "exchangeRateId": "rate-example-a",
    "currency": "JPY",
    "rate": "9.12",
    "exchangeMethod": "CASH",
    "exchangeDate": "2026-11-05",
    "createdAt": "2026-11-05T03:00:00Z"
  }
}
```

생성한 대상의 조회 URL을 `Location`에 제공한다.

> 논의 필요
>
> 조회 경로를 아직 정의하지 않은 자원은 그 경로를 함께 확정해야 한다.

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

서버는 지원 통화, 0보다 큰 환율과 소수점 이하 2자리 입력을 검증한다.
서버는 기록자 ID를 직접 정한다.
같은 여행에서 본인이 통화, 일자, 수단과 환율이 모두 같은 기록을 다시 저장하면 서버는 저장을 거절한다.
날짜가 다르면 서버는 별도 기록으로 저장한다.

> 논의 필요
>
> 환율 상한과 KRW 처리, 같은 기록의 재요청 응답과 중복 오류 매핑

## 환율 조회와 선택 API

### FX-03 결제일 기준 환율 조회

`GET /api/v1/reference-exchange-rates`

> 논의 필요
>
> 경로, 요청, 응답과 오류 코드의 확정 여부

#### 인증

로그인한 가입 완료 회원만 요청할 수 있다. 인증 없이 공개하지 않으며 가입 전용 토큰으로는 호출할 수 없다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

지출 결제일의 참고 환율과 실제 제공 기관, 기준일을 조회한다.

#### 요청

##### Query Parameter

선택 필터를 생략하면 그 조건을 적용하지 않는다.

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `currency` | String | 필수 | 없음 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `paidDate` | String | 필수 | 없음 | YYYY-MM-DD. 사용자가 입력한 현지 결제 날짜를 조회 기준으로 사용. 미래 날짜는 거절. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [ReferenceRate](#referencerate)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [ReferenceRate](#referencerate) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "currency": "JPY",
    "rate": "9.12",
    "provider": "EXAMPLE_PROVIDER",
    "baseDate": "2026-11-05"
  }
}
```

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음. 미래 날짜 요청 포함 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 503 | `EXCHANGE_RATE_PROVIDER_UNAVAILABLE` | 외부 환율을 일시적으로 조회할 수 없음 |

#### 처리 규칙

요청 날짜에 고시 환율이 없으면 서버는 그 날짜 이전에 고시된 값 중 가장 최근 환율을 반환한다.
서버는 `baseDate`에 요청 날짜가 아닌 실제 고시 기준일을 담는다.
미래 날짜의 환율을 요청하면 서버는 `400`으로 거절한다.
요청한 `paidDate`와 반환된 `baseDate`가 다르면 앱은 실제 기준일과 이전 고시값을 썼다는 안내를 표시한다.

요청 날짜와 그 이전에 고시된 환율이 없으면 서버는 조회 불가 오류를 반환한다.
서버는 빈 성공 응답이나 임의 환율을 반환하지 않는다.
조회 불가 오류를 받으면 앱은 사용자가 환율을 직접 입력하게 한다.
서버는 외부 서비스의 장애나 타임아웃을 그 날짜의 고시값이 없는 상태로 취급하지 않는다.
외부 조회에 실패하면 서버는 직접 입력이 필요한 상태를 고시값 부재와 구분해 응답한다.

> 논의 필요
>
> - 미래 날짜를 판정하는 업무 시간대와 오류 코드
> - 이전 고시값의 탐색 범위
> - 고시값이 없을 때의 HTTP 상태와 오류 코드
> - 환율 제공 기관을 표현할 값

### FX-04 지출 기본 환율 결정: 조건부

`POST /api/v1/trips/{tripId}/exchange-rate-resolutions`

> 논의 필요
>
> - 경로, 요청, 응답과 오류 코드의 확정 여부
> - 제공 범위나 구현 방식을 정한 뒤의 채택 여부

#### 인증

여행 확정 후 로그인한 여행 참여자가 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

지출 입력 조건에 따라 기본 환율을 선택해 반환한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

##### Body

Content-Type: `application/json`

요청 필드: [RateResolutionInput](#rateresolutioninput).

```json
{
  "splitType": "EQUAL",
  "payerTripMemberId": "participant-example-a",
  "paymentMethod": "CARD",
  "currency": "JPY",
  "paidDate": "2026-11-05"
}
```

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [ReferenceRate](#referencerate)이다.
필드 정의는 그 모델을 따른다.
이 모델은 일부 필드만 정의한 구조이며, 아래 예시는 정의된 필드 범위만 보여준다.
전체 요청과 응답 규격은 이 API의 논의 사항을 반영해 완성해야 한다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [ReferenceRate](#referencerate) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "currency": "JPY",
    "rate": "9.12",
    "provider": "EXAMPLE_PROVIDER",
    "baseDate": "2026-11-05"
  }
}
```

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

기본 환율은 [환율 정책의 기본값 선택 규칙](../policy/exchange-rate.md#지출-환율)을 따른다.
서버가 기본 환율을 결정하기로 정할 때만 서버는 이 API를 제공한다.
이 API를 호출해도 서버는 지출을 저장하지 않는다.
서버는 두 분할 방식에 같은 기본 환율 규칙을 적용한다.
본인 현금이나 트래블 카드 결제이면 서버는 같은 통화의 최근 개인 기록을 쓴다.
일반 카드 결제나 타인 결제이면 서버는 결제일 기준 API 환율을 쓴다.

> 논의 필요
>
> - 서버가 결정하는 API를 제공함 vs 앱이 규칙을 적용함
> - 응답 모델. 응답 예시는 API 환율을 선택했을 때의 후보이며, 개인 기록과 수동 입력을 포함한 최종 모델이 아니다.
> - 개인 기록을 고를 때 환전 수단도 일치시킬지 여부
> - 기록이 없을 때의 오류나 안내 응답

## 개인 기록 변경 API 후보

개인 환율의 수정과 삭제를 제공하기로 정하면 `PATCH /api/v1/trips/{tripId}/exchange-rates/{exchangeRateId}`와 `DELETE /api/v1/trips/{tripId}/exchange-rates/{exchangeRateId}`를 둔다.
1차 MVP에서 서버는 직접 수정한 환율을 개인 환율 기록에 자동 저장한다([개인 환율 기록](../policy/exchange-rate.md#개인-환율-기록), [지출 환율](../policy/exchange-rate.md#지출-환율)).

> 논의 필요
>
> - 개인 환율의 수정과 삭제 제공 여부
> - 자동 저장 시점, 지출 저장 API와의 연계, 중복 저장 때의 응답

## 공통 모델

> 논의 필요
>
> 모델 이름과 필드 구성의 확정 여부

요청 모델은 필수 여부와 기본값을, 응답 모델은 nullable과 포함 조건을 구분한다. 조건부 요청 필드는 예시에 사용한 조건을 만족할 때만 포함한다. 배열 원소는 링크한 모델을 따른다.

### PersonalRate

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `exchangeRateId` | String | 아니요 | 항상 | 본인이 등록한 개인 환율 기록 ID. |
| `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `rate` | String | 아니요 | 항상 | 외화 1단위당 원화 금액, 0보다 큼. 소수점 이하 2자리 문자열로 전달한다. |
| `exchangeMethod` | String | 아니요 | 항상 | `CASH`는 현금, `TRAVEL_CARD`는 트래블 카드다. |
| `exchangeDate` | String | 아니요 | 항상 | YYYY-MM-DD. 사용자가 입력한 현지 환전 날짜. 시각은 수집하지 않으며 UTC 변환 없음. |
| `createdAt` | String | 아니요 | 항상 | ISO 8601 UTC. 등록 시각. 최근 기록 선택 기준으로 사용할지는 논의 필요. |

### PersonalRateInput

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `currency` | String | 필수 | 없음 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `rate` | String | 필수 | 없음 | 외화 1단위당 원화 금액, 0보다 큼. 소수점 이하 2자리 문자열로 전달한다. |
| `exchangeMethod` | String | 필수 | 없음 | `CASH`는 현금, `TRAVEL_CARD`는 트래블 카드다. |
| `exchangeDate` | String | 필수 | 없음 | YYYY-MM-DD. 사용자가 입력한 현지 환전 날짜. 시각은 수집하지 않으며 UTC 변환 없음. |

### ReferenceRate

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `currency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `rate` | String | 아니요 | 항상 | 외화 1단위당 원화 금액. 소수점 이하 2자리 문자열로 전달한다. |
| `provider` | String | 아니요 | 항상 | 실제 환율 제공 기관 식별. 기관 코드 체계는 논의 필요. |
| `baseDate` | String | 아니요 | 항상 | YYYY-MM-DD. 환율의 실제 기준일. 요청 결제일과 다를 수 있다. |

### RateResolutionInput

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `splitType` | String | 필수 | 없음 | `EQUAL`은 전체 1/N, `BY_ITEM`은 품목별 분배다. |
| `payerTripMemberId` | String | 필수 | 없음 | 실제 결제자의 여행 참여자 ID. 같은 여행의 회원 참여자만 지정할 수 있다. |
| `paymentMethod` | String | 필수 | 없음 | `CARD`는 일반 카드, `CASH`는 현금, `TRAVEL_CARD`는 트래블 카드다. |
| `currency` | String | 필수 | 없음 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
| `paidDate` | String | 필수 | 없음 | YYYY-MM-DD. 사용자가 입력한 현지 결제 날짜. |

## 오류 코드

> 논의 필요
>
> 도메인 오류 코드의 이름, HTTP 대응과 안내 문구의 확정 여부

공통 코드 이름은 [백엔드 CommonErrorCode](https://github.com/JECT-Study/Trip-Sheet-5th-Server/blob/09dd7bfbaf37974fecfa3bb2fa2c22ea3378d20c/src/main/kotlin/com/ject/global/error/CommonErrorCode.kt)와 맞췄다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 503 | `EXCHANGE_RATE_PROVIDER_UNAVAILABLE` | 외부 환율을 일시적으로 조회할 수 없음 |

> 논의 필요
>
> 개별 API의 오류 조건과 도메인 코드 이름은 팀 검토 후 확정한다.
