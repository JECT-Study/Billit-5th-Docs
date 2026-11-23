# 인증 설계서

관련 정책: [로그인](../policy/authentication.md#로그인), [인증 유지](../policy/authentication.md#인증-유지), [로그아웃](../policy/authentication.md#로그아웃)

관련 문서: [인증 API 스펙](../api-spec/authentication.md), [가입 전용 토큰](#가입-전용-토큰), [인증 테이블과 컬럼](#인증-테이블과-컬럼)

## 설계 결정

서버는 Access Token, Refresh Token과 가입 전용 토큰을 모두 JWT로 발급하고, 로그인 세션은 DB에 저장한다. Access Token에는 세션 ID를 넣고 요청마다 세션이 종료됐는지 확인한다. 로그아웃하거나 Refresh Token 재사용을 감지하면 Access Token의 남은 유효 기간과 관계없이 해당 세션의 요청을 바로 거절한다.

이 설계는 2026-11-14 회의에서 검토했다. 회의에서 결론이 나지 않은 항목은 해당 절에 `논의 필요`로 남긴다.

구현은 아래 PR로 나눠 올려 `develop`에 병합했고, 개발 서버에 배포했다.

| 순서 | 범위 | PR | 상태 |
| --- | --- | --- | --- |
| 1 | 토큰 발급과 검증, 요청 인증, `auth_session` | [Billit-5th-Server PR #30](https://github.com/JECT-Study/Billit-5th-Server/pull/30) | `develop` 병합 |
| 2 | Google, Kakao 소셜 로그인 | [Billit-5th-Server PR #31](https://github.com/JECT-Study/Billit-5th-Server/pull/31) | `develop` 병합 |
| 3 | 토큰 갱신과 로그아웃 | [Billit-5th-Server PR #32](https://github.com/JECT-Study/Billit-5th-Server/pull/32) | `develop` 병합 |
| 4 | Apple 소셜 로그인과 연결 해제 토큰 | [Billit-5th-Server PR #33](https://github.com/JECT-Study/Billit-5th-Server/pull/33) | `develop` 병합 |
| 5 | 가입 정보 조회, 닉네임 확인과 가입 완료 | [Billit-5th-Server PR #34](https://github.com/JECT-Study/Billit-5th-Server/pull/34) | `develop` 병합 |
| 6 | 가입 시 임의 닉네임 부여, 닉네임 중복 확인 제거 | [Billit-5th-Server PR #43](https://github.com/JECT-Study/Billit-5th-Server/pull/43) | `develop` 병합(닉네임 단어는 임시, [Docs 이슈 #5](https://github.com/JECT-Study/Billit-5th-Docs/issues/5)에서 결정) |

## 토큰 종류와 검증

| 토큰 | `token_type` | 유효 기간 | 사용하는 요청 |
| --- | --- | --- | --- |
| Access Token | `ACCESS` | 15분 | 회원 API의 `Authorization` 헤더 |
| Refresh Token | `REFRESH` | 30일 | 토큰 갱신, Access Token이 만료된 뒤의 로그아웃 |
| 가입 전용 토큰 | `REGISTRATION` | 30분 | 가입 정보 조회, 가입 완료 |

유효 기간의 기준은 [인증 만료 정책](../policy/authentication.md#인증-만료)과 [회원 식별 정책](../policy/authentication.md#회원-식별)이 관리한다. 위 표는 구현에서 사용하는 값을 함께 보여주기 위한 것이다.

- 세 토큰은 서버의 서명 키로 HS512 서명한다. 서명 키는 환경 변수 `JWT_SECRET`으로 받으며 개발 서버와 운영 서버에 서로 다른 값을 사용한다. 값이 없으면 서버가 시작되지 않는다.
- 서버는 토큰을 검증할 때 `token_type`도 확인한다. 회원 API에 Refresh Token이나 가입 전용 토큰을 보내면 `401`을 반환한다. 가입 API에서 허용하는 토큰은 [가입 전용 토큰 설계](#인증-대상과-허용-범위)를 따른다.
- Access Token과 Refresh Token에는 로그인 세션 ID를 `sid` 클레임으로 넣는다. 가입 전용 토큰에는 세션 ID가 없다.

## 로그인 세션 저장

로그인 세션은 `auth_session` 테이블에 한 행으로 저장한다. 소셜 로그인이나 가입 완료로 회원 토큰을 발급할 때 세션을 만들고, 갱신과 로그아웃은 요청 토큰의 `sid`로 찾은 세션을 기준으로 처리한다. 기기 식별자는 저장하지 않는다.

| 컬럼 | 타입 | 값과 NULL 의미 |
| --- | --- | --- |
| `id` | `UUID` | 세션 ID. 회원 토큰의 `sid` 값 |
| `member_id` | `BIGINT` | 세션의 회원. `member.id` 참조 |
| `refresh_token_hash` | `VARCHAR(64)` | 현재 Refresh Token의 SHA-256 해시. 토큰 원문은 저장하지 않는다. |
| `refresh_token_expires_at` | `TIMESTAMPTZ` | 현재 Refresh Token의 만료 시각 |
| `previous_refresh_token_hash` | `VARCHAR(64)`, NULL 허용 | 직전 Refresh Token의 해시. NULL이면 아직 갱신한 적이 없다. |
| `refresh_token_rotated_at` | `TIMESTAMPTZ`, NULL 허용 | 마지막 갱신 시각. NULL이면 아직 갱신한 적이 없다. |
| `revoked_at` | `TIMESTAMPTZ`, NULL 허용 | 세션 종료 시각. NULL이면 유효한 세션이다. |
| `version` | `BIGINT` | 같은 세션의 동시 갱신과 종료를 감지하는 낙관적 잠금 버전 |

서버는 Access Token의 서명과 만료를 확인한 뒤 `sid`로 세션을 조회한다. 세션이 없거나 `revoked_at`이 기록돼 있으면 `401`을 반환한다. 요청마다 DB 조회가 한 번 늘어나며, 부하가 문제가 되면 캐시를 검토한다.

인증 데이터는 Redis에 두지 않는다. 현재 Redis는 메모리가 가득 차면 키를 무작위로 지우는 캐시 설정(`allkeys-lru`)이어서, 세션이나 차단 목록을 두면 사용자가 의도하지 않게 로그아웃되거나 차단이 풀릴 수 있다.

> 논의 필요
>
> 회의에서 Redis의 메모리 정리 정책을 다시 검토하기로 했다. 무작위 삭제 문제가 해결되면 세션을 Redis에 둘지, 지금처럼 DB에 둘지 다시 정한다. 정책을 바꾸면 Redis에 두는 다른 기능의 키에도 영향을 주므로 함께 확인한다.

## 인증 없이 허용하는 경로

| 경로 | 허용하는 이유 |
| --- | --- |
| `/healthz`, `/readyz`, 관리 포트의 actuator | 배포 중 상태 확인에 사용한다. 막히면 배포가 실패로 판정돼 이전 버전으로 되돌아간다. |
| `/swagger-ui.html`, `/api-docs/**`, `/scalar/**` | API 문서 경로다. 개발 서버는 게이트웨이에서 별도 인증으로 보호하고, 운영 서버는 게이트웨이에서 차단한다. |
| 소셜 로그인, 토큰 갱신, 로그아웃 | [인증 API 스펙](../api-spec/authentication.md#api-목록)에서 Access Token 없이 호출할 수 있는 API다. 로그아웃은 Access Token이 있으면 그 토큰으로 세션을 확인한다. |

그 밖의 경로는 회원 Access Token이 필요하다. 가입 API는 가입 전용 토큰으로 확인하며, 허용 범위는 [가입 전용 토큰](#가입-전용-토큰)를 따른다.

## 소셜 로그인

서버는 앱이 보낸 ID 토큰을 제공자의 공개키로 검증하고, 제공자와 제공자 내부 사용자 ID의 조합으로 회원을 찾는다. 이메일로 회원을 찾거나 합치지 않는다.

| 제공자 | ID 토큰의 발급 대상(`aud`) | 서버 설정 |
| --- | --- | --- |
| Google | 웹 클라이언트 ID. 개발 서버는 개발용, 운영 서버는 출시용 클라이언트를 사용한다. | `GOOGLE_CLIENT_ID` |
| Kakao | 네이티브 앱 키 | `KAKAO_NATIVE_APP_KEY` |
| Apple | 앱 번들 ID `app.billit.mobile` | `APPLE_CLIENT_ID` |

- 서버는 서명, 발급자, 발급 대상, 만료 시각과 사용자 ID를 검증한다. 제공자의 공개키 목록은 10분간 캐시한다.
- 같은 소셜 계정으로 회원이 두 번 생기지 않도록 `member`의 `(social_provider, social_provider_user_id)`에 UNIQUE 제약을 둔다.
- 처음 인증한 소셜 계정은 가입 중(`PENDING`) 회원 행을 만들고 가입 전용 토큰만 발급한다. 가입 완료(`ACTIVE`) 회원에게는 세션을 만들고 Access Token과 Refresh Token을 발급한다.
- 첫 소셜 로그인 응답을 받지 못한 사용자가 다시 소셜 로그인하면, 서버는 기존 가입 중 회원 행을 그대로 사용하고 새 가입 전용 토큰을 발급한다. 같은 소셜 계정의 회원 행을 새로 만들지 않는다.
- 제공자 장애나 타임아웃은 `503 AUTH_SOCIAL_PROVIDER_UNAVAILABLE`, 토큰 검증 실패는 `401 AUTH_SOCIAL_AUTHENTICATION_FAILED`로 구분한다.

탈퇴(`WITHDRAWN`) 회원의 소셜 계정으로 로그인하면 `401 AUTH_SOCIAL_AUTHENTICATION_FAILED`를 반환한다. 이 처리는 회의에서 합의한 현재 동작이며, 탈퇴 처리가 끝나 소셜 계정 ID가 지워진 사용자는 새 회원으로 가입한다.

> 논의 필요
>
> 탈퇴 회원의 재로그인과 재가입 시점은 회원 탈퇴 정책이 확정된 뒤 다시 논의한다. 정책의 미결 사항은 [재가입](../policy/user.md#재가입)을 따른다.

## Apple 연결 해제 토큰

Apple로 가입한 회원이 탈퇴하면 Apple에서 발급한 토큰을 폐기해 연결을 해제해야 한다. 기준은 [회원 탈퇴 정책](../policy/user.md#계정과-개인정보)을 따른다. 서버는 이 해제에 사용할 Apple Refresh Token을 로그인 시점에 받아 보관한다.

1. 앱은 Apple 로그인 요청에 `idToken`과 `authorizationCode`를 보낸다.
2. 서버는 ID 토큰을 검증한 뒤 인가 코드를 Apple 서버에서 교환해 Apple Refresh Token을 받는다.
3. 서버는 Apple Refresh Token을 AES-256-GCM으로 암호화해 `apple_refresh_token`에 회원당 한 행으로 저장한다. 다시 로그인해 새 토큰을 받으면 기존 값을 교체한다. 암호화 키는 환경 변수 `AUTH_TOKEN_ENCRYPTION_KEY`로 받는다.
4. 탈퇴 처리에서는 저장한 토큰으로 Apple에 연결 해제를 요청한다. Apple이 해제를 성공으로 응답한 경우에만 저장한 토큰을 삭제한다.

Apple 인가 코드는 한 번만 교환할 수 있다. 로그인 응답을 받지 못하면 앱은 Apple 인증부터 다시 진행해 새 인가 코드를 받는다. 서버도 같은 코드의 교환을 자동으로 재시도하지 않는다.

서버는 Apple `fullName`, Google `name`, Kakao `nickname` 같은 소셜 표시 이름을 저장하지 않는다.
가입할 때 닉네임을 받지 않고 임의 닉네임을 부여해 쓰는 곳이 없기 때문이다([USER-01](../api-spec/user.md#user-01-가입-정보-조회)).

> 논의 필요
>
> 암호화 키를 바꾸면 기존 행을 복호화할 수 없다. 탈퇴할 때 복호화하지 못한 행과 Apple이 해제를 거절한 경우의 처리를 회원 탈퇴 구현 전에 정한다.

## 토큰 갱신

갱신에 성공하면 서버는 새 Access Token과 Refresh Token을 발급하고 기존 Refresh Token을 폐기한다. 세션의 `refresh_token_hash`를 새 토큰의 해시로 바꾸고, 기존 해시는 `previous_refresh_token_hash`에 옮긴다.

| 제출한 Refresh Token | 처리 결과 |
| --- | --- |
| 현재 토큰 | 새 토큰을 발급한다. |
| 직전 토큰, 마지막 갱신 후 30초 이내의 첫 재요청 | 갱신 응답을 받지 못한 재시도로 보고 새 토큰을 한 번 더 발급한다. |
| 직전 토큰, 위 조건을 벗어난 요청 | 재사용으로 보고 해당 세션을 종료한 뒤 `401 AUTH_REFRESH_TOKEN_INVALID`를 반환한다. 다른 세션은 유지한다. |
| 만료됐거나 종료된 세션의 토큰 | `401 AUTH_REFRESH_TOKEN_INVALID`를 반환한다. |

같은 Refresh Token으로 동시에 들어온 갱신 요청은 `version` 검사로 한 건만 성공한다. 동시 요청 때문에 유효한 후속 토큰이 둘 이상 생기지 않는다. 앱은 갱신 요청을 한 번에 하나만 보내고, 다른 요청은 그 결과로 받은 새 Access Token으로 다시 보내야 한다.

> 논의 필요
>
> 갱신 응답을 받지 못했을 때의 재시도 횟수와 30초 유예 시간을 정한다. 회의에서는 한 번만 받을지, 여러 번 받을지 더 검토하기로 했다. [Docs PR #4](https://github.com/JECT-Study/Billit-5th-Docs/pull/4)가 [인증 만료 정책](../policy/authentication.md#인증-만료)에 지금 구현과 같은 30초 안의 첫 재요청 허용을 반영했고, [인증 API 스펙](../api-spec/authentication.md#auth-02-access-token-갱신)도 같은 기준으로 맞췄다.

> 논의 필요
>
> 회의에서 Access Token이 만료되기 전에는 일반 API 요청을 처리하면서 서버가 토큰을 갱신하고, 만료된 뒤에는 앱이 갱신 API를 호출하는 방식이 언급됐다. 현재 계약은 만료된 Access Token에 `401`을 반환하고 앱이 갱신 API를 호출하는 방식이다. 이 방식을 도입할지, 도입한다면 새 토큰을 응답의 어디에 담고 만료 시 어떤 상태 코드를 쓸지 정한다.

## 로그아웃

로그아웃은 요청 토큰의 `sid`로 찾은 세션 하나만 종료한다. 유효한 Access Token이 있으면 헤더로, Access Token이 만료됐으면 본문의 Refresh Token으로 세션을 확인한다. 세션의 `revoked_at`을 기록하면 해당 세션의 Access Token과 Refresh Token은 더 이상 사용할 수 없다. 다른 세션은 유지한다.

만료된 Access Token을 헤더에 붙여 보내면 본문을 확인하기 전에 `401`을 반환한다. 이미 종료된 세션으로 다시 요청해도 `401 COMMON_UNAUTHORIZED`를 반환하며, 앱은 로컬에 남은 토큰을 정리한다. 요청과 응답 규격은 [현재 기기 로그아웃 API](../api-spec/authentication.md#auth-03-현재-기기-로그아웃)를 따른다.

## 서버 설정

| 환경 변수 | 사용하는 곳 |
| --- | --- |
| `JWT_SECRET` | Billit 토큰 서명. 64바이트 이상의 값을 base64로 인코딩해 넣는다. |
| `GOOGLE_CLIENT_ID` | Google ID 토큰의 발급 대상 검증 |
| `KAKAO_NATIVE_APP_KEY` | Kakao ID 토큰의 발급 대상 검증 |
| `APPLE_TEAM_ID`, `APPLE_KEY_ID`, `APPLE_CLIENT_ID`, `APPLE_PRIVATE_KEY_BASE64` | Apple ID 토큰 검증과 인가 코드 교환 |
| `AUTH_TOKEN_ENCRYPTION_KEY` | Apple Refresh Token 암호화 |

개발 서버와 운영 서버에 값을 등록했다. 값은 저장소에 기록하지 않는다.

> 확인 필요
>
> Kakao ID 토큰은 Kakao 개발자 콘솔에서 OpenID Connect를 활성화해야 발급된다. 활성화 여부를 확인하지 못했다.

## 구현 시 검증

- 회원 API에 Refresh Token이나 가입 전용 토큰을 보내면 `401`을 반환하는지 확인한다.
- 로그아웃 직후 같은 세션의 Access Token으로 요청하면 만료 전이라도 `401`을 반환하는지 확인한다.
- 같은 Refresh Token으로 동시에 갱신하면 한 건만 성공하는지 확인한다.
- 직전 Refresh Token의 재요청이 유예 조건 안에서는 한 번만 성공하고, 조건을 벗어나면 세션을 종료하는지 확인한다.
- 같은 소셜 계정으로 동시에 처음 로그인해도 회원 행이 하나만 생기는지 확인한다.
- 탈퇴 회원의 소셜 계정으로 로그인하면 거절하는지 확인한다.
- 인증 없이 허용한 상태 확인 경로가 배포 중 상태 확인에서 막히지 않는지 확인한다.

문서 검토는 위 동작의 실행 검증을 대신하지 않는다. 구현 PR에서 실행한 테스트를 따로 기록한다.

## 가입 전용 토큰

관련 정책: [회원 식별](../policy/authentication.md#회원-식별), [가입 완료 조건](../policy/user.md#가입-완료-조건), [가입 중단과 재개](../policy/user.md#가입-중단과-재개)

### 설계 결정

소셜 인증에 성공했지만 가입을 완료하지 않은 사용자에게 가입 전용 토큰을 발급한다. 가입 전용 토큰은 가입 절차를 진행할 사용자를 확인하며, 일반 회원용 Access Token과 구분한다. 이 방향은 2026-11-06 API 스펙 검토에서 채택했다.

기존 회원은 소셜 로그인 성공 시 회원용 토큰을 발급받는다. 가입 미완료 사용자는 필수 가입 정보를 검증하고 가입을 완료한 뒤 회원용 Access Token과 Refresh Token을 발급받는다. 가입 전용 토큰으로 일반 회원 API를 호출하거나 회원용 토큰을 갱신할 수 없다.

### 인증 대상과 허용 범위

앱은 가입 전용 토큰을 `Authorization: Bearer {registrationToken}`으로 전달한다.

서버가 검증한 소셜 제공자와 제공자 내부 계정 ID를 가입 주체에 연결한다. 클라이언트가 가입 요청에 보낸 소셜 계정 ID를 신뢰해 가입 대상을 바꾸지 않는다. 가입 전용 토큰의 인증 주체를 가입 완료 회원의 memberId로 간주하지 않는다.

가입 정보 조회와 최초 가입 완료 요청은 유효한 가입 전용 토큰으로 확인한 가입 미완료 사용자에게 허용한다. 가입 완료 후에는 같은 가입 요청의 토큰 수령 재시도만 예외적으로 허용한다.

가입 전용 토큰은 회원용 토큰과 같은 방식의 JWT로 발급하고 `token_type`을 `REGISTRATION`으로 구분한다. 서버는 각 API에서 허용하는 토큰 종류를 검사한다. 토큰 형식과 검증은 [인증 세션과 토큰](#토큰-종류와-검증)을 따른다.

가입 주체는 처음 소셜 로그인할 때 만드는 가입 중(`PENDING`) 회원 행이다. 이 행에 소셜 제공자와 제공자 내부 계정 ID를 저장하고, 가입 전용 토큰에는 그 회원 ID를 넣는다. 가입을 완료하면 같은 행을 가입 완료(`ACTIVE`)로 바꾼다.

> 구현 제안
>
> 가입을 완료하지 않은 가입 중 회원 행은 PostgreSQL의 `pg_cron`으로 주기적으로 삭제한다. 삭제 기준은 생성 후 1시간으로 시작하고, 운영하면서 조정한다. 가입 전용 토큰의 유효 기간인 30분보다 길게 두어 진행 중인 가입을 지우지 않는다.

### 처리 흐름

1. 서버가 소셜 인증 결과를 검증하고 제공자와 계정 ID로 가입 완료 회원인지 확인한다.
2. 기존 회원이면 회원용 로그인 결과를 반환한다. 가입 미완료 사용자이면 가입 전용 토큰을 발급한다.
3. 가입 정보 조회와 가입 완료 요청에서 가입 전용 토큰의 유효성과 가입 주체를 확인한다.
4. 가입 완료 시 필수 동의, 만 18세 이상 확인과 소셜 계정의 기존 가입 여부를 최종 검증한다. 동시에 가입을 요청해도 같은 소셜 계정의 회원을 중복 생성하지 않는다.
5. 가입이 완료되면 회원용 토큰을 발급한다. 가입 완료 후 가입 전용 토큰으로 가입 절차를 새로 진행할 수 없도록 한다.

가입 전용 토큰은 발급 후 30분간 유효하다. 만료되면 다시 소셜 인증해 가입을 진행한다. 가입을 중단하고 다시 시작하는 경우 정책에 따라 소셜 인증을 다시 진행한다. 가입 전용 토큰의 만료는 가입 완료나 회원 탈퇴를 의미하지 않는다.

> 구현 제안
>
> 가입 처리 실패 시 회원용 토큰을 반환하지 않고, 가입 완료 여부는 DB 커밋 결과를 기준으로 판단한다.

> 논의 필요
>
> 다시 소셜 인증해 새 가입 전용 토큰을 받았을 때 이전 토큰의 처리와, 여러 토큰으로 동시에 가입한 경우의 응답을 결정한다.

### 가입 완료 응답 유실과 재시도

가입 완료 응답을 받지 못한 앱은 같은 가입 전용 토큰과 같은 가입 내용으로 가입 완료 요청을 재시도해 회원용 Access Token과 Refresh Token을 받는다. 서버는 기존 가입 결과를 식별하며 회원이나 약관 동의 기록을 중복 생성하지 않고 가입 정보를 다시 변경하지 않는다. 가입이 커밋된 뒤 토큰 발급에 실패한 경우에도 같은 복구 흐름을 사용한다.

재시도는 가입 전용 토큰의 유효 기간 안에서만 허용한다. 유효 기간은 발급 시점부터 30분이며 가입 완료나 재시도로 연장하지 않는다. 가입 전용 토큰의 회원이 이미 가입을 완료했다면 서버는 요청 내용을 다시 비교하지 않고 새 로그인 세션을 만들어 회원용 토큰을 새로 발급한다. 최초 응답의 토큰을 저장해 다시 반환하지 않는다.

이 예외는 해당 가입 완료 요청의 복구에만 적용한다. 가입 완료 후 가입 정보 조회를 계속 허용하거나, 가입 전용 토큰을 일반 로그인 및 Refresh Token 대용으로 사용하지 않는다. 소셜 로그인 응답 자체가 유실된 경우에는 별도 인증 API 규칙에 따라 소셜 인증부터 다시 진행한다.

> 논의 필요
>
> 재시도할 때마다 새 로그인 세션이 생기므로, 유효 기간 안에 여러 번 재시도하면 같은 회원의 세션이 여러 개 남는다. 이전 세션을 종료할지, 회원 로그아웃이나 탈퇴에서 함께 정리할지 정한다.

### 구현 상태

가입 전용 토큰의 발급과 검증은 [Billit-5th-Server PR #30](https://github.com/JECT-Study/Billit-5th-Server/pull/30), 가입 API는 [Billit-5th-Server PR #34](https://github.com/JECT-Study/Billit-5th-Server/pull/34)에서 리뷰 중이며 아직 `develop`에 반영되지 않았다. 전체 PR 순서는 [인증 세션과 토큰](#설계-결정)을 따른다.

### 구현 시 검증

- 가입 전용 토큰으로 가입 정보 조회와 가입 완료를 요청할 수 있는지 확인한다.
- 가입 전용 토큰으로 일반 회원 API와 회원용 토큰 갱신을 요청하면 거절되는지 확인한다.
- 토큰에 연결된 소셜 계정과 다른 계정으로 가입 대상을 바꿀 수 없는지 확인한다.
- 만료되거나 변조된 토큰을 거절하고, 가입 완료 후 토큰 사용을 같은 가입 요청의 복구 범위로 제한하는지 확인한다.
- 동일 소셜 계정의 동시 가입과 성공 응답 유실 후 재시도에서 회원과 약관 동의 기록이 중복 생성되지 않는지 확인한다.
- 가입 커밋 후 토큰 발급 실패 또는 응답 유실 시 재시도로 회원용 토큰을 받을 수 있는지 확인한다.
- 필수 동의 또는 만 18세 이상 확인 검증 실패 시 가입이 완료되지 않고 회원용 토큰을 반환하지 않는지 확인한다.

## 인증 테이블과 컬럼

인증 구현에서 기존 ERD에 없는 테이블, 컬럼과 제약을 추가했다. 아래 변경은 구현 PR의 마이그레이션이며 아직 `develop`에 반영되지 않았다. `auth_session`은 [PR #30](https://github.com/JECT-Study/Billit-5th-Server/pull/30), 소셜 계정 UNIQUE는 [PR #31](https://github.com/JECT-Study/Billit-5th-Server/pull/31), `apple_refresh_token`과 `social_display_name`은 [PR #33](https://github.com/JECT-Study/Billit-5th-Server/pull/33)에 있다. 설계 근거는 [인증 설계서의 앞 절](#설계-결정)을 따른다.

| 대상 | 변경 | 용도 |
| --- | --- | --- |
| `auth_session` | 테이블 추가 | 로그인 세션과 현재, 직전 Refresh Token의 해시를 저장한다. 컬럼은 [로그인 세션 저장](#로그인-세션-저장)을 따른다. |
| `apple_refresh_token` | 테이블 추가, `member_id` UNIQUE | Apple 연결 해제에 사용할 Apple Refresh Token을 회원당 한 행으로 암호화해 저장한다. |
| `member.social_display_name` | `VARCHAR(100) NULL` 추가 후 삭제 | 소셜 표시 이름을 쓰지 않게 되어 [서버 PR #49](https://github.com/JECT-Study/Billit-5th-Server/pull/49)에서 컬럼을 지운다. |
| `member` | `UNIQUE (social_provider, social_provider_user_id)` 추가 | 같은 소셜 계정으로 회원이 두 번 생기지 않게 한다. 탈퇴 처리로 `social_provider_user_id`가 NULL인 행끼리는 중복을 허용한다. |
