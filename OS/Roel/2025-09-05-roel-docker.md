---
date: 2025-09-05
user: Roel4990
topic: 도커(Docker)가 뭐고 왜 쓰고 어떻게 돌아가는지 — 동작 원리와 실전 팁
---

# 1. 도커(docker)란 무엇인가?
도커는 애플리케이션과 그 실행환경을 ‘이미지’로 묶어, 한 커널 위에서 격리된 ‘컨테이너’ 프로세스로 빠르게 실행하도록 돕는 도구.

## 왜 쓸까?
- 일관성: “내 로컬/서버 어디서든 동일하게”

- 속도/밀도: 부팅 없는 프로세스 수준 격리 → 생성/종료 수백 ms

- 배포 단순화: 이미지 한 개로 코드+라이브러리+설정을 휴대


# 2. 도커를 알기 위한 필수 개념
- 이미지(Image): 실행 스냅샷(읽기 전용 레이어 묶음). ubuntu:22.04처럼 태그로 식별.

- 컨테이너(Container): 이미지를 실제 프로세스로 띄운 것(읽기-쓰기 레이어 추가).

- 레이어/Copy-on-Write(COW): 파일 변경 시 전체 복사 대신 변경분만 기록.

- 레지스트리(Registry): 이미지를 저장/배포(예: Docker Hub).

- 볼륨(Volume): 컨테이너 수명과 독립된 지속 저장소.

- 네트워크(Network): 컨테이너 간 통신(기본은 bridge).

## 정리
- “이미지=실행 스냅샷”, “컨테이너=그 이미지를 프로세스로 실행한 것”.

- VM과의 차이: VM은 게스트 커널 포함, 컨테이너는 호스트 커널 공유 → 부팅 없음, 오버헤드 작음.

- 이점: 환경 일관성(개발/스테이징/운영 동일), 빠른 배포/롤백, 높은 밀도(같은 서버에 더 많이).

# 3. 내부 구성과 동작 원리

## 3.1. 구성요소 역할
- Docker CLI → dockerd(데몬)에게 명령 전달

- dockerd → 컨테이너 수명주기·네트워킹·로그·빌드 관리, 그리고 containerd에게 실행 위임

- containerd → 이미지 풀/스냅샷/컨테이너 관리, runc 호출

- runc(OCI 런타임) → 실제로 네임스페이스/유저/마운트/cgroups 설정하고 execve로 프로세스 시작

- 리눅스 커널 기능
  - Namespaces: PID/NET/MNT/IPC/UTS/USER → “보이는 세계” 격리
  
  - cgroups v2: CPU/메모리/IO 제한
  
  - Capabilities · seccomp · LSM(AppArmor/SELinux): 권한 최소화
```text
CLI → dockerd → containerd → runc → (Linux 커널: namespaces, cgroups) → 실행 프로세스
```
## 3.2. docker run 내부 흐름(예: docker run -d -p 8080:80 --name web nginx:latest)

- 이미지 확인/풀: 로컬에 없으면 레지스트리에서 메타데이터+레이어 다운로드

- 파일시스템 준비: overlay2 드라이버로 다중 RO 레이어 + 컨테이너 전용 RW 레이어를 merged에 마운트

- 격리/제한 설정: runc가 pid/net/mnt/uts/ipc/user 네임스페이스와 cgroups 세팅

- 프로세스 시작: 컨테이너 내부 PID 1로 엔트리포인트 실행(예: nginx -g 'daemon off;')

- 네트워킹 연결: 기본 bridge에 veth 페어 붙이고, 호스트에서 8080→컨테이너 80 NAT(포트포워딩)

- 로그/모니터링: 표준출력/에러 수집, 상태/리소스 추적

# 4. 저장소 동작: overlay2 & COW 직관

- 이미지 레이어는 읽기 전용, 컨테이너 시작 시 얇은 RW 레이어가 위에 놓임.

- 파일 수정 시 “변경된 파일만” RW 레이어에 기록 → 빠르고 공간 절약.

- 컨테이너 제거해도 볼륨에 저장한 데이터는 남는다.

> **꿀팁**: 빈번히 바뀌는 데이터는 볼륨/바인드 마운트로 빼면 좋다(이미지 크기/빌드 속도 개선).

# 5. 네트워킹 핵심

