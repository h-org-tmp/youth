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
     - "부산 23살, 통학 버스비랑 자취방 보증금이 너무 부담돼요. 청년 버스카드나 주거비 지원제도가 있는지 알고 싶어요."
     - "대구 25살, 야간 알바하면서 토익 준비 중인데, 시험 응시료나 교재비를 지원해 주는 정책이 있을까요?"
     - "인천 29살, 1인 가구인데 월세 지원이 있는지, 청년 전용 전세자금 대출 같은 정책도 알고 싶어요."

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
   - "누가 / 무엇을 / 얼마 동안 / 어떻게 신청하는지" 기준으로 정제된 요약 제공
   - 사용자가 추가 질문("이 버스카드랑 월세 지원을 동시에 받을 수 있나요?")을 하면, 정책 텍스트를 근거로 Q&A 응답

5. **📝 신청서 자동 작성 (핵심 기능)**
   - **정책별 신청서 양식 구조 파악 및 저장**
     - 각 정책의 신청서에 필요한 항목(지원동기, 활용계획, 가족사항, 경력사항 등)을 DB에 템플릿으로 관리
     - OCR로 PDF 신청서 양식 분석 또는 수동 등록
   - **사용자 상세 프로필 수집**
     - 신청서 작성에 필요한 정보(학력, 전공, 경력, 소득, 가족구성, 특기사항 등)를 대화형으로 수집
     - 한 번 입력한 정보는 재사용 가능하도록 저장
   - **고품질 신청서 초안 생성**
     - OpenAI Chat API + 정교한 프롬프트 엔지니어링
     - 정책 취지와 사용자 상황을 연결하는 논리적 서술
     - Few-shot learning으로 실제 합격 사례 참고
   - **다중 버전 생성 및 선택**
     - 2~3가지 버전의 신청서를 생성하여 사용자가 선택
     - 각 버전은 톤앤매너(진지함/열정적/실용적)를 다르게 구성
   - **신청서 품질 검증**
     - 필수 키워드 포함 여부 체크
     - 글자 수 제한 준수 확인
     - 정책 목적과의 연관성 점수 계산
   - **사용자 편집 및 저장**
     - 생성된 초안을 에디터에서 자유롭게 수정
     - 복사하기 버튼으로 실제 신청 사이트에 붙여넣기

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
    - Chat 모델: `gpt-4o` 또는 `gpt-4-turbo` (상황 파싱, 정책 요약, **신청서 초안**)
    - Embedding 모델: `text-embedding-3-small` (정책/사용자 문장 임베딩)

- **외부 OPEN API**
  - 청년 정책 관련 공공 Open API
    - 예: 청년 교통비(버스카드, 대중교통 할인), 어학 시험 응시료 지원, 청년 월세/보증금 지원, 청년 저축지원 등
    - REST 기반 JSON/XML 응답
    - API Key 인증 방식

- **네이버 CLOVA (필수 또는 선택)**
  - CLOVA OCR API
    - 정책 안내문이나 **신청서 양식이 이미지/PDF인 경우** 텍스트 추출
    - 추출 텍스트를 Embedding / 요약 / Q&A / **신청서 템플릿 구조 파악**에 활용

- **기타**
  - HTTP Client: `axios` 또는 `node-fetch`
  - 환경 변수 관리: `dotenv`
  - 로깅: `morgan` + 콘솔 로그
  - 개발용 재시작: `nodemon`
  - 텍스트 검증: 정규표현식 또는 간단한 NLP 라이브러리 (`natural`, `compromise`)

### 2.2 Frontend

- **언어 / 프레임워크**
  - React 18+
  - TypeScript (권장)

- **빌드**
  - Vite

- **상태관리 / 데이터 패칭**
  - React Query (TanStack Query) – API 호출 및 캐싱
  - Zustand 또는 Context API – 사용자 프로필 전역 상태 관리

- **UI / 스타일**
  - Tailwind CSS 또는 MUI / Chakra UI 중 택 1

- **폼**
  - `react-hook-form` – AI 요약 카드 수정/추가 미니 폼, 상세 프로필 입력 폼 구성

- **텍스트 에디터 (신청서 편집용)**
  - `react-quill` 또는 `draft-js` 또는 단순 `<textarea>` + 커스텀 스타일링
  - 복사하기 기능: Clipboard API 활용

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
                  │    ├─ 상황 파싱
                  │    ├─ 정책 요약 & Q&A
                  │    └─ **신청서 초안 생성 (프롬프트 엔지니어링)**
                  ├─ 청년 정책 OPEN API (정책 메타데이터 수집)
                  ├─ 네이버 CLOVA OCR (정책 이미지/신청서 양식 → 텍스트)
                  └─ PostgreSQL (+ pgvector)
                       ├─ 정책 & 임베딩
                       ├─ 사용자 질의 & 프로필
                       ├─ **신청서 템플릿 (application_form_templates)**
                       └─ **신청서 초안 & 버전 관리 (drafts)**
