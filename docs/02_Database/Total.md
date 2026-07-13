| Service                  | PostgreSQL           | Redis                  | InfluxDB | Elasticsearch | 비고        |
| ------------------------ | -------------------- | ---------------------- | -------- | ------------- | --------- |
| **Auth Service**         | ✅ auth_user          | ✅ RefreshToken, 이메일 인증 | ❌        | ❌             | 인증 전용     |
| **User Service**         | ✅ users              | (선택)                   | ❌        | ❌             | 프로필 관리    |
| **Workspace Service**    | ✅ workspace 관련       | (선택)                   | ❌        | ❌             | 핵심 비즈니스   |
| **AI Service**           | ❌                    | ✅ AI 응답 캐시(선택)         | ❌        | ❌             | Stateless |
| **Embedding Service**    | ❌                    | ❌                      | ❌        | ✅ VectorDB    | 벡터 검색     |
| **Collector Service**    | ❌                    | ❌                      | ❌        | ❌             | MQTT 수집만  |
| **Rule Engine Service**  | ❌                    | ❌                      | ❌        | ❌             | 필터/분기     |
| **Sensor Service**       | ✅ AI Report, EventLog | (선택)                   | ✅ 센서 데이터 | ❌             | 센서 저장     |
| **Notification Service** | ❌                    | ❌                      | ❌        | ❌             | 알림 전송     |
| **Gateway**              | ❌                    | ❌                      | ❌        | ❌             | 라우팅       |
| **Datasource Service**   | ✅                     | ❌                      | ❌        | ❌             | CSV 파싱    |

### PostgreSQL
* Auth
* User
* Workspace
* Sensor

### Redis
* Auth(Refresh Token, 이메일 인증 코드)
* AI (AI 환경 추천 결과, AI 진단 결과, AI 리포트)
* Workspace (목록 캐시, 상세 정보 캐시, 초대 코드 캐시)

### InfluxDB
* Sensor

### Elasticsearch
* Embedding