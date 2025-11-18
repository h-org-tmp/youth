# 청년정책 올인원 – AI Youth Policy Assistant (MVP)

자연어로 현재 상황을 입력하면,  
**교통비(청년 버스카드 등) / 어학 시험 응시료(예: 토익 지원금) / 주거비(월세·보증금 지원)** 같은  
청년 정책을 찾아 **추천 + 요약 + 신청서 초안**까지 도와주는 **AI 기반 정책 어시스턴트**의 MVP입니다.

이 README는 이 서비스를 실제로 구현하기 위해 필요한 **기술적인 내용(MVP 기준)**만 정리합니다.

---

## 1. MVP 기능 범위

### 1.1 핵심 사용자 플로우

1. **상황 입력**
   - React 기반 챗/입력창에서 자유롭게 입력
   - 예시
     - “부산 23살, 통학 버스비랑 자취방 보증금이 너무 부담돼요. 청년 버스카드나 주거비 지원제도가 있는지 알고 싶어요.”
     - “대구 25살, 야간 알바하면서 토익 준비 중인데, 시험 응시료나 교재비를 지원해 주는 정책이 있을까요?”
     - “인천 29살, 1인 가구인데 월세 지원이 있는지, 청년 전용 전세자금 대출 같은 정책도 알고 싶어요.”

2. **AI 구조화 & 요약 카드**
   - 입력 문장을 OpenAI Chat API로 분석
   - 나이, 지역, 고용 상태, 관심·고민 카테고리(교통, 교육/어학, 주거, 저축, 빚·채무 등)를 JSON으로 파싱
   - 프론트에서 아래 형태의 **자동 요약 카드 + 미니 폼**으로 표시:
     - (AI) 지금 이렇게 이해했어요
       - 나이: 23세  
       - 지역: 부산  
       - 상태: 대학생, 알바 중  
       - 주요 고민: 통학 버스비, 자취방 보증금, 청년 버스카드/주거비 지원  
     - [수정/추가하기] 버튼 → 사용자가 직접 보정

3. **정책 검색 및 추천**
   - 공공 청년 정책 OPEN API(청년 정책, 교통카드, 어학 시험 지원, 주거비 지원 등)로 가져온 정책 목록을 DB에 적재
   - 정책 설명/대상/혜택 텍스트를 OpenAI Embedding API로 벡터화
   - 사용자의 상황 텍스트를 임베딩 후, **코사인 유사도 + 조건 필터(나이/지역 등)**로 정책 Top-N 추천
   - 예: 청년 버스카드, 통학 교통비 지원, 어학 시험 응시료/교재비 지원, 청년 월세·보증금 지원, 청년 저축 장려 정책 등

4. **정책 설명 요약 & Q&A**
   - 선택한 정책의 상세 공고문을 RAG 형태로 요약
   - “누가 / 무엇을 / 얼마 동안 / 어떻게 신청하는지” 기준으로 정제된 요약 제공
   - 사용자가 추가 질문(“이 버스카드랑 월세 지원을 동시에 받을 수 있나요?”)을 하면, 정책 텍스트를 근거로 Q&A 응답

5. **신청서 텍스트 초안 생성 (문장 중심)**
   - 정책별 신청서에서 자주 등장하는 항목(신청 사유, 사용 계획 등)을 템플릿으로 정의
   - OpenAI Chat API로 400~800자 수준의 한국어 신청 사유/활동 계획 초안 생성
   - 사용자가 수정/복사하여 실제 온라인 신청서에 붙여넣을 수 있도록 제공

---

## 2. 기술 스택

### 2.1 Backend

- **언어 / 런타임**
  - Node.js 20+
  - TypeScript (권장, JS도 가능)

- **웹 프레임워크**
  - Express 4+

- **데이터베이스**
  - PostgreSQL 15+
  - (옵션) `pgvector` 확장 – 정책 텍스트 임베딩 벡터 저장 및 유사도 검색

- **ORM / Query**
  - Prisma 또는 TypeORM 중 택 1

- **AI / Embedding**
  - OpenAI API
    - Chat 모델: gpt-4.x 계열 (상황 파싱, 정책 요약, 신청서 초안)
    - Embedding 모델: `text-embedding-3-small` (정책/사용자 문장 임베딩)

- **외부 OPEN API**
  - 청년 정책 관련 공공 Open API
    - 예: 청년 교통비(버스카드, 대중교통 할인), 어학 시험 응시료 지원, 청년 월세/보증금 지원, 청년 저축지원 등
    - REST 기반 JSON/XML 응답
    - API Key 인증 방식