```

### 3.2 디렉터리 구조 (예시)

```text
.
├─ backend/
│  ├─ src/
│  │  ├─ app.ts               # Express 앱 초기화
│  │  ├─ routes/
│  │  │  ├─ health.route.ts
│  │  │  ├─ chat.route.ts     # 상황 입력 분석
│  │  │  ├─ policy.route.ts   # 정책 추천/검색/Q&A
│  │  │  ├─ ingest.route.ts   # OPEN API → 정책 DB 적재
│  │  │  ├─ draft.route.ts    # 신청서 초안 생성/버전 관리
│  │  │  └─ profile.route.ts  # 사용자 상세 프로필 관리 (신규)
│  │  ├─ services/
│  │  │  ├─ openai.service.ts
│  │  │  ├─ embedding.service.ts
│  │  │  ├─ youth-openapi.service.ts
│  │  │  ├─ clova-ocr.service.ts
│  │  │  ├─ scoring.service.ts
│  │  │  ├─ draft-generator.service.ts  # 신청서 생성 전용 서비스 (신규)
│  │  │  └─ prompt-engineering.service.ts  # 프롬프트 관리 (신규)
│  │  ├─ repositories/
│  │  ├─ db/
│  │  ├─ types/
│  │  └─ utils/
│  │     └─ text-validator.ts  # 신청서 품질 검증 유틸 (신규)
│  └─ prisma/ or ormconfig.ts
└─ frontend/
   ├─ src/
   │  ├─ pages/
   │  ├─ components/
   │  │  ├─ ChatInput.tsx
   │  │  ├─ AiSummaryCard.tsx
   │  │  ├─ PolicyCard.tsx
   │  │  ├─ DraftEditor.tsx     # 신청서 편집기
   │  │  ├─ DraftVersionSelector.tsx  # 버전 선택 (신규)
   │  │  ├─ ProfileForm.tsx     # 상세 프로필 입력 폼 (신규)
   │  │  └─ TimelineBoard.tsx
   │  ├─ api/
   │  └─ hooks/
   └─ vite.config.ts
