# 인증 세션과 토큰

관련 정책: [로그인](../policy/authentication.md#로그인), [인증 유지](../policy/authentication.md#인증-유지), [로그아웃](../policy/authentication.md#로그아웃)

관련 문서: [인증 API 스펙](../api-spec/authentication.md), [가입 전용 토큰](registration-token.md), [ERD 변경 목록](erd-changes.md#인증-테이블과-컬럼)

## 설계 결정

서버는 Access Token, Refresh Token과 가입 전용 토큰을 모두 JWT로 발급하고, 로그인 세션은 DB에 저장한다. Access Token에는 세션 ID를 넣고 요청마다 세션이 종료됐는지 확인한다. 로그아웃하거나 Refresh Token 재사용을 감지하면 Access Token의 남은 유효 기간과 관계없이 해당 세션의 요청을 바로 거절한다.

이 설계는 2026-11-14 회의에서 검토했다. 회의에서 결론이 나지 않은 항목은 해당 절에 `논의 필요`로 남긴다.

구현은 아래 PR로 나눠 올렸다. 앞 PR이 병합된 뒤 다음 PR을 병합한다.

| 순서 | 범위 | PR | 상태 |
| --- | --- | --- | --- |
| 1 | 토큰 발급과 검증, 요청 인증, `auth_session` | [Billit-5th-Server PR #30](https://github.com/JECT-Study/Billit-5th-Server/pull/30) | `develop` 병합 |
| 2 | Google, Kakao 소셜 로그인 | [Billit-5th-Server PR #31](https://github.com/JECT-Study/Billit-5th-Server/pull/31) | 리뷰 중 |
| 3 | 토큰 갱신과 로그아웃 | [Billit-5th-Server PR #32](https://github.com/JECT-Study/Billit-5th-Server/pull/32) | 리뷰 중 |
| 4 | Apple 소셜 로그인과 연결 해제 토큰 | [Billit-5th-Server PR #33](https://github.com/JECT-Study/Billit-5th-Server/pull/33) | 리뷰 중 |
| 5 | 가입 정보 조회, 닉네임 확인과 가입 완료 | [Billit-5th-Server PR #34](https://github.com/JECT-Study/Billit-5th-Server/pull/34) | 리뷰 중 |
| 6 | 닉네임 선택 입력과 임의 닉네임 부여 | [Billit-5th-Server 이슈 #42](https://github.com/JECT-Study/Billit-5th-Server/issues/42) | 착수 전 |

## 토큰 종류와 검증

| 토큰 | `token_type` | 유효 기간 | 사용하는 요청 |
| --- | --- | --- | --- |
| Access Token | `ACCESS` | 15분 | 회원 API의 `Authorization` 헤더 |
| Refresh Token | `REFRESH` | 30일 | 토큰 갱신, Access Token이 만료된 뒤의 로그아웃 |
| 가입 전용 토큰 | `REGISTRATION` | 30분 | 가입 정보 조회, 가입 완료 |

유효 기간의 기준은 [인증 만료 정책](../policy/authentication.md#인증-만료)과 [회원 식별 정책](../policy/authentication.md#회원-식별)이 관리한다. 위 표는 구현에서 사용하는 값을 함께 보여주기 위한 것이다.

- 세 토큰은 서버의 서명 키로 HS512 서명한다. 서명 키는 환경 변수 `JWT_SECRET`으로 받으며 개발 서버와 운영 서버에 서로 다른 값을 사용한다. 값이 없으면 서버가 시작되지 않는다.
- 서버는 토큰을 검증할 때 `token_type`도 확인한다. 회원 API에 Refresh Token이나 가입 전용 토큰을 보내면 `401`을 반환한다. 가입 API에서 허용하는 토큰은 [가입 전용 토큰 설계](registration-token.md#인증-대상과-허용-범위)를 따른다.
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

그 밖의 경로는 회원 Access Token이 필요하다. 가입 API는 가입 전용 토큰으로 확인하며, 허용 범위는 [가입 전용 토큰 설계](registration-token.md)를 따른다.

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

1. 앱은 Apple 로그인 요청에 `idToken`, `authorizationCode`와 최초 권한 부여 시 받은 `fullName`을 보낸다.
2. 서버는 ID 토큰을 검증한 뒤 인가 코드를 Apple 서버에서 교환해 Apple Refresh Token을 받는다.
3. 서버는 Apple Refresh Token을 AES-256-GCM으로 암호화해 `apple_refresh_token`에 회원당 한 행으로 저장한다. 다시 로그인해 새 토큰을 받으면 기존 값을 교체한다. 암호화 키는 환경 변수 `AUTH_TOKEN_ENCRYPTION_KEY`로 받는다.
4. 탈퇴 처리에서는 저장한 토큰으로 Apple에 연결 해제를 요청한다. Apple이 해제를 성공으로 응답한 경우에만 저장한 토큰을 삭제한다.

Apple 인가 코드는 한 번만 교환할 수 있다. 로그인 응답을 받지 못하면 앱은 Apple 인증부터 다시 진행해 새 인가 코드를 받는다. 서버도 같은 코드의 교환을 자동으로 재시도하지 않는다.

Apple은 사용자 이름을 최초 권한 부여 때만 주므로, 서버는 받은 `fullName`을 `member.social_display_name`에 저장한다. Google은 ID 토큰의 `name`, Kakao는 `nickname`을 같은 컬럼에 저장한다. 닉네임을 입력하지 않으면 임의 닉네임을 부여하기로 해서, 이 값은 Apple을 포함해 닉네임 초기값으로 쓰지 않는다(BE 논의). 저장을 계속할지는 [USER-01 논의 필요](../api-spec/user.md#user-01-가입-정보-조회)에서 정한다.

> 논의 필요
>
> 회의에서 Apple 토큰을 Refresh Token과 같은 이름으로 다루지 말고, Apple 전용 테이블에 토큰과 `fullName`을 함께 보관하는 방안이 나왔다. 지금처럼 `apple_refresh_token`과 `member.social_display_name`으로 나눌지, Apple 전용 테이블로 합칠지 정한다.
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
