---
title: "Linux 네트워크 포트 상태(LISTEN, ESTABLISHED, TIME_WAIT, CLOSE_WAIT)와 CLOSE_WAIT 누수 장애"
date: 2026-11-27T19:45:00+09:00
draft: false
summary: "Linux의 ss/netstat으로 확인하는 4가지 TCP 포트 상태(LISTEN, ESTABLISHED, TIME_WAIT, CLOSE_WAIT)와 4-Way Handshake 기준 차이, 파일 디스크립터(FD) 고갈(Too many open files)로 이어지는 CLOSE_WAIT 누수 원인 및 스레드 덤프·코드·타임아웃 기반 해결 전략을 정리합니다."
categories: ["Backend"]
tags: ["Linux", "Network", "TCP", "CLOSE_WAIT", "TIME_WAIT", "File Descriptor", "Troubleshooting", "Backend"]
---

## 1. Linux TCP 네트워크 포트 4가지 핵심 상태

Linux에서 `ss`나 `netstat` 명령어로 확인하는 TCP 네트워크 포트 상태는 소켓이 연결을 대기하고, 통신하며, 종료하는 생명주기를 보여준다. 그중 장애 분석에서 가장 자주 마주치는 4가지 핵심 상태는 다음과 같다.

| 상태 | 의미 | 발생 시점 | 정상 / 주의 여부 |
| :--- | :--- | :--- | :--- |
| **`LISTEN`** | 연결 대기 중 | 서버 프로세스가 특정 포트를 열고 클라이언트의 연결 요청(`SYN`)을 기다림 | 정상 (서버 포트 개방 상태) |
| **`ESTABLISHED`** | 연결 완료 및 통신 중 | 3-Way Handshake가 끝나고 실제 데이터를 주고받는 상태 | 정상 (급증 시 트래픽 폭주·커넥션 풀 점검) |
| **`TIME_WAIT`** | 연결 종료 후 대기 | 먼저 연결 종료(`FIN`)를 요청한 쪽(Active Close)이 지연 패킷 유실 방지 등을 위해 일정 시간(Linux 기본 60초) 대기 | 일시적 증가는 정상 (과도하면 로컬 포트 고갈 유발) |
| **`CLOSE_WAIT`** | 상대방 종료 후 로컬 종료 대기 | 상대방으로부터 `FIN`을 받고 `ACK`를 보낸 뒤, **로컬 애플리케이션이 `close()`를 호출하기를 기다리는 상태** | **지속 누적 시 장애 (애플리케이션 버그·리소스 누수)** |

---

## 2. `TIME_WAIT` vs `CLOSE_WAIT`: 왜 차이가 날까?

두 상태는 TCP 연결을 끊는 **4-Way Handshake** 과정에서 <strong>누가 먼저 연결 종료를 요청했는지(Active vs Passive)</strong>에 따라 갈린다.

```text
[먼저 끊는 쪽: Active Close]                 [끊김을 당하는 쪽: Passive Close]
  ├─ FIN 전송 ─────────────────────────────▶  ├─ FIN 수신 및 ACK 응답
  ├─ ACK 수신                                 ├─ CLOSE_WAIT 진입!
  └─ TIME_WAIT 진입 (커널이 60초 후 자동 회수) └─ 애플리케이션이 close()를 호출해야만 LAST_ACK ➔ CLOSED
```

- **Active Close (먼저 끊은 쪽) ➔ `TIME_WAIT`**: 커널이 정해진 시간(2MSL, Linux 기본 60초)이 지나면 자동으로 소켓을 회수한다.
- **Passive Close (상대방이 먼저 끊은 쪽) ➔ `CLOSE_WAIT`**: 커널이 임의로 소켓을 정리해 주지 않는다. **로컬 애플리케이션 코드가 명시적으로 소켓을 닫아야(`close()`)** 비로소 다음 단계(`LAST_ACK` ➔ `CLOSED`)로 넘어가며 완전히 정리된다.

---

## 3. `CLOSE_WAIT` 누수 장애의 원인과 증상

