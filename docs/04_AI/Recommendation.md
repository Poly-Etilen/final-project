# Environment Recommendation

## 목적

사용자가 새로운 Workspace를 생성할 때 AI가 추천 환경을 생성한다.

---

## Sequence

```mermaid
sequenceDiagram

User->>Workspace: 새 Workspace 생성

Workspace->>AI: 추천 요청

AI->>Embedding: Vector Search

Embedding-->>AI: Context

AI->>LLM: Prompt 생성

LLM-->>AI: JSON 반환

AI-->>Workspace: 추천 DTO

Workspace-->>User: 추천 화면
```

---

## 특징

AI는 추천만 수행한다.

데이터 저장은 하지 않는다.

---

## 저장 시점

사용자가

"환경 구성 완료"

버튼을 누를 때

Workspace Service가

EnvironmentSetting을 저장한다.