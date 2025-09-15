# Docker_Container_Network_Lab
# 컨테이너 간 네트워크 통신으로 Spring Boot ↔ MySQL 연동하기

## 1) 목표

- Docker **커스텀 브리지 네트워크**를 만들고
    
    **Spring Boot 컨테이너**와 **MySQL 컨테이너**가 통신하도록 구성
    
- JAR 실행(호스트) → 컨테이너 간 통신(네트워크) → 이미지 빌드/배포

---

## 2) 배경 지식

```
[Container eth0: 172.17.x.x] ─> [docker0(172.17.0.1)] ─> NAT(iptables) ─> [호스트 NIC(enp0s3)] ─> 외부
```

- **docker0(브리지)**: 컨테이너를 스위치처럼 연결, 기본 대역 172.17.0.0/16
- **컨테이너 IP**: 172.17.x.x 사용자 브리지 대역(예: 172.19.0.0/16)
- **컨테이너 이름으로 DNS**: `mysql://mysqldb:3306` 식으로 이름으로 접근

네트워크 드라이버:

- `bridge`(기본): 컨테이너↔컨테이너, 컨테이너↔호스트 통신 OK
- `host`: 호스트 네트워크를 그대로 사용(성능↑/격리↓, 포트매핑 불필요)
- `none`: 네트워크 차단

---

## 3) Docker Network Architecture

```
           ┌────────────────────────────────────────────┐
           │        docker network: springboot-mysql-net│
           │   Subnet: 172.19.0.0/16, GW: 172.19.0.1    │
           │                                            │
           │  ┌──────────────┐        ┌──────────────┐  │
           │  │  bootapp     │        │   mysqldb    │  │
Host:8081 ─┼─▶│:8081         │  JDBC  │:3306         │  │
 (PortFWD) │  │URL→ jdbc:mysql://mysqldb:3306/fisa   │  │
           │  └──────────────┘        └──────────────┘  │
           └────────────────────────────────────────────┘
```

## 4) 단계별 실습

### A. 사용자 브리지 네트워크 생성

```bash
# 커스텀 브리지 네트워크 생성
docker network create springboot-mysql-net

# 상세 확인 (서브넷/GW, 컨테이너 연결 상태 등)
docker inspect springboot-mysql-net
```

### B. MySQL 컨테이너 실행 & 초기화

```bash
# MySQL 8.0 컨테이너 실행 (컨테이너명: mysqldb)
docker run --name mysqldb \
  --network springboot-mysql-net \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=fisa \
  -d mysql:8.0
```

```bash
# 컨테이너 진입(한글 깨질 때 LC_ALL 설정)
docker exec -it -e LC_ALL=C.UTF-8 mysqldb bash

mysql -u root -p
```

필수 스키마(예시):

```sql
USE fisa;

-- 일반계정 사용 시
CREATE USER 'user01'@'%' IDENTIFIED BY 'user01';
GRANT ALL PRIVILEGES ON fisa.* TO 'user01'@'%';
FLUSH PRIVILEGES;

-- 테이블 & 샘플 데이터
CREATE TABLE dept (
  deptno INT NOT NULL,
  dname  VARCHAR(20),
  loc    VARCHAR(20),
  CONSTRAINT pk_dept PRIMARY KEY (deptno)
);

INSERT INTO dept VALUES (10,'ACCOUNTING','NEW YORK'),
                        (20,'RESEARCH','DALLAS'),
                        (30,'SALES','CHICAGO'),
                        (40,'OPERATIONS','BOSTON');
COMMIT;

SELECT * FROM dept;
```

### C. Spring Boot 컨테이너 이미지 빌드

`Dockerfile` (런타임용, JAR만 복사)

```docker
# 1) 런타임 이미지
FROM openjdk:17-slim
WORKDIR /app

# app.jar 파일을 같은 폴더에 둔 상태에서 빌드
COPY app.jar app.jar

# 컨테이너 환경에 최적화된 JVM 옵션
ENTRYPOINT ["java",
  "-XX:+UnlockExperimentalVMOptions",
  "-XX:+UseContainerSupport",
  "-XX:MaxRAMPercentage=75.0",
  "-jar","app.jar"]
```

빌드:

```bash
docker build -t bootapp:1.0 .
docker images
```

### D. 애플리케이션 설정(application.yml) 핵심

- **JDBC URL에서 호스트명이 아니라 컨테이너 이름을 사용**: `mysqldb`
- 포트/계정은 위에서 만든 것과 일치시킵니다.

```yaml
server:
  port: 8081

spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://mysqldb:3306/fisa?characterEncoding=UTF-8&useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
    username: user01       # root를 쓰면 보안상 비권장
    password: user01
  jpa:
    database: mysql
    database-platform: org.hibernate.dialect.MySQL8Dialect
    generate-ddl: true
    hibernate:
      ddl-auto: none
    show-sql: true
```

### E. Spring Boot 컨테이너 실행

```bash
docker run --name bootapp \
  --network springboot-mysql-net \
  -p 8081:8081 \
  -d bootapp:1.0

# 로그 확인
docker logs -f bootapp
```

접속 확인(호스트 브라우저/터미널):

```bash
curl http://127.0.0.1:8081/
```

---
## 5) 자주 나오는 이슈 & 해결

- **이미지 삭제 충돌**: `image is being used by running container`
    
    ```bash
    docker ps -a               # 어떤 컨테이너가 쓰는지 확인
    docker stop <container>
    docker rm   <container>
    docker rmi  <image>:<tag>
    
    ```
    
- **포트 충돌(예: 3306 이미 사용)**: 다른 프로세스/컨테이너가 점유 중 → 다른 호스트 포트 매핑 사용(`p 3307:3306`) 또는 점유한 프로세스 종료
- **DB 접속 실패**: JDBC URL에 `localhost`를 쓰지 말고 **컨테이너 이름**(`mysqldb`) 사용
- **한글 깨짐**: 컨테이너 셸 진입 시 `e LC_ALL=C.UTF-8` 옵션
- **태그 latest**: 환경에 따라 **`mysql:latest`가 다른 주요 버전**을 가리킬 수 있음 → **`mysql:8.0` 고정** 권장
- **가상화 환경 접근(포트 포워딩)**: VirtualBox NAT 사용 시 호스트 포트→게스트 포트 포워딩 설정 필요. 예) 호스트 `8081` → 게스트(컨테이너) `8081`

---

## 6) Docker Compose

여러 컨테이너와 네트워크를 **파일로 선언**하고 `up -d` 한 번에 올림. 반복 실행/공유/협업에 유리.

`docker-compose.yml` 예시:

```yaml
version: "3.9"
services:
  mysqldb:
    image: mysql:8.0
    container_name: mysqldb
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    ports:
      - "3306:3306"
    networks:
      - appnet

  bootapp:
    image: bootapp:1.0
    container_name: bootapp
    depends_on:
      - mysqldb
    ports:
      - "8081:8081"
    networks:
      - appnet

networks:
  appnet:
    driver: bridge
```

```bash
docker compose up -d
docker compose logs -f
docker compose down
```
---

## 7) 참고 명령어 모음

```bash
# 네트워크
docker network ls
docker network create <name>
docker network rm <name>
docker inspect <network-or-container>

# 컨테이너
docker ps -a
docker logs -f <container>
docker exec -it <container> bash
docker stop <container> && docker rm <container>

# 이미지
docker images
docker build -t <repo>:<tag> .
docker rmi <repo>:<tag>
```