```

---

## 4. 데이터 모델 (초안)

### 4.1 주요 테이블

#### `user_queries` – 사용자 상황 입력 로그

- **id** (UUID)
- **raw_text** (TEXT) – 사용자가 입력한 문장
- **parsed_profile_json** (JSONB) – LLM이 분석한 나이/지역/상태/관심 카테고리 등
- **created_at** (TIMESTAMPTZ)

#### `user_profiles` – 사용자 상세 프로필 (신청서 작성용) ✨ 신규

- **id** (PK)
- **user_query_id** (FK → user_queries.id) 또는 별도 user 테이블 연결
- **age** (INT)
- **region** (VARCHAR)
- **employment_status** (VARCHAR) – 예: 재학, 재직, 구직, 프리랜서 등
- **education_level** (VARCHAR) – 예: 고졸, 대학재학, 대졸 등
- **major** (VARCHAR) – 전공
- **work_experience** (TEXT) – 경력사항 (자유 서술)
- **household_income** (INT) – 가구 소득 (선택)
- **household_members** (INT) – 가구원 수
- **special_notes** (TEXT) – 특기사항, 장애 여부, 병역 등
- **goal** (TEXT) – 사용자의 목표/계획 (예: "토익 점수 향상으로 해외 취업")
- **created_at**, **updated_at** (TIMESTAMPTZ)

#### `policies` – 정책 메타데이터

- **id** (PK)
- **external_id** – Open API 정책 ID
- **title** – 정책명
  - 예: 청년 버스카드 교통비 할인, 청년 어학시험 응시료 지원, 청년 월세·보증금 지원 등
- **summary** – 요약 (내부 생성)
- **category** – 예: `["교통"]`, `["교육/어학"]`, `["주거"]`, `["저축/자산형성"]`
- **target_age_min**, **target_age_max**
- **target_region_code**
- **income_condition** – 소득 기준 (예: 중위소득 n% 이하)
- **employment_condition** – 재학/재직/구직 등 조건
- **benefit_type** – 예: 교통비 지원, 응시료/교육비 지원, 월세/보증금 지원
- **application_start_date**, **application_end_date**
- **application_url** – 실제 신청 사이트 URL ✨ 추가
- **detail_url** – 상세 공고문 링크
- **raw_text** – 공고문 전체 또는 주요 본문

#### `application_form_templates` – 정책별 신청서 양식 구조 ✨ 신규

- **id** (PK)
- **policy_id** (FK → policies.id)
- **form_fields** (JSONB) – 신청서 필드 정의
  - 예시 구조:
    ```json
    [
      {
        "fieldName": "지원동기",
        "fieldType": "textarea",
        "required": true,
        "minLength": 300,
        "maxLength": 800,
        "description": "본 사업에 지원하게 된 동기를 구체적으로 작성해주세요."
      },
      {
        "fieldName": "활용계획",
        "fieldType": "textarea",
        "required": true,
        "minLength": 200,
        "maxLength": 600,
        "description": "지원금을 어떻게 활용할 계획인지 작성해주세요."
      },
      {
        "fieldName": "경력사항",
        "fieldType": "textarea",
        "required": false,
        "maxLength": 500
      }
    ]
    ```
- **example_applications** (JSONB) – Few-shot learning용 합격 사례 (익명화) ✨ 선택사항
- **created_at**, **updated_at**

#### `policy_embeddings`

- **id**
- **policy_id** (FK → policies.id)
- **chunk_index**
- **embedding** – vector(1536) (pgvector) 또는 JSON 배열
- **created_at**

#### `policy_recommendations`

- **id**
- **user_query_id** (FK → user_queries.id)
- **policy_id** (FK → policies.id)
- **score_similarity** – 임베딩 유사도 점수
- **score_rule_based** – 나이/지역/카테고리 매칭 점수
- **score_total**
- **created_at**

#### `drafts` – 신청서 초안 및 버전 관리

- **id** (PK)
- **user_query_id** (FK → user_queries.id) 또는 user_profile_id
- **policy_id** (FK → policies.id)
- **version** (INT) – 같은 정책에 대해 여러 버전 생성 가능 ✨ 추가
- **field_name** (VARCHAR) – 예: "지원동기", "활용계획" 등
- **content** (TEXT) – 신청서 문단 초안
- **tone** (VARCHAR) – 예: "formal", "passionate", "practical" ✨ 추가
- **quality_score** (FLOAT) – 품질 점수 (선택) ✨ 추가
- **is_selected** (BOOLEAN) – 사용자가 선택한 버전인지 ✨ 추가
- **created_at**, **updated_at**

---

## 5. 주요 API 설계 (MVP 기준)

### 5.1 상황 입력 → AI 요약 카드

#### `POST /api/chat/analyze`

##### Request
```json
{
  "message": "부산 23살, 학교 다니면서 알바하는데 통학 버스비랑 자취방 보증금이 너무 부담돼요."
}
```

##### 처리 흐름

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

##### Response
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

### 5.2 사용자 상세 프로필 수집 ✨ 신규

#### `POST /api/profile/update`

##### Request
```json
{
  "userQueryId": "uuid-123",
  "profile": {
    "age": 25,
    "region": "대구",
    "employmentStatus": "편의점 야간 알바",
    "educationLevel": "대학 재학",
    "major": "경영학",
    "workExperience": "편의점 알바 1년 6개월, 카페 알바 6개월",
    "householdIncome": 3000000,
    "householdMembers": 3,
    "specialNotes": "특이사항 없음",
    "goal": "어학 점수 향상을 통한 사무직 취업"
  }
}
```

##### 처리 흐름
1. `user_profiles` 테이블에 upsert
2. 이후 신청서 생성 시 이 프로필 데이터 활용

##### Response
```json
{
  "success": true,
  "profileId": "uuid-456"
}
```

---

### 5.3 정책 데이터 인입 (OPEN API)

#### `POST /api/ingest/policies`

##### 역할
- 공공 청년 정책 OPEN API 호출
- 청년 버스카드(교통비), 어학시험 응시료 지원, 월세·보증금 지원, 청년 저축지원 같은 정책을 `policies` 테이블에 upsert

##### 처리 흐름
1. 여러 페이지에 걸친 정책 목록 GET
2. 개별 정책 상세 정보 GET
3. DB에 저장 (없으면 insert, 있으면 update)

---

### 5.4 신청서 양식 구조 파싱 및 저장 ✨ 신규

#### `POST /api/ingest/application-forms`

##### 역할
- 정책별 신청서 양식을 수집하고 구조화하여 `application_form_templates` 테이블에 저장
- 방법:
  1. **수동 등록**: 관리자가 각 정책 신청서를 보고 필드 정보를 직접 입력
  2. **OCR 기반 자동 파싱**: PDF/이미지 신청서를 CLOVA OCR로 텍스트 추출 후, GPT로 구조 파싱
  3. **HTML 파싱**: 온라인 신청서 form HTML을 크롤링하여 필드 추출

##### Request (수동 등록 예시)
```json
{
  "policyId": 101,
  "formFields": [
    {
      "fieldName": "지원동기",
      "fieldType": "textarea",
      "required": true,
      "minLength": 300,
      "maxLength": 800,
      "description": "본 사업에 지원하게 된 동기를 구체적으로 작성해주세요."
    },
    {
      "fieldName": "활용계획",
      "fieldType": "textarea",
      "required": true,
      "minLength": 200,
      "maxLength": 600,
      "description": "지원금을 어떻게 활용할 계획인지 작성해주세요."
    }
  ]
}
```

##### 처리 흐름 (OCR 기반)
1. 신청서 PDF/이미지를 CLOVA OCR로 텍스트 추출
2. OpenAI Chat API로 "다음 텍스트에서 신청서 필드를 파싱해줘" 프롬프트 전송
3. 반환된 JSON을 `application_form_templates`에 저장

---

### 5.5 정책 텍스트 임베딩

#### `POST /api/ingest/policies/embed`

##### 역할
- `policies.raw_text` 또는 `title + summary`를 chunk로 나누고
- OpenAI Embedding API (`text-embedding-3-small`)로 벡터를 만든 뒤,
- `policy_embeddings` 테이블에 저장

##### 처리 흐름
1. 아직 임베딩이 없는 정책들 조회
2. OpenAI Embedding API 호출
3. `policy_id`, `chunk_index`, `embedding` 저장

---

### 5.6 정책 추천

#### `POST /api/policies/recommend`

##### Request
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

##### 처리 흐름
1. `message + parsedProfile`을 하나의 텍스트로 합쳐 Embedding 생성
2. pgvector의 `cosine_distance` 등을 사용해 `policy_embeddings`에서 Top-N 검색
3. 나이/지역/카테고리(교육/어학, 교통, 주거 등) 매칭으로 룰 기반 점수 추가
4. 상위 K개 정책 리턴

##### Response (예시)
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
      "applicationEndDate": "2025-06-30",
      "applicationUrl": "https://..."
    },
    {
      "policyId": 202,
      "title": "청년 직무역량 강화 바우처",
      "summary": "직무·어학 교육비를 바우처 형태로 지원합니다.",
      "category": ["교육/어학"],
      "benefitType": "교육비/교재비 지원",
      "score": 0.82,
      "applicationUrl": "https://..."
    }
  ]
}
```

