# Prompt Engineering

## 목적

LLM이 항상 동일한 JSON 형식으로 응답하도록 Prompt를 구성한다.

---

## System Prompt

당신은 식물 전문가입니다.

사용자의 질문에 대해 반드시 JSON으로만 응답하십시오.

---

## User Prompt

몬스테라를 키우려고 합니다.

적절한 환경을 추천해주세요.

---

## Context

Embedding Service가 검색한 식물 환경 정보

---

## 출력 형식

```json
{
    "temperature":24,
    "humidity":65,
    "co2":700,
    "ph":6.5,
    "reason":"..."
}
```

---

## 규칙

- JSON만 출력
- 설명은 reason 필드 사용
- 수치는 실수 허용