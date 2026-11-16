# Ostone — 상담 원본 저장과 후속 분석의 실패 경계

**4인 팀 캡스톤 · Spring Boot, PostgreSQL, S3/MinIO, Airflow**

상담 로그를 분석하려면 원본 파일을 보관하면서 화자와 발화 순서를 구조화해야 했습니다. 원본 저장과 DB 기록이 서로 다른 시스템에서 일어나므로 한쪽만 끝난 상태에서 요청이 실패할 수 있었습니다. 저는 원본 업로드 API와 상담 파서를 구현하고, 업로드를 Airflow 분석 작업에 연결했습니다.

## 원본 보관과 분석 입력

긴 상담 로그의 분석을 업로드 요청에서 동기 처리하면 요청이 서버 자원을 오래 점유하고, API와 분석 서버의 역할이 겹칠 수 있었습니다. 원본 JSON은 감사나 재처리 때 참조할 수 있도록 객체 저장소에 두고, 분석이 조회하는 대화·발화와 원본을 찾는 메타데이터는 PostgreSQL에 기록했습니다. 당시 분석 작업을 AWS에 배포할 계획이어서 S3를 선택했습니다. 로컬에서는 같은 SDK로 연결할 수 있는 MinIO를 사용했습니다. [저장소 선정 기록](https://github.com/ajou-2026-1-capstone-5/ostone/blob/b1925c1dc88cedae17d57d2e8c76c0d249fcb506/.agent/specs/114/adr-storage-selection.md)

파일은 최대 **50MB**로 제한했습니다. 워크스페이스 접근 권한과 데이터셋 키 중복을 확인한 뒤 객체를 저장했습니다. 객체 키에는 워크스페이스·데이터셋과 파일을 구분하는 값을 넣고, 업로드 전에 수신한 원본 바이트의 SHA-256을 계산해 메타데이터에 함께 기록했습니다. [원본 업로드 PR #58](https://github.com/ajou-2026-1-capstone-5/ostone/pull/58)

분석 입력을 만들 때 `상담사:`는 상담사, `고객:`과 `손님:`은 고객으로 매핑했습니다. 빈 줄을 건너뛰며 발화 순번을 붙였습니다. `안내:`나 `요청:` 같은 지원 범위 밖의 말머리는 발화 주체를 임의로 정할 경우 분류를 왜곡할 수 있어 입력 오류로 처리했습니다. 화자 분류·빈 줄·잘못된 입력을 다루는 단위 테스트 7개를 작성했습니다. [파싱 PR #13](https://github.com/ajou-2026-1-capstone-5/ostone/pull/13) · [화자 파서](https://github.com/ajou-2026-1-capstone-5/ostone/blob/03cbe279b136204f2be8670119d6eef1ddf75895/backend/src/main/java/com/init/corpus/application/ConsultingContentParser.java#L32-L68) · [파서 테스트](https://github.com/ajou-2026-1-capstone-5/ostone/blob/03cbe279b136204f2be8670119d6eef1ddf75895/backend/src/test/java/com/init/corpus/application/ConsultingContentParserTest.java)

## 업로드 이후의 부분 실패

객체 저장 이후 JSON 파싱이나 DB 저장에서 예외가 발생하면, 객체 삭제를 시도하고 원래 예외를 다시 던지도록 구현했습니다. 삭제 호출도 실패하면 객체 키를 경고 로그에 남겼습니다. #58에서 업로드 서비스 단위 테스트 **8개**를 추가해 정상 수신·접근 권한·중복 키, 파싱·저장 실패 시 삭제 호출, 삭제 실패 시 원래 오류 보존을 검증했습니다. 당시 백엔드 CI가 통과했습니다. [보상 삭제 구현](https://github.com/ajou-2026-1-capstone-5/ostone/blob/b1925c1dc88cedae17d57d2e8c76c0d249fcb506/backend/src/main/java/com/init/corpus/application/RawFileUploadService.java#L79-L135) · [업로드 테스트](https://github.com/ajou-2026-1-capstone-5/ostone/blob/b1925c1dc88cedae17d57d2e8c76c0d249fcb506/backend/src/test/java/com/init/corpus/application/RawFileUploadServiceTest.java) · [업로드 CI](https://github.com/ajou-2026-1-capstone-5/ostone/actions/runs/24521458072/job/71680088176)

## 데이터셋 커밋과 분석 트리거

#58에서 분석 트리거 인터페이스를 정의하고, #227에서 `dataset_id`와 `object_key`를 넘기는 Airflow 호출을 연결했습니다. 연동 테스트에서 실행 요청에 원본 객체 키가 포함되는 것과 트리거 성공 후 작업이 `RUNNING`으로 전환되는 것을 확인했습니다. [Airflow 연동 PR #227](https://github.com/ajou-2026-1-capstone-5/ostone/pull/227) · [Airflow 연동 테스트](https://github.com/ajou-2026-1-capstone-5/ostone/blob/50be6fa16d58161e31542161f1ae8685b3464dbc/backend/src/test/java/com/init/pipelinejob/application/TriggerIngestionUseCaseTest.java)

별도 트랜잭션에서 생성하는 파이프라인 작업이 PostgreSQL 외래 키로 데이터셋을 참조하려면, 데이터셋 저장 트랜잭션이 먼저 커밋돼야 했습니다. 팀원은 후속 PR #245에서 업로드 전체를 감싸던 트랜잭션을 제거해 데이터셋 커밋 뒤 분석 트리거가 호출되도록 바꿨습니다. [팀 후속 PR #245](https://github.com/ajou-2026-1-capstone-5/ostone/pull/245)

[관련 사례 · 기존 계정을 보존한 인증 스키마 변경](ostone.md) · [포트폴리오 목록](README.md)