---

### 5.7 정책 Q&A

#### `POST /api/policies/:id/qa`

##### Request
```json
{
  "question": "이 토익 응시료 지원은 1년에 몇 번까지 신청할 수 있나요?"
}
```

##### 처리 흐름
1. `policies.raw_text` + 관련 텍스트를 RAG 컨텍스트로 구성
2. OpenAI Chat API 호출 시, 시스템 프롬프트에 아래 원칙 적용:
   - 공고문에 근거가 있는 내용만 답변
   - 없으면 **"공고문에 명시되어 있지 않다"**고 명확히 말하기

##### Response (예시)
```json
{
  "answer": "공고문에 따르면 연 2회까지 응시료 지원이 가능합니다."
}
```

---

### 5.8 신청서 초안 생성 (다중 버전) ✨ 개선

#### `POST /api/drafts/generate`

##### Request
```json
{
  "policyId": 101,
  "userQueryId": "uuid-123",
  "profileId": "uuid-456",
  "numberOfVersions": 3
}
```

##### 처리 흐름

1. **정책 정보 조회**
   - `policies` 테이블에서 정책 상세 정보 가져오기
   - `application_form_templates`에서 해당 정책의 신청서 필드 구조 가져오기

2. **사용자 프로필 조회**
   - `user_profiles`에서 사용자 상세 프로필 가져오기

