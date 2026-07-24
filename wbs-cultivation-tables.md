# WBS — Cultivation DB 담당 테이블 (cultivation_member / cultivation / harvest / cultivation_photo)

완료: ERD 설계, Entity 클래스 작성
기간: 2026-07-23(목) ~ 2026-09-11(금)
Repo: `nhnacademy-aiot3-yes-ai-do/Cultivation_server`

순서: **API 명세서 → (기능별) 구현 → 테스트**. 1~3일 단위로 잘랐고, 09/07 ~ 09/11은 여유(버퍼) 기간입니다.

## 일정표

| # | 이슈 | 기간 | 일수 |
|---|---|---|---|
| 0 | API 명세서 작성 | 07/23(목) ~ 07/24(금) | 2일 |
| 1a | 재배 생성 API — 구현 | 07/27(월) ~ 07/28(화) | 2일 |
| 1b | 재배 생성 API — 테스트 | 07/29(수) | 1일 |
| 2a | 재배 조회/이력 API — 구현 | 07/30(목) ~ 07/31(금) | 2일 |
| 2b | 재배 조회/이력 API — 테스트 | 08/03(월) | 1일 |
| 3a | 재배 종료 API — 구현 | 08/04(화) | 1일 |
| 3b | 재배 종료 API — 테스트 | 08/05(수) | 1일 |
| 4a | 재배 멤버 초대/관리 API — 구현 | 08/06(목) ~ 08/07(금) | 2일 |
| 4b | 재배 멤버 초대/관리 API — 테스트 | 08/10(월) | 1일 |
| 5a | 재배 멤버 목록/탈퇴 API — 구현 | 08/11(화) ~ 08/12(수) | 2일 |
| 5b | 재배 멤버 목록/탈퇴 API — 테스트 | 08/13(목) | 1일 |
| 6a | 수확 기록 저장 API — 구현 | 08/14(금) ~ 08/17(월) | 2일 |
| 6b | 수확 기록 저장 API — 테스트 | 08/18(화) | 1일 |
| 7a | 수확 이력/통계 조회 API — 구현 | 08/19(수) ~ 08/20(목) | 2일 |
| 7b | 수확 이력/통계 조회 API — 테스트 | 08/21(금) | 1일 |
| 8a | 생육 사진 업로드 API — 구현 | 08/24(월) ~ 08/26(수) | 3일 |
| 8b | 생육 사진 업로드 API — 테스트 | 08/27(목) | 1일 |
| 9a | 생육 사진 조회/삭제 API — 구현 | 08/28(금) ~ 08/31(월) | 2일 |
| 9b | 생육 사진 조회/삭제 API — 테스트 | 09/01(화) | 1일 |
| 10 | 공통 마무리 (Swagger/통합테스트/PR) | 09/02(수) ~ 09/04(금) | 3일 |
| - | 버퍼(리뷰 반영/QA) | 09/07(월) ~ 09/11(금) | 5일 |

---

## 0. API 명세서 작성

`기간: 07/23(목) ~ 07/24(금)`

### 어떤 기능인가요?
> cultivation_member/cultivation/harvest/cultivation_photo 4개 테이블 관련 전체 엔드포인트의 API 명세서를 작성한다. 이후 모든 구현 작업은 이 명세서를 기준으로 진행한다.

### 작업 상세 내용
- [ ] 엔드포인트 목록 정의 (Method/URL) — 재배/멤버/수확/사진 전체
- [ ] Request/Response 스펙(필드/타입) 정의
- [ ] 에러 코드/예외 응답 정의
- [ ] Swagger(OpenAPI) 초안 작성 또는 문서로 정리

---

## 1a. 재배 생성 API — 구현

`기간: 07/27(월) ~ 07/28(화)`

### 어떤 기능인가요?
> 사용자가 새로운 버섯 재배를 생성한다. 재배 이름, 버섯 종류, 센서 장치(선택)를 입력받아 저장한다.

### 작업 상세 내용
- [ ] CultivationRepository 작성
- [ ] CultivationService — 생성 로직 구현
- [ ] CultivationController — POST API 작성 (명세서 기준)
- [ ] Request/Response DTO 작성
- [ ] 예외 처리 (버섯 종류 유효성 등)

---

## 1b. 재배 생성 API — 테스트

`기간: 07/29(수)`

### 어떤 기능인가요?
> 재배 생성 API의 정상/예외 케이스를 검증한다.

