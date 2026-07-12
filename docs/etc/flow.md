# 🌐 EcoSphere 전체 요청 및 이벤트 흐름 가이드

이 문서는 EcoSphere 플랫폼에서 클라이언트의 요청 및 시스템 이벤트에 따라 발생하는 전체 아키텍처 흐름(Data Flow)을 정리한 가이드입니다.

---

## 1. 인증 및 회원 관리 흐름 (Auth & User Flow)
사용자의 로그인, 회원가입 및 개인정보 관리는 **동기(REST API)** 방식으로 처리되며, 인증 서비스와 유저 서비스가 분리되어 보안성을 높입니다.

### 1-1. 회원가입 및 이메일 인증
* **흐름:** Client → API Gateway → Auth Service
* **상세 단계:**
    1. 사용자가 이메일 회원가입을 요청합니다.
    2. Auth Service는 회원 정보를 PostgreSQL에 저장하고 이메일 인증 코드를 발급합니다.
    3. 발급된 인증 코드는 유효시간(5분) 동안 Redis에 캐싱됩니다.
    4. 사용자가 코드를 입력해 인증을 완료하면 Redis 내 데이터가 검증됩니다.

### 1-2. 로그인 및 토큰 관리 (JWT / OAuth2)
* **흐름:** Client → API Gateway → Auth Service
* **상세 단계:**
    1. 사용자가 이메일 또는 OAuth2(구글 등)로 로그인을 요청합니다.
    2. Auth Service는 PostgreSQL(또는 OAuth 연동 테이블)에서 계정을 검증합니다.
    3. 검증 성공 시 Access Token과 Refresh Token을 발급합니다.
    4. Refresh Token은 보안 및 빠른 만료 처리를 위해 Redis에 14일 동안 저장됩니다.

### 1-3. 내 정보 조회 및 수정
* **흐름:** Client → API Gateway → User Service
* **상세 단계:**
    1. API Gateway가 클라이언트 요청의 JWT 인증을 수행합니다.
    2. 인증이 완료된 요청은 User Service로 라우팅됩니다.
    3. User Service는 오직 사용자의 개인정보 데이터(프로필, 닉네임 등)만 PostgreSQL(User)에서 조회하거나 수정하여 반환합니다.

---

## 2. 워크스페이스 및 디바이스 관리 흐름 (Workspace & Device Flow)
식물을 관리하는 핵심 공간인 워크스페이스의 생성, 멤버 초대 및 센서 등록 과정입니다.

### 2-1. 워크스페이스 기본 CRUD 및 초대
* **흐름:** Client → API Gateway → Workspace Service
* **상세 단계:**
    1. 사용자가 워크스페이스 생성, 조회, 수정, 삭제를 요청합니다.
    2. Workspace Service는 해당 요청을 처리하고 비즈니스 데이터를 PostgreSQL(Workspace)에 반영합니다.
    3. 멤버 초대 시 생성된 초대 코드는 Redis 캐시에 저장되어 초대 수락 시 빠른 검증에 사용됩니다.

### 2-2. IoT 디바이스(센서) 등록
* **흐름:** Client → API Gateway → Workspace Service
* **상세 단계:**
    1. 사용자가 특정 워크스페이스에 연결할 MQTT 센서 정보(MQTT Client ID, Topic 등)를 입력해 등록을 요청합니다.
    2. Workspace Service는 이를 검증하고 디바이스 메타데이터를 PostgreSQL에 저장하여 파이프라인 수신 준비를 마칩니다.

---

## 3. AI 식물 환경 추천 흐름 (AI Recommendation Sync Flow)
사용자가 워크스페이스를 생성할 때 최적의 생육 환경을 지능적으로 제안받는 **동기(REST)** 기반 RAG 흐름입니다.

```text
사용자 ──> API Gateway ──> Workspace Service ──> AI Service ──> Embedding Service ──> Elasticsearch(Vector DB)
                                                                                          │ (Top-K 결과)
사용자 <── 환경 저장 완료 <── Workspace Service <── AI Service <── LLM 응답 생성 <────────┘

```

---

