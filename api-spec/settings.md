# 설정 API 스펙 (1차 MVP)

## 목차

- [문서 범위](#문서-범위)
- [API 목록](#api-목록)
- [공지 API](#공지-api)
  - [SET-07 공지 목록](#set-07-공지-목록)
  - [SET-08 공지 상세](#set-08-공지-상세)
- [클라이언트 기능과 다른 도메인](#클라이언트-기능과-다른-도메인)
- [공통 모델](#공통-모델)
  - [NoticeSummary](#noticesummary)
  - [NoticeDetail](#noticedetail)
- [오류 코드](#오류-코드)

## 문서 범위

공지사항은 앱 내 화면으로 제공한다. 설정 화면에 있다는 이유만으로 별도 API를 두지 않는다.

[목록](README.md) | [공통 API 스펙](common.md) | [정책](../policy/settings.md)

API 식별자는 화면 정의서 식별자와 별개다.

## API 목록

| 기능 | Method | Path | 인증 |
| --- | --- | --- | --- |
| [공지 목록](#set-07-공지-목록) | `GET` | `/api/v1/notices` |  |
| [공지 상세](#set-08-공지-상세) | `GET` | `/api/v1/notices/{noticeId}` |  |

## 공지 API

### SET-07 공지 목록

`GET /api/v1/notices`

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

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `data` | Array&lt;[NoticeSummary](#noticesummary)&gt; | 아니요 | 항상 | 이 API의 처리 결과. |

응답 최상위 `page`는 [Page](common.md#page)를 따른다.

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

### SET-08 공지 상세

`GET /api/v1/notices/{noticeId}`

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

## 클라이언트 기능과 다른 도메인

프로필과 탈퇴는 [유저 API](user.md), 로그아웃은 [인증 API](authentication.md)를 사용한다. 설치된 앱 버전은 앱 자체 정보로 표시한다. 버그 제보는 공식 이메일, 관리자 문의는 앱스토어와 플레이스토어의 개발자 연락처로 연결하므로 앱 내 문의 접수 API를 제공하지 않는다. 언어와 표시 통화 변경은 현재 MVP에서 제공하지 않는다.

## 공통 모델

요청 모델은 필수 여부와 기본값을, 응답 모델은 nullable과 포함 조건을 구분한다. 조건부 요청 필드는 예시에 사용한 조건을 만족할 때만 포함한다. 배열 원소는 링크한 모델을 따른다.

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
| `body` | String | 아니요 | 항상 | 공지 본문. |

## 오류 코드

공통 코드 이름은 [백엔드 CommonErrorCode](https://github.com/JECT-Study/Trip-Sheet-5th-Server/blob/09dd7bfbaf37974fecfa3bb2fa2c22ea3378d20c/src/main/kotlin/com/ject/global/error/CommonErrorCode.kt)와 맞췄다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 404 | `COMMON_NOT_FOUND` | 경로에 지정한 ID에 해당하는 대상을 찾을 수 없음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