3. **프롬프트 엔지니어링**
   - **System Prompt** 구성:
     ```text
     당신은 청년 정책 신청서 작성 전문가입니다.
     아래 정책과 사용자 정보를 바탕으로, 합격 가능성이 높은 신청서를 작성해주세요.
     
     [작성 원칙]
     1. 정책의 목적과 취지를 명확히 이해하고, 사용자의 상황과 연결하세요.
     2. 구체적인 사례와 숫자를 활용하여 신뢰성을 높이세요.
     3. 개인의 목표와 정책 혜택이 어떻게 연결되는지 논리적으로 서술하세요.
     4. 진정성 있고 자연스러운 한국어로 작성하세요.
     5. 지정된 글자 수 범위를 준수하세요.
     ```
   
   - **User Prompt** 구성:
     ```text
     [정책 정보]
     - 정책명: 청년 어학시험 응시료 지원 사업
     - 목적: 청년들의 취업 역량 강화 및 어학 능력 향상 지원
     - 혜택: 토익/토플/텝스 등 응시료 최대 5만원 지원 (연 2회)
     - 대상: 만 19~34세 대구 거주 청년
     
     [사용자 정보]
     - 나이: 25세
     - 지역: 대구
     - 상태: 편의점 야간 알바
     - 학력: 대학 재학 (경영학과)
     - 경력: 편의점 알바 1년 6개월, 카페 알바 6개월
     - 목표: 어학 점수 향상을 통한 사무직 취업
     
     [작성할 항목]
     - 필드명: 지원동기
     - 글자 수: 300~800자
     - 설명: 본 사업에 지원하게 된 동기를 구체적으로 작성해주세요.
     
     [톤앤매너]
     - Version 1: 진지하고 신중한 톤 (formal)
     - Version 2: 열정적이고 적극적인 톤 (passionate)
     - Version 3: 실용적이고 현실적인 톤 (practical)
     ```

4. **Few-shot Learning (선택사항)**
   - `application_form_templates.example_applications`에 합격 사례가 있다면 참고
   - 프롬프트에 "다음은 합격한 신청서 예시입니다: ..." 추가

5. **OpenAI Chat API 호출**
   - 각 버전(톤앤매너)별로 개별 호출 또는 한 번에 3개 버전 요청
   - Temperature 조정 (0.7~0.9) 으로 다양성 확보

6. **품질 검증**
   - 생성된 텍스트 길이가 제한 범위 내인지 확인
   - 필수 키워드 포함 여부 체크 (정책명, 핵심 혜택 등)
   - 정책 목적과의 연관성 점수 계산 (선택사항)

7. **DB 저장**
   - 각 버전을 `drafts` 테이블에 저장
   - `version`, `tone`, `quality_score` 필드 업데이트

##### Response (예시)
```json
{
  "policyId": 101,
  "drafts": [
    {
      "draftId": "draft-001",
      "version": 1,
      "fieldName": "지원동기",
      "tone": "formal",
      "content": "저는 현재 대구에서 야간 편의점 알바를 하며 경영학을 전공하고 있는 25세 청년입니다. 대학 재학 중이지만 학비와 생활비를 스스로 마련하기 위해 1년 6개월간 편의점에서 근무해왔습니다. 이 과정에서 단순 노동보다는 전문성을 갖춘 사무직으로 진로를 전환하고 싶다는 목표가 명확해졌습니다...",
      "qualityScore": 0.92,
      "charCount": 650
    },
    {
      "draftId": "draft-002",
      "version": 2,
      "fieldName": "지원동기",
      "tone": "passionate",
      "content": "대학생이면서 야간 알바를 병행하는 것이 쉽지 않지만, 저는 반드시 어학 능력을 갖춘 사무직 직장인으로 성장하고 싶습니다! 1년 6개월간 편의점에서 일하며 고객 응대와 책임감을 배웠지만, 이제는 영어 실력을 갖춰 더 넓은 세계로 나아가고 싶습니다...",
      "qualityScore": 0.88,
      "charCount": 580
    },
    {
      "draftId": "draft-003",
      "version": 3,
      "fieldName": "지원동기",
      "tone": "practical",
      "content": "저는 대구에 거주하는 25세 대학생으로, 현재 편의점 야간 알바로 학비를 충당하고 있습니다. 사무직 취업을 위해서는 토익 점수가 필수적이지만, 응시료가 부담되어 계속 미뤄왔습니다. 이번 지원 사업을 통해 연 2회 응시 기회를 확보하고, 6개월 내에 목표 점수(750점)를 달성하여 하반기 채용에 지원할 계획입니다...",
      "qualityScore": 0.90,
      "charCount": 520
    }
  ]
}
```

---

### 5.9 신청서 버전 선택 및 수정 ✨ 신규

#### `PATCH /api/drafts/:draftId`

##### Request
```json
{
  "content": "사용자가 수정한 신청서 내용...",
  "isSelected": true
}
```

##### 처리 흐름
1. `drafts` 테이블에서 해당 draft 업데이트
2. `is_selected` 플래그 설정

##### Response
```json
{
  "success": true,
  "draftId": "draft-002"
}
```

---

## 6. 신청서 생성 프롬프트 엔지니어링 전략 ✨ 핵심

### 6.1 System Prompt 설계

