# 당낭콩 (DNC)

CGM 연속 혈당 데이터와 식단 사진 분석을 결합해 식후 혈당 변화를 예측하고, 사용자의 당뇨 유형과 건강 정보를 바탕으로 식단 코칭을 제공하는 모바일 헬스케어 서비스입니다.

> 본 프로젝트는 포트폴리오 및 학습 목적의 서비스입니다. 의료 진단 또는 치료 목적의 판단 도구로 사용할 수 없습니다.

## 프로젝트 개요

당뇨 관리에서 중요한 것은 식사, 운동, 혈당 변화를 한 흐름으로 이해하는 것입니다. 당낭콩은 Dexcom CGM 데이터를 주기적으로 수집하고, 사용자가 촬영한 음식 이미지를 AI 서버로 분석해 영양 정보와 식후 혈당 예측 그래프를 제공합니다. 이후 Gemini 기반 코칭 문구와 일별/센서별 리포트를 통해 사용자가 자신의 혈당 패턴을 확인할 수 있도록 구성했습니다.

## 주요 기능

| 구분 | 기능 |
| --- | --- |
| 사용자 인증 | 자체 회원가입/로그인, JWT 인증, Google/Kakao/Naver 소셜 로그인 |
| CGM 연동 | Dexcom OAuth 연동, 토큰 갱신, 실시간/과거 혈당 데이터 수집 |
| 혈당 파이프라인 | 5분 주기 수집, Redis 버퍼링, MySQL 저장, 중복 데이터 방지, 센서 이력 관리 |
| 식단 분석 | 음식 이미지 업로드, AI 음식 탐지, 영양 성분 매핑, 식사 기록 저장/수정/삭제 |
| 혈당 예측 | 음식 영양 정보와 사용자 건강 정보를 기반으로 식후 혈당 변화 예측 |
| AI 코칭 | Gemini API를 활용한 개인 맞춤 식단 가이드 비동기 생성 |
| 리포트 | 평균/최대/최소 혈당, 표준편차, Time In Range, 센서별 리포트 |
| 알림 | 혈당 경고 설정, 사용자 알림 목록, FCM 푸시 토큰 관리 |

## 서비스 흐름

```text
사용자
  |
  v
Expo React Native App
  |
  v
Spring Boot API
  |-- Auth / User / Meal / Report / Alert
  |-- Dexcom OAuth -> CGM Scheduler -> Redis Buffer -> MySQL
  |-- FastAPI AI Server -> Food Detection / Glucose Prediction
  |-- Gemini API -> Personalized Guide
  |-- Firebase Cloud Messaging
  `-- S3 or Local Storage
