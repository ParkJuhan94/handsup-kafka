# HandsUp Kafka

HandsUp 프로젝트의 Kafka 브로커 인프라. Apache Kafka KRaft 모드 사용 (Zookeeper 불필요).

## Prerequisites

- Docker Engine 24+
- Docker Compose v2

## Local Development

```bash
# 1. 환경변수 파일 생성 (최초 1회)
cp .env.dev.example .env.dev
# KAFKA_ADVERTISED_HOST=localhost 로 설정 (기본값)

# 2. 브로커 + Kafka UI 시작
docker compose -f docker-compose.dev.yml --env-file .env.dev up -d

# 3. Kafka UI 접속
# http://localhost:8080

# 4. 애플리케이션 연결
# bootstrap-servers: localhost:29092

# 5. 중지
docker compose -f docker-compose.dev.yml --env-file .env.dev down

# 데이터까지 삭제하려면
docker compose -f docker-compose.dev.yml --env-file .env.dev down -v
```

## Dev Server

`dev` 브랜치에 push 시 GitHub Actions CD 파이프라인이 자동으로 dev EC2에 배포.

### GitHub Secrets 설정 필요 (Dev)

| Secret | 설명 |
|--------|------|
| `ENV_DEV_CONTENT` | .env.dev 파일 내용 (`KAFKA_ADVERTISED_HOST`에 EC2 IP 설정) |
| `EC2_DEV_HOST` | dev EC2 퍼블릭 IP |
| `EC2_DEV_USERNAME` | dev EC2 SSH 사용자 |
| `EC2_DEV_SSH_PRIVATE_KEY` | dev EC2 SSH 키 |
| `EC2_DEV_PORT` | dev EC2 SSH 포트 |

## Production

`main` 브랜치에 push 시 GitHub Actions CD 파이프라인이 자동으로 prod EC2에 배포.

### GitHub Secrets 설정 필요 (Prod)

| Secret | 설명 |
|--------|------|
| `ENV_PROD_CONTENT` | .env.prod 파일 내용 |
| `EC2_HOST` | EC2 퍼블릭 IP |
| `EC2_USERNAME` | EC2 SSH 사용자 |
| `EC2_SSH_PRIVATE_KEY` | EC2 SSH 키 |
| `EC2_PORT` | EC2 SSH 포트 |

### Production Kafka UI (SSH 터널)

kafka-ui는 `127.0.0.1`로만 바인딩되어 외부 접근 불가. SSH 터널로 접속:

```bash
# 1. SSH 터널 열기
ssh -L 8080:localhost:8080 -i <키파일> ubuntu@<EC2_IP> -p <SSH포트>

# 2. 브라우저에서 접속
# http://localhost:8080
```

## Architecture

- **Image**: `apache/kafka` (KRaft 모드)
- **Dev**: CONTROLLER + INTERNAL(9092) + EXTERNAL(9094) 리스너, kafka-ui 공개 접근
- **Production**: CONTROLLER + INTERNAL(9092) + EXTERNAL(9094) 리스너, 메모리 제한, kafka-ui SSH 터널 전용