**`CLOSE_WAIT` 누수 장애**란 상대방 서버나 클라이언트가 연결을 정상적으로 끊었음(`FIN` 수신)에도, **로컬 서버의 애플리케이션(내 애플리케이션) 코드가 소켓을 닫지 않아 소켓이 영구적으로 `CLOSE_WAIT` 상태에 머무르는 현상**을 말한다.

### 대표적인 발생 원인 3가지

1. **리소스 해제 누락**: 예외가 발생했을 때 `finally` 블록이나 `try-with-resources`, `defer`, `with` 구문 등으로 소켓·스트림을 안전하게 닫지 않은 경우
2. **HTTP 클라이언트 사용 실수**: 외부 API 호출 후 HTTP 응답 바디를 끝까지 소비(`EntityUtils.consume()`)하지 않거나 `response.close()`를 호출하지 않아 커넥션이 풀에 반환되지 않는 경우
3. **애플리케이션 지연 또는 데드락**: 스레드가 DB 쿼리 지연이나 락 경합으로 멈춰 있어, 소켓을 닫는 로직까지 도달하지 못하는 경우

### 왜 치명적인 장애로 이어질까? (`Too many open files` 연결)

이전 글([Linux 파일 디스크립터(File Descriptor)와 소켓 고갈, ulimit 설정과 Too many open files 해결](https://bonggyunjo.github.io/backend/linux-file-descriptor-and-too-many-open-files/))에서도 정리했듯, Linux에서는 모든 네트워크 소켓이 <strong>파일 디스크립터(FD)</strong>를 하나씩 점유한다.

따라서 `CLOSE_WAIT` 상태의 소켓이 회수되지 않고 계속 쌓이면 결국 프로세스에 할당된 FD 한도(`ulimit -n`)에 도달하게 되며, 다음과 같은 연쇄 장애로 이어진다.

- 로그에 `Too many open files` 에러 발생
- 새로운 네트워크 연결 수락 불가 및 외부 API·DB 연결 실패
- 서비스 전체 응답 불가(Hang) 상태

---

## 4. 대응 및 근본 해결 방법

### 왜 커널 튜닝으로는 해결할 수 없을까?

커널 입장에서는 자바(또는 서버) 애플리케이션이 아직 소켓 수신 버퍼에 남아 있는 잔여 데이터를 처리 중인지 알 수 없다. 따라서 **애플리케이션이 스스로 `socket.close()`를 호출하기 전까지는 커널이 소켓을 절대 강제로 닫을 수 없도록 설계**되어 있다.

그렇기 때문에 `tcp_tw_reuse`, `tcp_fin_timeout` 같은 커널 파라미터 튜닝으로는 `CLOSE_WAIT`를 절대 해결할 수 없으며, **오직 애플리케이션 코드와 클라이언트 설정을 고치는 것만이 유일한 근본 해결책**이다.

### 운영 장애 시 권장 대응 순서

1. **긴급 복구 ➔ 재가동 전 반드시 '스레드 덤프' 먼저 확보하기**
   - 운영 서버에서 `Too many open files`와 함께 `CLOSE_WAIT`가 수천 개 쌓여 서비스가 마비되었다면, 점유된 소켓과 FD를 회수하기 위해 원인 프로세스를 재가동해야 한다.
   - 단, **프로세스를 재가동하기 전에 반드시 스레드 덤프(`jstack`, `pstack` 등)를 먼저 떠두는 것이 좋다.** 재시작해 버리면 당시 스레드 상태가 사라지지만, 덤프 파일을 남겨두면 나중에 열어보며 어느 외부 API 호출 구간에서 타임아웃이나 누수가 발생했는지 정확히 찾아낼 수 있기 때문이다.
2. **근본 해결 ➔ 코드 수정 및 타임아웃·유휴 커넥션 정리 적용**
   - 원인이 된 외부 호출 코드에 **`try-with-resources`**(또는 `finally` 블록)를 적용해 예외가 발생해도 반드시 `close()`가 호출되도록 수정한다.
   - HTTP 클라이언트와 커넥션 풀에 **Connect Timeout / Read Timeout**을 반드시 지정하고, 오래 놀고 있는 유휴 커넥션을 정리하는 스레드(**`evictIdleConnections`** 등)를 활성화해 배포하는 것이 좋다.
