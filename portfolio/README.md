# 기술 포트폴리오

## 프로젝트

- **[Ostone — 기존 계정을 보존한 인증 스키마 변경](ostone.md)** — 비밀번호 해시가 없던 기존 운영자 계정에 인증을 도입하기 위해 단계적 DB 변경과 인증 API를 구현했습니다. *Spring Boot · PostgreSQL · Liquibase*
- **[Ostone — 상담 원본 저장과 후속 분석의 실패 경계](ostone-corpus.md)** — 원본 저장 뒤 파싱·DB 기록에 실패하면 객체 삭제를 시도하도록 구현하고, 업로드를 후속 분석에 연결했습니다. *Spring Boot · PostgreSQL · S3/MinIO · Airflow*
- **[Polyglot Chat — TCP 프레이밍과 파일 전송](polyglot-chat.md)** — TCP 스트림에서 메시지 경계를 복원하는 C 라이브러리를 만들고, Java 서버와 Python 클라이언트 사이의 파일 전송을 검증했습니다. *C · JNA · ctypes · Spring Boot*

## 비즈앤뱅크 인턴십

- **[SQL Chatbot — 금융 업무 의미를 반영한 SQL 조회](bizbank-sql-chatbot.md)** — 지급 질의가 올바른 업무 정의를 찾도록 검색을 개선하고, SAP 금액 보정과 실행 전 검증을 적용했습니다. *Dify · SQLGlot · SAP HANA*
- **[SQL Chatbot — 업무 정확성을 고려한 모델·검색 비용 최적화](bizbank-sql-evaluation.md)** — 같은 질의로 비용·재시도·실행 오류를 비교해 경량 모델을 적용할 단계와 기존 모델을 유지할 단계를 정했습니다. *Dify · Gemini*
- **[Dify Workflow Automation — 그래프 변경과 배포 설정 검증](dify-dev-automation.md)** — 끊긴 변수 참조를 빌드에서 찾고 포크 후 환경 설정을 복원하는 컴파일러와 CLI를 구현했습니다. *Node.js · Dify DSL*
- **[GCMS Registry — SAP 스키마를 업무 조회 규칙으로 연결](gcms-semantic-registry.md)** — SAP 원천과 현업 규칙을 대조해 잘못된 업무 분류·조회 조건을 고치고, 검토 문서와 실행용 레지스트리를 같은 정의에서 생성했습니다. *SAP GUI·ADT · ABAP DDIC · Python · Dify*
- **[Windows Deployment Kit — 서비스 정지 전 배포 산출물 검증](windows-deployment-kit.md)** — JAR 내부까지 검사한 뒤 서비스를 교체하도록 하고, 새 버전 기동 실패에 대비해 이전 버전 복구 절차를 구현했습니다. *PowerShell · NSSM · Spring Boot · React*

[프로필로 돌아가기](../README.md)
