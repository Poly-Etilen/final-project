# Workspace Database

## 개요

Workspace는 하나의 식물을 관리하는 공간이다.

Workspace 생성 시 생성자는 ADMIN 권한을 가진다.

---

# workspace

관리 공간

---

# workspace_member

Workspace 참여자

- ADMIN
- MEMBER

---

# environment_setting

사용자가 최종 저장한 환경 설정

AI 추천값은 저장하지 않는다.

저장 대상

- 목표 온도
- 목표 습도
- 목표 CO₂
- 목표 pH

허용 범위

- Temperature Min
- Temperature Max

- Humidity Min
- Humidity Max

- CO₂ Min
- CO₂ Max

- pH Min
- pH Max

---

# device

MQTT 센서

저장

- MQTT Client ID
- Topic
- Sensor Type

---

# workspace_invite

초대 코드

---

# ai_report

AI가 생성한

- Daily
- Weekly
- Monthly

리포트 저장