- 기본 드라이버는 bridge. 각 컨테이너는 가상 NIC로 이 다리에 붙는다.

- veth 페어: 호스트 이름공간 ↔ 컨테이너 NET 네임스페이스 연결 터널.

- 포트 매핑: -p 8080:80은 호스트 8080 → 컨테이너 80 NAT.

- 사용자 정의 브리지 네트워크를 쓰면 컨테이너끼리 서비스명으로 DNS 통신 가능.

# 6. 보안 기초

- Capabilities: 루트 권한을 잘게 쪼개 필요한 것만 부여.

- User namespace: 컨테이너 루트를 호스트 비특권 사용자로 매핑 가능.

- Rootless Docker: 데몬 없이 비root로 실행(초보는 기본 모드부터 시작 추천).

- 읽기 전용 루트 + tmpfs: 변조 면역력 향상.
```bash
docker run --read-only -u 1000:1000 --cap-drop ALL --cap-add NET_BIND_SERVICE \
-v appdata:/data -p 8080:80 myimage:latest
```

# 7. 연습 실행 방법

## 7.1. 컨테이너 띄우기/접속/정리

```bash
# 1) nginx 웹 서버 실행
docker run -d --name web -p 8080:80 nginx:latest

# 2) 상태/로그 확인
docker ps
docker logs -f web

# 3) 컨테이너 안에서 셸로 보기
docker exec -it web bash   # (alpine 계열이면 sh)

# 4) 파일 바꿔 보기: 바인드 마운트로 실시간 반영
mkdir -p $(pwd)/site
echo "Hello Docker!" > $(pwd)/site/index.html
docker rm -f web
docker run -d --name web -p 8080:80 -v $(pwd)/site:/usr/share/nginx/html:ro nginx:latest

# 5) 리소스 제한 맛보기
docker rm -f web
docker run -d --name web -p 8080:80 --memory=256m --cpus="0.50" nginx:latest

```

# 7.2. Dockerfile 빌드 & 멀티스테이지 맛보기

```dockerfile
# 빌드 단계 (node 빌드 예시)
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 실행 단계 (가벼운 nginx)
FROM nginx:stable-alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

```bash
docker build -t myapp:1.0 .
docker run -d --name myapp -p 8080:80 myapp:1.0
```

> **포인트**: 빌드 도구(node 등)는 최종 이미지에 포함되지 않아 이미지가 **작고 안전**해진다.

---

## 8. 자주 겪는 오류 & 빠른 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `port is already allocated` | 호스트 포트 사용 중 | `lsof -i :8080`로 점유 프로세스 종료 또는 다른 포트 사용 |
| `permission denied` (마운트) | 호스트 디렉터리 권한 문제 | `chmod/chown` 조정 또는 `-u 1000:1000`로 사용자 맞추기 |
| 컨테이너는 떴는데 접속 불가 | 내부 서비스가 `0.0.0.0`이 아닌 `127.0.0.1` 바인딩 | 앱을 `0.0.0.0`에 바인딩하도록 설정 |
| 변경이 이미지에 안 남음 | COW 특성 | 지속 데이터는 **볼륨**에, 코드 반영은 **새 빌드** |
| 이미지 용량 폭증 | 불필요 레이어/캐시 | `.dockerignore` 추가, 멀티스테이지, 패키지 캐시 정리 |

### `.dockerignore` 예
```plaintext
node_modules
.git
*.log
dist
```

---

## 9. 성능·용량 최적화 꿀팁

- **베이스 이미지 최소화**: `-alpine` 계열(단, glibc 필요 여부 확인)
- **레이어 수 줄이기**: 관련 명령 묶어서 `RUN` 최소화
- **캐시 활용**: 자주 바뀌는 파일(`COPY` 소스)을 마지막 단계에 배치
- **헬스체크**: 비정상 상태 조기 감지

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:80 || exit 1
```

---

## 10. 운영 실무 한 숟갈

- **재시작 정책**: `--restart=unless-stopped`
- **로그**: 기본 json-logs → 로테이션 옵션 고려
- **Compose**: 여러 컨테이너를 하나의 앱처럼 정의

```yaml
# docker-compose.yml
services:
  web:
    image: nginx:latest
    ports: ["8080:80"]
    volumes: ["./site:/usr/share/nginx/html:ro"]
    restart: unless-stopped
```