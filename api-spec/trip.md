# 여행 API 스펙 초안 (1차 MVP)

## 목차

- [문서 범위](#문서-범위)
- [API 목록](#api-목록)
- [여행 조회와 관리 API](#여행-조회와-관리-api)
  - [TRIP-01 내 여행 목록](#trip-01-내-여행-목록)
  - [TRIP-02 여행 생성](#trip-02-여행-생성)
  - [TRIP-03 여행 상세](#trip-03-여행-상세)
  - [TRIP-04 여행 정보 수정](#trip-04-여행-정보-수정)
- [초대와 참여자 API](#초대와-참여자-api)
  - [TRIP-05 초대 정보 조회](#trip-05-초대-정보-조회)
  - [TRIP-06 초대 코드 검색](#trip-06-초대-코드-검색)
  - [TRIP-07 여행 참여](#trip-07-여행-참여)
  - [TRIP-08 비로그인 참여자 등록](#trip-08-비로그인-참여자-등록)
  - [TRIP-09 비로그인 이름 수정](#trip-09-비로그인-이름-수정)
  - [TRIP-10 비로그인 참여자 삭제](#trip-10-비로그인-참여자-삭제)
- [확정과 종료 API](#확정과-종료-api)
  - [TRIP-11 여행 확정](#trip-11-여행-확정)
  - [TRIP-12 여행 나가기](#trip-12-여행-나가기)
  - [TRIP-13 여행 삭제](#trip-13-여행-삭제)
- [공통 모델](#공통-모델)
  - [TripSummary](#tripsummary)
  - [TripListItem](#triplistitem)
  - [TripDetail](#tripdetail)
  - [TripCreate](#tripcreate)
  - [Invitation](#invitation)
  - [InvitationSearch](#invitationsearch)
  - [InviteCodeInput](#invitecodeinput)
  - [Membership](#membership)
  - [Participant](#participant)
  - [GuestInput](#guestinput)
  - [Guest](#guest)
  - [TripConfirmation](#tripconfirmation)
- [오류 코드](#오류-코드)

## 문서 범위

여행 확정 상태와 일정 상태를 구분한다. 관리자는 생성자이며 권한 양도는 MVP 범위가 아니다.

[목록](README.md) | [공통 API 스펙](common.md) | [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

이 문서는 API 스펙 검토용 초안이다. API 식별자는 화면 정의서 식별자와 별개다. 팀에서 확정한 계약은 본문에 작성하고, 아직 검토 중인 경로, 요청과 응답 필드 및 오류 코드는 각 API의 `논의 필요` 인용문 안에 제안 모델과 예시로 구분한다.

## API 목록

| 기능 | Method | Path | 인증 | operationId |
| --- | --- | --- | --- | --- |
| [내 여행 목록](#trip-01-내-여행-목록) | `GET` | `/api/v1/trips` | 로그인 필요 | `listTrips` |
| [여행 생성](#trip-02-여행-생성) | `POST` | `/api/v1/trips` | 로그인 필요 | `createTrip` |
| [여행 상세](#trip-03-여행-상세) | `GET` | `/api/v1/trips/{tripId}` | 여행 참여자 | `getTrip` |
| [여행 정보 수정](#trip-04-여행-정보-수정) | `PATCH` | `/api/v1/trips/{tripId}` | 여행 관리자 | `updateTrip` |
| [초대 정보 조회](#trip-05-초대-정보-조회) | `GET` | `/api/v1/trips/{tripId}/invitation` | 여행 관리자 (여행 확정 전) | `getTripInvitation` |
| [초대 코드 검색](#trip-06-초대-코드-검색) | `GET` | `/api/v1/trip-invitations` | 로그인 필요 | `searchTripInvitation` |
| [여행 참여](#trip-07-여행-참여) | `POST` | `/api/v1/trip-memberships` | 로그인 필요 | `joinTripByInviteCode` |
| [비로그인 참여자 등록](#trip-08-비로그인-참여자-등록) | `POST` | `/api/v1/trips/{tripId}/guests` | 여행 관리자 (여행 확정 전) | `createTripGuest` |
| [비로그인 이름 수정](#trip-09-비로그인-이름-수정) | `PATCH` | `/api/v1/trips/{tripId}/guests/{tripMemberId}` | 여행 관리자 | `updateTripGuest` |
| [비로그인 참여자 삭제](#trip-10-비로그인-참여자-삭제) | `DELETE` | `/api/v1/trips/{tripId}/guests/{tripMemberId}` | 여행 관리자 (여행 확정 전) | `deleteTripGuest` |
| [여행 확정](#trip-11-여행-확정) | `POST` | `/api/v1/trips/{tripId}/confirm` | 여행 관리자 | `confirmTrip` |
| [여행 나가기](#trip-12-여행-나가기) | `POST` | `/api/v1/trips/{tripId}/leave` | 여행 참여자 (관리자 제외) | `leaveTrip` |
| [여행 삭제](#trip-13-여행-삭제) | `DELETE` | `/api/v1/trips/{tripId}` | 여행 관리자 | `deleteTrip` |

operationId는 백엔드 [API 문서 기준](https://github.com/JECT-Study/Billit-5th-Server/blob/develop/docs/contributing/api-documentation.md#태그와-operationid)의 이름 규칙을 따른다.

> 논의 필요
>
> 회원이 자기 여행방 닉네임을 수정하는 API의 경로와 요청 형식을 정한다. 회원은 여행에 참여한 뒤에도 자신의 여행방 닉네임을 수정할 수 있다([여행방 닉네임](../policy/trip.md#여행방-닉네임)).

## 여행 조회와 관리 API

### TRIP-01 내 여행 목록

`GET /api/v1/trips`

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

요청한 회원이 참여 중인 모든 여행을 조회한다. 여행 목록 화면(`HOME-01`)의 여행 카드에 필요한 정보를 반환한다.

#### 요청

##### Query Parameter

페이지는 [Page](common.md#page)의 offset 방식을 따른다.

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `page` | Integer | 선택 | 0 | 0 이상. 다음 페이지는 현재 page+1. |
| `size` | Integer | 선택 | 20 | 1~100, 범위 밖이면 400. |
| `filter` | String | 선택 | 없음 | `INVITING`은 참여자 초대중, `UPCOMING`은 여행 예정, `ENDED`는 여행 종료. 확정 전 여행은 여행 시작일과 관계없이 `INVITING`에만 들어가고, `UPCOMING`과 `ENDED`에는 확정 후 여행만 들어간다([여행 상태](../policy/trip.md#여행-상태)와 같은 기준). 생략하면 진행 중 여행을 포함한 전체 여행 목록을 조회한다. |

필터는 서버가 적용하며, 페이지는 필터를 적용한 결과에 나눈다. 여행 목록 화면의 기본 필터는 전체 여행 목록이다.

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다. 공통 응답의 `data`는 Array&lt;[TripListItem](#triplistitem)&gt;이고, 응답 최상위 `page`는 [Page](common.md#page)를 따른다.

서버는 최근에 만든 여행부터 정렬해 반환한다. 만든 시각이 같으면 여행 ID가 큰 여행을 먼저 둔다.

```json
{
  "data": [
    {
      "tripId": "trip-example-a",
      "name": "도쿄 여행",
      "countryCode": "JP",
      "customCountryName": null,
      "startDate": "2026-11-05",
      "endDate": "2026-11-07",
      "memberCapacity": 3,
      "registeredMemberCount": 3,
      "status": "UNCONFIRMED",
      "myRole": "ADMIN",
      "myShareAmountKrw": "0.00"
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

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |

#### 처리 규칙

요청한 회원이 참여 중인 모든 여행을 조회하며, `filter`를 보내면 그 조건에 맞는 여행만 조회한다. [나간 여행](../policy/trip.md#나가기와-기록-유지)과 [삭제된 여행](../policy/trip.md#여행-삭제)은 목록에서 제거되므로 포함하지 않는다. 여행 카드의 사용 금액은 여행 전체의 지출 합계가 아니라 요청한 회원의 누적 부담액을 원화로 나타낸 값이다.

진행 중 여행은 목록 상단의 별도 영역에 표시한다. 여행 예정, 진행 중과 여행 종료를 나누는 기준은 [여행 상태](../policy/trip.md#여행-상태)를 따르며, 이때 '오늘'은 사용자 기기의 현지 날짜다.

> 논의 필요
>
> - 상단의 진행 중 여행 영역을 이 API로 조회할지, 별도 영역과 페이지의 관계를 결정한다.
> - 서버가 필터를 적용하므로 서버가 사용자 기기의 현지 날짜를 알아야 한다. 앱이 그 날짜를 요청에 담아 보낼지 등 서버에 전달하는 방식을 정한다. 응답 모델에는 일정 상태 필드를 추가하지 않았다.
> - 앱이 받은 여행 목록을 기기에 저장해 화면에 먼저 보여주고, 이후에는 서버가 추가, 수정, 삭제된 여행만 보내는 방향을 검토한다. 변경분을 요청하는 기준, 삭제된 여행의 전달 방식과 offset 페이지와의 관계를 정한다.
>
> 출처: [정책](../policy/trip.md#여행-조회) | 기존 검토 DDL `trip.sql`

### TRIP-02 여행 생성

`POST /api/v1/trips`

> 논의 필요
>
> 요청과 응답 모델의 필드 구성은 검토 제안이다.
> 출처: [정책](../policy/trip.md#여행-생성) | 기존 검토 DDL `trip.sql`

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

여행 기본 정보와 정원을 입력해 새 여행을 생성한다.

#### 요청

##### Header

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `Idempotency-Key` | String | 필수 | 없음 | UUID 형식의 여행 생성 고유키. 앱은 생성 시도마다 새 키를 만들고, 응답을 받지 못해 같은 요청을 다시 보낼 때만 같은 키를 사용한다. 없거나 UUID 형식이 아니면 400. |

##### Body

> 논의 필요
>
> 다음 요청 예시는 검토 제안이다.
>
> Content-Type: `application/json`
>
> 요청 필드: [TripCreate](#tripcreate).
>
> ```json
> {
>   "name": "도쿄 여행",
>   "countryCode": "JP",
>   "baseCurrency": "JPY",
>   "startDate": "2026-11-05",
>   "endDate": "2026-11-07",
>   "memberCapacity": 3,
>   "nickname": "민지"
> }
> ```
>
> 기타 국가를 선택한 예시:
>
> ```json
> {
>   "name": "하노이 여행",
>   "countryCode": "ETC",
>   "customCountryName": "베트남",
>   "baseCurrency": "USD",
>   "startDate": "2026-12-01",
>   "endDate": "2026-12-04",
>   "memberCapacity": 4,
>   "nickname": "민지"
> }
> ```

#### 성공 응답

HTTP `201 Created`와 생성한 여행의 주소를 담은 `Location` 헤더를 반환한다. 같은 회원이 같은 고유키로 다시 요청해도 처음과 같이 `201 Created`와 `Location`을 반환하며, `data`에는 처음 만든 여행의 현재 정보를 담는다.

> 논의 필요
>
> 공통 응답의 `data`는 [TripSummary](#tripsummary)이다. 필드 정의는 해당 모델을 따른다. 아래 예시는 제안 필드 범위만 보여준다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [TripSummary](#tripsummary) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "tripId": "trip-example-a",
>     "name": "도쿄 여행",
>     "countryCode": "JP",
>     "customCountryName": null,
>     "startDate": "2026-11-05",
>     "endDate": "2026-11-07",
>     "baseCurrency": "JPY",
>     "memberCapacity": 3,
>     "registeredMemberCount": 1,
>     "status": "UNCONFIRMED",
>     "myRole": "ADMIN"
>   }
> }
> ```

#### 오류

[공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | `Idempotency-Key`가 없거나 UUID 형식이 아님 |
| 409 | `TRIP_IDEMPOTENCY_KEY_REUSED` | 다른 회원이 이미 사용한 고유키이거나, 그 키로 만든 여행이 삭제됨 |

> 논의 필요
>
> 아래 오류 매핑은 제안이다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음. `countryCode`가 `ETC`가 아닌데 `customCountryName`을 보내거나, `ETC`인데 누락한 경우도 포함 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |

#### 처리 규칙

생성자를 첫 참여자로 등록한다. 정원 1~10명, 종료일은 시작일 이상이다.

고유키는 여행 생성에만 사용한다. 같은 회원이 같은 키로 다시 보낸 요청만 재요청으로 보고, 여행과 생성자 참여 정보를 다시 만들지 않는다. 서버는 재요청의 본문을 처음 요청과 비교하지 않으므로 본문이 달라도 처음 만든 여행을 반환한다. 같은 키의 요청이 동시에 와도 여행은 하나만 만든다. 다른 회원이 이미 사용한 키이거나 그 키로 만든 여행이 삭제됐으면 `409 TRIP_IDEMPOTENCY_KEY_REUSED`를 반환한다. 키의 보관과 처리 방식은 [여행 생성 멱등성](../design/idempotency.md#여행-생성-멱등성)을 따른다.

> 논의 필요
>
> - 검토 제안: `countryCode`가 `ETC`일 때만 `customCountryName`을 필수로 받으며, `countryCode`가 `ETC`가 아니면 `customCountryName`을 생략해야 한다.
> `countryCode`가 `ETC`가 아닌데 `customCountryName`을 보내면 무시하지 않고 `400 COMMON_INVALID_REQUEST`로 거절한다(`null` 불가).
> - 같은 키의 요청이 동시에 왔을 때 늦게 처리된 요청의 응답과 `trip`에 생성자 컬럼을 추가할지는 [여행 생성 멱등성](../design/idempotency.md#여행-생성-멱등성)의 논의 사항을 따른다.
>
> 출처: [정책](../policy/trip.md#여행-생성) | 기존 검토 DDL `trip.sql`

### TRIP-03 여행 상세

`GET /api/v1/trips/{tripId}`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

#### 인증

로그인한 회원 중 해당 여행에 참여하고 여행방에 접근할 수 있는 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

여행 상세 진입에 필요한 기본 정보와 참여자 목록을 함께 조회한다. 참여자 목록 포함 여부는 확정됐으며 구체적인 필드 구성은 아래 제안을 검토한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [TripDetail](#tripdetail)이다. 필드 정의는 해당 모델을 따른다. 이 모델은 일부 필드만 정의한 구조이며, 아래 예시는 정의된 필드 범위만 보여준다. 전체 요청과 응답 규격은 이 API의 논의 사항을 반영해 완성해야 한다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [TripDetail](#tripdetail) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "tripId": "trip-example-a",
>     "name": "도쿄 여행",
>     "countryCode": "JP",
>     "customCountryName": null,
>     "startDate": "2026-11-05",
>     "endDate": "2026-11-07",
>     "baseCurrency": "JPY",
>     "memberCapacity": 3,
>     "registeredMemberCount": 3,
>     "status": "UNCONFIRMED",
>     "myRole": "ADMIN",
>     "participants": [
>       {
>         "tripMemberId": "participant-example-a",
>         "memberId": "member-example-a",
>         "displayName": "민지",
>         "role": "ADMIN",
>         "shareAmountKrw": "0.00"
>       },
>       {
>         "tripMemberId": "participant-example-b",
>         "memberId": "member-example-b",
>         "displayName": "지훈",
>         "role": "MEMBER",
>         "shareAmountKrw": "0.00"
>       },
>       {
>         "tripMemberId": "participant-example-c",
>         "memberId": null,
>         "displayName": "서연",
>         "role": "MEMBER",
>         "shareAmountKrw": "0.00"
>       }
>     ]
>   }
> }
> ```

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

여행 종료일이 지나도 조회 가능하다. 삭제와 생성자 탈퇴로 차단된 여행은 진입을 차단한다.

참여자 목록은 관리자, 로그인 참여자(참여한 순서), 비로그인 참여자(관리자가 추가한 순서) 순으로 제공한다. 빈자리는 목록에 넣지 않는다. 앱은 총인원(`memberCapacity`)에서 등록 인원(`registeredMemberCount`)을 뺀 수만큼 빈자리를 목록 맨 위에 표시한다. 참여자별 부담액은 참여자 목록의 각 참여자([Participant](#participant))의 `shareAmountKrw`에 담는다. 부담액은 각 지출에 등록한 환율로 원화로 환산해 합산하며, 지출 날짜 필터와 관계없이 여행 전체 기간을 기준으로 한다([여행 조회](../policy/trip.md#여행-조회)와 같은 규칙).

참여자 목록만 조회하는 API는 따로 두지 않는다. 참여자 상태는 `MEMBER`(로그인 참여자), `GUEST`(비로그인 참여자), `LEFT`(나간 회원), `WITHDRAWN`(탈퇴 회원) 네 값으로 구분한다.

### TRIP-04 여행 정보 수정

`PATCH /api/v1/trips/{tripId}`

> 논의 필요
>
> 최종 경로, 요청 필드 이름과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md#정보와-인원-수정) | 기존 검토 DDL `trip.sql`

#### 인증

로그인한 해당 여행의 관리자만 요청할 수 있다. 여행 생성자가 관리자다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

여행 기본 정보와 변경 가능한 정원을 PATCH로 일부 수정한다. 요청에 보낸 필드만 변경하고 생략한 필드는 기존 값을 유지한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

##### Body

바꿀 필드만 보낸다. 모든 필드는 선택이며 `null`을 보낼 수 없다. 필드에 `null`을 보내거나 필드를 하나도 보내지 않으면 400을 반환한다.

> 논의 필요
>
> 다음 요청 필드는 [TripCreate](#tripcreate)의 제안 이름을 따른다.
>
> Content-Type: `application/json`
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `name` | String | 선택 | 없음 | 공백을 포함해 최대 15자, 한글과 영문, 숫자와 공백 허용. 특수문자 금지. 앞뒤 공백을 지운 값으로 검사하고 저장하며, 공백만 입력하면 거절한다([TripCreate](#tripcreate)의 `name`과 같은 규칙). `null` 불가. |
> | `countryCode` | String | 선택 | 없음 | 대표 국가. `KR`, `JP`, `US`, `CN`, `EU`, `ETC` 중 하나([TripCreate](#tripcreate)의 `countryCode`와 같은 규칙). `null` 불가. |
> | `customCountryName` | String | 조건부 | 없음 | 기타 국가로 직접 입력한 국가명. `countryCode`를 `ETC`로 바꾸면 필수이며, 기존 여행의 대표 국가가 `ETC`이면 `customCountryName`만 단독으로 수정할 수 있다. 최종 `countryCode`가 `ETC`가 아닌데 `customCountryName`을 보내면 무시하지 않고 `400 COMMON_INVALID_REQUEST`로 거절한다. 최대 10자, 한글과 영문만 허용. 가운데 공백, 숫자와 특수문자 금지. 영문 대소문자 구분. 앞뒤 공백을 지운 값으로 검사하고 저장하며, 공백만 입력하면 거절한다([TripCreate](#tripcreate)의 `customCountryName`과 같은 규칙). `null` 불가. |
> | `baseCurrency` | String | 선택 | 없음 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나([TripCreate](#tripcreate)의 `baseCurrency`와 같은 규칙). `null` 불가. |
> | `startDate` | String | 선택 | 없음 | YYYY-MM-DD. 여행 시작일. `null` 불가. |
> | `endDate` | String | 선택 | 없음 | YYYY-MM-DD, 여행 시작일 이상. `null` 불가. |
> | `memberCapacity` | Integer | 선택 | 없음 | 생성자 포함 1~10명. 확정 전에만 변경할 수 있고 등록된 참여자 수보다 작게 줄일 수 없다. `null` 불가. |
>
> ```json
> {
>   "name": "오사카 여행",
>   "memberCapacity": 4
> }
> ```

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [TripSummary](#tripsummary)이다. 필드 정의는 해당 모델을 따른다. 이 모델은 일부 필드만 정의한 구조이며, 아래 예시는 정의된 필드 범위만 보여준다. 전체 요청과 응답 규격은 이 API의 논의 사항을 반영해 완성해야 한다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [TripSummary](#tripsummary) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "tripId": "trip-example-a",
>     "name": "오사카 여행",
>     "countryCode": "JP",
>     "customCountryName": null,
>     "startDate": "2026-11-05",
>     "endDate": "2026-11-07",
>     "baseCurrency": "JPY",
>     "memberCapacity": 4,
>     "registeredMemberCount": 3,
>     "status": "UNCONFIRMED",
>     "myRole": "ADMIN"
>   }
> }
> ```

#### 오류

[공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필드에 `null`을 보내거나 필드를 하나도 보내지 않음 |

> 논의 필요
>
> 아래 오류 매핑은 제안이다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 형식 또는 값의 범위가 올바르지 않음. 최종 `countryCode`가 `ETC`가 아닌데 `customCountryName`을 보내거나, `ETC`로 바꾸면서 누락한 경우도 포함 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
> | 409 | `TRIP_CAPACITY_CHANGE_NOT_ALLOWED` | 확정 후 인원을 변경하거나 등록 인원보다 정원을 줄이려 함 |

#### 처리 규칙

확정 후 정원 변경은 금지한다. 등록 인원보다 줄일 수 없다. 앱이 확정 후 정원 입력을 비활성화해도 서버는 확정 여부와 등록 인원을 다시 확인해 조건에 맞지 않는 변경을 거절한다. 기존 지출과 정산에 소급 반영하지 않는다.

> 논의 필요
>
> - 검토 제안: 요청에 일부 필드만 보낸 경우 저장된 여행 정보와 합친 뒤 `countryCode`와 `customCountryName`의 조합을 검증한다.
> 최종 `countryCode`가 `ETC`가 아닌데 `customCountryName`을 보내면 무시하지 않고 `400 COMMON_INVALID_REQUEST`로 거절한다.
> 대표 국가를 `ETC`에서 다른 국가로 바꾸면 `customCountryName`은 생략하며, 서버는 기존에 저장된 기타 국가명을 비운다(`null`).

## 초대와 참여자 API

### TRIP-05 초대 정보 조회

`GET /api/v1/trips/{tripId}/invitation`

#### 인증

여행 확정 전, 로그인한 해당 여행의 관리자만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

공유할 초대 코드와 URL을 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다. 공통 응답의 `data`는 [Invitation](#invitation)이다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [Invitation](#invitation) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "inviteCode": "A7K2PQ",
    "inviteUrl": "https://example.com/invitations/A7K2PQ"
  }
}
```

#### 오류

[공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | `tripId`가 숫자 형식이 아님 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 403 | `COMMON_FORBIDDEN` | 요청한 회원이 해당 여행의 관리자가 아님. 여행에 참여하지 않은 회원도 포함 |
| 404 | `COMMON_NOT_FOUND` | `tripId`에 해당하는 여행이 없거나 삭제됨 |
| 409 | `TRIP_ALREADY_CONFIRMED` | 확정 후 초대 정보를 요청함 |

#### 처리 규칙

확정 후 신규 초대를 허용하지 않는다. 초대 코드는 여행을 생성할 때 발급하며, 확정 전까지 시간 제한 없이 유효하고 확정하면 초대가 자동으로 종료된다. 초대를 수동으로 중지하는 API는 두지 않는다. 1차 MVP에서는 초대 코드를 재발급하지 않는다([초대 코드와 링크](../policy/trip.md#초대-코드와-링크)와 같은 규칙).

초대 URL은 환경별로 설정한 초대 링크 기본 주소 뒤에 초대 코드를 붙여 `{기본 주소}/{초대 코드}` 형식으로 만든다. 기본 주소는 웹 초대 화면의 `/invitations` 경로를 가리킨다.

### TRIP-06 초대 코드 검색

`GET /api/v1/trip-invitations`

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

초대 코드에 해당하는 여행의 참여 전 확인 정보를 조회한다.

#### 요청

##### Query Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `code` | String | 필수 | 없음 | 영문 대문자와 숫자를 조합한 6자리 초대 코드. 서버는 앞뒤 공백을 지우고 대소문자를 구분하지 않고 비교한다. 비어 있거나 공백뿐이면 400, 형식이 맞지 않으면 없는 코드와 같이 404. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다. 공통 응답의 `data`는 [InvitationSearch](#invitationsearch)이다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [InvitationSearch](#invitationsearch) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "tripId": "trip-example-a",
    "name": "도쿄 여행",
    "startDate": "2026-11-05",
    "endDate": "2026-11-07",
    "registeredMemberCount": 2,
    "memberCapacity": 3,
    "status": "UNCONFIRMED",
    "isAlreadyJoined": false
  }
}
```

#### 오류

[공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | `code`가 없거나 비어 있거나 공백뿐임 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 404 | `TRIP_INVITATION_NOT_FOUND` | `code`에 해당하는 여행이 없거나 삭제됨. 형식이 맞지 않는 코드도 포함 |

#### 처리 규칙

검색만으로 참여하거나 빈자리를 배정하지 않는다. 서버는 입력한 코드의 앞뒤 공백을 지우고 대소문자를 구분하지 않고 비교한다.

정원이 찼거나 확정된 여행도 검색 결과를 반환한다. 앱은 등록 인원, 총인원과 확정 상태(`status`)로 [검색과 참여](../policy/trip.md#검색과-참여)의 `여행 참여 불가`를 표시한다.

삭제된 여행의 코드는 없는 코드와 같이 `404 TRIP_INVITATION_NOT_FOUND`를 반환한다. 여행 API에서 삭제된 대상을 `404`로 응답하는 방식과 맞춘다.

> 논의 필요
>
> - [여행 삭제](../policy/trip.md#여행-삭제) 정책은 기존 초대 URL과 코드로 접근하면 삭제된 여행임을 안내하라고 하지만, 현재 응답은 없는 코드와 같은 `TRIP_INVITATION_NOT_FOUND`라서 앱이 둘을 구분할 수 없다. 별도 오류 코드로 구분할지, 정책의 안내 문구를 없는 코드와 같게 고칠지 결정한다. [여행 상세](#trip-03-여행-상세)와 [여행 삭제](#trip-13-여행-삭제)에서 삭제된 여행을 응답하는 방식도 함께 정한다.
> - [참여 조건](../policy/trip.md#참여-조건)의 '접근이 차단된 여행'은 정의된 조건이 없어 현재 응답에서 구분하지 않는다. 차단 조건을 정하면 응답 방식을 함께 결정한다.
>
> 출처: [정책](../policy/trip.md)

### TRIP-07 여행 참여

`POST /api/v1/trip-memberships`

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

초대 코드와 이 여행에서 사용할 여행방 닉네임으로 여행에 참여하고 본인 참여 식별자를 반환한다.

#### 요청

##### Body

Content-Type: `application/json`

요청 필드: [InviteCodeInput](#invitecodeinput).

```json
{
  "inviteCode": "A7K2PQ",
  "nickname": "지훈"
}
```

#### 성공 응답

최초 참여 성공 시 HTTP `201 Created`를 반환한다. 이미 참여 중인 본인의 재요청은 기존 참여 정보를 HTTP `200 OK`로 반환한다. 참여 정보를 가리킬 조회 주소가 없으므로 `Location` 헤더는 넣지 않는다.

공통 응답의 `data`는 [Membership](#membership)이다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [Membership](#membership) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "tripId": "trip-example-a",
    "tripMemberId": "participant-example-b"
  }
}
```

#### 오류

[공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | `inviteCode`가 없거나 비어 있거나 공백뿐이거나, `nickname`이 형식을 벗어남 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 404 | `TRIP_INVITATION_NOT_FOUND` | 초대 코드에 해당하는 여행이 없거나 삭제됨. 형식이 맞지 않는 코드도 포함 |
| 409 | `TRIP_ALREADY_CONFIRMED` | 신규 참여자가 확정된 여행에 참여하려 함 |
| 409 | `TRIP_CAPACITY_FULL` | 신규 참여에 필요한 빈자리가 없음 |
| 409 | `TRIP_NICKNAME_DUPLICATED` | 입력한 여행방 닉네임을 같은 여행의 다른 참여자가 사용 중임 |

#### 처리 규칙

이미 참여 중인 본인은 추가 등록 없이 기존 tripId와 tripMemberId를 반환한다. 여행이 확정됐거나 정원이 찼어도, 다른 닉네임을 보내도 같다. 참여 성공 응답을 받지 못해 재요청해도 같은 참여 정보를 200으로 반환한다. 나간 회원이 확정 전에 다시 참여하면 새 참여 정보를 만들지 않고 나가기 전의 `tripMemberId`를 다시 쓰며, 여행방 닉네임도 나가기 전에 쓰던 닉네임을 그대로 쓴다([재참여](../policy/trip.md#재참여)와 같은 규칙). 신규 참여는 여행 유효성, 확정 여부, 빈자리와 여행방 닉네임 중복을 재검증한다. 여행 종료일이 지났더라도 확정 전이면 참여할 수 있다([참여 조건](../policy/trip.md#참여-조건)과 같은 규칙).

동시 요청으로 중복 참여와 정원 초과를 만들지 않는다. 서버는 참여를 처리하는 동안 여행을 잠가 같은 여행의 참여 요청을 차례로 처리하고, 같은 회원의 중복 참여는 `(trip_id, member_id)` UNIQUE 제약으로도 막는다. 처리 방식은 [ERD 변경 목록](../design/erd-changes.md#반영할-변경)을 따른다.

서버는 초대 코드의 앞뒤 공백을 지우고 대소문자를 구분하지 않고 비교한다. 삭제된 여행의 코드는 없는 코드와 같이 처리하며, 남은 결정은 [초대 코드 검색](#trip-06-초대-코드-검색)의 논의 사항을 따른다. 신규 참여자가 입력한 여행방 닉네임이 같은 여행의 로그인 참여자나 비로그인 참여자의 여행방 닉네임과 같으면 참여를 거절한다. 같은 이름의 비로그인 참여자 때문에 거절됐다면 관리자가 [비로그인 참여자 삭제](#trip-10-비로그인-참여자-삭제)로 그 참여자를 삭제한 뒤 참여할 수 있다.

### TRIP-08 비로그인 참여자 등록

`POST /api/v1/trips/{tripId}/guests`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

#### 인증

여행 확정 전, 로그인한 해당 여행의 관리자만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

빈자리에 이름을 입력해 비로그인 참여자를 한 명씩 등록한다. 요청 한 건은 이름 하나를 받으며 여러 명을 일괄 등록하지 않는다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

##### Body

> 논의 필요
>
> 다음 요청 모델과 예시는 검토 제안이다.
>
> Content-Type: `application/json`
>
> 요청 필드: [GuestInput](#guestinput).
>
> ```json
> {
>   "name": "서연"
> }
> ```

#### 성공 응답

HTTP `201 Created`와 등록한 비로그인 참여자의 주소를 담은 `Location` 헤더를 반환한다. 주소는 `/api/v1/trips/{tripId}/guests/{tripMemberId}` 형식이다.

> 논의 필요
>
> 공통 응답의 `data`는 [Guest](#guest)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [Guest](#guest) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "tripMemberId": "participant-example-c",
>     "name": "서연"
>   }
> }
> ```

#### 오류

[공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 409 | `TRIP_GUEST_NAME_DUPLICATED` | 같은 여행의 다른 참여자가 이미 같은 여행방 닉네임을 사용 중임 |

> 논의 필요
>
> 아래 오류 매핑은 제안이다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
> | 409 | `TRIP_ALREADY_CONFIRMED` | 확정 후 비로그인 참여자를 추가하려 함 |
> | 409 | `TRIP_CAPACITY_FULL` | 등록할 빈자리가 없음 |

#### 처리 규칙

요청 한 건으로 빈자리 한 개에 비로그인 참여자 한 명을 등록한다. 이름은 필수이며 [여행방 닉네임](../policy/trip.md#여행방-닉네임)으로 사용하므로, 같은 여행의 로그인 참여자나 다른 비로그인 참여자의 여행방 닉네임과 겹칠 수 없다. 여행 참여, 확정과 정원 변경이 동시에 요청돼도 정원을 넘기거나 확정 후 참여자를 추가하지 않는다.

### TRIP-09 비로그인 이름 수정

`PATCH /api/v1/trips/{tripId}/guests/{tripMemberId}`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

#### 인증

로그인한 해당 여행의 관리자만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

비로그인 참여자 이름을 수정한다. 여행 확정 전과 후 모두 수정할 수 있다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `tripMemberId` | String | 필수 | 없음 | 여행별 참여자 ID. 회원과 비로그인 참여자를 구분하는 ID다. |

##### Body

> 논의 필요
>
> 다음 요청 모델과 예시는 검토 제안이다.
>
> Content-Type: `application/json`
>
> 요청 필드: [GuestInput](#guestinput).
>
> ```json
> {
>   "name": "서연"
> }
> ```

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [Guest](#guest)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [Guest](#guest) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "tripMemberId": "participant-example-c",
>     "name": "서연"
>   }
> }
> ```

#### 오류

[공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 409 | `TRIP_GUEST_NAME_DUPLICATED` | 같은 여행의 다른 참여자가 이미 같은 여행방 닉네임을 사용 중임 |

> 논의 필요
>
> 아래 오류 매핑은 제안이다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

대상은 같은 여행의 비로그인 참여자여야 한다. 바꿀 이름이 같은 여행의 로그인 참여자나 다른 비로그인 참여자의 여행방 닉네임과 겹치면 거절한다([비로그인 참여자 등록](#trip-08-비로그인-참여자-등록)과 같은 규칙).

### TRIP-10 비로그인 참여자 삭제

`DELETE /api/v1/trips/{tripId}/guests/{tripMemberId}`

#### 인증

여행 확정 전, 로그인한 해당 여행의 관리자만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

확정 전 여행의 비로그인 참여자를 삭제한다. 등록된 참여자 수가 한 명 줄며 총인원은 바뀌지 않는다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `tripMemberId` | String | 필수 | 없음 | 삭제할 비로그인 참여자의 여행별 참여자 ID. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `204 No Content`를 반환한다. 응답 Body는 없다.

#### 오류

[공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 409 | `TRIP_ALREADY_CONFIRMED` | 확정 후 비로그인 참여자를 삭제하려 함 |

> 논의 필요
>
> 아래 오류 매핑은 제안이다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

대상은 같은 여행의 비로그인 참여자여야 한다. 삭제하면 그 참여자의 정보를 지우고 등록 인원(`registeredMemberCount`)이 1 줄어든다. 총인원(`memberCapacity`)은 바뀌지 않는다. 예를 들어 5/5인 여행에서 비로그인 참여자 한 명을 삭제하면 4/5가 되고, 다른 회원이 참여할 수 있다. 여행 참여, 확정과 정원 변경이 동시에 요청돼도 확정한 여행의 참여자를 삭제하지 않는다.

같은 이름의 비로그인 참여자 때문에 [여행 참여](#trip-07-여행-참여)가 거절된 회원은 관리자가 이 API로 그 참여자를 삭제한 뒤 참여할 수 있다.

## 확정과 종료 API

### TRIP-11 여행 확정

`POST /api/v1/trips/{tripId}/confirm`

> 논의 필요
>
> 응답 모델의 필드 구성은 검토 제안이다.
> 출처: [정책](../policy/trip.md#확정-조건) | 기존 검토 DDL `trip.sql`

#### 인증

로그인한 해당 여행의 관리자만 요청할 수 있다. 여행 생성자가 관리자다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

참여자 구성을 확정하고 지출과 정산을 시작할 수 있는 상태로 전환한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

확정 요청에는 HTTP `200 OK`와 요청을 처리한 뒤의 여행 확정 상태를 반환한다. 빈자리가 남아 확정하지 못한 경우와 이미 확정된 여행에 다시 요청한 경우도 `409`가 아니라 `200 OK`를 반환한다. 앱은 응답의 `status`로 확정 여부를 판단한다.

> 논의 필요
>
> 공통 응답의 `data`는 [TripConfirmation](#tripconfirmation)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [TripConfirmation](#tripconfirmation) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> 확정했거나 이미 확정된 여행의 예시:
>
> ```json
> {
>   "data": {
>     "tripId": "trip-example-a",
>     "status": "CONFIRMED"
>   }
> }
> ```
>
> 빈자리가 남아 확정하지 못한 여행의 예시:
>
> ```json
> {
>   "data": {
>     "tripId": "trip-example-a",
>     "status": "UNCONFIRMED"
>   }
> }
> ```

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

빈자리가 0개일 때만 확정한다. 빈자리가 남아 있으면 여행을 바꾸지 않고 확정 전 상태를 반환한다. 이미 확정된 여행은 확정 상태를 유지한 채 그대로 반환하므로, 확정 응답을 받지 못해 다시 요청해도 같은 결과를 받는다. 확정할 때 확정 상태 값과 함께 확정 시각을 저장하며, 확정 시각은 응답에 제공하지 않는다.

앱이 확정 버튼을 비활성화해도 서버는 확정 조건을 다시 확인한다. 앱은 응답의 `status`를 기준으로 확정 버튼을 막아 사용자가 확정 요청을 다시 보내지 않게 한다. 확정 후 모집으로 되돌리지 않는다. 참여자 일괄 등록을 이 요청에 포함하지 않는다.

> 논의 필요
>
> - 확정 요청을 처리하는 동안 다른 사용자의 참여처럼 인원이 바뀌는 요청이 함께 들어오는 예외 상황의 처리를 정한다. 정원과 확정 조건을 위반하지 않아야 한다는 기준은 [동시 참여와 재시도](../policy/trip.md#동시-참여와-재시도)를 따른다.
>
> 출처: [정책](../policy/trip.md#확정-조건) | 기존 검토 DDL `trip.sql`

### TRIP-12 여행 나가기

`POST /api/v1/trips/{tripId}/leave`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

#### 인증

로그인한 일반 여행 참여자만 요청할 수 있다. 관리자는 나가기를 요청할 수 없고, 여행을 삭제해야만 여행에서 나갈 수 있다([나가기와 기록 유지](../policy/trip.md#나가기와-기록-유지)와 같은 규칙). Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

본인의 여행 참여를 종료하고 참여 중 목록에서 제거한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `204 No Content`를 반환하는 안이다. 응답 Body는 없다.

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

미정산 지출이 있어도 일반 회원의 나가기를 허용한다. 앱이 관리자의 나가기 버튼을 비활성화해도 서버는 요청자의 역할을 다시 확인해 관리자의 나가기 요청을 거절한다. 영향 안내 데이터의 별도 조회 여부는 미정이다. 나간 회원은 나가기 전과 후 언제 정산하든 자신이 포함된 지출의 정산에 포함되며, 정산이 발행되면 내 공간에서 조회할 수 있다. 나간 회원이 수령인인 미확인 송금 관계는 수령 확인 불가로 표시한다([나가기와 기록 유지](../policy/trip.md#나가기와-기록-유지)와 같은 규칙).

> 논의 필요
>
> - 미정산 건수와 영향 조회 방식을 결정한다.
>
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

### TRIP-13 여행 삭제

`DELETE /api/v1/trips/{tripId}`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

#### 인증

로그인한 해당 여행의 관리자만 요청할 수 있다. 여행 생성자가 관리자다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

삭제 조건을 충족한 여행을 삭제하고 모든 참여자의 접근을 차단한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `204 No Content`를 반환하는 안이다. 응답 Body는 없다.

#### 오류

[공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 404 | `COMMON_NOT_FOUND` | 이미 삭제한 여행임. 삭제 응답을 받지 못해 다시 보낸 요청도 포함 |

> 논의 필요
>
> 아래 오류 매핑은 제안이다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 여행을 찾을 수 없음 |
> | 409 | `TRIP_RECORDS_EXIST` | 삭제를 막는 지출 또는 정산 기록이 존재함 |

#### 처리 규칙

지출과 정산 기록이 모두 0건이어야 한다. 삭제 후 전원 목록에서 제거하고 초대와 상세 접근을 차단한다. 이미 삭제한 여행을 다시 삭제하면 `404`를 반환한다. 삭제 응답을 받지 못해 다시 보낸 요청도 같다.

> 논의 필요
>
> - 다시 보낸 삭제 요청이 받은 `404`를 앱이 이미 삭제된 것으로 보고 여행 목록으로 이동하는 안을 FE와 확인한다.
>
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

## 공통 모델

`논의 필요`로 표시한 모델의 이름과 필드 구성은 검토 제안이다. 요청 모델은 필수 여부와 기본값을, 응답 모델은 nullable과 포함 조건을 구분한다. 조건부 요청 필드는 예시에 사용한 조건을 만족할 때만 포함한다. 배열 원소는 링크한 모델을 따른다.

### TripSummary

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
> | `name` | String | 아니요 | 항상 | 여행명. |
> | `countryCode` | String | 아니요 | 항상 | 대표 국가. 한국 `KR`, 일본 `JP`, 미국 `US`, 중국 `CN`, 유럽 `EU`, 기타 `ETC`. |
> | `customCountryName` | String | 예 | 항상 | 기타 국가로 직접 입력한 국가명. `countryCode`가 `ETC`가 아니면 null. |
> | `startDate` | String | 아니요 | 항상 | YYYY-MM-DD. 여행 시작일. |
> | `endDate` | String | 아니요 | 항상 | YYYY-MM-DD. 여행 종료일. |
> | `baseCurrency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
> | `memberCapacity` | Integer | 아니요 | 항상 | 총 정원, 1~10명. |
> | `registeredMemberCount` | Integer | 아니요 | 항상 | 회원과 비로그인 참여자의 합. 빈자리 제외. |
> | `status` | String | 아니요 | 항상 | 여행 확정 상태. `UNCONFIRMED`는 확정 전, `CONFIRMED`는 확정 후. |
> | `myRole` | String | 아니요 | 항상 | `ADMIN`은 이 여행의 관리자, `MEMBER`는 일반 참여자다. |

여행 확정 상태는 상태 값(`status`)으로 관리한다. 기타 국가는 `countryCode`를 `ETC`로 두고, 사용자가 입력한 국가명을 국가 코드로 바꾸지 않고 `customCountryName`에 그대로 담는다. 역할 값은 `ADMIN`과 `MEMBER`다.

### TripListItem

[내 여행 목록](#trip-01-내-여행-목록) 응답의 `data` 배열 원소이며, 여행 목록 화면(`HOME-01`)의 여행 카드에 필요한 필드만 담는 모델이다. 카드에 여행 기본 통화를 표시하지 않으므로 `baseCurrency`는 포함하지 않는다. 카드의 사용 금액은 여행 전체의 지출 합계가 아니라 요청한 회원의 누적 부담액이며 원화로 제공한다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
| `name` | String | 아니요 | 항상 | 여행명. |
| `countryCode` | String | 아니요 | 항상 | 대표 국가. 한국 `KR`, 일본 `JP`, 미국 `US`, 중국 `CN`, 유럽 `EU`, 기타 `ETC`. |
| `customCountryName` | String | 예 | 항상 | 기타 국가로 직접 입력한 국가명. `countryCode`가 `ETC`가 아니면 null. |
| `startDate` | String | 아니요 | 항상 | YYYY-MM-DD. 여행 시작일. |
| `endDate` | String | 아니요 | 항상 | YYYY-MM-DD. 여행 종료일. |
| `memberCapacity` | Integer | 아니요 | 항상 | 총 정원, 1~10명. |
| `registeredMemberCount` | Integer | 아니요 | 항상 | 회원과 비로그인 참여자의 합. 빈자리 제외. |
| `status` | String | 아니요 | 항상 | 여행 확정 상태. `UNCONFIRMED`는 확정 전, `CONFIRMED`는 확정 후. |
| `myRole` | String | 아니요 | 항상 | `ADMIN`은 이 여행의 관리자, `MEMBER`는 일반 참여자다. |
| `myShareAmountKrw` | String | 아니요 | 항상 | 요청한 회원의 누적 부담액. 원화이며 소수점 이하 2자리 문자열로 전달한다. |

### TripDetail

여행 상세는 기본 정보와 참여자 목록을 함께 반환한다. 목록 조회와 생성 응답의 TripSummary에 참여자 목록을 추가하는 것은 아니다.

> 구현 제안
>
> TripSummary의 기본 필드에 `participants`를 추가한 응답 모델이다. 기본 필드의 정의는 [TripSummary](#tripsummary)를 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `participants` | Array&lt;[Participant](#participant)&gt; | 아니요 | 항상 | 참여자 목록. 관리자, 로그인 참여자, 비로그인 참여자 순. 빈자리는 넣지 않는다. |

### TripCreate

> 논의 필요
>
> 요청 모델 후보:
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `name` | String | 필수 | 없음 | 공백을 포함해 최대 15자, 한글과 영문, 숫자와 공백 허용. 특수문자 금지. 앞뒤 공백을 지운 값으로 검사하고 저장하며, 공백만 입력하면 거절한다. |
> | `countryCode` | String | 필수 | 없음 | 대표 국가. `KR`, `JP`, `US`, `CN`, `EU`, `ETC` 중 하나. 기타 국가는 `ETC`. |
> | `customCountryName` | String | 조건부 | 없음 | 기타 국가로 직접 입력한 국가명. `countryCode`가 `ETC`이면 필수이며, `ETC`가 아니면 생략한다. `countryCode`가 `ETC`가 아닌데 `customCountryName`을 보내면 무시하지 않고 `400 COMMON_INVALID_REQUEST`로 거절한다. 최대 10자, 한글과 영문만 허용. 가운데 공백, 숫자와 특수문자 금지. 영문 대소문자 구분. 앞뒤 공백을 지운 값으로 검사하고 저장하며, 공백만 입력하면 거절한다([기본 정보](../policy/trip.md#기본-정보)와 같은 규칙). `null` 불가. |
> | `baseCurrency` | String | 필수 | 없음 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
> | `startDate` | String | 필수 | 없음 | YYYY-MM-DD. 여행 시작일. 과거 날짜 허용. |
> | `endDate` | String | 필수 | 없음 | YYYY-MM-DD, startDate 이상. 당일 여행 허용. |
> | `memberCapacity` | Integer | 필수 | 없음 | 생성자 포함 1~10명. |
> | `nickname` | String | 필수 | 없음 | 생성자의 여행방 닉네임. 1\~10자, 한글 완성형(가\~힣), 영문(A\~Z, a\~z)과 숫자(0\~9)만 허용([InviteCodeInput](#invitecodeinput)의 `nickname`과 같은 규칙). |

### Invitation

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `inviteCode` | String | 아니요 | 항상 | 영문 대문자와 숫자를 조합한 6자리 초대 코드. 1차 MVP에서는 재발급하지 않으며, 확정 전까지 시간 제한 없이 유효하다. |
| `inviteUrl` | String | 아니요 | 항상 | 초대 링크. `{기본 주소}/{초대 코드}` 형식이며, 열기만 해서는 참여하지 않는다. |

### InvitationSearch

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
| `name` | String | 아니요 | 항상 | 여행명. |
| `startDate` | String | 아니요 | 항상 | YYYY-MM-DD. 여행 시작일. |
| `endDate` | String | 아니요 | 항상 | YYYY-MM-DD. |
| `registeredMemberCount` | Integer | 아니요 | 항상 | 현재 등록 인원. |
| `memberCapacity` | Integer | 아니요 | 항상 | 총 정원. |
| `status` | String | 아니요 | 항상 | 여행 확정 상태. `UNCONFIRMED`는 확정 전, `CONFIRMED`는 확정 후([TripSummary](#tripsummary)의 `status`와 같은 값). |
| `isAlreadyJoined` | Boolean | 아니요 | 항상 | 현재 요청자의 기존 참여 여부. |

### InviteCodeInput

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `inviteCode` | String | 필수 | 없음 | 공유받은 6자리 초대 코드. 서버는 앞뒤 공백을 지우고 대소문자를 구분하지 않고 비교한다. 비어 있거나 공백뿐이면 400, 형식이 맞지 않으면 없는 코드와 같이 404. |
| `nickname` | String | 필수 | 없음 | 이 여행에서 사용할 여행방 닉네임. 1\~10자, 한글 완성형(가\~힣), 영문(A\~Z, a\~z)과 숫자(0\~9)만 허용([계정 닉네임](../policy/user.md#닉네임)과 같은 규칙). 같은 여행의 다른 참여자와 겹칠 수 없다. |

### Membership

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
| `tripMemberId` | String | 아니요 | 항상 | 여행별 참여자 ID. 회원과 비로그인 참여자를 구분하는 ID다. |

### Participant

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `tripMemberId` | String | 아니요 | 항상 | 여행별 참여자 ID. 회원과 비로그인 참여자를 구분하는 ID다. |
> | `memberId` | String | 예 | 항상 | Billit 회원 계정 ID. 비로그인 참여자이면 null이며, 탈퇴 회원의 값은 보존 정책에 따라 결정한다. |
> | `displayName` | String | 아니요 | 항상 | 참여자의 여행방 닉네임. 이름을 식별자로 사용하지 않는다. |
> | `role` | String | 아니요 | 항상 | `ADMIN`은 이 여행의 관리자, `MEMBER`는 일반 참여자다. |
> | `shareAmountKrw` | String | 아니요 | 항상 | 참여자별 부담액. 각 지출에 등록한 환율로 원화로 환산해 합산하며, 지출 날짜 필터와 관계없이 여행 전체 기간을 기준으로 한다. 소수점 이하 2자리 문자열로 전달한다. |

### GuestInput

> 논의 필요
>
> 요청 모델 후보:
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `name` | String | 필수 | 없음 | 비로그인 참여자의 여행방 닉네임. 1\~10자, 한글 완성형(가\~힣), 영문(A\~Z, a\~z)과 숫자(0\~9)만 허용([계정 닉네임](../policy/user.md#닉네임)과 같은 규칙). 같은 여행의 다른 참여자와 겹칠 수 없다. |

### Guest

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `tripMemberId` | String | 아니요 | 항상 | 여행별 참여자 ID. 회원과 비로그인 참여자를 구분하는 ID다. |
> | `name` | String | 아니요 | 항상 | 등록된 비로그인 이름. |

### TripConfirmation

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
> | `status` | String | 아니요 | 항상 | 요청을 처리한 뒤의 여행 확정 상태. `UNCONFIRMED`는 확정 전, `CONFIRMED`는 확정 후([TripSummary](#tripsummary)의 `status`와 같은 값). |

## 오류 코드

[공통 오류 응답](common.md#오류-응답)을 따른다. 각 API에서 쓰는 조건은 해당 API의 오류 표를 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 409 | `TRIP_ALREADY_CONFIRMED` | 확정된 여행에 확정 전에만 허용하는 요청을 함 |
| 409 | `TRIP_CAPACITY_FULL` | 등록할 빈자리가 없음 |
| 409 | `TRIP_GUEST_NAME_DUPLICATED` | 비로그인 참여자를 등록하거나 이름을 수정할 때 같은 여행의 다른 참여자가 이미 같은 여행방 닉네임을 사용 중임 |
| 409 | `TRIP_IDEMPOTENCY_KEY_REUSED` | 여행 생성 고유키를 다른 회원이 이미 사용했거나, 그 키로 만든 여행이 삭제됨 |
| 404 | `TRIP_INVITATION_NOT_FOUND` | 초대 코드에 해당하는 여행이 없거나 삭제됨. 형식이 맞지 않는 코드도 포함 |
| 409 | `TRIP_NICKNAME_DUPLICATED` | 여행에 참여할 때 입력한 여행방 닉네임을 같은 여행의 다른 참여자가 사용 중임 |

> 논의 필요
>
> 아래 오류 코드의 이름, HTTP 대응과 안내 문구는 제안이다. 공통 코드 이름은 [백엔드 CommonErrorCode](https://github.com/JECT-Study/Trip-Sheet-5th-Server/blob/09dd7bfbaf37974fecfa3bb2fa2c22ea3378d20c/src/main/kotlin/com/ject/global/error/CommonErrorCode.kt)와 맞췄다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 409 | `TRIP_CAPACITY_CHANGE_NOT_ALLOWED` | 확정 후 인원을 변경하거나 등록 인원보다 정원을 줄이려 함 |
> | 409 | `TRIP_RECORDS_EXIST` | 삭제를 막는 지출 또는 정산 기록이 존재함 |
