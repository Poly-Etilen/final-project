# Photo Storage (MinIO / 로컬)

## 개요

사용자가 직접 촬영하여 업로드한 버섯 생육 사진 원본은 객체 저장소(기본값 MinIO)에
저장합니다. 카메라 센서가 아닌 사용자의 앱/웹 업로드로 사진이 생성되며, Cultivation
Service가 사진을 저장하고, AI Service는 Vision 분석을 위해 읽기 전용으로 조회합니다.

사진의 실제 저장 위치(MinIO 객체 저장소 또는 로컬 파일시스템)와 무관하게 동작하도록,
DB에는 완성된 URL을 저장하지 않고 저장소 중립적인 `object_key` + `storage_type`만
저장합니다. 실제로 접근 가능한 경로/URL을 조합하는 로직은 서비스 설정(저장소별 base
path/endpoint)에 있으며, 저장소를 MinIO ↔ 로컬로 바꾸더라도 이미 저장된 DB 행을 고칠
필요가 없습니다.

---

# 사용 서비스

| Service | 역할 |
|----------|------|
| Cultivation Service | 사진 업로드/저장 |
| AI Service | 사진 조회 (Vision 분석용) |

---

# 저장소 종류

| storage_type | 설명 |
|--------------|------|
| MINIO | 기본 저장소. 객체 저장소(S3 호환)에 파일을 저장 |
| LOCAL | 로컬 파일시스템에 파일을 저장 (소규모 배포/개발 환경 등에서 선택 가능) |

`cultivation_photo`/`profile_image` 테이블의 `storage_type` 컬럼으로 사진마다 저장
위치를 구분합니다. 같은 시스템 안에서도 사진마다 다른 저장소를 쓸 수 있습니다(예:
마이그레이션 과도기). AI DB의 `growth_record`는 더 이상 `storage_type`을 직접 갖지
않고, `cultivation_photo_id`로 `cultivation_photo`를 소프트 참조합니다.

---

# Bucket (MinIO)

```
mushroom-photos
```

---

# object_key 구조

저장소에 무관하게 파일을 가리키는 상대 경로입니다.

```
{cultivationId}/{uploadedAt}.jpg
```

예시

```
3/20260815-090000.jpg
```

`storage_type = MINIO`이면 이 값은 버킷 안의 Object Key로, `storage_type = LOCAL`이면
기준 디렉터리 하위의 상대 경로로 해석됩니다.

---

# 메타데이터

실제 이미지 파일은 저장소(MinIO 또는 로컬)에 저장하고, 아래 메타데이터는 Cultivation DB의
`cultivation_photo` 테이블에서 관리합니다.

| Field | 설명 |
|--------|------|
| object_key | 저장소 내 상대 경로 |
| storage_type | 저장 위치 (MINIO/LOCAL) |
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

설정된 저장소(MinIO 또는 로컬)에 파일 저장

↓

object_key 발급

↓

PostgreSQL에 cultivation_photo 메타데이터 저장 (object_key, storage_type)

↓

AI Service에 분석 요청 (object_key, storage_type 전달)

---

# 조회 흐름

Cultivation Service

↓

AI Service

↓

전달받은 (object_key, storage_type)으로 실제 접근 경로를 조합해 이미지 조회

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

저장소(MinIO 또는 로컬) 장애 발생 시

- 사진 업로드 불가
- AI Vision 분석(생육 분석) 기능 일시 이용 불가
- 기존에 캐시된 생육 분석 결과(Redis)는 조회 가능

---

# 고려 사항

- 관계형 데이터는 저장소에 저장하지 않으며, 이미지 파일만 저장합니다.
- 사진 메타데이터(object_key, storage_type, 업로드 시각)는 Cultivation DB에서 관리하고,
  저장소는 파일 저장 역할만 합니다.
- 완성된 URL이 아니라 저장소 중립적인 `object_key`를 저장하는 이유는, 저장소를 전환해도
  기존 데이터를 다시 쓸 필요가 없도록 하기 위함입니다. 실제 URL/경로 조합은 조회 시점에
  서비스 설정을 기준으로 이루어집니다.
- 여러 장의 사진이 등록된 경우 AI Service는 가장 최근 업로드본을 사용합니다.
- PostgreSQL과 이미지 데이터를 중복 저장하지 않습니다.
- 카메라 센서가 아닌 사용자가 직접 촬영하여 업로드하는 방식입니다.

---

# 추후 개발 예정

- 사진 업로드 전 압축/리사이징
- Presigned URL 기반 클라이언트 직접 업로드
- 오래된 사진 아카이빙 정책
- 재촬영 이력(사진 버전) 관리
- storage_type별 저장소 이관(마이그레이션) 도구
