# Ostone — 기존 계정을 보존한 인증 스키마 변경

**4인 팀 캡스톤 · Spring Boot, PostgreSQL, Liquibase**

Ostone은 상담 기록에서 업무 지식과 워크플로 초안을 만들고 운영자가 검토하는 시스템입니다. 인증을 추가할 때 기존 운영자 계정에는 비밀번호 해시가 없었습니다. 저는 기존 행을 유지하면서 새 제약을 적용하는 DB 변경과 로그인·재발급·비밀번호 재설정 API를 구현했습니다. [프로젝트 저장소](https://github.com/ajou-2026-1-capstone-5/ostone) · [인증 PR #6](https://github.com/ajou-2026-1-capstone-5/ostone/pull/6)

## 기존 행을 보정한 뒤 제약 적용

비밀번호 해시를 필수 컬럼으로 바로 추가하면 기존 행이 새 제약을 만족할 수 없었습니다. Liquibase에서 `password_hash`를 nullable로 추가하고, 해시가 없는 행의 `password_reset_required`를 켰습니다. 해당 행에 전이용 해시를 채운 뒤 `NOT NULL`을 적용했습니다. 전이용 해시는 기존 행을 새 스키마에 맞추기 위한 값이며, 재설정 플래그는 실제 비밀번호 설정이 필요하다는 상태를 나타냅니다. 당시 기존 계정의 비밀번호는 DB에서 직접 설정했습니다. 재설정 토큰 해시는 값이 있는 행에만 유일성을 적용하는 부분 인덱스로 관리했습니다. [초기 구현 커밋](https://github.com/ajou-2026-1-capstone-5/ostone/commit/6a07a6866d781fbc181f85967e120ba9e5cae858) · [Liquibase 변경](https://github.com/ajou-2026-1-capstone-5/ostone/blob/6a07a6866d781fbc181f85967e120ba9e5cae858/backend/src/main/resources/db/changelog/db.changelog-master.sql#L584-L650)

## API 인증과 재발급 상태

보호 API 요청은 Access JWT로 인증했습니다. Refresh Token은 해시·만료·폐기 상태를 DB에 기록하고, 재발급 요청에서 토큰과 저장된 상태를 검사했습니다. 검사를 통과하면 사용한 토큰을 폐기하고 새 토큰을 발급했습니다. [초기 재발급 구현](https://github.com/ajou-2026-1-capstone-5/ostone/blob/6a07a6866d781fbc181f85967e120ba9e5cae858/backend/src/main/java/com/init/auth/application/AuthService.java#L93-L139)

팀원은 후속 PR #627에서 같은 토큰의 동시 재발급을 직렬화하도록 조회에 PostgreSQL 비관적 행 잠금(`PESSIMISTIC_WRITE`)을 적용했습니다. 먼저 처리된 요청이 기존 토큰을 폐기하면 뒤따르는 요청은 폐기 상태를 읽고 인증 실패로 종료됩니다. PostgreSQL 컨테이너에서 같은 토큰으로 두 요청을 보내 한 건만 성공하는지 검사하는 테스트도 추가했습니다. [팀 후속 PR #627](https://github.com/ajou-2026-1-capstone-5/ostone/pull/627) · [동시 재발급 테스트](https://github.com/ajou-2026-1-capstone-5/ostone/blob/15577e45696ac89dbe1d8fa29eab106c43f73638/backend/src/test/java/com/init/auth/application/AuthServiceConcurrencyTest.java#L65-L102)

## 검증 범위

초기 인증 구현은 백엔드 CI를 통과했습니다. #19에서는 잘못된 비밀번호, 만료된 Refresh Token, 비밀번호 재설정을 다루는 단위 테스트를 추가해 인증 실패와 재설정 경로를 검증했습니다. 해당 PR의 백엔드 CI도 통과했습니다. [초기 인증 CI](https://github.com/ajou-2026-1-capstone-5/ostone/actions/runs/24063666195/job/70185016650) · [인증 테스트 PR #19](https://github.com/ajou-2026-1-capstone-5/ostone/pull/19) · [후속 인증 CI](https://github.com/ajou-2026-1-capstone-5/ostone/actions/runs/24249099134/job/70803281057)

팀은 **2026년 아주대학교 SoftCon 개발부문(캡스톤디자인) 우수상**을 받았습니다.

[관련 사례 · 상담 원본 저장과 후속 분석](ostone-corpus.md) · [포트폴리오 목록](README.md)
