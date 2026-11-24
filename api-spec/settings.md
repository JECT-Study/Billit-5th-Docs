# 설정 API 스펙 초안 (1차 MVP)

## 목차

- [문서 범위](#문서-범위)
- [API 목록](#api-목록)
- [알림 설정과 기기 API](#알림-설정과-기기-api)
  - [SET-01 알림 설정 조회: 조건부](#set-01-알림-설정-조회-조건부)
  - [SET-02 알림 설정 수정: 조건부](#set-02-알림-설정-수정-조건부)
  - [SET-03 푸시 기기 등록: 조건부](#set-03-푸시-기기-등록-조건부)
  - [SET-04 푸시 기기 해제: 조건부](#set-04-푸시-기기-해제-조건부)
- [알림함 API](#알림함-api)
  - [SET-05 알림 목록: 조건부](#set-05-알림-목록-조건부)
  - [SET-06 알림 읽음: 조건부](#set-06-알림-읽음-조건부)
- [공지 API](#공지-api)
  - [SET-07 공지 목록](#set-07-공지-목록)
  - [SET-08 공지 상세](#set-08-공지-상세)
- [클라이언트 기능과 다른 도메인](#클라이언트-기능과-다른-도메인)
- [공통 모델](#공통-모델)
  - [PushDeviceInput](#pushdeviceinput)
  - [PushDevice](#pushdevice)
  - [NotificationSummary](#notificationsummary)
  - [NotificationRead](#notificationread)
  - [NoticeSummary](#noticesummary)
  - [NoticeDetail](#noticedetail)
- [오류 코드](#오류-코드)

## 문서 범위

알림 API는 지출과 정산 기능을 완료한 뒤 1차 MVP 포함 여부를 검토하는 조건부 후보다. 공지사항은 앱 내 화면 제공이 확정됐으며 구체적인 API 계약은 검토 제안이다. 설정 화면에 있다는 이유만으로 별도 API를 확정하지 않는다.

[목록](README.md) | [공통 API 스펙](common.md) | [정책](../policy/settings.md)

이 문서는 API 스펙 검토용 초안이다.
API 식별자는 화면 정의서 식별자와 별개다.
경로, 요청과 응답 필드, 오류 코드를 팀에서 확정하지 않은 API는 Method와 Path 아래에 `논의 필요`로 표시한다.

## API 목록

| 기능 | Method | Path | 인증 |
| --- | --- | --- | --- |
| [알림 설정 조회: 조건부](#set-01-알림-설정-조회-조건부) | `GET` | `/api/v1/me/notification-settings` | 로그인 필요 |
| [알림 설정 수정: 조건부](#set-02-알림-설정-수정-조건부) | `PATCH` | `/api/v1/me/notification-settings` | 로그인 필요 |
| [푸시 기기 등록: 조건부](#set-03-푸시-기기-등록-조건부) | `POST` | `/api/v1/me/push-devices` | 로그인 필요 |
| [푸시 기기 해제: 조건부](#set-04-푸시-기기-해제-조건부) | `DELETE` | `/api/v1/me/push-devices/{pushDeviceId}` | 로그인 필요 (본인이 등록한 기기) |
| [알림 목록: 조건부](#set-05-알림-목록-조건부) | `GET` | `/api/v1/me/notifications` | 로그인 필요 |
| [알림 읽음: 조건부](#set-06-알림-읽음-조건부) | `PATCH` | `/api/v1/me/notifications/{notificationId}` | 로그인 필요 (본인이 받은 알림) |
| [공지 목록](#set-07-공지-목록) | `GET` | `/api/v1/notices` | 인증 범위 논의 필요 |
| [공지 상세](#set-08-공지-상세) | `GET` | `/api/v1/notices/{noticeId}` | 인증 범위 논의 필요 |

## 알림 설정과 기기 API

### SET-01 알림 설정 조회: 조건부

`GET /api/v1/me/notification-settings`

> 논의 필요
>
> - 경로, 요청, 응답과 오류 코드의 확정 여부
> - 제공 범위나 구현 방식을 정한 뒤의 채택 여부

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 회원의 알림 수신 설정을 조회한다.

#### 요청

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

> 논의 필요
>
> - 알림 종류별 수신 여부를 반환한다.
> - 지원할 알림 종류를 정한 뒤 응답 필드와 예시를 작성한다.

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |

#### 처리 규칙

알림의 MVP 포함 여부와 설정 제공 범위를 정하면 이 API의 채택 여부를 정한다.

> 논의 필요
>
> 기본 수신값과 계정별 또는 기기별 적용 범위. 알림의 MVP 포함 여부와 종류는 [알림 정책](../policy/settings.md#알림)의 논의 필요를 따르며, 정할 때까지 필드 표와 예시는 보류한다.

### SET-02 알림 설정 수정: 조건부

`PATCH /api/v1/me/notification-settings`

> 논의 필요
>
> - 경로, 요청, 응답과 오류 코드의 확정 여부
> - 제공 범위나 구현 방식을 정한 뒤의 채택 여부

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 회원의 알림 수신 설정을 변경한다.

#### 요청

> 논의 필요
>
> - 수신 여부를 바꿀 알림 종류와 값을 전달한다.
> - 알림 종류와 요청 필드가 정해지면 필드 표와 예시를 작성한다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

> 논의 필요
>
> - 변경된 알림 수신 설정을 반환한다.
> - 지원할 알림 종류를 정한 뒤 응답 필드와 예시를 작성한다.

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |

#### 처리 규칙

지원하지 않는 알림 종류를 보내면 서버는 요청을 거절한다.

> 논의 필요
>
> 지원 알림 종류, 수신값과 `null`을 보냈을 때의 처리, PATCH 요청의 필드 모델

### SET-03 푸시 기기 등록: 조건부

`POST /api/v1/me/push-devices`

> 논의 필요
>
> - 경로, 요청, 응답과 오류 코드의 확정 여부
> - 제공 범위나 구현 방식을 정한 뒤의 채택 여부

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

현재 기기로 알림을 보낼 때 사용할 푸시 토큰을 등록한다.

#### 요청

##### Body

Content-Type: `application/json`

요청 필드: [PushDeviceInput](#pushdeviceinput).

```json
{
  "pushToken": "example-push-token",
  "installationId": "example-installation"
}
```

#### 성공 응답

HTTP `201 Created`를 반환한다.

공통 응답의 `data`는 [PushDevice](#pushdevice)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [PushDevice](#pushdevice) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "pushDeviceId": "push-device-example-a"
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

#### 처리 규칙

서비스가 푸시를 도입하면 이 API를 채택한다.

> 논의 필요
>
> - 푸시 제공 여부와 플랫폼 표현
> - 설치 식별과 계정 전환, 중복 등록과 토큰 교체의 처리
> - 푸시 발송 방식에 따른 `pushToken`의 형식. Expo Push Service를 쓰는지 FCM과 APNs를 직접 쓰는지에 따라 달라지므로 [설정 정책](../policy/settings.md#알림)의 결정과 함께 정한다.

### SET-04 푸시 기기 해제: 조건부

`DELETE /api/v1/me/push-devices/{pushDeviceId}`

> 논의 필요
>
> - 경로, 요청, 응답과 오류 코드의 확정 여부
> - 제공 범위나 구현 방식을 정한 뒤의 채택 여부

#### 인증

로그인한 회원은 본인이 등록한 기기만 해제할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

지정한 앱 설치의 푸시 수신 등록을 해제한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `pushDeviceId` | String | 필수 | 없음 | 본인 계정에 등록된 푸시 수신 기기 ID. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `204 No Content`를 반환한다.
응답 Body는 없다.

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

서버는 다른 회원과 다른 기기의 등록을 임의로 해제하지 않는다.

> 논의 필요
>
> 로그아웃과 탈퇴 때의 해제 연계, 반복 해제의 응답

## 알림함 API

### SET-05 알림 목록: 조건부

`GET /api/v1/me/notifications`

> 논의 필요
>
> - 경로, 요청, 응답과 오류 코드의 확정 여부
> - 제공 범위나 구현 방식을 정한 뒤의 채택 여부

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

본인에게 제공된 앱 내 알림 목록을 조회한다.

#### 요청

##### Query Parameter

선택 필터를 생략하면 그 조건을 적용하지 않는다.

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `page` | Integer | 선택 | 0 | 0 이상. 다음 페이지는 현재 page+1. |
| `size` | Integer | 선택 | 20 | 1~100, 범위 밖이면 400. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 Array&lt;[NotificationSummary](#notificationsummary)&gt;이다.
필드 정의는 그 모델을 따른다.
이 모델은 일부 필드만 정의한 구조이며, 아래 예시는 정의된 필드 범위만 보여준다.
전체 요청과 응답 규격은 이 API의 논의 사항을 반영해 완성해야 한다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | Array&lt;[NotificationSummary](#notificationsummary)&gt; | 아니요 | 항상 | 이 API의 처리 결과. |

응답 최상위 `page`는 [Page](common.md#page)를 따른다.
페이지 적용 자체와 개별 기본 정렬은 검토 대상이다.

```json
{
  "data": [
    {
      "notificationId": "notification-example-a",
      "title": "정산서가 도착했어요",
      "body": "여행 정산서를 확인해 주세요.",
      "createdAt": "2026-11-05T03:00:00Z",
      "isRead": false
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

#### 처리 규칙

서비스가 앱 안의 알림함을 제공할 때만 이 API를 채택한다.
푸시를 제공한다고 알림함을 제공하는 것은 아니다.

> 논의 필요
>
> 알림함 도입 여부, 알림 종류와 이동 대상 모델, 정렬 기준과 동률 순서, 발송 중복과 보관 기간

### SET-06 알림 읽음: 조건부

`PATCH /api/v1/me/notifications/{notificationId}`

> 논의 필요
>
> - 경로, 요청, 응답과 오류 코드의 확정 여부
> - 제공 범위나 구현 방식을 정한 뒤의 채택 여부

#### 인증

로그인한 회원은 본인이 받은 알림만 변경할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

본인 알림의 읽음 상태를 변경한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `notificationId` | String | 필수 | 없음 | 본인에게 발송된 알림 ID. |

##### Body

Content-Type: `application/json`

요청 필드: [NotificationRead](#notificationread).

```json
{
  "isRead": true
}
```

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [NotificationRead](#notificationread)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [NotificationRead](#notificationread) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "isRead": true
  }
}
```

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

> 논의 필요
>
> 읽음 해제 제공 여부, 중복 읽음의 처리와 보관 기간

## 공지 API

### SET-07 공지 목록

`GET /api/v1/notices`

> 논의 필요
>
> - 경로, 요청, 응답과 오류 코드의 확정 여부
> - 제공 범위나 구현 방식을 정한 뒤의 채택 여부

#### 인증

> 논의 필요
>
> 로그인 없이 공개함 vs 로그인한 회원에게만 허용함

#### 설명

앱에서 제공할 공지 목록을 조회한다.

#### 요청

##### Query Parameter

선택 필터를 생략하면 그 조건을 적용하지 않는다.

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `page` | Integer | 선택 | 0 | 0 이상. 다음 페이지는 현재 page+1. |
| `size` | Integer | 선택 | 20 | 1~100, 범위 밖이면 400. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 Array&lt;[NoticeSummary](#noticesummary)&gt;이다.
필드 정의는 그 모델을 따른다.
이 모델은 일부 필드만 정의한 구조이며, 아래 예시는 정의된 필드 범위만 보여준다.
전체 요청과 응답 규격은 이 API의 논의 사항을 반영해 완성해야 한다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | Array&lt;[NoticeSummary](#noticesummary)&gt; | 아니요 | 항상 | 이 API의 처리 결과. |

응답 최상위 `page`는 [Page](common.md#page)를 따른다.
페이지 적용 자체와 개별 기본 정렬은 검토 대상이다.

```json
{
  "data": [
    {
      "noticeId": "notice-example-a",
      "title": "서비스 안내",
      "publishedAt": "2026-11-05T03:00:00Z"
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

#### 처리 규칙

서비스가 공지를 앱 안의 서버 데이터로 제공할 때만 이 API를 채택한다.

> 논의 필요
>
> 공지를 서버로 제공함 vs 외부 페이지로 제공함, 외부 페이지로 제공할 때 이 API의 필요 여부, 인증 범위, 정렬. 신규 표시의 해제 기준은 [문의와 공지 정책](../policy/settings.md#문의와-공지)의 논의 필요를 따른다.

### SET-08 공지 상세

`GET /api/v1/notices/{noticeId}`

> 논의 필요
>
> - 경로, 요청, 응답과 오류 코드의 확정 여부
> - 제공 범위나 구현 방식을 정한 뒤의 채택 여부

#### 인증

> 논의 필요
>
> 로그인 없이 공개함 vs 로그인한 회원에게만 허용함

#### 설명

지정한 공지의 제목과 본문을 조회한다.

#### 요청

##### Path Parameter

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `noticeId` | String | 필수 | 없음 | 공지 ID. |

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

공통 응답의 `data`는 [NoticeDetail](#noticedetail)이다.
필드 정의는 그 모델을 따른다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | [NoticeDetail](#noticedetail) | 아니요 | 항상 | 이 API의 처리 결과. |

```json
{
  "data": {
    "noticeId": "notice-example-a",
    "title": "서비스 안내",
    "publishedAt": "2026-11-05T03:00:00Z",
    "body": "서비스 이용 안내입니다."
  }
}
```

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |

#### 처리 규칙

> 논의 필요
>
> 본문 표현 형식, 인증과 노출 기준, 조회와 읽음 기록의 분리 여부. 업데이트 표시를 해제하는 조건은 [문의와 공지 정책](../policy/settings.md#문의와-공지)의 논의 필요를 따른다.

## 클라이언트 기능과 다른 도메인

프로필과 탈퇴는 [유저 API](user.md), 로그아웃은 [인증 API](authentication.md)를 사용한다. 설치된 앱 버전은 앱 자체 정보로 표시한다. 버그 제보는 공식 이메일, 관리자 문의는 앱스토어와 플레이스토어의 개발자 연락처로 연결하므로 앱 내 문의 접수 API를 제공하지 않는다. 공지 확인 기록 API는 신규 표시 해제 기준이 정해지면 검토한다. 언어와 표시 통화 변경은 현재 MVP에서 제공하지 않는다.

## 공통 모델

> 논의 필요
>
> 모델 이름과 필드 구성의 확정 여부

요청 모델은 필수 여부와 기본값을, 응답 모델은 nullable과 포함 조건을 구분한다. 조건부 요청 필드는 예시에 사용한 조건을 만족할 때만 포함한다. 배열 원소는 링크한 모델을 따른다.

### PushDeviceInput

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `pushToken` | String | 필수 | 없음 | 푸시 제공자가 발급한 토큰. 이 값은 실제 토큰이 아닌 예시다. |
| `installationId` | String | 필수 | 없음 | 앱 설치 식별자. 계정 전환과 토큰 교체 기준은 논의 필요. |

### PushDevice

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `pushDeviceId` | String | 아니요 | 항상 | 본인 계정에 등록된 푸시 수신 기기 ID. |

### NotificationSummary

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `notificationId` | String | 아니요 | 항상 | 본인에게 발송된 알림 ID. |
| `title` | String | 아니요 | 항상 | 알림 제목. |
| `body` | String | 아니요 | 항상 | 알림 내용. |
| `createdAt` | String | 아니요 | 항상 | ISO 8601 UTC. 생성 시각. |
| `isRead` | Boolean | 아니요 | 항상 | 읽음 여부. |

### NotificationRead

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `isRead` | Boolean | 필수 | 없음 | true만 허용하는 안. 읽음 해제 제공 여부는 논의 필요. |

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `isRead` | Boolean | 아니요 | 항상 | true만 허용하는 안. 읽음 해제 제공 여부는 논의 필요. |

### NoticeSummary

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `noticeId` | String | 아니요 | 항상 | 공지 ID. |
| `title` | String | 아니요 | 항상 | 공지 제목. |
| `publishedAt` | String | 아니요 | 항상 | ISO 8601 UTC. 게시 시각. |

### NoticeDetail

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `noticeId` | String | 아니요 | 항상 | 공지 ID. |
| `title` | String | 아니요 | 항상 | 공지 제목. |
| `publishedAt` | String | 아니요 | 항상 | ISO 8601 UTC. 게시 시각. |
| `body` | String | 아니요 | 항상 | 공지 본문. 텍스트, HTML 또는 Markdown 형식은 결정해야 함. |

## 오류 코드

> 논의 필요
>
> 도메인 오류 코드의 이름, HTTP 대응과 안내 문구의 확정 여부

공통 코드 이름은 [백엔드 CommonErrorCode](https://github.com/JECT-Study/Trip-Sheet-5th-Server/blob/09dd7bfbaf37974fecfa3bb2fa2c22ea3378d20c/src/main/kotlin/com/ject/global/error/CommonErrorCode.kt)와 맞췄다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |

> 논의 필요
>
> 개별 API의 오류 조건과 도메인 코드 이름은 팀 검토 후 확정한다.
