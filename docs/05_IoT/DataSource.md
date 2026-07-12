# Datasource Service

## 목적

개발 환경에서는 실제 IoT 센서를 사용할 수 없기 때문에 CSV 데이터를 이용하여 센서 데이터를 시뮬레이션한다.

---

## 역할

- CSV 파일 읽기
- 데이터 파싱
- MQTT Publish
- 실시간 데이터 시뮬레이션

---

## Publish Topic

```
sensor/{deviceId}
```

예시

```
sensor/device001
```

---

## Payload

```json
{
  "deviceId":"device001",
  "temperature":24.5,
  "humidity":61,
  "co2":630,
  "timestamp":"2026-07-12T10:00:00"
}
```

---

## 특징

- 실제 센서를 대체
- Publish 간격 조절 가능
- 테스트 데이터 재사용 가능