### 작업 상세 내용
- [ ] CultivationService 단위 테스트
- [ ] CultivationController 단위 테스트
- [ ] 예외 케이스 테스트 (잘못된 버섯 종류 등)

---

## 2a. 재배 조회/이력 API — 구현

`기간: 07/30(목) ~ 07/31(금)`

### 어떤 기능인가요?
> 사용자의 재배 목록/상세를 조회하고, 종료된 재배의 이력을 조회한다.

### 작업 상세 내용
- [ ] CultivationRepository — 목록/상세 조회 쿼리
- [ ] CultivationService — 조회 로직 구현
- [ ] CultivationController — GET API 작성 (목록/상세/이력, 명세서 기준)
- [ ] Response DTO 작성
- [ ] 페이징 처리

---

## 2b. 재배 조회/이력 API — 테스트

`기간: 08/03(월)`

### 어떤 기능인가요?
> 재배 조회/이력 API의 정상/예외 케이스를 검증한다.

### 작업 상세 내용
- [ ] CultivationService 단위 테스트
- [ ] CultivationController 단위 테스트
- [ ] 페이징 경계값 테스트

---

## 3a. 재배 종료 API — 구현

`기간: 08/04(화)`

### 어떤 기능인가요?
> 사용자가 진행 중인 재배를 종료한다. 수확 기록과는 별개 동작이며 수확 정보를 받지 않는다.

### 작업 상세 내용
- [ ] CultivationService — 종료 로직 구현 (상태 전환 CREATED/RUNNING → COMPLETED)
- [ ] CultivationController — PATCH API 작성 (명세서 기준)
- [ ] 이미 종료된 재배에 대한 예외 처리

---

## 3b. 재배 종료 API — 테스트

`기간: 08/05(수)`

### 어떤 기능인가요?
> 재배 종료 API의 정상/예외 케이스를 검증한다.

### 작업 상세 내용
- [ ] 상태 전환 단위 테스트
- [ ] 이미 종료된 재배 재종료 시 예외 테스트

---

## 4a. 재배 멤버(cultivation_member) 초대/관리 API — 구현

`기간: 08/06(목) ~ 08/07(금)`

### 어떤 기능인가요?
> 하나의 재배에 여러 사용자가 참여할 수 있도록 멤버를 초대하고 권한(OWNER/MEMBER)을 관리한다.

### 작업 상세 내용
- [ ] CultivationMemberRepository 작성
- [ ] CultivationMemberService — 초대/권한 부여 로직 구현
- [ ] CultivationMemberController — 초대/권한변경 API 작성 (명세서 기준)
- [ ] Request/Response DTO 작성
- [ ] 권한 검증 로직 (OWNER만 가능한 동작 구분)

---

## 4b. 재배 멤버 초대/관리 API — 테스트

`기간: 08/10(월)`

### 어떤 기능인가요?
> 멤버 초대/권한 관리 API의 정상/예외 케이스를 검증한다.

### 작업 상세 내용
- [ ] CultivationMemberService 단위 테스트
- [ ] 권한 검증(OWNER 아닌 사용자의 권한변경 시도) 예외 테스트

---

## 5a. 재배 멤버 목록/탈퇴 API — 구현

`기간: 08/11(화) ~ 08/12(수)`

### 어떤 기능인가요?
> 특정 재배에 속한 멤버 목록을 조회하고, 멤버 탈퇴(또는 추방)를 처리한다.

### 작업 상세 내용
- [ ] CultivationMemberRepository — 목록 조회 쿼리
- [ ] CultivationMemberService — 탈퇴/추방 로직 구현
- [ ] CultivationMemberController — GET/DELETE API 작성 (명세서 기준)
- [ ] 마지막 OWNER 탈퇴 시 예외 처리

---

## 5b. 재배 멤버 목록/탈퇴 API — 테스트

`기간: 08/13(목)`

### 어떤 기능인가요?
> 멤버 목록/탈퇴 API의 정상/예외 케이스를 검증한다.

### 작업 상세 내용
- [ ] CultivationMemberService 단위 테스트
- [ ] 마지막 OWNER 탈퇴 시도 예외 테스트

---

## 6a. 수확(Harvest) 기록 저장 API — 구현

`기간: 08/14(금) ~ 08/17(월)`

### 어떤 기능인가요?
> 재배 진행 중 반복적으로 실제 수확량과 메모를 저장한다(1차/2차/3차 등 여러 번 기록 가능).

