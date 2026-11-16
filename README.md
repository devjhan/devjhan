# Hello!

Java/Spring 기반 백엔드 개발을 중심으로, **인증·데이터 정합성·실행 안정성**을 다루고 있습니다.

AI 서비스와 내부 개발 도구를 구현하면서, 시스템의 실패 조건을 명확하게 정의하고 검증하는 데 관심이 있습니다.

[기술 포트폴리오](portfolio/README.md)

- **Email:** [egnever4434@gmail.com](mailto:egnever4434@gmail.com)
- **GitHub:** [github.com/devjhan](https://github.com/devjhan)

## Experience

### 비즈앤뱅크 (Biz&Bank)

**Software Engineering Intern | July 2026 – November 2026**

금융 업무용 AI 조회 시스템과 내부 개발·운영 자동화 도구 개발에 참여했습니다.

- 금융 데이터 조회를 위한 Text-to-SQL 시스템의 업무 의미 규칙, SQL 검증 및 오류 처리 구조 개선 · [SQL 조회 사례](portfolio/bizbank-sql-chatbot.md)
- Dify 워크플로 구성 요소와 모델 설정·변수 바인딩을 검증하는 빌드 및 회귀 테스트 자동화 도구 개발 · [Dify 자동화 사례](portfolio/dify-dev-automation.md)
- Spring Boot 기반 API와 Windows 환경의 서비스 배포·관리 도구 개발 · [Windows 배포 사례](portfolio/windows-deployment-kit.md)

## Selected Projects

### 01. [Ostone — 상담 로그 기반 업무 지식화 시스템](https://github.com/ajou-2026-1-capstone-5/ostone)

**4인 팀 캡스톤 | Spring Boot · PostgreSQL · AWS S3**

**2026 아주대학교 SoftCon 개발부문 우수상 (팀 수상)**

상담 기록을 분석해 업무 지식과 워크플로 초안을 생성하고, 사람의 검토를 거쳐 활성화하는 시스템입니다.

**My Contributions**

- JWT 로그인·재발급·비밀번호 재설정 API 구현
- Refresh Token 해시·폐기 상태·버전 정보와 데이터베이스 제약 관리
- 상담 데이터 모델과 S3/MinIO 원본 저장 API 구현
- 인증 스키마의 단계적 변경 및 관련 테스트 작성

[Repository](https://github.com/ajou-2026-1-capstone-5/ostone) · [Implementation Evidence](https://github.com/ajou-2026-1-capstone-5/ostone/commit/6a07a6866d781fbc181f85967e120ba9e5cae858) · [인증 스키마](portfolio/ostone.md) · [상담 원본 저장](portfolio/ostone-corpus.md)

### 02. [Polyglot Chat — C/Java/Python 통신 시스템](https://github.com/devjhan/25-1-mogakso)

**개인 학습 프로젝트 | C · Spring Boot · Python**

네이티브 소켓 라이브러리를 구현하고, 이를 Java 서버와 Python 클라이언트에서 공통으로 사용하는 채팅 시스템입니다.

**My Contributions**

- C 기반 소켓 통신·메시지 프로토콜·명령 큐 구현
- JNA와 ctypes를 이용한 네이티브 라이브러리 연동
- 파일 전송의 청크 처리, 수신 크기 및 SHA-256 무결성 검증
- 불완전한 데이터 수신과 전송 취소에 대한 상태 정리 구현

[Repository](https://github.com/devjhan/25-1-mogakso) · [Implementation Evidence](https://github.com/devjhan/25-1-mogakso/commit/6ae35601e9726ec40554dc2c246eef2f37e83625) · [TCP 프레이밍과 파일 전송](portfolio/polyglot-chat.md)

### Additional Projects

- [Massive Board Study](https://github.com/devjhan/massive-board-study) — Spring Boot 기반 CQRS, Kafka, Redis 및 Transactional Outbox 학습 및 구현
- [Kimchi Discount Pipeline](https://github.com/devjhan/kimchi-discount-pipeline) — Python 기반 금융 데이터 수집·분석 및 규칙 검증 자동화

## Tech Stack

- **Languages:** Java, Python, JavaScript, TypeScript, C, SQL
- **Backend:** Spring Boot, Spring Security, JPA, JDBC, Node.js
- **Database:** PostgreSQL, MySQL, SQLite, DuckDB
- **Infrastructure & Tools:** Git, Docker, Gradle, PowerShell, Playwright
- **AI & Workflow:** Dify, MCP, RAG, SQLGlot

## Education

**Ajou University — Department of Software**

Mar 2021 – Feb 2027 (Expected Graduation), GPA 3.63/4.5, 129/140 Credits

## Certifications

- Engineer Information Processing (정보처리기사)
- AWS Certified Security – Specialty
- AWS Certified Solutions Architect – Associate
- TOPCIT Level 3 — 624 / 1000
- SQL Developer (SQLD)

