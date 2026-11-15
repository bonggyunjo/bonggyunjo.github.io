---
title: "Linux 파일 디스크립터(File Descriptor)와 소켓 고갈, ulimit 설정과 Too many open files 해결"
date: 2026-11-15T16:50:00+09:00
draft: false
summary: "부하 테스트나 대규모 트래픽 상황에서 서버가 갑자기 멈추는 대표적인 원인인 'Too many open files' 오류를 파일 디스크립터(FD) 개념부터 ulimit 1024 기본 제한, CLOSE_WAIT 및 FD Leak 원인, 그리고 limits.conf·systemd LimitNOFILE 해결 방법까지 정리합니다."
categories: ["Backend"]
tags: ["Linux", "Backend", "File Descriptor", "Socket", "ulimit", "systemd", "Troubleshooting", "Spring Boot"]
---

## 1. 대규모 트래픽에서 서버가 허무하게 뻗는 이유: `Too many open files`

트래픽이 몰리거나 부하 테스트를 진행할 때 CPU나 메모리가 아직 여유 있는데도 서버가 갑자기 응답을 멈추며 아래와 같은 에러를 뱉고 죽는 경우가 있다.

```text
java.io.IOException: Too many open files
OSError: [Errno 24] Too many open files
accept4 failed (24: Too many open files)
socket: too many open files
```

이 현상을 정확히 파악하고 해결하려면 리눅스의 **파일 디스크립터(File Descriptor, FD)** 개념부터 이해해야 한다.

---

## 2. 파일 디스크립터(FD)와 "리눅스는 모든 것이 파일이다"

### 주요 개념

- **파일 디스크립터 (File Descriptor, FD)**: 운영체제(OS)가 실행 중인 프로세스에서 열려 있는 파일, 네트워크 소켓, 파이프, 디렉토리 등의 **입출력 자원을 식별하기 위해 부여하는 정수 값**이다.
- **Too many open files 오류**: 특정 프로세스가 OS로부터 할당받아 동시에 열 수 있는 파일 디스크립터의 **최대 허용 개수(Limit)를 초과**했을 때 발생하는 시스템 오류다.

리눅스 계열 운영체제는 <strong>"모든 것이 파일이다(Everything is a file)"</strong>라는 철학을 따른다. 하드디스크에 있는 텍스트 파일이든, 네트워크 랜선으로 들어오는 TCP 연결이든, 모니터 출력이든 전부 똑같은 파일 인터페이스로 취급해 `read`하고 `write`한다.

프로세스가 파일이나 네트워크 연결을 하나 열 때마다 리눅스 커널은 이를 구분하기 위해 0부터 시작하는 작은 정수 번호표를 하나씩 붙여주는데, 이것이 바로 <strong>파일 디스크립터(FD)</strong>다.

```text
• FD 0 : 표준 입력 (stdin)
• FD 1 : 표준 출력 (stdout)
• FD 2 : 표준 에러 (stderr)
• FD 3 : app.log 파일 기록용
• FD 4 : MySQL DB 커넥션 (TCP 소켓)
• FD 5 : 1번 고객과의 HTTPS 연결 (TCP 소켓)
• FD 6 : 2번 고객과의 HTTPS 연결 (TCP 소켓)
...
```

즉, **사용자가 웹 서버에 접속할 때마다 리눅스 커널은 소켓용 FD 번호표를 1개씩 소비**한다.

---

## 3. 왜 `Too many open files`가 쉽게 발생할까? (`ulimit -n 1024`)

리눅스 OS는 한 프로세스가 파일과 소켓을 무제한으로 열어 시스템 전체를 마비시키는 것을 막기 위해, 프로세스당 동시에 열 수 있는 FD 개수의 상한선(`ulimit -n`)을 걸어둔다. 문제는 대부분의 리눅스 배포판에서 이 **기본 제한값(Soft Limit)이 고작 1,024개**로 매우 낮게 설정되어 있다는 점이다.

스프링 부트 서버 하나가 떠 있을 때 실제로 소비하는 FD 개수를 계산해 보면 1,024개가 얼마나 금방 차는지 바로 체감할 수 있다.

```text
- 스프링 부트 서버 자체 라이브러리(JAR) 파일 오픈: ~100개
- DB 커넥션 풀(HikariCP) 유지용 소켓: ~50개
- 로그 파일(Logback 등): ~10개
- 동시 접속자 850명 웹사이트 접속(TCP 소켓): 850개
────────────────────────────────────────────────────
▶ 합계: 이미 1,010개 사용 (기본 한도 1,024개 턱밑 도달)
```

이 상태에서 동시 접속자가 **겨우 20명만 더 들어오면** 즉시 참사가 벌어진다.

