# 유저 설계서

## 개요

이 설계서는 회원 행의 저장 구조와 회원이 탈퇴할 때 서버가 데이터를 삭제하고 보존하는 처리를 다룬다.
가입 중 회원 행과 가입 전용 토큰은 [인증 설계서](authentication.md#가입-전용-토큰)를 따른다.
업무 규칙은 [유저 정책서](../policy/user.md)를 따른다.

## 저장 데이터

| 이름 | 타입 | 제약 | 뜻 |
| --- | --- | --- | --- |
| `member.social_provider` | `auth_provider` | NOT NULL | 소셜 제공자. `GOOGLE`, `APPLE`, `KAKAO` 중 하나 |
| `member.social_provider_user_id` | `VARCHAR(255)` | NULL | 제공자 내부 사용자 ID. NULL이면 탈퇴 처리로 지운 회원임 |
| `member.nickname` | `VARCHAR(10)` | NULL | 계정 닉네임. NULL이면 가입 중이거나 탈퇴한 회원임 |
| `member.status` | `member_status` | NOT NULL | 회원 상태. `PENDING`, `ACTIVE`, `WITHDRAWN` 중 하나 |

- `member`에는 `UNIQUE (social_provider, social_provider_user_id)` 제약을 둔다. `social_provider_user_id`가 NULL인 행끼리는 중복을 허용한다.
- 서버는 계정 닉네임의 중복을 허용한다. 여행방에서는 [여행방 닉네임](../policy/trip.md#여행방-닉네임)을 쓴다.
- `member`에는 이름과 나이 컬럼을 두지 않는다. 가입할 때 이름과 나이를 입력받지 않는다([선택 정보](../policy/user.md#선택-정보)).
- 서버는 탈퇴 사유를 회원 ID 없이 `withdrawal_reason` 테이블에 저장한다.

## 처리 흐름

다음 표는 회원이 탈퇴할 때 서버가 데이터를 처리하는 방식이다.

| 대상 | 처리 |
| --- | --- |
| 소셜 식별값, 계정 닉네임, 선택 정보 | 탈퇴 즉시 지움 |
| 여행방 닉네임(`trip_member.trip_nickname`) | 탈퇴 즉시 비움 |
| 회원 행 | 지우지 않고 `WITHDRAWN`으로 바꿈 |
| Apple Refresh Token | Apple 로그인 회원이면 탈퇴 즉시 Apple에 해지를 요청함 |
| 송금 관계와 금액 | 계정과 연결을 끊은 별도 기록으로 남김 |
| 탈퇴 사유 | 회원 ID 없이 `withdrawal_reason`에 저장함 |

- 서버는 탈퇴한 회원의 `trip_nickname`을 다른 문구로 바꾸지 않고 NULL로 비운다.
- 공동 기록에서 탈퇴한 회원은 참여자 상태 `WITHDRAWN`으로 구분한다. 표시 문구는 [공동 기록과 미완료 정산](../policy/user.md#공동-기록과-미완료-정산)을 따른다.
- `UNIQUE (trip_id, trip_nickname)`은 NULL끼리 중복으로 보지 않으므로, 같은 여행방에서 여러 회원이 탈퇴해도 서버는 제약 위반 없이 닉네임을 비운다. 제약은 [여행 설계서](trip.md#저장-데이터)를 따른다.
- Apple 로그인 회원이 탈퇴하면 서버는 저장한 Apple Refresh Token으로 Apple 토큰 해지 API(`/auth/revoke`)를 호출한다. 토큰의 저장 구조는 [소셜 로그인](authentication.md#소셜-로그인)을 따른다.
- Apple이 해지를 성공으로 응답하면 서버는 저장한 Apple Refresh Token을 삭제한다.
- 정산이 완료되었거나 탈퇴일로부터 90일이 지나면 서버 스케줄러는 탈퇴한 회원의 별도 기록을 매일 삭제한다.
- 서버는 개인 리시트를 저장하지 않고, 정산과 송금 데이터로 조회할 때마다 만든다.
- 서버는 탈퇴한 회원의 개인 리시트를 [공동 기록 표시 기준](../policy/user.md#공동-기록과-미완료-정산)에 따라 다른 참여자에게 반환하지 않는다.

> 논의 필요
>
> 탈퇴한 회원의 별도 기록을 90일 보존하는 법적 근거. [공동 기록과 미완료 정산](../policy/user.md#공동-기록과-미완료-정산)의 논의 필요를 따른다.

> 추후 반영
>
> 프로필 이미지의 저장 컬럼과 스토리지 경로를 정할 때 탈퇴 즉시 Object Storage에서 이미지를 지우는 처리를 함께 정하고 위 표에 추가한다.

## 실패 처리

- Apple이 해지를 거절하면 서버는 저장한 Apple Refresh Token을 삭제하지 않는다.

> 논의 필요
>
> - Apple이 해지를 거절했을 때 탈퇴 처리
> - 암호화 키를 바꿔 복호화할 수 없는 Apple Refresh Token의 탈퇴 처리