## 1. 워크스페이스 생성 및 AI 추천 요청 (웹 요청)
* **사용자 행동:** 웹 대시보드에서 '새 워크스페이스 생성' 버튼을 누르고 관리할 식물 이름(예: "몬스테라")과 함께 자연어로 요청을 입력합니다.

* **시스템 흐름:** 
1. 클라이언트의 POST /api/workspaces/recommend 요청이 API Gateway로 진입합니다.
2. API Gateway는 사용자의 JWT 토큰을 인증한 후 요청을 Workspace Service로 라우팅합니다.
3. Workspace Service는 즉시 비즈니스 DB에 저장하지 않고, 내부 REST 통신을 통해 AI Service(POST /internal/ai/recommend)로 추천 요청을 위임합니다.

## 2. RAG 기반 최적 생육 환경 도출 (AI 레이어)
* 시스템 흐름:

1. AI Service는 반복적인 LLM 연산 비용을 줄이기 위해 먼저 Redis Semantic Cache를 조회하여 동일하거나 유사한 질문이 있었는지 확인합니다.

2. 캐시 미스가 발생하면, AI Service는 Embedding Service를 호출하여 사용자의 질문을 벡터화합니다.

3. Embedding Service는 벡터 DB인 Elasticsearch의 plant_environment 인덱스에서 KNN 유사도 검색을 수행하여 해당 식물과 가장 잘 맞는 전문 생육 데이터(Context)를 추출합니다.

4. 검색된 식물 정보와 시스템 프롬프트를 조합하여 LLM을 호출합니다.

5. LLM은 설정된 출력 규칙에 따라 정확한 JSON 포맷(목표 온도, 습도, CO₂, pH 및 추천 사유)으로 응답을 생성합니다.

## 3. 최종 환경 설정 및 워크스페이스 저장 (메타데이터 적재)
* 사용자 행동: 사용자는 웹 화면에 표시된 AI의 추천 수치(예: 온도 24℃, 습도 65% 등)를 확인하고, 본인의 재배 환경에 맞게 상/하한선 임계치를 수정한 뒤 '환경 구성 완료' 버튼을 누릅니다.

**시스템 흐름:**

1. 웹 브라우저가 POST /api/workspaces/{workspaceId}/environment API를 호출합니다.

2. 요청을 받은 Workspace Service가 드디어 PostgreSQL 데이터베이스의 workspace 테이블과 environment_setting 테이블에 관리 공간 정보 및 허용 범위 임계치 데이터를 저장합니다.

3. 이 시점부터 해당 워크스페이스는 고유한 workspace_id를 부여받고 활성화됩니다.

## 4. IoT 디바이스(Data Source) 등록
* 사용자 행동: 사용자는 식물 주위에 배치한 센서 디바이스를 시스템에 연동하기 위해 디바이스 고유 ID와 수집할 센서 타입을 입력합니다.

**시스템 흐름:**

1. 웹에서 POST /api/workspaces/{workspaceId}/devices API로 등록을 요청합니다.

2. Workspace Service는 해당 센서가 통신할 MQTT 클라이언트 ID와 구독할 토픽 구조(sensor/{deviceId}) 정보를 PostgreSQL의 device 테이블에 저장합니다.

3. 이 메타데이터 등록을 통해 IoT 데이터 파이프라인에서 해당 디바이스의 메시지를 수용할 준비가 완료됩니다.

## 5. 데이터 소스 연결 및 MQTT 수집 파이프라인 활성화
**시스템 흐름:**

1. 하드웨어 센서나 FBP(Flow-Based Programming) 엔진 같은 Data Source가 활성화되어 실제 온/습도 및 CO₂ 값을 측정하기 시작합니다.

2. Data Source는 지정된 MQTT 토픽(sensor/device001)으로 센서 데이터 JSON 페이로드를 MQTT Broker로 발행(Publish)합니다.

3. EcoSphere의 Collector Service는 MQTT Broker를 상시 구독(Subscribe)하고 있으므로, 유입된 메시지를 실시간으로 수신하여 DTO로 변환한 뒤 Rule Engine으로 넘겨주며 비동기 데이터 파이프라인이 정상 작동하게 됩니다.