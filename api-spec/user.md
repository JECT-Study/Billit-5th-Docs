# 유저 API 스펙 초안 (1차 MVP)

## 목차

- [문서 범위](#문서-범위)
- [API 목록](#api-목록)
- [가입 API](#가입-api)
  - [USER-01 가입 정보 조회](#user-01-가입-정보-조회)
  - [USER-03 가입 완료](#user-03-가입-완료)
- [프로필 API](#프로필-api)
  - [USER-04 내 프로필 조회](#user-04-내-프로필-조회)
  - [USER-05 프로필 이미지 변경](#user-05-프로필-이미지-변경)
- [회원 탈퇴 API](#회원-탈퇴-api)
  - [USER-06 탈퇴 영향 조회](#user-06-탈퇴-영향-조회)
  - [USER-07 회원 탈퇴](#user-07-회원-탈퇴)
- [오류 코드](#오류-코드)

## 문서 범위

계정 닉네임은 다른 회원과 겹쳐도 되며 가입 후 변경할 수 있다([유저 정책](../policy/user.md#닉네임)). 그래서 닉네임 중복 확인 API(기존 USER-02)는 두지 않는다. 계정 닉네임 변경 API는 아직 정의하지 않았다. 프로필 이미지 처리 방식은 기존 ERD만으로 확정하지 않는다.

[목록](README.md) | [공통 API 스펙](common.md) | [정책](../policy/user.md) | 기존 검토 DDL `member.sql`

가입 API는 합의한 계약을 본문에 작성하고, 미결정 사항은 `논의 필요`로 표시한다. 프로필과 회원 탈퇴 API는 검토 중이다. 합의된 가입 계약도 [API 계약 관리 기준](<../guide/API 스펙 문서 가이드.md#계약-원본과-문서의-역할>)에 따른 OpenAPI 반영 확인이 필요하며 구현 완료를 뜻하지 않는다.

## API 목록

| 기능 | Method | Path | 인증 |
| --- | --- | --- | --- |
| [가입 정보 조회](#user-01-가입-정보-조회) | `GET` | `/api/v1/me/registration` | 소셜 로그인 완료, 가입 미완료 |
| [가입 완료](#user-03-가입-완료) | `POST` | `/api/v1/me/registration` | 소셜 로그인 완료, 가입 미완료 |
| [내 프로필 조회](#user-04-내-프로필-조회) | `GET` | `/api/v1/me` | 로그인 필요 |
| [프로필 이미지 변경](#user-05-프로필-이미지-변경) | `PUT` | `/api/v1/me/profile-image` | 로그인 필요 |
| [탈퇴 영향 조회](#user-06-탈퇴-영향-조회) | `GET` | `/api/v1/me/withdrawal-preview` | 로그인 필요 |
| [회원 탈퇴](#user-07-회원-탈퇴) | `POST` | `/api/v1/me/withdrawal` | 로그인 필요 |

## 가입 API

### USER-01 가입 정보 조회

`GET /api/v1/me/registration`

#### 인증

소셜 로그인은 완료했지만 Billit 가입을 완료하지 않은 사용자만 요청할 수 있다. [가입 전용 토큰](../design/registration-token.md)으로 본인을 확인한다. 토큰은 `Authorization: Bearer {registrationToken}`으로 전달한다.

#### 설명

가입 시 동의할 약관 목록을 조회한다.

#### 요청

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

응답 본문의 `data` 필드는 다음과 같다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `status` | String | 아니요 | 항상 | 가입 미완료 상태인 `SIGNING_UP`을 반환한다. |
| `socialDisplayName` | String | 예 | 항상 | 소셜 제공자가 준 표시 이름. 제공값이 없으면 `null`. 닉네임 초기값으로 쓰지 않는다(아래 논의 필요). |
| `terms` | Array&lt;RegistrationTerms&gt; | 아니요 | 항상 | 가입 화면에서 표시할 필수 및 선택 약관 목록. |

**RegistrationTerms**

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `termsId` | String | 아니요 | 항상 | 약관 버전 식별자. 가입 완료 요청의 동의 항목에 같은 값을 보낸다. |
| `title` | String | 아니요 | 항상 | 약관 제목 |
| `isRequired` | Boolean | 아니요 | 항상 | 가입에 필요한 필수 약관인지 여부 |
| `contentUrl` | String | 아니요 | 항상 | 해당 버전의 약관 전문 URL |

```json
{
  "data": {
    "status": "SIGNING_UP",
    "socialDisplayName": "김빌릿",
    "terms": [
      {
        "termsId": "1",
        "title": "개인정보 수집 및 이용 동의",
        "isRequired": true,
        "contentUrl": "https://example.com/terms/privacy/v1"
      }
    ]
  }
}
```

#### 오류

[공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 401 | `COMMON_UNAUTHORIZED` | 소셜 로그인을 마친 가입 미완료 사용자인지 확인할 수 없음 |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |

#### 처리 규칙

가입 중 사용자 본인 정보만 제공한다. `socialDisplayName`은 Google ID 토큰의 `name`, Kakao ID 토큰의 `nickname`, Apple 로그인 요청의 `fullName`에서 가져온다. 제공값이 없으면 `null`을 반환한다.

닉네임을 입력하지 않으면 서비스가 임의 닉네임을 부여하므로, 소셜 표시 이름을 닉네임 입력란의 초기값으로 쓰지 않는다. Apple `fullName`도 같다(BE 논의).

> 논의 필요
>
> - 닉네임 초기값으로 쓰지 않으면 `socialDisplayName`을 응답할 이유가 없다. 응답에서 빼고 `member.social_display_name` 저장도 중단할지 정한다(구현 제안: 쓰지 않는 개인정보이므로 둘 다 중단).
>
> 출처: [정책](../policy/user.md) | 기존 검토 DDL `member.sql`

### USER-03 가입 완료

`POST /api/v1/me/registration`

#### 인증

최초 요청은 소셜 로그인을 마친 가입 미완료 사용자에게 허용한다. 가입 완료 응답을 받지 못한 경우에는 같은 가입 요청의 토큰 수령 재시도를 허용한다. [가입 전용 토큰](../design/registration-token.md)으로 본인을 확인한다. 토큰은 `Authorization: Bearer {registrationToken}`으로 전달한다.

#### 설명

필수 약관 동의와 만 18세 이상 확인을 제출하고 일반 회원으로 가입을 완료한다. 정산 시 번거로운 점은 선택 항목이며 최대 2개를 전달할 수 있다. 기타를 선택하면 직접 입력한 내용을 함께 전달할 수 있다.

#### 요청

##### Body

Content-Type: `application/json`

| 필드 | 타입 | 필수 | 기본값 | 규칙 |
| --- | --- | --- | --- | --- |
| `nickname` | String | 선택 | 없음 | 계정 닉네임. 1~10자, 한글 완성형(가~힣), 영문(A~Z, a~z)과 숫자(0~9)만 허용한다. 생략하면 서버가 임의 닉네임을 부여한다. 다른 회원과 겹쳐도 된다. |
| `isAgeEligible` | Boolean | 필수 | 없음 | 만 18세 이상인지 확인한 값. `true`만 허용하며 값 자체는 저장하지 않는다. |
| `platform` | String | 필수 | 없음 | 약관에 동의한 기기의 운영체제. `IOS`, `ANDROID` |
| `appVersion` | String | 필수 | 없음 | 약관에 동의한 시점의 앱 버전. 1~20자. 약관 동의 기록에 저장한다. |
| `agreements` | Array | 필수 | 없음 | 앱에 표시한 약관 버전별 동의 여부 |
| `agreements[].termsId` | String | 필수 | 없음 | 앱에 표시한 약관 버전의 식별자 |
| `agreements[].isAgreed` | Boolean | 필수 | 없음 | 해당 버전에 대한 동의 여부 |
| `survey` | Object | 선택 | 없음 | 선택 설문. 응답하지 않으면 생략한다. |
| `survey.gender` | String | 선택 | 없음 | 선택한 성별 코드 |
| `survey.isTripOrganizer` | Boolean | 선택 | 없음 | 여행 총무 여부 |
| `survey.settlementPainPoints` | Array&lt;String&gt; | 선택 | 없음 | 정산 시 번거로운 점을 최대 2개까지 전달한다. 중복은 허용하지 않는다. |
| `survey.otherSettlementPainPoint` | String | 선택 | 없음 | `settlementPainPoints`에 `OTHER`가 있을 때만 전달할 수 있다. |

설문에 일부만 응답하면 응답한 필드만 보낸다. 별도로 허용하지 않은 필드의 `null` 처리 규칙은 [공통 규격](common.md#데이터와-예시의-해석)을 따른다.

필수 약관의 동의가 누락되거나 `false`이면 `400 USER_REQUIRED_AGREEMENT_MISSING`을 반환한다. `isAgeEligible`이 `false`이면 `400 COMMON_INVALID_REQUEST`를 반환한다. 선택 약관은 생략하거나 `false`로 보낼 수 있다.

```json
{
  "nickname": "빌릿여행",
  "isAgeEligible": true,
  "platform": "IOS",
  "appVersion": "1.0.0",
  "agreements": [
    { "termsId": "1", "isAgreed": true },
    { "termsId": "2", "isAgreed": false }
  ],
  "survey": {
    "isTripOrganizer": true,
    "settlementPainPoints": ["OTHER"],
    "otherSettlementPainPoint": "여러 통화로 결제하면 정산하기 어려워요."
  }
}
```

> 논의 필요
>
> - 이름과 나이를 선택 항목으로 받을지 정한다. [Docs PR #4](https://github.com/JECT-Study/Billit-5th-Docs/pull/4)는 선택 항목으로 다시 넣었고, 기존 결정은 받지 않는 것이라 PM에게 확인 중이다. 받기로 하면 이 요청에 `name`, `age`를 선택 필드로 추가한다.
> - 임의 닉네임의 단어와 조합 방식은 [Docs 이슈 #5](https://github.com/JECT-Study/Billit-5th-Docs/issues/5)에서 정한다.
> - 성별과 정산 설문의 선택지 코드, `OTHER` 선택 시 직접 입력의 필수 여부를 정한다.

#### 성공 응답

HTTP `200 OK`를 반환한다.

회원 ID와 가입 완료 여부의 응답 필드는 별도로 확정한다.

회원용 토큰은 응답 Body의 `data`에 담아 반환한다. 아래 예시는 합의된 토큰 필드만 보여준다. 회원 ID와 가입 완료 여부를 포함한 전체 응답 예시는 해당 필드의 계약을 확정한 뒤 작성한다.

```json
{
  "data": {
    "accessToken": "example-access-token",
    "refreshToken": "example-refresh-token"
  }
}
```

#### 오류

[공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 요청 형식이나 필드가 올바르지 않음 |
| 401 | `COMMON_UNAUTHORIZED` | 가입 전용 토큰으로 최초 가입 또는 동일 가입 요청의 복구 권한을 확인할 수 없음 |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없음 |
| 400 | `USER_REQUIRED_AGREEMENT_MISSING` | 필수 약관의 동의가 누락되거나 동의하지 않음 |

#### 처리 규칙

서버는 필수 동의, 만 18세 이상 확인과 닉네임 형식을 검증한다. 닉네임을 생략하면 임의 닉네임을 부여한다. 설문을 제출하면 선택 개수가 2개 이하인지, 기타 입력이 기타 선택과 함께 제출됐는지 검증한다. 가입 완료 응답을 받지 못한 앱은 같은 가입 전용 토큰과 같은 가입 내용으로 재시도해 회원용 Access Token과 Refresh Token을 받는다. 서버는 회원과 동의 기록을 중복 생성하거나 가입 정보를 다시 변경하지 않는다. 복구 범위와 남은 구현 선택은 [가입 전용 토큰 설계](../design/registration-token.md)를 따른다.

> 논의 필요
>
> - 가입 완료 재시도의 식별 및 허용 기간과 토큰 재전달 또는 재발급 방식을 결정한다.
>
> 출처: [정책](../policy/user.md) | 기존 검토 DDL `member.sql`

## 프로필 API

> 검토 중
>
> 이 절의 경로, 요청, 응답과 오류 코드는 초안이다.

### USER-04 내 프로필 조회

`GET /api/v1/me`

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

설정과 프로필 화면에 표시할 본인 계정 정보를 조회한다.

#### 요청

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환하는 안이다.

응답 본문의 `data` 필드는 다음과 같다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `memberId` | String | 아니요 | 항상 | Billit 회원 계정 ID. 문자열로 전달한다. |
| `nickname` | String | 아니요 | 항상 | 가입 시 설정한 계정 닉네임. |
| `profileImageUrl` | String | 예 | 항상 | 미등록이면 null. |
| `socialProvider` | String | 아니요 | 항상 | `GOOGLE`, `APPLE`, `KAKAO` 중 하나. |

```json
{
  "data": {
    "memberId": "member-example-a",
    "nickname": "민지",
    "profileImageUrl": "https://cdn.example.com/profile/example.png",
    "socialProvider": "KAKAO"
  }
}
```

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |

#### 처리 규칙

탈퇴 계정 접근을 차단한다. 가입 중 정보는 가입 조회 API로 구분한다.

### USER-05 프로필 이미지 변경

`PUT /api/v1/me/profile-image`

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

새 이미지로 본인의 프로필 이미지를 변경한다. 앱은 이미지 파일을 `multipart/form-data`로 Billit 서버에 직접 업로드한다.

#### 요청

Content-Type: `multipart/form-data`

##### Part

| Part | Content-Type | 필수 | 규칙 |
| --- | --- | --- | --- |
| `image` | `image/*` | 필수 | 이미지 1개. 실제 허용 형식과 최대 용량은 논의 필요. |

#### 성공 응답

HTTP `200 OK`를 반환하는 안이다.

응답 본문의 `data` 필드는 다음과 같다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `profileImageUrl` | String | 아니요 | 항상 | 최종 반영된 이미지 접근 URL. |

```json
{
  "data": {
    "profileImageUrl": "https://cdn.example.com/profile/example.png"
  }
}
```

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 형식 또는 값의 범위가 올바르지 않음 |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |
| 413 | `USER_PROFILE_IMAGE_TOO_LARGE` | 허용 용량을 초과한 이미지 |
| 415 | `USER_PROFILE_IMAGE_UNSUPPORTED` | 지원하지 않는 실제 이미지 형식 |

#### 처리 규칙

실제 파일 형식과 용량을 검사한다. 실패하면 이전 프로필을 유지한다. 크기 초과는 413, 미지원 형식은 415를 검토한다.

> 논의 필요
>
> - 파일 형식과 용량을 결정한다. 5MB는 확정값이 아니다.
>
> 출처: [정책](../policy/user.md) | 기존 검토 DDL `member.sql`

## 회원 탈퇴 API

> 검토 중
>
> 이 절의 경로, 요청, 응답과 오류 코드는 초안이다.

### USER-06 탈퇴 영향 조회

`GET /api/v1/me/withdrawal-preview`

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

탈퇴 전 미완료 송금 관계의 총건수와 생성한 여행에 미치는 영향을 조회한다. 앱은 삭제 정보, 정산 완료 시 즉시 삭제와 탈퇴 후 최대 90일 보존 기준을 안내한다.

#### 요청

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `200 OK`를 반환하는 안이다.

응답 본문의 `data` 필드는 다음과 같다.

| 필드 | 타입 | nullable | 포함 조건 | 설명 |
| --- | --- | --- | --- | --- |
| `incompleteSettlementCount` | Integer | 아니요 | 항상 | 미완료 송금 관계의 총건수, 0 이상. 정산서 수가 아닌 관계 수를 센다. 구체적인 상태별 집계 기준은 논의 필요. |
| `affectedCreatedTripIds` | Array&lt;String&gt; | 아니요 | 항상 | 생성자 탈퇴로 다른 참여자의 접근도 차단되는 여행 식별자 목록. |

```json
{
  "data": {
    "incompleteSettlementCount": 1,
    "affectedCreatedTripIds": [
      "trip-example-a"
    ]
  }
}
```

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |

#### 처리 규칙

정산 건수가 있어도 탈퇴를 막지 않는다. 최종 탈퇴 전에 다시 상태를 확인한다.

> 논의 필요
>
> - 영향을 표시할 여행 목록의 정보와 미완료 송금 관계의 상태별 집계 기준을 결정한다.
>
> 출처: [정책](../policy/user.md) | 기존 검토 DDL `member.sql`

### USER-07 회원 탈퇴

`POST /api/v1/me/withdrawal`

#### 인증

가입을 완료한 회원만 요청할 수 있다. Access Token 전달 방식은 [공통 인증](common.md#인증과-권한)을 따른다.

#### 설명

최종 확인한 회원의 탈퇴를 처리하고 해당 계정의 접근을 차단한다.

#### 요청

요청 Body는 보내지 않는다.

#### 성공 응답

HTTP `204 No Content`를 반환하는 안이다. 응답 Body는 없다.

#### 오류

공통 오류 형식은 [공통 오류 응답](common.md#오류-응답)을 따른다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 401 | `COMMON_UNAUTHORIZED` | Access Token이 없거나 만료되었거나 올바르지 않음 |

#### 처리 규칙

안내 사항 확인 체크박스 동의와 최종 확인 팝업을 거쳐 요청한다. 미완료 정산이 있어도 탈퇴를 허용한다. 계정 접근을 차단하고 계정 정보와 개인 리시트를 삭제한다. 생성한 여행은 모든 참여자의 접근을 차단하고 기존 여행방 기록을 삭제하되 진행 중인 정산에 필요한 최소 기록은 보존한다. 다른 참여자는 내 정산 내역에서 기존 여행명, 필요한 송금 관계와 금액을 조회하고 수령을 확인할 수 있다.

탈퇴자 표시, 상세 품목 비노출, 양쪽 탈퇴 관계 비노출, 정산 완료 시 즉시 삭제 및 각 탈퇴일 기준 최대 90일 보존은 [회원 탈퇴 정책](../policy/user.md#공동-기록과-미완료-정산)을 따른다. 삭제 이후에도 다른 참여자의 수령 확인은 유지한다.

> 논의 필요
>
> - 탈퇴 사유 수집 여부와 확인 동의 전달 형식, 항목별 삭제 구현, 재가입 제한을 결정한다. 분리 보존의 적법한 근거와 완료 즉시 삭제 및 최대 90일 보존 기준은 법률 검토가 필요하다.
>
> 출처: [정책](../policy/user.md) | 기존 검토 DDL `member.sql`

## 오류 코드

아래는 개별 API에 사용한 코드의 대응표다. 가입 API와 검토 중인 프로필 및 탈퇴 API의 상태는 각 절을 따른다. 프로필 이미지의 허용 형식과 용량은 확정된 값이 아니다.

| HTTP | 코드 | 조건 |
| --- | --- | --- |
| 400 | `COMMON_INVALID_REQUEST` | 필수값이 없거나 요청 형식 또는 값의 범위가 올바르지 않다. |
| 400 | `USER_REQUIRED_AGREEMENT_MISSING` | 필수 약관 동의가 누락되거나 동의하지 않았다. |
| 401 | `COMMON_UNAUTHORIZED` | 요청에 필요한 회원 또는 가입 전용 인증을 확인할 수 없다. |
| 403 | `COMMON_FORBIDDEN` | 로그인했지만 요청 대상에 대한 조회 또는 변경 권한이 없다. |
| 413 | `USER_PROFILE_IMAGE_TOO_LARGE` | 이미지가 확정할 허용 용량을 초과한다. |
| 415 | `USER_PROFILE_IMAGE_UNSUPPORTED` | 확정할 허용 이미지 형식에 해당하지 않는다. |

오류 본문은 [공통 오류 응답](common.md#오류-응답)에 따라 `code`, `message`, `errors`를 반환한다. 앱은 `code`를 기준으로 분기한다.

인증 정보를 확인할 수 없으면 `401 COMMON_UNAUTHORIZED`를 반환한다. 인증된 사용자에게 요청을 수행할 권한이 없으면 `403 COMMON_FORBIDDEN`을 반환한다. API별 오류 조건과 코드는 각 API의 오류 절에 정의한다.
