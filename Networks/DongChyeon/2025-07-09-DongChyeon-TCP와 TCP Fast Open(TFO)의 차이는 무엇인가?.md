---
date: 2025-07-09
user: DongChyeon
topic: TCP와 TCP Fast Open(TFO)의 차이는 무엇인가?
---

# TCP와 TCP Fast Open(TFO)의 차이는 무엇인가?

TCP와 TCP Fast Open은 `연결 과정과 데이터 전송 시점에서 차이`가 있다.

## 1. 기존 TCP 연결 과정

<img width="327" height="294" alt="Image" src="https://github.com/user-attachments/assets/93fd8f76-5b8c-4dad-988d-3be286230f02" />

기본적인 TCP 3-way handshake:
1. Client -> Server : SYN (연결 요청)
2. Server -> Client : SYN-ACK (연결 수락 응답)
3. Client -> Server : ACK (응답 확인)
4. 이후 데이터 전송 시작

### RTT (Round Trip Time) 이란?

- RTT는 왕복 시간을 의미한다.
- 클라이언트가 패킷을 보내고, 서버가 그 패킷에 대한 응답을 돌려받을 때까지 걸리는 총 시간이다.

기존 TCP 연결에서 1~3단계의 handshake가 1 RTT를 소모하며, 데이터 전송은 handshake 완료 후 시작된다.
즉, 최소 1 RTT 이후에야 데이터 수신이 가능하다.

## 2. TCP Fast Open 연결 과정

<img width="675" height="312" alt="Image" src="https://github.com/user-attachments/assets/1aab1e7c-9d2c-4c1d-9884-6e22ec6ad101" />

TCP Fast Open은 `초기 SYNC 패킷에 데이터(payload)를 함께 실어보낼 수 있는 기능`을 제공한다.

동작 흐름
1. Client -> Server : SYN + Fast Open Cookie 요청 (최초 접속 시, 데이터 X -> 서버에서 쿠키 발급)
2. Server -> Client : SYN-ACK + Fast Open Cookie 전송
3. Client -> Server : ACK, 이후 클라이언트는 서버로부터 받은 Fast Open Cookie를 저장한다.

- 최초 연결은 기존 TCP와 유사하지만, Fast Open Cookie 발급 과정이 추가된다.
- 이 단계에서도 데이터 전송은 handshake 완료 후에 가능하므로 1 RTT가 소모된다.

다음부터 같은 서버 접속 시 (쿠키 있음):
1. Client -> Server : SYN + Fast Open Cookie + 데이터
2. Server -> Client : SYN-ACK + (데이터 처리 응답)
3. Client -> Server : ACK

- 클라이언트는 SYN 패킷에 Cookie와 데이터를 함께 담아 전송한다.
- 서버는 Cookie를 검증하고 handshake와 동시에 데이터를 처리할 수 있다.
- 따라서, 0.5 RTT 혹은 1 RTT를 절감할 수 있다.

### 왜 0.5 RTT ~ 1 RTT 인가?
- 0.5 RTT 절감:  
  서버가 SYN-ACK를 보낸 후 연결이 확정되면, 클라이언트가 이미 보낸 데이터를 바로 처리할 수 있어 `추가 데이터 전송 RTT 없이` 응답 가능.  
  → 평균적으로 0.5 RTT 정도의 절감 효과

- 1 RTT 절감:  
  일부 구현에서는 SYN-ACK 응답 전에도 데이터를 처리하도록 최적화되어, `거의 1 RTT 절감 효과`를 보이기도 한다.

## 3. Cookie 기반 인증 이유
- TCP Fast Open은 `SYN Flood Attack과 같은 DoS 공격 방지`를 위해 Cookie 인증을 사용한다.
- 서버는 클라이언트의 `Fast Open Cookie가 유효할 경우` handshake가 끝나기 전에 데이터를 수신하고 처리한다.

## 4. 요약

| 항목 | 기존 TCP | TCP Fast Open |
|---|---|---|
| **데이터 전송 시점** | 3-way handshake 완료 후 | SYN 패킷에 데이터 포함 가능 |
| **RTT 소요** | 최소 1 RTT | 0.5 ~ 1 RTT 절감 가능 |
| **보안** | 없음 | Fast Open Cookie 기반 인증 |
| **사용 조건** | 모든 TCP 지원 서버 | 클라이언트와 서버 모두 TFO 지원 필요 |
