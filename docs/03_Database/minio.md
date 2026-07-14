# MinIO

## 개요

MinIO는 사용자가 직접 촬영하여 업로드한 버섯 생육 사진 원본을 저장하는 객체 저장소입니다.

카메라 센서가 아닌 사용자의 앱/웹 업로드로 사진이 생성되며,
Cultivation Service가 사진을 저장하고, AI Service는 Vision 분석을 위해 읽기 전용으로 조회합니다.

---

# 사용 서비스

| Service | 역할 |
|----------|------|
| Cultivation Service | 사진 업로드 |
| AI Service | 사진 조회 (Vision 분석용) |

---

# Bucket

```
mushroom-photos
```

---

# Key 구조

```
{cultivationId}/{uploadedAt}.jpg
```

예시

```
3/20260815-090000.jpg
```

---

# 메타데이터

실제 이미지 파일은 MinIO에 저장하고, 아래 메타데이터는 Cultivation DB의 `photo` 테이블에서 관리합니다.

| Field | 설명 |
|--------|------|
| image_url | MinIO Object URL |
| cultivation_id | 재배 ID |
| uploaded_at | 업로드 시각 |

---

# 업로드 흐름

사용자 (카메라 촬영)

↓

Client 앱/웹

↓

Cultivation Service

↓

MinIO 저장 (PUT Object)

↓

image_url 발급

↓

PostgreSQL에 photo 메타데이터 저장

↓

AI Service에 분석 요청 (image_url 전달)

---

# 조회 흐름

Cultivation Service

↓

AI Service

↓

전달받은 image_url로 MinIO에서 이미지 조회

↓

Vision 모델 입력으로 사용

---

# 파일 형식

지원 형식

```
JPG, PNG
```

최대 용량

```
10MB
```

---

# Retention Policy

사진 원본은 삭제하지 않고 보관합니다.

재분석이나 생육 이력 확인을 위해 재배가 종료된 이후에도 유지합니다.

※ 운영 환경에서는 오래된 사진에 대한 별도 아카이빙 정책을 적용할 수 있습니다.

---

# 장애 대응

MinIO 장애 발생 시

- 사진 업로드 불가
- AI Vision 분석(생육 분석) 기능 일시 이용 불가
- 기존에 캐시된 생육 분석 결과(Redis)는 조회 가능

---

# 고려 사항

- 관계형 데이터는 저장하지 않으며, 이미지 파일만 저장합니다.
- 사진 메타데이터(URL, 업로드 시각)는 Cultivation DB에서 관리하고, MinIO는 파일 저장소 역할만 합니다.
- 여러 장의 사진이 등록된 경우 AI Service는 가장 최근 업로드본을 사용합니다.
- PostgreSQL과 이미지 데이터를 중복 저장하지 않습니다.
- 카메라 센서가 아닌 사용자가 직접 촬영하여 업로드하는 방식입니다.

---

# 추후 개발 예정

- 사진 업로드 전 압축/리사이징
- Presigned URL 기반 클라이언트 직접 업로드
- 오래된 사진 아카이빙 정책
- 재촬영 이력(사진 버전) 관리