1. 리눅스 커널이 `EMFILE: Too many open files` 시스템 콜 에러를 발생시킨다.
2. 자바(JVM) 레벨로 `java.io.IOException: Too many open files` 예외가 전파된다.
3. 새로운 사용자의 **TCP 연결 수립 자체가 불가능**해져 서비스 접속이 막힌다.
4. 심지어 **에러 로그를 쓸 파일 FD조차 열지 못해 로그도 남지 않는 상황**에 빠진다.

### 주요 발생 원인 4가지

큰 틀에서 보면 <strong>정상적인 트래픽 증가(인프라 기본값 방치)</strong>와 **소스코드의 자원 누수(Leak)** 두 가지 축으로 나뉜다.

1. **파일 디스크립터 누수 (FD Leak, 가장 흔함)**: 코드에서 파일, 네트워크 소켓, HTTP 클라이언트(`HttpClient`, `RestTemplate` 등) 응답 스트림, DB 커넥션을 연(`open`) 뒤 작업이 끝났는데도 명시적으로 닫지(`close`) 않아 FD가 반환되지 않고 계속 쌓이는 현상이다.
2. **기본 제한값 부족 (Soft Limit 1024)**: 리눅스 기본 제한값이 1,024개로 낮게 잡혀 있어 트래픽이 조금만 몰려도 금방 한계에 부딪힌다.
3. **동시 접속 및 트래픽 급증**: 다수의 클라이언트가 동시에 접속하면서 정상적인 TCP 소켓이 한꺼번에 많이 열릴 때 발생한다.
4. **`CLOSE_WAIT` 상태 소켓 누적**: 네트워크 연결 종료 과정에서 상대방이 연결을 끊었음(`FIN`)에도 내 애플리케이션 프로세스가 소켓을 `close()`하지 않고 `CLOSE_WAIT` 상태로 방치해 FD를 계속 점유하는 경우다.

---

## 4. 해결 방법과 확인 순서

`Too many open files`를 마주했을 때는 무작정 한도 숫자만 올리기보다 **1) 먼저 FD 누수(`close` 누락) 여부를 확인하고, 2) 정상 트래픽 대비 한도가 부족하다면 `ulimit` / `LimitNOFILE`을 증설**하는 순서로 접근해야 한다.

### 1) 실시간 트러블슈팅 및 누수 확인

현재 실행 중인 프로세스가 실제로 몇 개의 FD를 점유하고 있는지, 터미널의 한도는 얼마인지부터 확인한다.

```bash
# 현재 내 프로세스(스프링 부트 등)가 잡고 있는 FD 개수 확인
lsof -p <PID> | wc -l

# 현재 셸 세션의 오픈 파일 한도 확인
ulimit -n
```

만약 트래픽이 빠졌는데도 `lsof` 개수가 줄어들지 않고 계속 늘어나거나 `CLOSE_WAIT` 소켓이 쌓여 있다면, 코드 내에서 스트림이나 소켓의 `close()`(`try-with-resources` 등)가 누락된 누수일 가능성이 높다.

### 2) 프로세스 한도 영구 증설 (`/etc/security/limits.conf`)

서버가 재부팅되더라도 적용되도록 OS 전역 설정 파일에 `nofile` 한도를 높여둔다.

```text
# /etc/security/limits.conf
* soft nofile 65535
* hard nofile 65535
```

### 3) `systemd` 서비스 설정 (`LimitNOFILE`, 데몬 실행 시 필수)

우분투나 CentOS/RHEL 환경에서 `systemctl start myapp`처럼 `systemd` 서비스로 스프링 부트 애플리케이션을 띄우는 경우, `/etc/security/limits.conf` 설정이 적용되지 않는 경우가 많다. 따라서 반드시 해당 서비스 유닛 파일(`.service`) 내부에 한도를 직접 명시해야 한다.

```ini
# /etc/systemd/system/myapp.service
[Service]
LimitNOFILE=65535
```

설정 변경 후에는 `sudo systemctl daemon-reload`와 서비스 재시작을 거쳐 실제 프로세스에 새 한도가 반영되었는지(`/proc/<PID>/limits` 확인) 점검하면 된다.

---

## 5. 핵심 요약

1. 리눅스는 일반 파일뿐 아니라 <strong>TCP 소켓, DB 커넥션, 파이프까지 모두 파일 디스크립터(FD)</strong>로 관리한다.
2. 리눅스 기본 `ulimit -n` 값이 **1,024**로 낮게 잡혀 있으면, 애플리케이션 기본 점유분 + 수백 명의 동시 접속만으로도 쉽게 한도에 걸려 접속 불가 및 로그 기록 불가 상태에 빠진다.
3. 대응 순서는 **① `lsof` 등으로 FD 누수(`close()` 누락, `CLOSE_WAIT`) 여부를 먼저 점검하고, ② 정상 트래픽 수용을 위해 `limits.conf`와 `systemd`의 `LimitNOFILE` 한도를 65535 이상으로 상향**하는 것이다.
