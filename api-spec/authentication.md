# 인증 API 스펙 초안 (1차 MVP)

## 목차

- [문서 범위](#문서-범위)
- [API 목록](#api-목록)
- [로그인과 인증 유지 API](#로그인과-인증-유지-api)
  - [AUTH-01 소셜 로그인](#auth-01-소셜-로그인)
  - [AUTH-02 Access Token 갱신](#auth-02-access-token-갱신)
- [로그아웃 API](#로그아웃-api)
  - [AUTH-03 현재 기기 로그아웃](#auth-03-현재-기기-로그아웃)
- [클라이언트 처리 범위](#클라이언트-처리-범위)
- [오류 코드](#오류-코드)

## 문서 범위

Google, Apple, Kakao 소셜 로그인, Access Token 갱신과 현재 기기 로그아웃을 다룬다. 소셜 로그인을 마쳤더라도 필수 약관 동의와 만 18세 이상 확인을 끝내기 전에는 가입 완료 회원으로 처리하지 않는다.

[목록](README.md) | [공통 API 스펙](common.md) | [인증 정책](../policy/authentication.md) | [가입 정책](../policy/user.md#회원가입)

아래 경로와 요청 필드는 검토 중인 초안이다. 합의 기록과 OpenAPI 반영 상태는 구분해 확인하며, [API 계약 관리 기준](<../guide/API 스펙 문서 가이드.md#계약-원본과-문서의-역할>)을 따른다. 소셜 제공자가 발급한 코드와 토큰, Billit이 발급하는 토큰의 차이는 [공통 인증](common.md#인증과-권한)을 따른다.

## API 목록

| 기능 | Method | Path | 인증 |
| --- | --- | --- | --- |
| [소셜 로그인](#auth-01-소셜-로그인) | `POST` | `/api/v1/auth/social-logins` | Billit Access Token 불필요, 소셜 제공자의 ID 토큰 필요 |
| [Access Token 갱신](#auth-02-access-token-갱신) | `POST` | `/api/v1/auth/token-refreshes` | 요청 Body에 Refresh Token 필요 (갱신 시 회전) |
| [현재 기기 로그아웃](#auth-03-현재-기기-로그아웃) | `POST` | `/api/v1/auth/logout` | 로그인 필요 (현재 기기) |

## 로그인과 인증 유지 API

### AUTH-01 소셜 로그인

`POST /api/v1/auth/social-logins`

이 경로는 백엔드 OpenAPI의 [`socialLogin`](https://github.com/JECT-Study/Billit-5th-Server/blob/develop/src/main/resources/api-docs/domains/auth.yml)에 반영돼 있다.

#### 인증

Billit Access Token은 보내지 않는다. 서버는 소셜 제공자의 공개키로 ID 토큰의 서명을 검증하고 발급자, 발급 대상, 만료 시간과 사용자 식별자를 확인한다.

#### 설명

소셜 인증 결과로 기존 회원에게 회원용 토큰을 발급하고, 가입 미완료 사용자에게 가입 전용 토큰을 발급한다.

#### 요청

##### Body

Content-Type: `application/json`

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `provider` | String | 필수 | 없음 | `GOOGLE`, `KAKAO`, `APPLE` 중 하나 |
| `idToken` | String | 필수 | 없음 | 모바일 SDK에서 받은 ID 토큰 |
| `authorizationCode` | String | 조건부 | 없음 | Apple 요청에서만 필수다. Apple 연결 해제에 필요한 토큰을 확보하기 위해 로그인 시 교환한다. |
| `fullName` | String | 선택 | 없음 | 사용 중단. 서버는 저장하지 않는다. 기존 앱과의 호환을 위해 Apple 요청에서만 받으며 최대 100자다. 앱이 보내지 않게 되면 필드를 지운다. |

문자열에 빈 값, 공백만 있는 값과 null은 허용하지 않는다.
Google과 Kakao 요청에는 `authorizationCode`와 `fullName`을 보내지 않는다.

#### 성공 응답

HTTP `200 OK`와 `data`를 반환한다.

| `data.status` | 대상 | 함께 반환하는 필드 |
| --- | --- | --- |
| `AUTHENTICATED` | 가입 완료 회원 | `accessToken`, `refreshToken` |
| `REGISTRATION_REQUIRED` | 가입 미완료 사용자 | `registrationToken` |

#### 오류

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수 필드가 없거나 입력 형식이 올바르지 않음 |
| 401 | `AUTH_SOCIAL_AUTHENTICATION_FAILED` | ID 토큰 또는 Apple 인가 코드 검증 실패 |
| 503 | `AUTH_SOCIAL_PROVIDER_UNAVAILABLE` | 제공자 장애나 타임아웃으로 로그인 처리 불가 |

`503`을 받으면 잠시 후 소셜 인증부터 다시 진행한다. Apple 로그인에서는 이전 요청에 사용한 `authorizationCode`를 자동으로 재전송하지 않는다.

#### 처리 규칙

소셜 제공자와 제공자 내부 계정 ID의 조합으로 회원을 찾는다. 다른 소셜 계정의 여행과 정산 기록을 자동으로 합치지 않는다. 소셜 로그인에 실패하거나 사용자가 취소하면 가입과 로그인을 완료하지 않는다.

가입 미완료 사용자에게는 [가입 전용 토큰](../design/registration-token.md)을 발급한다. 일반 회원용 Access Token과 Refresh Token은 가입 완료 후 발급한다.

Apple 로그인 응답을 받지 못하면 앱은 소셜 인증을 다시 진행해 새 `authorizationCode`를 받는다. 앱은 같은 코드로 로그인 요청을 자동 재시도하지 않으며, 서버도 같은 코드의 교환을 자동 재시도하지 않는다.

응답 유실 복구를 위해 이전 로그인 결과를 재전달하지 않는다. 현재 구현은 Google과 Kakao의 ID 토큰을 일회용으로 처리하지 않는다. 로그인 응답을 받지 못한 앱이 같은 `idToken`으로 다시 요청하면, 서버는 토큰이 만료되기 전까지 처음 요청과 같은 방식으로 검증하고 로그인 결과를 새로 발급한다. 코드 확인 기준은 [Billit-5th-Server `bd8d21f`](https://github.com/JECT-Study/Billit-5th-Server/tree/bd8d21f3b0ce289cedee201ec10a3cfbc21b863b)다.

> 논의 필요
>
> 같은 `idToken`의 재요청을 막으려면 `nonce` 검증이나 일회용 처리를 추가하고 앱과 서버를 함께 변경해야 한다. 1차 MVP에 도입할지 정한다.

ID 토큰의 검증 조건과 제공자별 발급 대상은 [소셜 로그인 설계](../design/authentication.md#소셜-로그인)를, Apple 인가 코드 교환은 [Apple 연결 해제 토큰 설계](../design/authentication.md#apple-연결-해제-토큰)를 따른다. 가입 전용 토큰의 형식과 유효 기간은 [토큰 종류와 검증](../design/authentication.md#토큰-종류와-검증)과 [가입 전용 토큰 설계](../design/registration-token.md)를 따른다.

### AUTH-02 Access Token 갱신

`POST /api/v1/auth/token-refreshes`

이 경로는 백엔드 OpenAPI의 [`refreshTokens`](https://github.com/JECT-Study/Billit-5th-Server/blob/develop/src/main/resources/api-docs/domains/auth.yml)에 반영돼 있다.

#### 인증

Refresh Token을 사용해 현재 기기의 로그인 상태를 확인한다. 갱신에 성공하려면 만료 전 Access Token도 함께 보내야 하는 것으로 가정하지 않는다.

#### 설명

로그인을 유지할 수 있는지 확인하고 새 Billit Access Token과 Refresh Token을 발급한다.

#### 요청

##### Body

Content-Type: `application/json`

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `refreshToken` | String | 필수 | 없음 | 현재 기기에 발급된 Refresh Token. 빈 문자열과 null은 허용하지 않음 |

```json
{
  "refreshToken": "example-refresh-token"
}
```

#### 성공 응답

HTTP `200 OK`를 반환한다. 응답 Body의 `data`에는 새 `accessToken`과 `refreshToken`만 포함한다.

Access Token은 15분, 새 Refresh Token은 30일간 유효하다.

#### 오류

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | `refreshToken` 필드가 없거나 요청 형식이 올바르지 않음 |
| 401 | `AUTH_REFRESH_TOKEN_INVALID` | 제출한 Refresh Token이 만료됐거나 유효하지 않음. 이미 갱신에 사용한 토큰도 포함 |

#### 처리 규칙

갱신에 성공하면 새 Access Token과 Refresh Token을 발급하고, 사용한 기존 Refresh Token은 폐기한다. 기존 토큰의 사용 처리와 새 Refresh Token의 유효화는 하나의 갱신으로 함께 처리하며, 동일 토큰의 동시 사용으로 여러 유효한 후속 토큰을 만들지 않는다.

갱신에 실패해도 만료된 Access Token으로 일반 회원 API를 사용할 수는 없다. Refresh Token이 만료되거나 유효하지 않으면 [인증 만료 정책](../policy/authentication.md#인증-만료)에 따라 다시 로그인하도록 안내한다.

네트워크 오류로 갱신 응답을 받지 못하면 앱은 같은 Refresh Token으로 한 번만 다시 요청한다. 서버가 첫 요청에서 이미 토큰을 회전했더라도, 마지막 갱신 후 30초 안에 직전 Refresh Token으로 들어온 첫 재요청은 응답을 받지 못한 재시도로 보고 새 토큰을 한 번 더 발급한다. 이 조건을 벗어나면 재사용으로 처리해 해당 세션을 차단하고 `401`을 반환한다. 앱은 Refresh Token이 유효하지 않다는 401 응답을 받으면 다시 로그인을 요청한다. 재요청에서도 응답을 받지 못하면 네트워크 오류로 처리한다.

MVP에서는 기기 식별자를 기준으로 Refresh Token을 저장하지 않는다. 갱신과 로그아웃은 요청에 사용한 토큰에 연결된 로그인 세션을 기준으로 처리한다.

갱신에 사용되어 폐기된 Refresh Token이 위 재시도 조건을 벗어나 다시 제출되면 해당 로그인 세션을 즉시 차단하고 `401`을 반환한다. 해당 세션의 Access Token과 현재 Refresh Token은 더 이상 사용할 수 없다. 다른 로그인 세션은 유지한다.
> 출처: [인증 유지 정책](../policy/authentication.md#인증-유지)

## 로그아웃 API

### AUTH-03 현재 기기 로그아웃

`POST /api/v1/auth/logout`

이 경로는 백엔드 OpenAPI의 [`logout`](https://github.com/JECT-Study/Billit-5th-Server/blob/develop/src/main/resources/api-docs/domains/auth.yml)에 반영돼 있다.

#### 인증

현재 기기의 유효한 Access Token으로 요청한다. Access Token이 만료됐더라도 현재 기기의 유효한 Refresh Token으로 로그아웃할 수 있다. 이때 새 토큰은 발급하지 않는다.

#### 설명

현재 기기에서 로그아웃하고 해당 기기의 기존 Access Token을 즉시 차단한다. 다른 기기의 로그인은 유지한다.

#### 요청

유효한 Access Token이 있으면 `Authorization: Bearer {accessToken}` 헤더로 전달한다. 요청 본문은 보내지 않는다.

Access Token이 만료됐다면 Authorization 헤더를 생략하고, 유효한 Refresh Token을 JSON 본문의 `refreshToken` 필드로 전달한다.

##### Header

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `Authorization` | String | 조건부 | 없음 | 유효한 Access Token을 사용하는 경우 `Bearer {accessToken}`으로 전달한다. Refresh Token으로 요청하면 생략한다. |

##### Body

Content-Type: `application/json`

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `refreshToken` | String | 조건부 | 없음 | Access Token이 만료된 경우 필수다. 유효한 Access Token으로 요청하면 Body를 보내지 않는다. |

```json
{"refreshToken": "example-refresh-token"}
```

#### 성공 응답

HTTP `204 No Content`를 반환한다. 응답 본문은 없다.

#### 오류

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 요청 형식이 올바르지 않음 |
| 401 | `COMMON_UNAUTHORIZED` | 세션을 확인할 유효한 토큰이 없거나 해당 세션이 이미 종료됨 |

이미 종료된 세션으로 다시 요청해 `401`을 받으면 앱은 로컬에 남은 해당 세션의 토큰을 정리한다.

#### 처리 규칙

로그아웃 처리가 완료되면 현재 기기에 발급한 기존 Access Token은 만료 전이라도 사용할 수 없다. 해당 기기의 Refresh Token을 폐기해 인증 갱신도 차단한다. 클라이언트에서 토큰을 삭제하는 것만으로 로그아웃 처리가 완료된 것으로 보지 않는다.

로그아웃해도 회원 계정과 여행의 지출 및 정산 기록은 유지한다. 다른 기기의 로그인 상태를 함께 해제하지 않는다.

서버는 Access Token과 Refresh Token에 담긴 로그인 세션 ID로 세션을 찾아 종료 시각을 기록한다. 같은 세션의 갱신과 로그아웃이 동시에 들어오면 서버는 세션 버전으로 충돌을 감지한다. 세션의 저장 구조는 [로그인 세션 저장](../design/authentication.md#로그인-세션-저장)을, 로그아웃 처리는 [로그아웃 설계](../design/authentication.md#로그아웃)를 따른다.

## 클라이언트 처리 범위

소셜 로그인 취소 후 화면 이동과 로그인 후 진입 화면은 앱에서 처리한다.

## 오류 코드

아래는 개별 API에 사용한 코드의 대응표다. 합의 여부와 미결인 매핑은 해당 API의 상태 표시를 따르며, OpenAPI 반영 여부는 [API 계약 관리 기준](<../guide/API 스펙 문서 가이드.md#계약-원본과-문서의-역할>)에 따라 별도로 확인한다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수 필드가 없거나 요청 형식이 올바르지 않다. |
| 401 | `AUTH_SOCIAL_AUTHENTICATION_FAILED` | ID 토큰 또는 Apple 인가 코드 검증에 실패한다. |
| 401 | `AUTH_REFRESH_TOKEN_INVALID` | Refresh Token이 만료되거나 유효하지 않다. 이미 갱신에 사용한 토큰도 포함한다. |
| 401 | `COMMON_UNAUTHORIZED` | 세션을 확인할 유효한 토큰이 없거나 해당 세션이 이미 종료됐다. |
| 403 | `COMMON_FORBIDDEN` | 인증된 사용자에게 요청을 수행할 권한이 없다. |
| 503 | `AUTH_SOCIAL_PROVIDER_UNAVAILABLE` | 소셜 제공자 장애나 타임아웃으로 로그인할 수 없다. |

오류 본문은 [공통 오류 응답](common.md#오류-응답)에 따라 `code`, `message`, `errors`를 반환한다. 앱은 `code`를 기준으로 분기한다.

인증 정보를 확인할 수 없으면 API별로 정의한 `401` 오류를 반환한다. 인증된 사용자에게 요청을 수행할 권한이 없으면 `403 COMMON_FORBIDDEN`을 반환한다.
