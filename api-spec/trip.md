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
  - [TRIP-08 참여자 목록](#trip-08-참여자-목록)
  - [TRIP-09 비로그인 참여자 등록](#trip-09-비로그인-참여자-등록)
  - [TRIP-10 비로그인 이름 수정](#trip-10-비로그인-이름-수정)
- [확정과 종료 API](#확정과-종료-api)
  - [TRIP-11 여행 확정](#trip-11-여행-확정)
  - [TRIP-12 여행 나가기](#trip-12-여행-나가기)
  - [TRIP-13 여행 삭제](#trip-13-여행-삭제)
- [공통 모델](#공통-모델)
  - [TripSummary](#tripsummary)
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

이 문서는 API 스펙 검토용 초안이다. API 식별자는 화면 정의서 식별자와 별개다. 구체적인 경로, 요청과 응답 필드 및 오류 코드는 아직 팀 검토를 마치지 않았다. 각 API의 `논의 필요` 인용문 안에 제안 모델과 예시를 구분한다.

## API 목록

| 기능 | Method | Path | 인증 |
| --- | --- | --- | --- |
| [내 여행 목록](#trip-01-내-여행-목록) | `GET` | `/api/v1/trips` | 로그인 필요 |
| [여행 생성](#trip-02-여행-생성) | `POST` | `/api/v1/trips` | 로그인 필요 |
| [여행 상세](#trip-03-여행-상세) | `GET` | `/api/v1/trips/{tripId}` | 여행 참여자 |
| [여행 정보 수정](#trip-04-여행-정보-수정) | `PATCH` | `/api/v1/trips/{tripId}` | 여행 관리자 |
| [초대 정보 조회](#trip-05-초대-정보-조회) | `GET` | `/api/v1/trips/{tripId}/invitation` | 여행 관리자 (여행 확정 전) |
| [초대 코드 검색](#trip-06-초대-코드-검색) | `GET` | `/api/v1/trip-invitations` | 로그인 필요 |
| [여행 참여](#trip-07-여행-참여) | `POST` | `/api/v1/trip-memberships` | 로그인 필요 |
| [참여자 목록](#trip-08-참여자-목록) | `GET` | `/api/v1/trips/{tripId}/members` | 여행 참여자 |
| [비로그인 참여자 등록](#trip-09-비로그인-참여자-등록) | `POST` | `/api/v1/trips/{tripId}/guests` | 여행 관리자 (여행 확정 전) |
| [비로그인 이름 수정](#trip-10-비로그인-이름-수정) | `PATCH` | `/api/v1/trips/{tripId}/guests/{tripMemberId}` | 여행 관리자 (여행 확정 전) |
| [여행 확정](#trip-11-여행-확정) | `POST` | `/api/v1/trips/{tripId}/confirm` | 여행 관리자 |
| [여행 나가기](#trip-12-여행-나가기) | `POST` | `/api/v1/trips/{tripId}/leave` | 여행 참여자 (관리자 나가기 논의 필요) |
| [여행 삭제](#trip-13-여행-삭제) | `DELETE` | `/api/v1/trips/{tripId}` | 여행 관리자 |

## 여행 조회와 관리 API

### TRIP-01 내 여행 목록

`GET /api/v1/trips`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 회원이 참여하고 접근할 수 있는 여행 목록을 조회한다.

#### 요청

##### Query Parameter

> 논의 필요
>
> 다음 조회 필드는 검토 제안이다. 선택 필터를 생략하면 해당 조건을 적용하지 않는다.
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `page` | Integer | 선택 | 0 | 0 이상. 다음 페이지는 현재 page+1. |
> | `size` | Integer | 선택 | 20 | 1~100, 범위 밖이면 400. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 Array&lt;[TripSummary](#tripsummary)&gt;이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | Array&lt;[TripSummary](#tripsummary)&gt; | 아니요 | 항상 | 이 API의 처리 결과. |
>
> 응답 최상위 `page`는 [Page](common.md#page)를 따른다. 페이지 적용 자체와 개별 기본 정렬은 검토 대상이다.
>
> ```json
> {
>   "data": [
>     {
>       "tripId": "trip-example-a",
>       "name": "도쿄 여행",
>       "startDate": "2026-11-05",
>       "endDate": "2026-11-07",
>       "baseCurrency": "JPY",
>       "memberCapacity": 3,
>       "registeredMemberCount": 3,
>       "membersLockedAt": null,
>       "myRole": "ADMIN"
>     }
>   ],
>   "page": {
>     "page": 0,
>     "size": 20,
>     "totalElements": 1,
>     "hasNext": false
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

#### 처리 규칙

현재 접근 가능한 참여 여행만 조회한다. 진행 중 여행의 별도 영역과 페이지 관계는 미정이다.

> 논의 필요
>
> - 확정 상태와 일정 상태의 필터 enum, 진행 중 여행 별도 영역, 시작일이 가까운 여행부터 정렬하는 기준의 정확한 정의와 시간대를 결정한다. 모델에 일정 상태를 임의 enum으로 추가하지 않았다.
>
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

### TRIP-02 여행 생성

`POST /api/v1/trips`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

여행 기본 정보와 정원을 입력해 새 여행을 생성한다.

#### 요청

##### Body

> 논의 필요
>
> 대표 국가 필드가 아직 없는 모델이다. 국가 입력 형식을 정한 뒤 전체 요청 모델을 완성한다.
>
> Content-Type: `application/json`
>
> 요청 필드: [TripCreate](#tripcreate).

#### 성공 응답

> 논의 필요
>
> HTTP `201 Created`를 반환하는 안이다.
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
>     "name": "도쿄 여행",
>     "startDate": "2026-11-05",
>     "endDate": "2026-11-07",
>     "baseCurrency": "JPY",
>     "memberCapacity": 3,
>     "registeredMemberCount": 3,
>     "membersLockedAt": null,
>     "myRole": "ADMIN"
>   }
> }
> ```
>
> 생성한 대상의 조회 URL을 `Location`에 제공한다. 조회 경로를 아직 정의하지 않은 자원은 해당 경로를 함께 확정해야 한다.

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

생성자를 첫 참여자로 등록한다. 정원 1~10명, 종료일은 시작일 이상이다.

앱은 여행 생성 요청에 고유키를 함께 보내며, 같은 생성 요청을 재시도할 때는 같은 키를 사용한다. 서버는 같은 키의 재시도에 기존 여행을 반환하며 여행과 생성자 참여 정보를 중복 생성하지 않는다. 상세한 보장과 남은 선택은 [여행 생성 멱등성](../design/idempotency.md#여행-생성-멱등성)을 따른다.

> 논의 필요
>
> - TripCreate는 국가 표현을 제외한 입력 후보이며 완전한 요청 모델이 아니다. 한국, 일본, 유럽, 미국, 중국과 기타의 코드 및 기타 직접 입력 필드를 결정한 뒤 전체 요청 예시를 작성한다.
> - 고유키의 형식과 전달 위치, 적용 범위와 보존 기간, 같은 키로 다른 내용을 보낸 경우와 동시 요청 처리, 기존 여행 반환 시 HTTP 상태와 응답 시점을 결정한다.
> - 여행명 글자 수와 공백 검증을 결정한다.
>
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

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
>     "startDate": "2026-11-05",
>     "endDate": "2026-11-07",
>     "baseCurrency": "JPY",
>     "memberCapacity": 3,
>     "registeredMemberCount": 3,
>     "membersLockedAt": null,
>     "myRole": "ADMIN",
>     "participants": [
>       {
>         "tripMemberId": "participant-example-a",
>         "memberId": "member-example-a",
>         "displayName": "민지",
>         "role": "ADMIN",
>         "isVacant": false
>       },
>       {
>         "tripMemberId": "participant-example-b",
>         "memberId": "member-example-b",
>         "displayName": "지훈",
>         "role": "MEMBER",
>         "isVacant": false
>       },
>       {
>         "tripMemberId": "participant-example-c",
>         "memberId": null,
>         "displayName": "서연",
>         "role": "MEMBER",
>         "isVacant": false
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

> 논의 필요
>
> - 대표 국가 모델, 참여자 상태와 빈자리의 응답 표현, 참여자별 누적 부담액과 총액의 통화 및 날짜 필터 기준을 결정한다.
>
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

### TRIP-04 여행 정보 수정

`PATCH /api/v1/trips/{tripId}`

> 논의 필요
>
> 부분 수정 메서드 PATCH는 확정했다. 최종 경로, 요청 필드와 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

#### 인증

로그인한 해당 여행의 관리자만 요청할 수 있다. 여행 생성자가 관리자다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

여행 기본 정보와 변경 가능한 정원을 PATCH로 일부 수정한다. 요청에 보낸 필드만 변경하고 생략한 필드는 기존 값을 유지한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |

> 논의 필요
>
> 변경할 여행 기본 정보 또는 `memberCapacity`를 전달한다. 대표 국가와 기타 국가명을 표현할 필드, 각 필드에 null을 보낼 수 있는지 정한 뒤 요청 표와 예시를 작성한다.

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
>     "name": "도쿄 여행",
>     "startDate": "2026-11-05",
>     "endDate": "2026-11-07",
>     "baseCurrency": "JPY",
>     "memberCapacity": 3,
>     "registeredMemberCount": 3,
>     "membersLockedAt": null,
>     "myRole": "ADMIN"
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
> | 409 | `TRIP_CAPACITY_CHANGE_NOT_ALLOWED` | 확정 후 인원을 변경하거나 등록 인원보다 정원을 줄이려 함 |

#### 처리 규칙

확정 후 정원 변경은 금지한다. 등록 인원보다 줄일 수 없다. 기존 지출과 정산에 소급 반영하지 않는다.

> 논의 필요
>
> - 국가 표현과 각 필드 null 정책 확정 후 요청 표와 예시를 작성한다.
>
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

## 초대와 참여자 API

### TRIP-05 초대 정보 조회

`GET /api/v1/trips/{tripId}/invitation`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

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

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [Invitation](#invitation)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [Invitation](#invitation) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "inviteCode": "EXAMPLEA",
>     "inviteUrl": "https://example.com/invitations/EXAMPLEA"
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
> | 409 | `TRIP_ALREADY_CONFIRMED` | 확정 후 신규 초대 정보를 요청함 |

#### 처리 규칙

확정 후 신규 초대를 허용하지 않는다. 코드 재발급과 중지 API는 도입 결정 후 추가한다.

> 논의 필요
>
> - 초대 코드 입력의 공백과 소문자 처리, 만료, 재발급과 중지 제공 여부를 결정한다.
>
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

### TRIP-06 초대 코드 검색

`GET /api/v1/trip-invitations`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

초대 코드에 해당하는 여행의 참여 전 확인 정보를 조회한다.

#### 요청

##### Query Parameter

> 논의 필요
>
> 다음 조회 필드는 검토 제안이다. 선택 필터를 생략하면 해당 조건을 적용하지 않는다.
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `code` | String | 필수 | 없음 | 영문 대문자와 숫자를 조합한 6자리 초대 코드. 공백과 소문자 입력 처리는 논의 필요. |

요청 Body는 보내지 않는다.

#### 성공 응답

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [InvitationSearch](#invitationsearch)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [InvitationSearch](#invitationsearch) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "tripId": "trip-example-a",
>     "name": "도쿄 여행",
>     "startDate": "2026-11-05",
>     "endDate": "2026-11-07",
>     "registeredMemberCount": 2,
>     "memberCapacity": 3,
>     "isAlreadyJoined": false
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
> | 404 | `TRIP_INVITATION_NOT_FOUND` | 해당 코드로 검색 가능한 여행이 없음 |

#### 처리 규칙

검색만으로 참여하거나 빈자리를 배정하지 않는다. 삭제와 접근 차단 안내, 확정 후 검색 응답 범위는 미정이다.

> 논의 필요
>
> - 삭제와 접근 차단 및 확정 후 검색 응답, 일정 종료 후 미확정 여행의 참여 허용을 결정한다.
>
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

### TRIP-07 여행 참여

`POST /api/v1/trip-memberships`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

초대 코드로 여행에 참여하고 본인 참여 식별자를 반환한다.

#### 요청

##### Body

> 논의 필요
>
> 다음 요청 모델과 예시는 검토 제안이다.
>
> Content-Type: `application/json`
>
> 요청 필드: [InviteCodeInput](#invitecodeinput).
>
> ```json
> {
>   "inviteCode": "EXAMPLEA"
> }
> ```

#### 성공 응답

최초 참여 성공 시 HTTP `201 Created`를 반환한다. 이미 참여 중인 본인의 재요청은 기존 참여 정보를 HTTP `200 OK`로 반환한다.

> 논의 필요
>
> 아래 응답 모델과 필드는 검토 제안이다.
>
> 공통 응답의 `data`는 [Membership](#membership)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [Membership](#membership) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "tripId": "trip-example-a",
>     "tripMemberId": "participant-example-a"
>   }
> }
> ```
>
> 생성한 대상의 조회 URL을 `Location`에 제공한다. 조회 경로를 아직 정의하지 않은 자원은 해당 경로를 함께 확정해야 한다.

#### 오류

> 논의 필요
>
> 아래 오류 매핑은 제안이다. 확정된 공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 409 | `TRIP_ALREADY_CONFIRMED` | 신규 참여자가 확정된 여행에 참여하려 함 |
> | 409 | `TRIP_CAPACITY_FULL` | 신규 참여에 필요한 빈자리가 없음 |

#### 처리 규칙

이미 참여 중인 본인은 추가 등록 없이 기존 tripId와 tripMemberId를 반환한다. 참여 성공 응답을 받지 못해 재요청해도 같은 참여 정보를 200으로 반환한다. 이 재요청 응답은 현재 참여 중인 회원에게 적용하며, 나간 뒤 재참여하는 경우의 참여 ID 재사용 여부와 구분한다. 신규 참여는 여행 유효성, 확정 여부와 빈자리를 재검증한다. 동시 요청으로 중복 참여와 정원 초과를 만들지 않는다.

> 논의 필요
>
> - 동시 요청에서 중복 참여와 정원 초과를 방지하는 저장 제약 및 트랜잭션 방식을 결정한다.
> - 확정 전 재참여 시 기존 참여 ID 재사용 여부를 결정한다.
>
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

### TRIP-08 참여자 목록

`GET /api/v1/trips/{tripId}/members`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

#### 인증

로그인한 회원 중 해당 여행에 참여하고 여행방에 접근할 수 있는 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

여행의 등록 참여자와 빈자리를 조회한다.

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
> 공통 응답의 `data`는 Array&lt;[Participant](#participant)&gt;이다. 필드 정의는 해당 모델을 따른다. 이 모델은 일부 필드만 정의한 구조이며, 아래 예시는 정의된 필드 범위만 보여준다. 전체 요청과 응답 규격은 이 API의 논의 사항을 반영해 완성해야 한다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | Array&lt;[Participant](#participant)&gt; | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": [
>     {
>       "tripMemberId": "participant-example-a",
>       "memberId": "member-example-a",
>       "displayName": "민지",
>       "role": "ADMIN",
>       "isVacant": false
>     }
>   ]
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

참여자 목록은 [여행 상세](#trip-03-여행-상세) 응답에 포함한다. 상세 화면 진입에 이 API의 추가 호출을 요구하지 않는다. 별도 참여자 조회 API를 유지할지는 다른 화면의 독립 조회 필요성을 확인한 뒤 결정한다. 빈자리를 부담자로 제공하지 않는다.

> 논의 필요
>
> - 별도 참여자 조회 API 유지 여부와 회원, 비로그인, 나감 및 탈퇴 상태를 어떤 응답 필드로 구분할지 결정한다.
>
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

### TRIP-09 비로그인 참여자 등록

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

> 논의 필요
>
> HTTP `201 Created`를 반환하는 안이다.
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
>
> 생성한 대상의 조회 URL을 `Location`에 제공한다. 조회 경로를 아직 정의하지 않은 자원은 해당 경로를 함께 확정해야 한다.

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
> | 409 | `TRIP_ALREADY_CONFIRMED` | 확정 후 비로그인 참여자를 추가하려 함 |
> | 409 | `TRIP_CAPACITY_FULL` | 등록할 빈자리가 없음 |

#### 처리 규칙

요청 한 건으로 빈자리 한 개에 비로그인 참여자 한 명을 등록한다. 이름은 필수이며 최대 10자이고 같은 여행의 비로그인 참여자끼리 이름 중복을 허용하지 않는다. 여행 참여, 확정과 정원 변경이 동시에 요청돼도 정원을 넘기거나 확정 후 참여자를 추가하지 않는다.

> 논의 필요
>
> - 이름 허용 문자와 공백, 등록 후 삭제 제공 여부와 회원 닉네임이 같은 경우의 표시를 결정한다.
>
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

### TRIP-10 비로그인 이름 수정

`PATCH /api/v1/trips/{tripId}/guests/{tripMemberId}`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

#### 인증

여행 확정 전, 로그인한 해당 여행의 관리자만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

확정 전 여행의 비로그인 참여자 이름을 수정한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `tripId` | String | 필수 | 없음 | 여행 ID. 문자열로 전달한다. |
| `tripMemberId` | String | 필수 | 없음 | 여행별 참여자 ID. 회원과 비로그인 참여자 및 빈자리를 구분하는 ID다. |

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
> | 409 | `TRIP_ALREADY_CONFIRMED` | 확정 후 비로그인 이름을 수정하려 함 |

#### 처리 규칙

대상은 같은 여행의 비로그인 참여자여야 한다.

## 확정과 종료 API

### TRIP-11 여행 확정

`POST /api/v1/trips/{tripId}/confirm`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

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

> 논의 필요
>
> HTTP `200 OK`를 반환하는 안이다.
>
> 공통 응답의 `data`는 [TripConfirmation](#tripconfirmation)이다. 필드 정의는 해당 모델을 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `data` | [TripConfirmation](#tripconfirmation) | 아니요 | 항상 | 이 API의 처리 결과. |
>
> ```json
> {
>   "data": {
>     "tripId": "trip-example-a",
>     "membersLockedAt": "2026-11-05T03:00:00Z"
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
> | 409 | `TRIP_VACANCIES_REMAIN` | 등록 참여자 수가 정원에 미달함 |

#### 처리 규칙

빈자리가 0개여야 한다. 확정 후 모집으로 되돌리지 않는다. 참여자 일괄 등록을 이 요청에 포함하지 않는다.

> 논의 필요
>
> - 확정 재요청의 성공 또는 충돌 응답을 결정한다.
>
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

### TRIP-12 여행 나가기

`POST /api/v1/trips/{tripId}/leave`

> 논의 필요
>
> 경로, 요청과 응답 규격은 검토 제안이다.
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

#### 인증

로그인한 일반 여행 참여자가 요청할 수 있다. 여행 관리자도 나갈 수 있는지는 결정해야 한다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

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

미정산 지출이 있어도 일반 회원의 나가기를 허용한다. 영향 안내 데이터의 별도 조회 여부는 미정이다. 완료된 내 정산은 유지한다.

> 논의 필요
>
> - 관리자 나가기 허용 여부, 미정산 건수와 영향 조회 방식, 나간 회원의 진행 중 정산 조회와 이후 정산 포함, 재참여 ID를 결정한다.
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
> | 409 | `TRIP_RECORDS_EXIST` | 삭제를 막는 지출 또는 정산 기록이 존재함 |

#### 처리 규칙

지출과 정산 기록이 모두 0건이어야 한다. 삭제 후 전원 목록에서 제거하고 초대와 상세 접근을 차단한다.

> 논의 필요
>
> - 삭제 지출과 취소한 준비 정산의 기록 건수 포함 여부, 이미 삭제한 여행의 재삭제 응답을 결정한다.
>
> 출처: [정책](../policy/trip.md) | 기존 검토 DDL `trip.sql`

## 공통 모델

모델 이름과 필드 구성은 모두 검토 제안이다. 요청 모델은 필수 여부와 기본값을, 응답 모델은 nullable과 포함 조건을 구분한다. 조건부 요청 필드는 예시에 사용한 조건을 만족할 때만 포함한다. 배열 원소는 링크한 모델을 따른다.

### TripSummary

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
> | `name` | String | 아니요 | 항상 | 여행명. |
> | `startDate` | String | 아니요 | 항상 | YYYY-MM-DD. 여행 시작일. |
> | `endDate` | String | 아니요 | 항상 | YYYY-MM-DD. 여행 종료일. |
> | `baseCurrency` | String | 아니요 | 항상 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
> | `memberCapacity` | Integer | 아니요 | 항상 | 총 정원, 1~10명. |
> | `registeredMemberCount` | Integer | 아니요 | 항상 | 회원과 비로그인 참여자의 합. 빈자리 제외. |
> | `membersLockedAt` | String | 예 | 항상 | ISO 8601 UTC. 확정 전에는 null. |
> | `myRole` | String | 아니요 | 항상 | `ADMIN`은 이 여행의 관리자, `MEMBER`는 일반 참여자다. |

### TripDetail

여행 상세는 기본 정보와 참여자 목록을 함께 반환한다. 목록 조회와 생성 응답의 TripSummary에 참여자 목록을 추가하는 것은 아니다.

> 구현 제안
>
> TripSummary의 기본 필드에 `participants`를 추가한 응답 모델이다. 기본 필드의 정의는 [TripSummary](#tripsummary)를 따른다.
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `participants` | Array&lt;[Participant](#participant)&gt; | 아니요 | 항상 | 참여자 목록. 상태 구분, 빈자리 표현과 정렬은 별도 확정 |

### TripCreate

> 논의 필요
>
> 요청 모델 후보:
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `name` | String | 필수 | 없음 | 최대 15자, 한글과 영문, 숫자와 공백 허용. 특수문자 금지. |
> | `baseCurrency` | String | 필수 | 없음 | `KRW`, `JPY`, `USD`, `EUR`, `CNY` 중 하나. |
> | `startDate` | String | 필수 | 없음 | YYYY-MM-DD. 여행 시작일. 과거 날짜 허용. |
> | `endDate` | String | 필수 | 없음 | YYYY-MM-DD, startDate 이상. 당일 여행 허용. |
> | `memberCapacity` | Integer | 필수 | 없음 | 생성자 포함 1~10명. |

### Invitation

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `inviteCode` | String | 아니요 | 항상 | 영문 대문자와 숫자를 조합한 6자리 초대 코드. 입력 정규화 및 만료는 논의 필요. |
> | `inviteUrl` | String | 아니요 | 항상 | 초대 링크. 열기만 해서는 참여하지 않는다. |

### InvitationSearch

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
> | `name` | String | 아니요 | 항상 | 여행명. |
> | `startDate` | String | 아니요 | 항상 | YYYY-MM-DD. 여행 시작일. |
> | `endDate` | String | 아니요 | 항상 | YYYY-MM-DD. |
> | `registeredMemberCount` | Integer | 아니요 | 항상 | 현재 등록 인원. |
> | `memberCapacity` | Integer | 아니요 | 항상 | 총 정원. |
> | `isAlreadyJoined` | Boolean | 아니요 | 항상 | 현재 요청자의 기존 참여 여부. |

### InviteCodeInput

> 논의 필요
>
> 요청 모델 후보:
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `inviteCode` | String | 필수 | 없음 | 공유받은 코드. 길이, 공백과 대소문자 규칙은 논의 필요. |

### Membership

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
> | `tripMemberId` | String | 아니요 | 항상 | 여행별 참여자 ID. 회원과 비로그인 참여자 및 빈자리를 구분하는 ID다. |

### Participant

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `tripMemberId` | String | 아니요 | 항상 | 여행별 참여자 ID. 회원과 비로그인 참여자 및 빈자리를 구분하는 ID다. |
> | `memberId` | String | 예 | 항상 | Billit 회원 계정 ID. 비로그인 참여자와 빈자리이면 null이며, 탈퇴 회원의 값은 보존 정책에 따라 결정한다. |
> | `displayName` | String | 예 | 항상 | 표시 이름. 빈자리이면 null. 중복 이름을 식별자로 사용하지 않는다. |
> | `role` | String | 아니요 | 항상 | `ADMIN`은 이 여행의 관리자, `MEMBER`는 일반 참여자다. |
> | `isVacant` | Boolean | 아니요 | 항상 | 아무 참여자도 배정되지 않은 자리이면 true. |

### GuestInput

> 논의 필요
>
> 요청 모델 후보:
>
> | 필드 | 타입 | 필수 | 기본값 | 규칙 |
> | --- | --- | --- | --- | --- |
> | `name` | String | 필수 | 없음 | 필수, 최대 10자. 중복 허용. 허용 문자와 공백 규칙은 논의 필요. |

### Guest

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `tripMemberId` | String | 아니요 | 항상 | 여행별 참여자 ID. 회원과 비로그인 참여자 및 빈자리를 구분하는 ID다. |
> | `name` | String | 아니요 | 항상 | 등록된 비로그인 이름. |

### TripConfirmation

> 논의 필요
>
> 응답 모델 후보:
>
> | 필드 | 타입 | nullable | 포함 조건 | 설명 |
> | --- | --- | --- | --- | --- |
> | `tripId` | String | 아니요 | 항상 | 여행 ID. 문자열로 전달한다. |
> | `membersLockedAt` | String | 아니요 | 항상 | ISO 8601 UTC. 여행 확정 시각. |

## 오류 코드

> 논의 필요
>
> 도메인 오류 코드의 이름, HTTP 대응과 안내 문구는 제안이다. 공통 코드 이름은 [백엔드 CommonErrorCode](https://github.com/JECT-Study/Trip-Sheet-5th-Server/blob/09dd7bfbaf37974fecfa3bb2fa2c22ea3378d20c/src/main/kotlin/com/ject/global/error/CommonErrorCode.kt)와 맞췄다. 개별 API의 오류 조건과 도메인 코드 이름은 팀 검토 후 확정한다.
>
> | HTTP | 코드 | 조건 |
> | --- | --- | --- |
> | 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
> | 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
> | 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
> | 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
> | 409 | `TRIP_ALREADY_CONFIRMED` | 확정 후 비로그인 이름을 수정하려 함 |
> | 409 | `TRIP_CAPACITY_CHANGE_NOT_ALLOWED` | 확정 후 인원을 변경하거나 등록 인원보다 정원을 줄이려 함 |
> | 409 | `TRIP_CAPACITY_FULL` | 등록할 빈자리가 없음 |
> | 404 | `TRIP_INVITATION_NOT_FOUND` | 해당 코드로 검색 가능한 여행이 없음 |
> | 409 | `TRIP_RECORDS_EXIST` | 삭제를 막는 지출 또는 정산 기록이 존재함 |
> | 409 | `TRIP_VACANCIES_REMAIN` | 등록 참여자 수가 정원에 미달함 |