- **네이버 CLOVA (선택 기능)**
  - CLOVA OCR API
    - 정책 안내문이 이미지/PDF인 경우 텍스트 추출
    - 추출 텍스트를 Embedding / 요약 / Q&A에 활용

- **기타**
  - HTTP Client: `axios` 또는 `node-fetch`
  - 환경 변수 관리: `dotenv`
  - 로깅: `morgan` + 콘솔 로그
  - 개발용 재시작: `nodemon`

### 2.2 Frontend

- **언어 / 프레임워크**
  - React 18+
  - TypeScript (권장)

- **빌드**
  - Vite

- **상태관리 / 데이터 패칭**
  - React Query (TanStack Query) – API 호출 및 캐싱

- **UI / 스타일**
  - Tailwind CSS 또는 MUI / Chakra UI 중 택 1

- **폼**
  - `react-hook-form` – AI 요약 카드 수정/추가 미니 폼 구성

- **HTTP**
  - `fetch` 기반 커스텀 훅 또는 `axios`

---

## 3. 아키텍처 개요

### 3.1 시스템 구조 (논리)

```text
사용자 브라우저
   └─ React (Chat UI, 요약 카드, 정책 카드, 신청서 에디터)
        └─ REST API 호출
             └─ Node.js + Express (Backend)
                  ├─ OpenAI API (Chat + Embedding)
                  ├─ 청년 정책 OPEN API (정책 메타데이터 수집)
                  ├─ 네이버 CLOVA OCR (정책 이미지 → 텍스트, 선택)
                  └─ PostgreSQL (+ pgvector) – 정책/임베딩/사용자 질의 저장

3.2 디렉터리 구조 (예시)
.
├─ backend/
│  ├─ src/
│  │  ├─ app.ts               # Express 앱 초기화
│  │  ├─ routes/
│  │  │  ├─ health.route.ts
│  │  │  ├─ chat.route.ts     # 상황 입력 분석
│  │  │  ├─ policy.route.ts   # 정책 추천/검색/Q&A
│  │  │  ├─ ingest.route.ts   # OPEN API → 정책 DB 적재
│  │  │  └─ draft.route.ts    # 신청서 초안 생성
│  │  ├─ services/
│  │  │  ├─ openai.service.ts
│  │  │  ├─ embedding.service.ts
│  │  │  ├─ youth-openapi.service.ts
│  │  │  ├─ clova-ocr.service.ts
│  │  │  └─ scoring.service.ts
│  │  ├─ repositories/
│  │  ├─ db/
│  │  └─ types/
│  └─ prisma/ or ormconfig.ts
└─ frontend/
   ├─ src/
   │  ├─ pages/
   │  ├─ components/
   │  │  ├─ ChatInput.tsx
   │  │  ├─ AiSummaryCard.tsx
   │  │  ├─ PolicyCard.tsx
   │  │  ├─ DraftEditor.tsx
   │  │  └─ TimelineBoard.tsx
   │  ├─ api/
   │  └─ hooks/
   └─ vite.config.ts

```


# 4. 데이터 모델 (초안)

## 4.1 주요 테이블

### user_queries – 사용자 상황 입력 로그

- **id** (UUID)
- **raw_text** (TEXT)
  - 사용자가 입력한 문장
- **parsed_profile_json** (JSONB)
  - LLM이 분석한 나이/지역/상태/관심 카테고리 등
- **created_at** (TIMESTAMPTZ)

### policies – 정책 메타데이터

- **id** (PK)
- **external_id**
  - Open API 정책 ID
- **title** – 정책명
  - 예시:
    - 청년 버스카드 교통비 할인
    - 청년 어학시험 응시료 지원
    - 청년 월세·보증금 지원 등
- **summary** – 요약(내부 생성)
- **category** – 예시:
  - `["교통"]`
  - `["교육/어학"]`
  - `["주거"]`
  - `["저축/자산형성"]`
- **target_age_min**, **target_age_max**
- **target_region_code**
- **income_condition** – 소득 기준
  - 예: 중위소득 n% 이하
- **employment_condition** – 재학/재직/구직 등 조건
- **benefit_type** – 예시:
  - 교통비 지원
  - 응시료/교육비 지원
  - 월세/보증금 지원
- **application_start_date**, **application_end_date**
- **detail_url** – 상세 공고문 링크
- **raw_text** – 공고문 전체 또는 주요 본문