### 작업 상세 내용
- [ ] HarvestRepository 작성
- [ ] HarvestService — 기록 저장 로직 구현
- [ ] HarvestController — POST API 작성 (명세서 기준)
- [ ] Request/Response DTO 작성
- [ ] 재배 상태(종료 여부) 검증 로직

---

## 6b. 수확 기록 저장 API — 테스트

`기간: 08/18(화)`

### 어떤 기능인가요?
> 수확 기록 저장 API의 정상/예외 케이스를 검증한다.

### 작업 상세 내용
- [ ] HarvestService 단위 테스트
- [ ] 종료된 재배에 수확 기록 시도 예외 테스트

---

## 7a. 수확 이력/통계 조회 API — 구현

`기간: 08/19(수) ~ 08/20(목)`

### 어떤 기능인가요?
> 재배별 개별 수확 내역과 총 수확 횟수/총 수확량을 조회한다.

### 작업 상세 내용
- [ ] HarvestRepository — 이력/집계 쿼리
- [ ] HarvestService — 통계 계산 로직 구현
- [ ] HarvestController — GET API 작성 (개별/통계, 명세서 기준)
- [ ] Response DTO 작성

---

## 7b. 수확 이력/통계 조회 API — 테스트

`기간: 08/21(금)`

### 어떤 기능인가요?
> 수확 이력/통계 조회 API의 정상/예외 케이스를 검증한다.

### 작업 상세 내용
- [ ] HarvestService 단위 테스트 (통계 계산 정확성 포함)
- [ ] 수확 기록이 없는 재배 조회 시 처리 테스트

---

## 8a. 생육 사진(cultivation_photo) 업로드 API — 구현

`기간: 08/24(월) ~ 08/26(수)`

### 어떤 기능인가요?
> 사용자가 직접 촬영한 생육 사진을 업로드하고 Photo Storage(MinIO/Local)에 저장한다. object_key + storage_type 방식으로 저장 위치를 추상화한다.

### 작업 상세 내용
- [ ] CultivationPhotoRepository 작성
- [ ] Storage 연동 (storage-common 라이브러리 적용)
- [ ] CultivationPhotoService — 업로드 로직 구현 (object_key 생성)
- [ ] CultivationPhotoController — POST API 작성 (명세서 기준)
- [ ] Request/Response DTO 작성
- [ ] 파일 형식/용량 검증 예외 처리

---

## 8b. 생육 사진 업로드 API — 테스트

`기간: 08/27(목)`

### 어떤 기능인가요?
> 생육 사진 업로드 API의 정상/예외 케이스를 검증한다.

### 작업 상세 내용
- [ ] CultivationPhotoService 단위 테스트
- [ ] 잘못된 파일 형식/용량 초과 예외 테스트
- [ ] Storage 연동 mock 테스트

---

## 9a. 생육 사진 조회/삭제 API — 구현

`기간: 08/28(금) ~ 08/31(월)`

### 어떤 기능인가요?
> 재배별 생육 사진 목록/상세를 조회하고, 필요 시 삭제한다. 조회 시점에 object_key로 실제 접근 경로를 계산한다.

### 작업 상세 내용
- [ ] CultivationPhotoRepository — 목록/상세 조회 쿼리
- [ ] CultivationPhotoService — 조회 시 URL 계산 로직, 삭제 로직 구현
- [ ] CultivationPhotoController — GET/DELETE API 작성 (명세서 기준)
- [ ] Response DTO 작성

---

## 9b. 생육 사진 조회/삭제 API — 테스트

`기간: 09/01(화)`

### 어떤 기능인가요?
> 생육 사진 조회/삭제 API의 정상/예외 케이스를 검증한다.

### 작업 상세 내용
- [ ] CultivationPhotoService 단위 테스트
- [ ] 존재하지 않는 사진 삭제 시도 예외 테스트

---

## 10. 공통 마무리

`기간: 09/02(수) ~ 09/04(금)`

### 어떤 기능인가요?
> 4개 테이블 관련 API 전체에 대한 통합 점검 및 문서화.

### 작업 상세 내용
- [ ] API 명세서와 실제 구현 일치 여부 최종 확인
- [ ] Swagger(OpenAPI) 문서화
- [ ] 통합 테스트 (Service 레이어 간 흐름 점검)
- [ ] 코드 리뷰 반영
- [ ] PR 작성 및 병합

---

## 버퍼 기간

`기간: 09/07(월) ~ 09/11(금)`

리뷰 피드백 반영, 예상치 못한 이슈 대응, 최종 QA를 위한 여유 기간.
