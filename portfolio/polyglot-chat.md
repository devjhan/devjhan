# [C Socket Chat and File Transfer] A Shared TCP Library for Java and Python

**개인 학습 프로젝트 · TCP 통신 라이브러리 구현과 서버·클라이언트 연동 · C, Spring Boot/JNA, Python/ctypes** | [프로젝트 저장소](https://github.com/devjhan/25-1-mogakso)

> ### 📌 Context & Constraints
>
> - **Objective** 소켓 API를 사용해 TCP 통신 라이브러리를 C로 직접 구현하고, 채팅과 파일 전송으로 동작을 검증하고자 했습니다.
> - **Problem** TCP에서는 한 번에 보낸 데이터도 여러 번에 나뉘어 도착할 수 있었습니다. 여러 터미널을 띄워 실행하던 중 새로 접속한 사용자가 채팅에 추가되지 않는 문제도 발견했습니다.
> - **Constraints** 같은 C 라이브러리를 Java 서버와 Python 터미널 클라이언트에서 사용하고, 파일을 청크 단위로 처리하면서 최종 저장 내용의 무결성을 확인해야 했습니다.
> - **Scope** C에서 메시지 프레이밍과 명령 큐를 구현하고, Java 서버에는 JNA로, Python 클라이언트에는 ctypes로 연결했습니다.

## Decisions & Troubleshooting

### 1. 여러 번에 나뉘어 들어오는 TCP 메시지 처리

시스템 프로그래밍 및 실습 과제에서 C로 파일을 전송하며 `send()`가 요청한 바이트 중 일부만 처리할 수 있고, 한 번에 보낸 데이터가 수신 측에서 여러 번의 `recv()`에 걸쳐 들어올 수 있다는 점을 배웠습니다. 과제에서는 보낸 바이트 수를 누적해 남은 데이터를 전송하고, 받는 쪽에서는 헤더에 적힌 파일 크기만큼 반복해서 읽었습니다. 채팅에서도 한 번의 `recv()`를 메시지 하나로 처리하면 나뉘어 도착한 데이터를 온전한 메시지로 읽을 수 없었습니다.

채팅 라이브러리에서는 메시지 유형 1바이트와 본문 길이 4바이트로 헤더를 구성하고, 길이는 네트워크 바이트 순서로 인코딩했습니다. 파서는 호출 사이에 헤더와 본문의 수신 상태를 유지했습니다. 프레임이 여러 번에 나뉘어 들어와도 메시지가 완성된 시점에 콜백을 호출할 수 있었습니다. [프레임 인코딩](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/C_chat_lib/common/src/protocol.c#L27-L50) · [스트림 파서](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/C_chat_lib/common/src/protocol.c#L76-L137)

### 2. 통신 루프와 터미널 입력 처리 분리

여러 터미널에서 테스트하던 중 새로 접속한 사용자가 채팅에 추가되지 않는 문제를 발견했습니다. `client_start_chat()`은 통신 루프가 끝날 때까지 호출한 스레드를 점유합니다. 같은 스레드에서 터미널 입력이나 테스트 메시지 전송까지 처리하면 후속 동작이 루프 종료를 기다리게 됩니다.

Python 클라이언트에서는 통신 루프를 별도 스레드에서 실행해 터미널 입력 처리를 계속할 수 있게 했습니다. C Echo 테스트에서도 블로킹 호출을 별도 스레드로 옮겨 메인 스레드가 테스트 메시지를 보낼 수 있게 했고, `pthread_join()`은 종료 단계에서 호출했습니다. [Python 클라이언트 스레드](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/py_chat_client/src/app/chat_manager.py#L76-L94) · [C Echo 테스트](https://github.com/devjhan/25-1-mogakso/blob/181badd3505918795ef2c2c52aaa93a6ef438350/C_chat_lib/test/echo_test.c#L94-L125)

### 3. 청크 전송과 파일 무결성 확인

파일을 통째로 읽으면 파일 크기만큼 메모리가 필요했습니다. 파일명과 크기를 먼저 보내고 본문을 청크로 나눴습니다. Python 클라이언트는 청크를 읽을 때마다 같은 SHA-256 계산에 반영하고, 마지막에 파일 전체의 해시를 보냈습니다.

Java 서버는 받은 청크를 디스크에 이어 쓰고 수신 바이트 수를 누적했습니다. 누적 크기가 처음에 받은 파일 크기와 같고, 저장된 파일을 다시 읽어 계산한 SHA-256이 송신 측과 일치할 때 전송을 완료했습니다. [Python 파일 송신](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/py_chat_client/src/core/chat_client.py#L42-L62) · [Java 파일 수신](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/java_chat_server/src/main/java/project/java_chat_server/service/FileTransferService.java#L39-L92)

## Validation & Results

**터미널 수동 검증.** 여러 터미널에서 Java 서버와 Python 클라이언트를 직접 실행했습니다. 새 사용자가 추가되고, 클라이언트끼리 메시지를 주고받으며, Python 클라이언트에서 Java 서버로 파일이 전송되는 것까지 확인했습니다.

**C 라이브러리 테스트.** 공개 커밋 `0d7731d`의 프레이밍·명령 큐 CTest **5개**가 통과했습니다. [C 테스트 정의](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/C_chat_lib/test/CMakeLists.txt#L82-L93)

**파일 전송 서비스 테스트.** 개발 당시 실행한 JUnit 테스트 **13개**가 모두 통과했습니다. *(통과 건수는 당시 로컬 실행 기록을 근거로 했으며, 실행 로그는 개인 자료로 비공개입니다.)* [파일 전송 서비스 테스트 코드](https://github.com/devjhan/25-1-mogakso/blob/0d7731d6bc938f6f23f501ac0984ef6a1c4fd7a6/java_chat_server/src/test/java/project/java_chat_server/service/FileTransferServiceTest.java)