```text
당신은 청년 정책 신청서 작성 전문가입니다.
수많은 청년들의 신청서를 검토하고 합격 사례를 분석해온 경험이 있습니다.

[핵심 작성 원칙]
1. 정책의 목적과 취지를 정확히 파악하고, 지원자의 상황과 자연스럽게 연결하세요.
2. 구체적인 숫자, 사례, 경험을 활용하여 신뢰성을 높이세요.
   - 나쁜 예: "저는 열심히 공부하고 있습니다."
   - 좋은 예: "매일 새벽 5시에 일어나 2시간씩 토익 공부를 하고 있으며, 3개월간 모의고사 점수를 550점에서 680점까지 올렸습니다."

3. 지원자의 배경(학력, 경력, 가정환경)과 정책 혜택이 어떻게 시너지를 낼 수 있는지 논리적으로 서술하세요.

4. 진정성 있고 자연스러운 한국어를 사용하세요.
   - AI스러운 표현 지양: "~에 대해 깊이 고민하게 되었습니다", "~의 기회를 통해"
   - 자연스러운 표현 지향: "~를 경험하며 느꼈습니다", "~를 활용해서"

5. 정해진 글자 수 범위를 준수하세요.

6. 과도한 겸손이나 자기비하는 피하고, 적절한 자신감을 표현하세요.

7. 미래 계획은 구체적이고 실현 가능해야 합니다.
   - 나쁜 예: "열심히 노력하여 좋은 결과를 얻겠습니다."
   - 좋은 예: "6개월 내 토익 750점을 달성하여, 2025년 하반기 대기업 신입 공채에 지원하겠습니다."
```

### 6.2 Few-shot Learning 예시

실제 합격 신청서 사례 (익명화)를 DB에 저장하고 프롬프트에 포함:

```text
[합격 사례 1 - 청년 어학시험 응시료 지원]
저는 부산에 거주하는 24세 대학생으로, 물류 회사 취업을 목표로 토익 공부를 하고 있습니다. 
가정 형편상 학비와 생활비를 스스로 벌어야 해서 주말마다 카페 알바를 하고 있지만, 
토익 응시료(약 50,000원)가 부담되어 실전 경험 없이 독학만 해왔습니다.
(이하 생략...)

[합격 사례 2 - 청년 월세 지원]
저는 서울에서 1인 가구로 자취하는 27세 프리랜서 디자이너입니다.
안정적인 수입이 없어 매달 월세 40만원을 내는 것이 큰 부담입니다.
이번 지원을 통해 주거비 부담을 줄이고, 그 시간에 포트폴리오 작업에 집중하여...
(이하 생략...)
```

### 6.3 톤앤매너별 프롬프트 조정

#### Version 1: Formal (진지하고 신중한 톤)
```text
공식적이고 예의 바른 어조를 유지하세요.
문장은 정중하고 논리적이어야 합니다.
"~입니다", "~하고자 합니다" 같은 격식체를 사용하세요.
```

#### Version 2: Passionate (열정적이고 적극적인 톤)
```text
지원자의 열정과 의지가 드러나도록 작성하세요.
감탄사나 강조 표현을 적절히 사용하세요. (예: "반드시", "꼭", "절실히")
에너지 있고 긍정적인 어조를 유지하세요.
```

#### Version 3: Practical (실용적이고 현실적인 톤)
```text
구체적인 계획과 실행 방안을 중심으로 작성하세요.
숫자, 기한, 목표를 명확히 제시하세요.
감정 표현보다는 사실과 계획 중심으로 서술하세요.
```

### 6.4 품질 검증 로직

생성된 신청서의 품질을 자동으로 평가:

```typescript
interface QualityCheckResult {
  charCountValid: boolean;      // 글자 수 범위 준수
  keywordsIncluded: boolean;    // 필수 키워드 포함 여부
  policyRelevance: number;      // 정책 연관성 (0~1)
  naturalness: number;          // 자연스러움 (0~1, AI 감지 방지)
  specificityScore: number;     // 구체성 점수 (숫자/사례 포함 여부)
  overallScore: number;         // 종합 점수 (0~1)
}

// 예시 검증 로직
function validateDraft(content: string, policy: Policy, template: FormField): QualityCheckResult {
  const charCount = content.length;
  const charCountValid = charCount >= template.minLength && charCount <= template.maxLength;
  
  // 필수 키워드 체크 (정책명, 혜택 종류 등)
  const keywords = [policy.title, policy.benefitType, ...policy.category];
  const keywordsIncluded = keywords.some(kw => content.includes(kw));
  
  // 구체적인 숫자나 날짜 포함 여부
  const hasNumbers = /\d+/.test(content);
  const specificityScore = hasNumbers ? 0.8 : 0.5;
  
  // AI스러운 표현 감지 및 감점
  const aiPhrases = ["깊이 고민하게 되었습니다", "기회를 통해", "노력하겠습니다"];
  const aiPhraseCount = aiPhrases.filter(phrase => content.includes(phrase)).length;
  const naturalness = Math.max(0, 1 - (aiPhraseCount * 0.2));
  
  // 정책 연관성: 간단한 키워드 매칭 또는 임베딩 유사도
  const policyRelevance = 0.85; // 실제로는 임베딩 cosine similarity
  
  const overallScore = (
    (charCountValid ? 0.2 : 0) +
    (keywordsIncluded ? 0.2 : 0) +
    (policyRelevance * 0.3) +
    (naturalness * 0.15) +
    (specificityScore * 0.15)
  );
  
  return {
    charCountValid,
    keywordsIncluded,
    policyRelevance,
    naturalness,
    specificityScore,
    overallScore
  };
}
```

