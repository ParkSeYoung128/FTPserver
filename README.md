# FTP 서버/클라이언트 구현 프로젝트

C 언어와 TCP/IP 소켓 프로그래밍으로 구현한 FTP(File Transfer Protocol) 서버·클라이언트입니다. 제어 연결과 데이터 연결을 분리한 표준 FTP 구조를 따르며, `fork()` 기반으로 여러 클라이언트를 동시에 처리합니다.

| 항목 | 내용 |
|---|---|
| 작성자 | 박세영 |
| 작성일 | 2023/10/28 |
| 개발 환경 | Ubuntu 20.04.06 LTS (64bit), GCC |
| 과제 | 시스템 프로그래밍 – FTP 서버/클라이언트 |

---

## 목차

1. [주요 기능](#주요-기능)
2. [프로젝트 구조](#프로젝트-구조)
3. [빌드 및 실행](#빌드-및-실행)
4. [지원 명령어](#지원-명령어)
5. [시스템 아키텍처](#시스템-아키텍처)
6. [접근 제어 및 인증](#접근-제어-및-인증)
7. [응답 코드](#응답-코드)
8. [한계 및 개선 방향](#한계-및-개선-방향)

---

## 주요 기능

### 클라이언트 (`cli.c`)

- 서버 연결 및 사용자 인증
- 사용자 명령을 FTP 명령으로 변환하여 전송
- 파일/디렉토리 조작: `ls`, `pwd`, `cd`, `mkdir`, `rmdir`, `delete`, `rename`
- 파일 전송: `get`(다운로드), `put`(업로드), 전송된 바이트 수 출력
- 전송 모드 설정: `bin`, `ascii`, `type binary`, `type ascii`
- PORT 모드 데이터 연결 자동 생성 및 관리
- 세션 종료: `quit`

### 서버 (`srv.c`)

- `fork()` 기반 동시 다중 클라이언트 처리 (Concurrent Server)
- `access.txt` 기반 IP 접근 제어 (와일드카드 지원)
- `passwd` 파일 기반 사용자 인증, 로그인 실패 3회 제한
- 제어 연결 / 데이터 연결 분리
- 바이너리 / ASCII 전송 모드 지원
- `SIGINT`(^C) 처리를 통한 정상 종료

---

## 프로젝트 구조

```
.
├── cli.c            # FTP 클라이언트
├── srv.c            # FTP 서버
├── Makefile         # 빌드 스크립트
├── access.txt       # 허용 IP 패턴 목록
├── passwd           # 사용자 인증 정보
└── README.md
```

**`access.txt`**: 접속을 허용할 IP 패턴을 한 줄에 하나씩 적습니다. `*` 와일드카드를 쓸 수 있습니다.

```
127.0.0.1
128.134.64.*
203.245.*.*
```

**`passwd`**: `/etc/passwd`와 유사한 형식으로 사용자명과 비밀번호를 저장하며, 로그인 검증에 사용합니다.

---

## 빌드 및 실행

### 요구사항

- Linux/Unix (Ubuntu 20.04 이상 권장), GCC, `make`
- 사용 헤더: `sys/socket.h`, `netinet/in.h`, `arpa/inet.h`, `sys/stat.h`, `sys/wait.h`, `dirent.h`, `pwd.h`, `grp.h`, `time.h` 등 표준 POSIX 헤더

### 빌드

```bash
make          # cli, srv 모두 빌드
make cli      # 클라이언트만
make srv      # 서버만
make clean    # 실행 파일 삭제
```

### 실행

```bash
# 서버: 포트 2121에서 대기 (종료: Ctrl+C)
./srv 2121

# 클라이언트: 서버 IP, 포트 지정
./cli 127.0.0.1 2121
```

접속 후 흐름은 다음과 같습니다.

1. 서버가 IP를 확인하고 Welcome 메시지 또는 거부 메시지를 보냅니다.
2. `Input ID :` 와 `Input Password :` 를 차례로 입력합니다.
3. 로그인에 성공하면 `ftp>` 프롬프트가 표시됩니다.

---

## 지원 명령어

| 사용자 명령 | 전송되는 FTP 명령 | 설명 | 데이터 연결 |
|---|---|---|:---:|
| `ls` | `NLST` | 파일 이름 목록 | O |
| `ls -a` | `NLST -a` | 숨김 파일 포함 | O |
| `ls -l` | `NLST -l` | 상세 정보 포함 | O |
| `ls -al` / `ls -la` | `NLST -al` | 숨김 파일 + 상세 정보 | O |
| `dir` | `LIST` | 디렉토리 상세 목록 | O |
| `pwd` | `PWD` | 현재 작업 디렉토리 | X |
| `cd <경로>` | `CWD <경로>` | 디렉토리 이동 | X |
| `cd ..` | `CDUP` | 상위 디렉토리 이동 | X |
| `mkdir <이름>` | `MKD <이름>` | 디렉토리 생성 | X |
| `rmdir <이름>` | `RMD <이름>` | 빈 디렉토리 삭제 | X |
| `delete <이름>` | `DELE <이름>` | 파일 삭제 | X |
| `rename <old> <new>` | `RNFR <old>` → `RNTO <new>` | 이름 변경 (2단계) | X |
| `get <파일>` | `RETR <파일>` | 다운로드 | O |
| `put <파일>` | `STOR <파일>` | 업로드 | O |
| `bin` / `type binary` | `TYPE I` | 바이너리 모드 (기본값) | X |
| `ascii` / `type ascii` | `TYPE A` | ASCII 모드 | X |
| `quit` | `QUIT` | 세션 종료 | X |

### 실행 예시

```
ftp> ls -l
200 Port command successful.
150 Opening data connection for directory list.
-rw-r--r--  1 user group  1024 May 28 10:00 file1.txt
drwxr-xr-x  2 user group  4096 May 28 10:01 directory1
226 Result is transmitted successfully.

ftp> get file1.txt
150 Opening binary mode data connection.
226 Complete transmission.
OK. 1024 bytes is received.

ftp> rename old.txt new.txt
350 File exists, ready to rename
250 RNTO command succeeds

ftp> quit
221 Goodbye.
```

> 위 출력 문구는 실제 구현한 메시지와 동일하게 맞춰서 작성하세요.

---

## 시스템 아키텍처

### 연결 구조

| 구분 | 포트 | 용도 |
|---|---|---|
| 제어 연결 (Control) | 실행 인자로 지정 (예: 2121) | 명령 전송, 응답 수신 |
| 데이터 연결 (Data) | 10001~30000 사이 랜덤 포트 | 파일 데이터, 목록 전송 |

데이터 연결은 **PORT 모드**로 맺습니다. 클라이언트가 데이터용 소켓을 열고 `PORT` 명령으로 자신의 IP와 포트를 알려주면, 서버가 그 주소로 접속합니다. 전송이 끝나면 데이터 연결은 닫히고, 다음 전송이 필요할 때 새로 만듭니다.

### 동시성 처리

- 부모 프로세스는 `accept()`로 새 연결을 계속 받습니다.
- 연결마다 `fork()`로 자식 프로세스를 만들고, 자식이 해당 클라이언트의 인증과 명령 처리를 맡습니다.
- 종료된 자식은 `waitpid()`로 회수하여 좀비 프로세스를 방지합니다.

### 명령 처리 흐름

```
[Client]                                   [Server]
  사용자 입력 (ls -l)
  FTP 명령으로 변환 (NLST -l)
  데이터 소켓 생성/대기
  PORT <ip,port>  ───────────────────────▶ 200 Port command successful
  NLST -l         ───────────────────────▶ 명령 파싱·실행
                  ◀─────────────────────── 150 Opening data connection
                  ◀════ 데이터 연결로 결과 전송 ════
                  ◀─────────────────────── 226 Complete transmission
  결과 출력, 데이터 연결 종료
```

---

## 접근 제어 및 인증

### 1단계: IP 접근 제어

`access.txt`의 패턴과 클라이언트 IP를 비교합니다.

- 허용되면 `220` Welcome 메시지를 보냅니다.
- 거부되면 `431` 메시지를 보낸 뒤 연결을 종료합니다.

### 2단계: 사용자 인증

```
USER <id>  ──▶  passwd에서 조회
                ├─ 존재: 331 Password required
                └─ 없음: 430 Invalid username
PASS <pw>  ──▶  비밀번호 검증
                ├─ 일치: 230 User logged in
                └─ 불일치: 430 Invalid password
```

- 실패하면 재시도할 수 있고, **3회 실패하면 `530 Failed to log-in`** 후 세션을 종료합니다.
- 비밀번호는 평문으로 비교합니다.

---

## 응답 코드

| 코드 | 의미 | 사용 시점 |
|:---:|---|---|
| 150 | 데이터 연결 열림 | 파일/목록 전송 시작 전 |
| 200 | 명령 성공 | `PORT` 성공 |
| 201 | 타입 설정 성공 | `TYPE` 성공 |
| 220 | 서비스 준비 완료 | 접속 시 Welcome |
| 221 | 세션 종료 | `QUIT` |
| 226 | 전송 완료 | 파일/목록 전송 후 |
| 230 | 로그인 성공 | 인증 성공 |
| 250 | 파일 작업 성공 | `CWD`, `MKD`, `RMD`, `DELE`, `RNTO` |
| 257 | 경로명 표시 | `PWD` 응답 |
| 331 | 비밀번호 필요 | `USER` 이후 |
| 350 | 이름 변경 대기 | `RNFR` 이후 |
| 430 | 잘못된 사용자명/비밀번호 | 인증 실패 |
| 431 | 접근 거부 | IP 접근 제어 실패 |
| 502 | 명령 오류 | 잘못된 `TYPE` 등 |
| 530 | 로그인 실패 | 인증 3회 실패 |
| 550 | 파일/디렉토리 작업 실패 | 파일 없음, 권한 없음 등 |

---

## 한계 및 개선 방향

- 비밀번호를 평문으로 저장·비교하므로 해시(예: crypt, SHA-256)로 개선이 필요합니다.
- 제어/데이터 통신이 암호화되지 않으므로 FTPS/SFTP로 확장할 수 있습니다.
- PORT 모드만 지원하여 방화벽/NAT 환경에서는 동작이 제한됩니다. PASV 모드를 추가할 수 있습니다.
- 접속자마다 프로세스를 생성하는 구조라 대규모 접속에는 `select`/`epoll` 기반 구조가 더 적합합니다.
