# DatasourceGenerator

## 역할

DatasourceGenerator는 센서 및 데이터 소스를 관리하고, 센서에서 수집된 데이터를 MQTT Broker로 발행(Publish)하는 서비스입니다.

(기존 명칭: Datasource Service)

실제 운영 환경에서는 IoT 센서와 연동되며, 개발 환경에서는 CSV 등을 반복적으로 읽어 센서 데이터를 시뮬레이션 생성합니다.
서비스 이름의 "Generator"는 이 시뮬레이션 데이터 생성 역할을 강조한 것입니다.

---

# 책임

- 데이터 소스 관리
- 센서 관리
- 센서 데이터 발행 (실제 또는 시뮬레이션)
- MQTT Publish
- 센서 상태 관리

---

# 주요 기능

## 데이터 소스 등록

센서가 연결될 데이터 소스를 등록합니다.

등록 정보

- 데이터 소스 이름
- 데이터 소스 타입
- 위치
- 설명

---

## 센서 등록

데이터 소스에 센서를 등록합니다.

등록 정보

- 센서 이름
- 센서 타입
- 센서 고유번호
- 연결된 데이터 소스

---

## 센서 데이터 발행

센서 데이터를 MQTT Broker로 발행합니다.

발행 데이터

- Temperature
- Humidity
- CO₂
- Light

---

## 센서 상태 관리

센서의 현재 상태를 관리합니다.

상태

- ONLINE
- OFFLINE
- ERROR

Rule Engine Service가 센서 오류/연결 해제를 감지하면 SensorErrorEvent를 발행하며,
DatasourceGenerator는 이를 구독하여 상태를 갱신합니다.

---

# API

## 데이터 소스 등록

POST /datasources

---

## 데이터 소스 조회

GET /datasources

---

## 센서 등록

POST /sensors

---

## 센서 조회

GET /sensors

---

## 센서 상태 조회

GET /sensors/{sensorId}

---

# Database

DatasourceGenerator는 PostgreSQL을 사용합니다.

### Table

- datasource
- sensor

---

# Redis

사용하지 않습니다.

---

# 다른 서비스와의 통신

## 호출하는 서비스

없음

---

## 호출받는 서비스

- API Gateway (관리 기능)

---

# Event

## 구독 이벤트

### SensorErrorEvent

Rule Engine Service가 발행합니다.

센서 오류/연결 해제가 감지되면 sensor 테이블의 status를 OFFLINE/ERROR로 갱신합니다.

---

발행하는 이벤트는 없습니다. 센서 데이터는 MQTT를 통해서만 전송합니다.

---

# MQTT

## Publish Topic

```
sensor/{sensorId}
```

### Payload

```json
{
  "sensorId": 1,
  "temperature": 21.8,
  "humidity": 92.4,
  "co2": 760,
  "light": 430,
  "measuredAt": "2026-08-15T12:30:00"
}
```

---

# Sequence

센서 (실제 또는 시뮬레이션)

↓

DatasourceGenerator

↓

MQTT Publish

↓

MQTT Broker

↓

Rule Engine Service

---

# 예외 상황

- 존재하지 않는 센서
- 존재하지 않는 데이터 소스
- MQTT Broker 연결 실패
- 센서 연결 실패
- 센서 데이터 형식 오류

---

# 추후 개발 예정

- 실제 IoT 센서 연동
- 센서 자동 등록
- 센서 Health Check
- 센서 Firmware 관리