### policy_embeddings

- **id**
- **policy_id** (FK → policies.id)
- **chunk_index**
- **embedding**
  - vector(1536) (pgvector) 또는 JSON 배열
- **created_at**

### policy_recommendations

- **id**
- **user_query_id** (FK → user_queries.id)
- **policy_id** (FK → policies.id)
- **score_similarity** – 임베딩 유사도 점수
- **score_rule_based** – 나이/지역/카테고리 매칭 점수
- **score_total**
- **created_at**

### (선택) drafts – 신청서 초안

- **id**
- **user_query_id** 또는 (user_id + policy_id)
- **draft_type** – reason / plan 등
- **content** – 신청서 문단 초안
- **created_at**, **updated_at**

---

# 5. 주요 API 설계 (MVP 기준)

## 5.1 상황 입력 → AI 요약 카드

### `POST /api/chat/analyze`

#### Request
```json
{
  "message": "부산 23살, 학교 다니면서 알바하는데 통학 버스비랑 자취방 보증금이 너무 부담돼요."
}
```

#### 처리 흐름

1. OpenAI Chat API 호출
2. 프롬프트에서 다음 스키마로 파싱 지시:

```typescript
type ParsedProfile = {
  age: number | null;
  region: string | null;
  employmentStatus?: string | null;
  educationLevel?: string | null;
  concerns: string[]; // 예: ["교통비", "주거비(보증금)"]
};
```

3. 결과를 `user_queries`에 저장

#### Response
```json
{
  "parsedProfile": {
    "age": 23,
    "region": "부산",
    "employmentStatus": "대학생, 알바 중",
    "concerns": ["교통비", "주거비(보증금)"]
  }
}
```

프론트는 이 응답으로 **(AI) 지금 이렇게 이해했어요** 요약 카드와 **[수정/추가하기]** 미니 폼을 렌더링한다.

---

## 5.2 정책 데이터 인입 (OPEN API)

### `POST /api/ingest/policies`

#### 역할
- 공공 청년 정책 OPEN API 호출
- 청년 버스카드(교통비), 어학시험 응시료 지원, 월세·보증금 지원, 청년 저축지원 같은 정책을 `policies` 테이블에 upsert

#### 처리 흐름
1. 여러 페이지에 걸친 정책 목록 GET
2. 개별 정책 상세 정보 GET
3. DB에 저장 (없으면 insert, 있으면 update)

---

## 5.3 정책 텍스트 임베딩

### `POST /api/ingest/policies/embed`

#### 역할
- `policies.raw_text` 또는 `title + summary`를 chunk로 나누고
- OpenAI Embedding API (`text-embedding-3-small`)로 벡터를 만든 뒤,
- `policy_embeddings` 테이블에 저장

#### 처리 흐름
1. 아직 임베딩이 없는 정책들 조회
2. OpenAI Embedding API 호출
3. `policy_id`, `chunk_index`, `embedding` 저장

---

## 5.4 정책 추천

### `POST /api/policies/recommend`

#### Request
```json
{
  "message": "대구 25살, 야간 알바하면서 토익 준비 중인데, 시험 응시료나 교재비를 지원해 주는 정책이 있을까요?",
  "parsedProfile": {
    "age": 25,
    "region": "대구",
    "employmentStatus": "알바 중",
    "concerns": ["토익 응시료", "어학시험", "교육비"]
  }
}
```

#### 처리 흐름
1. `message + parsedProfile`을 하나의 텍스트로 합쳐 Embedding 생성
2. pgvector의 `cosine_distance` 등을 사용해 `policy_embeddings`에서 Top-N 검색
3. 나이/지역/카테고리(교육/어학, 교통, 주거 등) 매칭으로 룰 기반 점수 추가
4. 상위 K개 정책 리턴

#### Response (예시)
```json
{
  "recommendations": [
    {
      "policyId": 101,
      "title": "청년 어학시험 응시료 지원 사업",
      "summary": "만 19~34세 청년의 토익·토플 등 공인어학시험 응시료 일부를 지원합니다.",
      "category": ["교육/어학"],
      "benefitType": "응시료 지원",
      "score": 0.89,
      "applicationEndDate": "2025-06-30"
    },
    {
      "policyId": 202,
      "title": "청년 직무역량 강화 바우처",
      "summary": "직무·어학 교육비를 바우처 형태로 지원합니다.",
      "category": ["교육/어학"],
      "benefitType": "교육비/교재비 지원",
      "score": 0.82
    }
  ]
}
```

