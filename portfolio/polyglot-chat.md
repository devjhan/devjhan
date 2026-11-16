# Polyglot Chat — TCP 프레이밍과 파일 전송

**개인 학습 프로젝트 · C, Spring Boot/JNA, Python/ctypes**

TCP는 바이트 스트림이므로 한 번의 `recv()`에 메시지 일부만 오거나 여러 메시지가 함께 올 수 있습니다. C로 메시지 경계를 복원하는 소켓 라이브러리와 명령 큐를 만들고, Java 서버와 Python 터미널 클라이언트에 연결해 채팅과 파일 전송에 사용했습니다. [프로젝트 저장소](https://github.com/devjhan/25-1-mogakso)

## 끊기거나 이어서 들어오는 메시지 복원

시스템 프로그래밍 과제에서 `send()`가 요청한 바이트의 일부만 처리할 수 있고, 수신 측에서는 받은 바이트 수를 누적해야 한다는 점을 배웠습니다. 채팅에서는 메시지 유형 **1바이트**와 네트워크 바이트 순서의 본문 길이 **4바이트**를 헤더로 정했습니다. C 파서는 호출 사이에 헤더와 본문의 수신 상태를 유지하고, 헤더가 가리키는 길이만큼 본문이 모이면 콜백을 호출한 뒤 다음 메시지를 읽습니다. [프레임 인코딩](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/C_chat_lib/common/src/protocol.c#L27-L50) · [스트림 파서](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/C_chat_lib/common/src/protocol.c#L76-L137)

수신 파서는 헤더에 선언된 본문 길이만큼 버퍼를 할당합니다. 할당에 실패하면 `-1`을 반환하고, C 서버의 수신 루프에서 파싱 오류를 보고한 뒤 해당 연결을 정리하도록 구현했습니다. [파서의 길이 처리](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/C_chat_lib/common/src/protocol.c#L96-L117) · [서버 수신 루프](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/C_chat_lib/server_lib/src/chat_server.c#L511-L526)

## 블로킹 통신 루프와 터미널 입력 분리

Java 서버는 JNA로 C 라이브러리를 로드하고 연결·메시지 콜백을 등록했습니다. Python은 ctypes로 C 클라이언트 함수와 콜백을 연결했습니다. `client_start_chat()`은 통신 루프가 끝날 때까지 호출 스레드를 점유하므로, Python에서는 별도 스레드에서 실행해 터미널 입력을 계속 처리할 수 있게 했습니다. C Echo 테스트에서도 통신 루프를 별도 스레드로 옮겨 메인 스레드에서 메시지를 보내고, 종료 단계에서 `pthread_join()`을 호출했습니다. [Java JNA 매핑](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/java_chat_server/src/main/java/project/java_chat_server/wrapper_library/c_wrapper/ChatServerLibrary.java#L20-L39) · [Python 통신 스레드](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/py_chat_client/src/app/chat_manager.py#L76-L94) · [C Echo 테스트](https://github.com/devjhan/25-1-mogakso/blob/181badd3505918795ef2c2c52aaa93a6ef438350/C_chat_lib/test/echo_test.c#L94-L125)

## 청크는 순차 기록하고 파일은 종료 시 검증

파일 전체를 메모리에 올리지 않도록 Python 클라이언트가 파일명과 크기를 먼저 보내고, 본문을 청크로 읽어 전송했습니다. 읽은 청크는 하나의 SHA-256 계산에 누적하고, 마지막에 전체 파일 해시를 보냈습니다. Java 서버는 청크를 디스크에 순차 기록하며 수신 크기를 누적했습니다. 종료 요청에서 예상 크기와 수신 크기를 비교하고, 저장된 파일 전체를 읽어 계산한 SHA-256이 송신 측 해시와 같을 때 전송을 완료했습니다. [Python 파일 송신](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/py_chat_client/src/core/chat_client.py#L42-L62) · [Java 파일 수신](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/java_chat_server/src/main/java/project/java_chat_server/service/FileTransferService.java#L39-L92)

크기가 틀리거나 해시가 다르면 전송 완료를 거부하고 부분 파일과 세션을 정리하는 경로로 들어갑니다. 명시적 취소 메서드는 세션을 제거하고 임시 파일 삭제를 시도합니다. 서비스 단위 테스트에서 크기·해시 불일치와 명시적 취소 경로를 검증했습니다. [취소 구현](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/java_chat_server/src/main/java/project/java_chat_server/service/FileTransferService.java#L99-L109) · [실패·취소 테스트](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/java_chat_server/src/test/java/project/java_chat_server/service/FileTransferServiceTest.java#L111-L178)

## 검증 범위

- **수동 통합 검증** — 여러 터미널에서 새 사용자가 추가되지 않는 상황을 발견했습니다. 수정 후 Java 서버와 Python 클라이언트를 다시 실행해 새 사용자 추가, 클라이언트 간 채팅, Python에서 Java 서버로 파일 전송을 확인했습니다.
- **C 프레이밍·명령 큐** — 공개 커밋 `0d7731d`에서 이 범위의 CTest **5개**가 통과했습니다. [C 테스트 정의](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/C_chat_lib/test/CMakeLists.txt#L82-L93)
- **Java 파일 전송 서비스** — 개발 당시 로컬에서 수신 크기·해시 불일치와 취소를 포함한 JUnit 테스트 **13개**를 실행해 모두 통과했습니다. 실행 로그는 개인 자료로 비공개입니다. [JUnit 테스트 코드](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/java_chat_server/src/test/java/project/java_chat_server/service/FileTransferServiceTest.java)

공개 Python 통합 테스트에서는 실행 중인 서버가 필요한 항목을 따로 표시했습니다. C·Java·Python을 연결한 경로는 서버와 클라이언트를 띄운 터미널에서 수동으로 검증했습니다. [Python 통합 테스트](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/py_chat_client/tests/integration/test_integration.py#L20-L50)

[포트폴리오 목록](README.md)