---

## 7. Frontend 주요 화면

### 7.1 Chat / Home

- **상단**: 서비스 간단 소개
  - 예: 청년 버스카드·어학시험·월세 지원 등
- **중앙**: 상황 입력창 + 대화 영역
  - 입력 후 `/api/chat/analyze` 호출 → AI 요약 카드 표시

### 7.2 AI 요약 카드 + 수정 폼

**카드 내용:**
- 나이 / 지역 / 상태 / 주요 고민(버스비, 월세, 응시료 등)
- **[수정/추가하기]** 버튼 클릭 시 모달 폼 오픈
  - 브라우저 상태 또는 `/api/profile/update`로 저장

### 7.3 상세 프로필 입력 폼 ✨ 신규

신청서 작성에 필요한 추가 정보를 대화형 또는 폼 형태로 수집:

**수집 항목:**
- 학력 (고졸/대학재학/대졸 등)
- 전공
- 경력사항 (자유 서술)
- 가구 소득 (선택)
- 가구원 수
- 특기사항 (병역, 장애 여부 등)
- 목표 및 계획

**UI 흐름:**
1. 사용자가 정책 추천 결과를 보고 "신청서 초안 만들기" 클릭
2. "신청서 작성에 필요한 추가 정보를 입력해주세요" 안내
3. 간단한 폼 또는 대화형 UI로 정보 수집
4. 수집 완료 후 → 신청서 생성 API 호출

### 7.4 정책 추천 리스트

- `/api/policies/recommend` 결과를 정책 카드 리스트로 렌더링

**예시 카드:**
- 청년 버스카드 교통비 지원
- 청년 어학시험 응시료 지원
- 청년 월세 지원 사업

**각 카드에:**
- 자세히 보기
- **신청서 초안 만들기** 버튼 ← 클릭 시 프로필 입력 폼으로 이동

### 7.5 정책 상세 + Q&A

- 정책 요약 / 상세 / 마감일 표시
- **"질문하기"** 입력창 → `/api/policies/:id/qa` 결과를 채팅 버블 형식으로 렌더링

### 7.6 신청서 초안 버전 선택 ✨ 신규

**화면 구성:**
1. 생성된 3가지 버전을 탭 또는 카드 형식으로 나란히 표시
2. 각 버전에 톤앤매너 설명 표시 (예: "진지하고 신중한 톤")
3. 품질 점수 표시 (예: ⭐⭐⭐⭐☆ 4.5/5.0)
4. 사용자가 마음에 드는 버전 선택 → "이 버전 사용하기" 버튼

### 7.7 신청서 초안 에디터

- 선택한 버전을 에디터에 로드
- 사용자가 직접 수정 가능
- **실시간 글자 수 표시** (예: 650 / 800자)
- **필수 키워드 체크 표시** (예: "✓ 정책명 포함", "✗ 구체적 날짜 없음")
- **"복사하기"** 버튼으로 클립보드 복사
- **"저장하기"** 버튼으로 수정 내용 DB 저장

---

## 8. 환경 변수 (.env 예시)

```bash
# 공통
NODE_ENV=development

# Backend
PORT=4000

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/youth_policy

# OpenAI
OPENAI_API_KEY=sk-...
OPENAI_MODEL_CHAT=gpt-4o  # 또는 gpt-4-turbo
OPENAI_MODEL_EMBEDDING=text-embedding-3-small

# 청년 정책 Open API
YOUTH_POLICY_API_KEY=...
YOUTH_POLICY_API_BASE=https://...

# Naver CLOVA OCR
NCLOUD_ACCESS_KEY=...
NCLOUD_SECRET_KEY=...
NCLOUD_OCR_ENDPOINT=https://.../ocr/general

# 신청서 생성 설정
DRAFT_DEFAULT_TEMPERATURE=0.8
DRAFT_NUMBER_OF_VERSIONS=3
```

