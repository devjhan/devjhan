# [Ostone] Preserving Operator Accounts and Connecting Logs to Analysis

**4인 캡스톤 · 운영자 인증, 상담 로그 수집, Airflow 연동 · Spring Boot, Liquibase, PostgreSQL, S3/MinIO** | [프로젝트 저장소](https://github.com/ajou-2026-1-capstone-5/ostone)

> ### 📌 Context & Constraints
>
> - **Objective** 상담 로그에서 의도, 정책, 워크플로 초안을 생성하고 운영자가 검토·승인할 수 있도록 했습니다.
> - **Problem** 기존 운영자 계정에는 비밀번호 해시가 없었습니다. 상담 로그를 받는 API와 로그를 분석하는 ML 서버를 연결하고, 입력마다 달라지는 화자 표기도 처리해야 했습니다.
> - **Constraints** 기존 계정을 보존하면서 인증을 도입해야 했습니다. 업로드 크기와 워크스페이스 접근 권한을 검사하고, 감사나 재처리에 필요한 상담 원본도 보관해야 했습니다.
> - **Scope** 팀은 상담을 처리하고 검토하는 제품 런타임과 로그 분석 파이프라인을 나눴습니다. 저는 운영자 인증과 상담 로그 수집 백엔드를 구현하고, 업로드한 로그로 후속 분석을 시작할 수 있도록 연결했습니다.

## Decisions & Troubleshooting

### 1. 기존 계정을 보존한 인증 스키마 변경

운영자 인증을 추가하기 전, 기존 계정에는 비밀번호 해시가 없어 곧바로 `NOT NULL` 제약을 적용하면 기존 행이 새 제약을 만족하지 못한다는 점을 발견했습니다. Liquibase로 nullable 컬럼을 먼저 추가하고, 해시가 없는 행에 재설정 플래그와 전이용 해시를 채운 뒤 `NOT NULL`을 적용했습니다. 재설정 토큰 해시는 값이 있는 행에만 유일성을 강제했습니다. 당시 기존 계정의 비밀번호 설정은 DB에서 직접 처리했습니다.

보호 API는 access JWT로 인증했습니다. refresh token은 해시·만료·폐기 상태를 DB에 기록해 갱신 요청에서 검사하도록 구현했습니다. [인증 PR #6](https://github.com/ajou-2026-1-capstone-5/ostone/pull/6) · [Liquibase 변경](https://github.com/ajou-2026-1-capstone-5/ostone/blob/6a07a6866d781fbc181f85967e120ba9e5cae858/backend/src/main/resources/db/changelog/db.changelog-master.sql#L584-L650)

### 2. 상담 원본 저장과 분석 분리

상담 로그를 백엔드 API에서 바로 처리하고 PostgreSQL만 쓰는 방안도 검토했습니다. 긴 로그를 요청 안에서 처리하면 응답이 늦어지고, 큰 입력이 서버 자원을 오래 점유할 수 있었습니다. 원본을 받는 API와 로그를 분석하는 ML 서버의 책임도 겹친다고 판단했습니다.

업로드 크기를 최대 `50MB`로 제한하고, 워크스페이스 접근 권한을 확인한 뒤 원본 JSON을 S3에 저장했습니다. 로컬에서는 같은 저장소 인터페이스로 MinIO를 사용했습니다. API는 대화·발화, 객체 키, 원본의 SHA-256을 PostgreSQL에 기록했습니다. 분석 파이프라인은 구조화된 발화를 주 입력으로 사용하고, 원본은 감사나 재처리에 필요할 때 참조할 수 있게 했습니다. [저장소 선정 기록](https://github.com/ajou-2026-1-capstone-5/ostone/blob/b1925c1dc88cedae17d57d2e8c76c0d249fcb506/.agent/specs/114/adr-storage-selection.md) · [원본 업로드 PR #58](https://github.com/ajou-2026-1-capstone-5/ostone/pull/58)

### 3. 상담 로그의 화자와 발화 순서 파싱

상담 로그에는 `안내:`나 `요청:`처럼 표준 접두어와 다른 말머리가 섞일 수 있었습니다. 알 수 없는 말머리에 화자를 임의로 부여하면 발화의 주체를 잘못 분류할 수 있어, 파서가 처리하는 접두어를 명시했습니다.

정규식으로 줄을 나누고 각 줄의 접두어를 검사했습니다. 초기 파서는 `상담사:`를 상담사로, `고객:`과 `손님:`을 고객으로 매핑했습니다. 빈 줄을 건너뛰고 발화 순번을 붙였으며, 지원 범위 밖의 접두어는 입력 오류로 처리했습니다. [상담 로그 파싱 PR #13](https://github.com/ajou-2026-1-capstone-5/ostone/pull/13) · [화자 파서](https://github.com/ajou-2026-1-capstone-5/ostone/blob/03cbe279b136204f2be8670119d6eef1ddf75895/backend/src/main/java/com/init/corpus/application/ConsultingContentParser.java#L32-L68)

### 4. 상담 로그 업로드와 Airflow 연동

분석 작업이 정제된 발화와 원본을 같은 데이터셋으로 추적할 수 있도록, 원본 저장 PR에서 업로드와 후속 분석을 연결할 인터페이스를 먼저 정의했습니다. 후속 PR에서 Airflow 트리거를 구현해 실행 요청에 `dataset_id`와 `object_key`를 넘겼습니다.

DB 저장 전에 분석을 시작하면 발화를 조회할 준비가 되지 않을 수 있었습니다. 팀의 후속 변경에서는 DB 저장이 끝난 뒤 트리거를 호출하도록 순서를 조정했습니다. [Airflow 연동 PR #227](https://github.com/ajou-2026-1-capstone-5/ostone/pull/227) · [팀 후속 PR #245](https://github.com/ajou-2026-1-capstone-5/ostone/pull/245)

## Validation & Results

**인증과 상담 파서.** 최초 인증 PR의 백엔드 CI가 통과했고, 후속 PR에서 인증 서비스와 컨트롤러 단위 테스트를 추가했습니다. 상담 파서에는 화자 분류, 빈 줄, 알 수 없는 접두어를 다루는 **7개 단위 테스트**를 작성했습니다. [인증 CI](https://github.com/ajou-2026-1-capstone-5/ostone/actions/runs/24063666195/job/70185016650) · [인증 테스트 PR #19](https://github.com/ajou-2026-1-capstone-5/ostone/pull/19) · [파서 테스트](https://github.com/ajou-2026-1-capstone-5/ostone/blob/03cbe279b136204f2be8670119d6eef1ddf75895/backend/src/test/java/com/init/corpus/application/ConsultingContentParserTest.java)

**원본 업로드와 Airflow 연동.** 정상 파일 수신, 워크스페이스 접근 권한, 중복 데이터셋 키 등을 다루는 **8개 단위 테스트**를 원본 업로드 서비스에 추가했고, 당시 테스트 통과를 PR에 기록했습니다. Airflow 연동 테스트에서는 실행 요청에 원본 객체 키가 포함되고 트리거가 성공하면 작업 상태가 `RUNNING`으로 바뀌는 경로를 검사했습니다. [업로드 테스트](https://github.com/ajou-2026-1-capstone-5/ostone/blob/b1925c1dc88cedae17d57d2e8c76c0d249fcb506/backend/src/test/java/com/init/corpus/application/RawFileUploadServiceTest.java) · [Airflow 연동 테스트](https://github.com/ajou-2026-1-capstone-5/ostone/blob/50be6fa16d58161e31542161f1ae8685b3464dbc/backend/src/test/java/com/init/pipelinejob/application/TriggerIngestionUseCaseTest.java)

**팀 성과.** 팀은 **2026년 아주대학교 SoftCon 개발부문(캡스톤디자인) 우수상**을 받았습니다.