---

## 5.5 정책 Q&A

### `POST /api/policies/:id/qa`

#### Request
```json
{
  "question": "이 토익 응시료 지원은 1년에 몇 번까지 신청할 수 있나요?"
}
```

#### 처리 흐름
1. `policies.raw_text` + 관련 텍스트를 RAG 컨텍스트로 구성
2. OpenAI Chat API 호출 시, 시스템 프롬프트에 아래 원칙 적용:
   - 공고문에 근거가 있는 내용만 답변
   - 없으면 **"공고문에 명시되어 있지 않다"**고 명확히 말하기

#### Response (예시)
```json
{
  "answer": "공고문에 따르면 연 2회까지 응시료 지원이 가능합니다."
}
```

---

## 5.6 신청서 초안 생성

### `POST /api/drafts/generate`

#### Request
```json
{
  "policyId": 101,
  "profile": {
    "age": 25,
    "region": "대구",
    "employmentStatus": "편의점 야간 알바",
    "goal": "어학 점수 향상을 통한 사무직 취업"
  }
}
```

#### 처리 흐름
1. 정책 텍스트 + profile을 기반으로 OpenAI Chat API 호출
2. "지원 동기", "활용 계획"을 각각 400~800자 / 300~600자 정도 길이로 생성
3. `drafts` 테이블에 저장 후, 프론트로 반환

#### Response (예시)
```json
{
  "policyId": 101,
  "drafts": {
    "reason": "저는 현재 대구에서 야간 알바를 하며 어학 공부를 병행 중인 25세 청년입니다...",
    "plan": "지원받은 응시료를 활용해 정기적인 모의시험과 실전 시험에 응시하고..."
  }
}
```

---

# 6. Frontend 주요 화면

## 6.1 Chat / Home

- **상단**: 서비스 간단 소개
  - 예: 청년 버스카드·어학시험·월세 지원 등
- **중앙**: 상황 입력창 + 대화 영역
  - 입력 후 `/api/chat/analyze` 호출 → AI 요약 카드 표시

## 6.2 AI 요약 카드 + 수정 폼

**카드 내용:**
- 나이 / 지역 / 상태 / 주요 고민(버스비, 월세, 응시료 등)
- **[수정/추가하기]** 버튼 클릭 시 모달 폼 오픈
  - 브라우저 상태 또는 `/api/profile/update`로 저장

## 6.3 정책 추천 리스트

- `/api/policies/recommend` 결과를 정책 카드 리스트로 렌더링

**예시 카드:**
- 청년 버스카드 교통비 지원
- 청년 어학시험 응시료 지원
- 청년 월세 지원 사업

**각 카드에:**
- 자세히 보기
- 신청서 초안 만들기 버튼

## 6.4 정책 상세 + Q&A

- 정책 요약 / 상세 / 마감일 표시
- **"질문하기"** 입력창 → `/api/policies/:id/qa` 결과를 채팅 버블 형식으로 렌더링

## 6.5 신청서 초안 에디터

- LLM이 생성한 지원 동기/활용 계획 문단을 에디터에 표시
- 사용자가 직접 수정 가능
- **"복사하기"** 버튼으로 클립보드 복사

---

# 7. 환경 변수 (.env 예시)

```bash
# 공통
NODE_ENV=development

# Backend
PORT=4000

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/youth_policy

# OpenAI
OPENAI_API_KEY=sk-...

# 청년 정책 Open API
YOUTH_POLICY_API_KEY=...
YOUTH_POLICY_API_BASE=https://...

# Naver CLOVA OCR (선택)
NCLOUD_ACCESS_KEY=...
NCLOUD_SECRET_KEY=...
NCLOUD_OCR_ENDPOINT=https://.../ocr/general
```

---

# 8. 로컬 개발 실행

## 8.1 Backend

```bash
cd backend
npm install

# (Prisma 사용 시)
npx prisma migrate dev

npm run dev
```

## 8.2 Frontend

```bash
cd frontend
npm install
npm run dev
```

- **프론트 dev 서버**: http://localhost:5173
- **백엔드 API 서버**: http://localhost:4000

---

# 9. 라이선스 & 주의사항

- 사용되는 공공 데이터 / OPEN API는 각 제공 기관의 이용 약관 및 표기 의무를 반드시 준수해야 한다.
- 이 레포지토리의 소스 코드 라이선스(MIT 등)는 팀에서 별도로 지정한다.
