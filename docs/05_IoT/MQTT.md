# MQTT Broker

## 목적

IoT 장치와 Collector Service 간의 메시지 전달을 담당한다.

---

## Topic 구조

```
sensor/{deviceId}
```

예시

```
sensor/device001
sensor/device002
sensor/device003
```

---

## Subscriber

Collector Service

---

## QoS

QoS 1

- 최소 1회 전달

---

## 장점

- Lightweight
- 빠른 전송
- IoT 최적화