```

## 기술 스택

| 영역 | 기술 |
| --- | --- |
| Mobile | React Native 0.81, Expo SDK 54, TypeScript, Expo Router, Expo Camera, Expo Notifications, React Native Chart Kit |
| Backend | Java 17, Spring Boot 3.5, Spring Security, JWT, Spring Data JPA, Spring Session Redis, Springdoc OpenAPI |
| AI Server | Python 3.11, FastAPI, Uvicorn, OpenCV, Ultralytics YOLOv8, PyTorch, NumPy, Pandas, SciPy |
| Database | MySQL 8.0, Redis |
| Infra | Docker, Docker Compose, Nginx, Jenkins |
| External API | Dexcom, Google/Kakao/Naver OAuth, Gemini API, Firebase Cloud Messaging, AWS S3 |

## 핵심 구현 포인트

### 1. CGM 데이터 수집 파이프라인

Dexcom API에서 수집한 EGV 데이터를 바로 DB에 저장하지 않고 사용자별 Redis List에 먼저 적재합니다. 이후 FIFO 방식으로 MySQL에 동기화해 API 수집과 DB 저장 책임을 분리했습니다.

- 5분 주기 실시간 수집
- 과거 혈당 데이터 30일 단위 분할 수집
- Dexcom access token 만료 시 refresh token 기반 재시도
- `dexcom_record_id` 기반 중복 저장 방지
- 센서 교체/재연결 감지 및 ACTIVE/PENDING/INACTIVE 상태 관리

관련 문서: [Dexcom CGM 데이터 파이프라인](./apps/docs/dexcom-cgm-pipeline.md)

### 2. 식단 이미지 분석과 혈당 예측

모바일 앱에서 업로드한 식단 이미지를 Spring Boot API가 수신하고, FastAPI AI 서버로 전달해 음식 탐지 결과를 받습니다. 탐지된 음식명은 영양 성분 메타데이터와 매칭되며, 사용자의 건강 정보와 함께 혈당 예측 모델 입력값으로 사용됩니다.

- 음식 탐지 결과: 음식명, confidence, bounding box
- 영양 정보: 칼로리, 탄수화물, 단백질, 지방, 당류, 나트륨
- 예측 결과: 식후 0분부터 120분까지의 혈당 변화
- 분석 이미지 저장: 로컬 파일 시스템 또는 S3 선택 가능

### 3. AI 코칭 비동기 처리

Gemini 기반 식단 코칭은 사용자 응답 지연을 줄이기 위해 별도 Executor에서 비동기로 생성합니다. 클라이언트는 `requestId`로 코칭 생성 상태를 조회하고, 서버는 완료/실패 상태를 관리합니다.

- 분석 응답과 AI 코칭 생성을 분리
- `PENDING`, `COMPLETED`, `FAILED` 상태 관리
- 오래된 요청 상태 TTL 정리
- 사용자별 건강 정보와 식단 정보를 조합한 프롬프트 생성

### 4. 개인화 리포트

혈당 데이터는 일별/주별/월별 리포트로 집계됩니다. 당뇨 유형에 따라 Time In Range 기준을 다르게 적용하고, 센서 사용 기간 기준으로 리포트를 확인할 수 있도록 구성했습니다.

- 평균, 최대, 최소 혈당
- 표준편차
- Time In Range 구간 비율
- 센서별 사용 이력 및 최신 리포트
- 일일 리포트 히스토리

## 프로젝트 구조

```text
.
├── apps
│   ├── ai-server          # FastAPI 기반 음식 탐지/혈당 예측 서버
│   ├── backend/dnc        # Spring Boot API 서버
│   ├── docs               # CGM 파이프라인 등 기술 문서
│   └── mobile             # Expo React Native 모바일 앱
├── exec                   # 배포 가이드, 외부 서비스, DB 덤프 문서
└── infra                  # Docker Compose, Nginx, Jenkins 설정
```

## 실행 방법

상세한 빌드 및 배포 절차는 [배포 가이드](./exec/DEPLOY_GUIDE.md)를 기준으로 확인할 수 있습니다.

### Backend

```bash
cd apps/backend/dnc
./gradlew bootRun
```

필수 환경 변수는 `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `JWT_SECRET`, `AI_SERVER_BASE_URL` 등입니다. OAuth, S3, FCM, Gemini 연동은 필요한 기능에 따라 추가 설정합니다.

### AI Server

```bash
cd apps/ai-server
pip install -r requirements.txt
uvicorn ai.main:app --host 0.0.0.0 --port 18000
```

음식 탐지 모델과 혈당 예측 모델 파일이 필요합니다. 모델 경로는 `YOLO_MODEL_PATH`, `RESNET_MODEL_PATH` 환경 변수로 지정할 수 있습니다.

### Mobile

```bash
cd apps/mobile
npm install
npm run android
```

앱에서 백엔드 API를 호출하려면 `EXPO_PUBLIC_API_BASE_URL`을 실행 환경에 맞게 설정해야 합니다.

## 문서

- [빌드 및 배포 가이드](./exec/DEPLOY_GUIDE.md)
- [외부 서비스 연동 정보](./exec/EXTERNAL_SERVICES.md)
- [DB 덤프 및 정보](./exec/DB_DUMP_INFO.md)
- [Dexcom CGM 데이터 파이프라인](./apps/docs/dexcom-cgm-pipeline.md)
- [시연 시나리오](./exec/당낭콩%20시연%20시나리오.pdf)

## 보안 메모

민감 정보는 환경 변수 또는 별도 `.env` 파일로 주입하는 것을 전제로 합니다. 개인 포트폴리오 저장소에 업로드할 때는 API Key, OAuth Secret, Firebase Service Account, DB 비밀번호, JWT Secret 등 운영 비밀값을 포함하지 않아야 합니다.
