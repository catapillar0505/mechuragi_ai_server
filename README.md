<div align="center">

# 🤖 메추라기 — AI Server

### AWS Bedrock (Claude 3 Haiku) 기반 개인화 음식 추천 마이크로서비스

**날씨·시간·재료·감정·대화, 5가지 컨텍스트로 맞춤 메뉴를 추천합니다.**

![Award](https://img.shields.io/badge/Programming%20GURU%20해커톤-Android%20부문%20대상%20🏆-gold?style=flat)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot%203.5-6DB33F?style=flat&logo=springboot&logoColor=white)
![Java](https://img.shields.io/badge/Java%2017-007396?style=flat&logo=openjdk&logoColor=white)
![AWS Bedrock](https://img.shields.io/badge/AWS%20Bedrock%20·%20Claude%203-232F3E?style=flat&logo=amazonwebservices&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat&logo=redis&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)

📹 **시연 영상**: [YouTube Shorts](https://youtube.com/shorts/QZbuvk-MRn8?feature=share)

> 이 저장소는 [teamMechuragi/mechuragi_ai_server](https://github.com/teamMechuragi/mechuragi_ai_server)의 포크로,
> **김진아([@catapillar0505](https://github.com/catapillar0505))의 기여를 중심으로 정리한 포트폴리오 문서**입니다.
> 이 서버의 머지 PR 23개 중 **20개**를 작성했습니다.

</div>

---

## 🏆 수상

> ### Programming GURU 해커톤 **Android 부문 대상**
> 서울여자대학교 · 팀 Mechuragi · **2024.02**

---

## 🏗️ 아키텍처 위치

<p align="center">
  <img src="./docs/images/architecture.png" width="800"/>
</p>

```
CloudFront → OpenResty Gateway (Lua JWT 검증)
                      ↓  Direct Path (Main Server 미경유)
              AI 서버 :8082  ←────→  AWS Bedrock (Claude 3 Haiku)
                      ↓  (비동기)
              메인 서버 :8080/8081  ←───→  MySQL
```

> 추천 요청은 Main Server를 거치지 않고 게이트웨이에서 AI 서버로 직행합니다.
> k6 부하 테스트로 Legacy(메인 서버 경유) 대비 **처리량 +32%, p95 -45%**를 검증하고 채택한 구조입니다.
> ([상세 분석 보기](https://github.com/catapillar0505/mechuragi_main_server#2-k6-부하-테스트로-증명한-게이트웨이-구조-개선--direct-vs-legacy-97))

---

## 🙋 나의 역할 (머지 PR 20 / 23개)

### AI 추천 엔진 구현 ([#9](https://github.com/teamMechuragi/mechuragi_ai_server/pull/9))
- AWS Bedrock Claude 3 Haiku 기반 추천 엔진 및 5가지 추천 타입(WEATHER/TIME/INGREDIENTS/FEELING/CONVERSATION)별 프롬프트 설계
- DTO를 `external`(프론트↔AI↔메인)과 `internal`(Bedrock 호출)로 분리해 통신 경계 명확화
- 추천 결과를 메인 서버 DB에 **비동기 저장**하는 파이프라인 구축

### 프롬프트 엔지니어링 고도화 ([#23](https://github.com/teamMechuragi/mechuragi_ai_server/pull/23), [#25](https://github.com/teamMechuragi/mechuragi_ai_server/pull/25))
- 타입별로 흩어져 있던 프롬프트를 **단일 `generatePrompt()` 구조로 통합** — 페르소나 → 현재 상황 → 사용자 취향 → 응답 규칙의 일관된 순서
- 알레르기·기피 음식을 합쳐 **[절대 추천 금지]** 제약으로 명시, "반드시 준수"·"허용 음식 종류 내 추천" 지시어로 사용자 바운더리 이탈 방지
- "3가지는 서로 다른 계열의 음식으로 구성" 지침으로 추천 다양성 확보
- enum 코드값(WEIGHT_LOSS, VEGAN 등)을 한국어로 변환하는 헬퍼로 모델 이해도 향상

### API·통신 구조
- JWT 헤더 기반 인증으로 전환 (x-member-id 제거) ([#16](https://github.com/teamMechuragi/mechuragi_ai_server/pull/16))
- 추천 저장 요청을 Nginx 게이트웨이 경유로 통일 + 도메인 변수 관리 ([#22](https://github.com/teamMechuragi/mechuragi_ai_server/pull/22))
- CORS 설정, 로컬/운영 개발환경 분리 ([#15](https://github.com/teamMechuragi/mechuragi_ai_server/pull/15), [#21](https://github.com/teamMechuragi/mechuragi_ai_server/pull/21))

### 배포·운영
- GitHub Actions 파이프라인 정비 — push 시에만 배포, 헬스체크 경로 수정, 공용 도커 네트워크 ([#4](https://github.com/teamMechuragi/mechuragi_ai_server/pull/4), [#6](https://github.com/teamMechuragi/mechuragi_ai_server/pull/6), [#13](https://github.com/teamMechuragi/mechuragi_ai_server/pull/13))
- CloudWatch 로깅, management 엔드포인트 활성화 ([#12](https://github.com/teamMechuragi/mechuragi_ai_server/pull/12))

---

## 🛠️ 트러블슈팅 — LLM 응답 비용·속도·품질 최적화 ([#23](https://github.com/teamMechuragi/mechuragi_ai_server/pull/23))

**문제**: 추천 응답이 느리고 Bedrock 호출 비용이 컸으며, "다시 추천받기" 시 같은 결과가 반복되는 UX 문제가 있었습니다.

**해결**

| 전략 | 내용 | 효과 |
|------|------|------|
| 응답 필드 다이어트 | `ingredients`·`cookingTime`·`difficulty` 필드 제거, `name`+`reason`만 유지 | 페이로드·출력 토큰 감소 |
| max_tokens 축소 | 4000 → **1000** | 비용 절감 + 생성 속도 개선 |
| 시간 윈도우 캐시 | 캐시 키에 `currentTimeMillis / 600000` 포함 → **10분 단위 캐시 갱신** | 같은 요청 반복 호출 방지 + 주기적 신선한 결과 |
| temperature 상향 | 0.7 → **0.9** | 캐시 갱신 시 더 다양한 메뉴 추천 |

단순 `hashCode` 캐시는 영원히 같은 추천을 반환하고, 캐시 제거는 비용이 폭증합니다. **"10분 윈도우"라는 중간 지점**으로 비용과 신선함의 균형을 잡았습니다.

---

## 추천 타입

| 타입 | 설명 | `context` 예시 |
|------|------|----------------|
| `WEATHER` | 현재 날씨 기반 | `["맑음", "따뜻함"]` |
| `TIME` | 식사 시간대 기반 | `["아침", "점심"]` |
| `INGREDIENTS` | 보유 재료 기반 | `["감자", "양파", "계란"]` |
| `FEELING` | 현재 감정/상태 기반 | `["피곤함", "스트레스"]` |
| `CONVERSATION` | 자유 대화형 | `["매콤하고 든든한 것"]` |

## API 엔드포인트

```
GET  /recommend/health
     → { "status": "UP", "service": "mechuragi-ai-service", "version": "1.0.2" }

POST /recommend/food
     Headers: Authorization: Bearer <JWT>
     Body:    FoodRecommendationRequest
     → FoodRecommendationResponse (추천 결과 + 사용 모델)
```

### 요청 예시

```json
{
  "type": "WEATHER",
  "context": ["맑음", "따뜻함"],
  "numberOfDiners": 2,
  "dietStatus": "NONE",
  "veganOption": "NONE",
  "spiceLevel": "MEDIUM",
  "foodTypes": ["KOREAN", "FUSION"],
  "tastes": ["SPICY", "SAVORY"],
  "avoidedFoods": ["버섯"],
  "allergies": ["갑각류"]
}
```

### 응답 예시

```json
{
  "recommendations": [
    {
      "name": "김치찌개",
      "reason": "따뜻한 날씨에 어울리는 매콤한 음식"
    }
  ],
  "model": "anthropic.claude-3-haiku-20240307-v1:0"
}
```

## 프로젝트 구조

```
src/main/java/com/mechuragi/ai/
├── controller/
│   └── RecommendationController.java   # GET /health, POST /food
├── service/
│   ├── RecommendationService.java      # 요청 조율, 비동기 저장
│   ├── BedrockService.java             # Bedrock 호출, Redis 캐싱
│   └── PromptTemplateService.java      # 통합 프롬프트 생성
├── client/
│   └── MainServiceClient.java          # 메인 서버 저장 API 호출
├── config/
│   ├── AwsConfig.java                  # BedrockRuntimeClient 빈
│   ├── WebClientConfig.java            # WebClient, CORS 설정
│   ├── AsyncConfig.java                # @Async 활성화
│   └── SwaggerConfig.java              # OpenAPI (운영 비활성)
├── dto/
│   ├── frontend/                       # 프론트엔드 요청/응답 DTO
│   ├── bedrock/                        # Bedrock 프롬프트/응답 DTO
│   └── main/                           # 메인 서버 저장 요청 DTO
└── type/
    └── RecommendationType.java         # WEATHER, TIME, INGREDIENTS, FEELING, CONVERSATION
```

## 기술 스택

| 분류 | 기술 |
|------|------|
| Framework | Spring Boot 3.5.5 (Java 17) |
| AI Platform | AWS Bedrock — Claude 3 Haiku |
| Cache | Redis (10분 윈도우 추천 캐싱) |
| HTTP Client | Spring WebFlux WebClient (비동기 저장) |
| Container | Docker (멀티스테이지 빌드) |
| CI/CD | GitHub Actions → DockerHub → EC2 (Bastion 경유 SSH) |
| Logging | CloudWatch Logs (`/aws/mechuragi/dev/ai-service`) |
| API Docs | SpringDoc OpenAPI (운영환경 비활성) |

> MySQL은 이 서버에서 직접 접근하지 않습니다. 추천 결과 저장은 메인 서버 API 호출로 위임합니다.

## 로컬 개발 환경

```bash
# .env 파일 설정 (spring-dotenv로 자동 로드)
cp .env.example .env
# .env에 AWS credentials, Redis, Nginx Gateway IP 등 입력

./gradlew bootRun
```

| 환경 변수 | 설명 | 기본값 |
|-----------|------|--------|
| `REDIS_HOST` | Redis 호스트 | `localhost` |
| `REDIS_PORT` | Redis 포트 | `6379` |
| `AWS_REGION` | AWS 리전 | `ap-northeast-2` |
| `BEDROCK_CHAT_MODEL` | Bedrock 모델 ID | `anthropic.claude-3-haiku-20240307-v1:0` |
| `NGINX_GATEWAY_PRIVATE_IP` | 메인 서버 라우팅용 Nginx IP | `localhost` |
| `CORS_ALLOWED_ORIGINS` | CORS 허용 Origin | `http://localhost:3000` |
| `LOG_LEVEL` | 로그 레벨 | `INFO` |

## 배포

GitHub Actions가 `main` / `dev` 브랜치 push 시 자동 실행됩니다.

```
[push] → 테스트 & 빌드 → DockerHub push → SSH(Bastion 경유) → EC2 배포
                                                                ↓
                                                   /recommend/health 헬스체크 (5회)
                                                                ↓
                                                   Discord 성공/실패 알림
```

**컨테이너 실행 방식**
- 포트: `8082`
- 로그 드라이버: `awslogs` → CloudWatch `/aws/mechuragi/dev/ai-service`
- 재시작 정책: `unless-stopped`
- 네트워크: `mechuragi-network` (메인 서버와 동일)

## 프로파일

| 프로파일 | 로그 레벨 | Swagger | 비고 |
|----------|----------|---------|------|
| `local` | DEBUG | 활성 | localhost 직접 접속 |
| `dev` | DEBUG | 활성 | EC2 개발 환경 |
| `production` | INFO | **비활성** | CloudWatch 통합 |

---

## 관련 저장소

- 🐦 [mechuragi_main_server (포크)](https://github.com/catapillar0505/mechuragi_main_server) — Spring Boot 메인 백엔드 (기여 요약 포함)
- 💻 [mechuragi_client (포크)](https://github.com/catapillar0505/mechuragi_client) — Next.js 웹 클라이언트
- 🏗️ [mechuragi_infra (포크)](https://github.com/catapillar0505/mechuragi_infra) — Terraform · Ansible · OpenResty 게이트웨이 (기여 100%)
- 🤖 [팀 원본 저장소](https://github.com/teamMechuragi/mechuragi_ai_server)