---

## 9. 로컬 개발 실행

### 9.1 Backend

```bash
cd backend
npm install

# (Prisma 사용 시)
npx prisma migrate dev

# 정책 데이터 수집 (최초 1회)
npm run ingest:policies

# 정책 임베딩 생성 (최초 1회 또는 정책 업데이트 시)
npm run ingest:embeddings

# 신청서 양식 수집 (최초 1회, 수동 또는 반자동)
npm run ingest:application-forms

npm run dev
```

### 9.2 Frontend

```bash
cd frontend
npm install
npm run dev
```

- **프론트 dev 서버**: http://localhost:5173
- **백엔드 API 서버**: http://localhost:4000

---

## 10. MVP 개발 우선순위

### Phase 1: 기본 추천 시스템 (2주)
- [ ] 정책 Open API 연동 및 DB 적재
- [ ] 정책 임베딩 생성
- [ ] 사용자 상황 입력 및 파싱
- [ ] 정책 추천 알고리즘 (유사도 + 룰 기반)
- [ ] 기본 프론트엔드 (챗 UI, 정책 카드)

### Phase 2: 신청서 자동 작성 (3주) ⭐ 핵심
- [ ] 사용자 상세 프로필 수집 기능
- [ ] 신청서 양식 템플릿 DB 설계 및 데이터 수집
- [ ] 신청서 생성 프롬프트 엔지니어링
- [ ] 다중 버전 생성 로직
- [ ] 품질 검증 로직
- [ ] 신청서 에디터 UI
- [ ] 버전 선택 및 비교 UI

### Phase 3: 고도화 (2주)
- [ ] 정책 Q&A 기능
- [ ] Few-shot learning용 합격 사례 DB 구축
- [ ] OCR 기반 신청서 양식 자동 파싱
- [ ] 사용자 피드백 수집 및 품질 개선
- [ ] 반응형 UI 개선

---

## 11. 기술적 도전 과제 및 해결 방안

### 11.1 신청서 양식의 다양성
**문제**: 정책마다 신청서 양식이 천차만별
**해결**:
- MVP에서는 주요 정책(교통비, 어학시험, 주거비) 5~10개만 수동으로 양식 등록
- 향후 OCR + GPT로 자동 파싱 시스템 구축

### 11.2 자연스러운 한국어 생성
**문제**: GPT가 생성한 텍스트가 AI스럽게 느껴질 수 있음
**해결**:
- 프롬프트에 "AI스러운 표현 지양" 명시
- Few-shot learning으로 실제 합격 사례 학습
- Temperature 조정 (0.7~0.9)
- 사후 검증으로 부자연스러운 표현 감점

### 11.3 개인정보 보호
**문제**: 사용자의 민감한 정보(소득, 가족사항 등) 저장
**해결**:
- 최소한의 정보만 수집
- DB 암호화
- 개인정보 처리방침 명시
- 사용자 동의 절차

### 11.4 OpenAI API 비용
**문제**: 신청서 생성 시 여러 버전을 만들면 토큰 비용 증가
**해결**:
- MVP에서는 버전 수를 3개로 제한
- 캐싱 활용: 동일한 정책+유사한 프로필은 재사용
- Batch API 활용 (비용 50% 절감)

---

## 12. 라이선스 & 주의사항

- 사용되는 공공 데이터 / OPEN API는 각 제공 기관의 이용 약관 및 표기 의무를 반드시 준수해야 한다.
- 개인정보 보호법을 준수하여 사용자 정보를 안전하게 관리해야 한다.
- OpenAI API 사용 시 콘텐츠 정책을 준수해야 한다.
- 이 레포지토리의 소스 코드 라이선스(MIT 등)는 팀에서 별도로 지정한다.

---

## 13. 참고 자료

### 관련 API 문서
- OpenAI API: https://platform.openai.com/docs
- 네이버 CLOVA OCR: https://api.ncloud-docs.com/docs/ai-naver-clovaocr
- 청년 정책 Open API: (정부 제공 문서 참고)

### 프롬프트 엔지니어링
- OpenAI Prompt Engineering Guide: https://platform.openai.com/docs/guides/prompt-engineering
- Few-shot Learning: https://arxiv.org/abs/2005.14165

### pgvector
- pgvector GitHub: https://github.com/pgvector/pgvector
- PostgreSQL Vector Search: https://www.postgresql.org/docs/current/functions-vector.html
