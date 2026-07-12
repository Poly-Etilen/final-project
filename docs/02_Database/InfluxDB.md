# InfluxDB

## 목적

센서 데이터를 저장하는 TSDB이다.

---

# Measurement

sensor_data

---

# Tags

- workspace_id
- device_id
- sensor_type

---

# Fields

- temperature
- humidity
- co2
- ph

---

# Time

InfluxDB Timestamp

---

# 조회

Dashboard는 Storage Service를 호출하여 조회한다.

Frontend가 InfluxDB를 직접 조회하지 않는다.

---

# AI Report

Storage Service가

주간

월간

데이터를 집계하여 AI Service에 전달한다.