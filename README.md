# 청년정책 올인원 – 48시간 해커톤 MVP

자연어로 상황을 입력하면  
**청년 버스카드(교통비) / 토익·어학시험 응시료 / 월세·보증금 지원** 같은 청년 정책을 추천해 주고,  
선택한 정책에 대해 **신청서(지원동기·활용계획)를 3가지 버전으로 자동 생성**하는 서비스입니다.

> 목표: 48시간 안에 **“추천 → 신청서 자동 생성 → 수정·복사”**까지 실제로 돌아가는 MVP 완성

---

## 1. 팀 구성

- 프론트엔드: 1명
- 풀스택: 1명
- 백엔드: 2명
- AI & 데이터: 1명

---

## 2. MVP 범위

### 2.1 Phase 1 (필수, P0 – 30시간 내 완성 목표)

- 상황 입력 → AI 파싱 (나이, 지역, 고민 태그 등)
- **정책 추천 (pgvector 임베딩 + 간단 룰 기반 하이브리드)**
- 정책 DB 20~30개 (수동/스크립트로 입력)
- **신청서 자동 생성 (3버전: 진지/열정/실용)**  
  - 지원동기 + 활용계획
- 간단한 품질 점수(qualityScore) 계산
- 버전 선택 UI
- 에디터에서 수정 후 복사(Clipboard)

### 2.2 Phase 2 (시간 남으면, P1/P2)

- 정책 Q&A (정책별 RAG 기반 질의응답)
- OCR (정책 공고문/신청서 양식 텍스트 추출)
- 사용자 피드백 수집 (좋았어요/별로였어요)
- 반응형 UI, 디자인 폴리싱

---

## 3. 기술 스택

### 3.1 Backend

- Node.js 20+
- Express
- PostgreSQL
- pgvector (벡터 검색)
- OpenAI API (Chat + Embedding)

### 3.2 Frontend

- React 18
- Vite
- Tailwind CSS
- React Query (데이터 패칭/캐싱)

### 3.3 기타

- DB 마이그레이션: 초기에는 SQL 스크립트(`database/init.sql`) 직접 실행
- HTTP 클라이언트: `axios` 또는 기본 `fetch`
- 환경 변수 관리: `.env`

---

## 4. 데이터베이스 설계

### 4.1 스키마 (Option A – 해커톤용 단순화 버전)

```sql
-- pgvector 확장 설치
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE policies (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    summary TEXT,
    category VARCHAR(50),          -- 교통 / 교육/어학 / 주거 등
    target_age_min INT,
    target_age_max INT,
    target_region VARCHAR(50),     -- 부산 / 서울 / 전국 등
    benefit_type VARCHAR(100),     -- 교통비 지원 / 응시료 지원 / 월세 지원 등
    application_url TEXT,
    raw_text TEXT,                 -- 공고문 전체 또는 주요 내용
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE policy_embeddings (
    id SERIAL PRIMARY KEY,
    policy_id INT REFERENCES policies(id),
    chunk_index INT,
    embedding vector(1536),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS policy_embeddings_embedding_idx
ON policy_embeddings USING ivfflat (embedding vector_cosine_ops);

CREATE TABLE user_queries (
    id SERIAL PRIMARY KEY,
    raw_text TEXT,        -- 사용자가 처음 입력한 문장
    parsed_json JSONB,    -- LLM이 파싱한 나이/지역/관심사 등
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE user_profiles (
    id SERIAL PRIMARY KEY,
    user_query_id INT REFERENCES user_queries(id),
    age INT,
    region VARCHAR(50),
    status VARCHAR(100),        -- 대학생, 취준생, 알바 등
    experience TEXT,            -- 간단 경력/상황
    goal TEXT,                  -- 목표 (예: 교통비 절감, 월세 부담 완화)
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE drafts (
    id SERIAL PRIMARY KEY,
    user_query_id INT REFERENCES user_queries(id),
    policy_id INT REFERENCES policies(id),
    version INT,                -- 1, 2, 3
    tone VARCHAR(20),           -- formal / passionate / practical
    reason TEXT,                -- 지원동기
    plan TEXT,                  -- 활용계획
    quality_score FLOAT,        -- 0.0 ~ 1.0 간단 점수
    is_selected BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 5. API 설계 (MVP 기준)

### 5.1 상황 분석 – `POST /api/analyze`

사용자 자연어 입력을 받아 나이/지역/관심사를 파싱하고, `user_queries`에 저장합니다.

**Request**

```json
{
  "message": "부산 23살, 통학 버스비랑 자취방 보증금 부담"
}
```

**Response**

```json
{
  "queryId": 1,
  "parsed": {
    "age": 23,
    "region": "부산",
    "concerns": ["교통비", "주거비"]
  }
}
```

### 5.2 프로필 저장 (선택) – `POST /api/profile`

프론트에서 수정/추가한 정보를 저장합니다. (시간 없으면 생략하고 클라이언트 상태로만 관리해도 됨)

**Request**

```json
{
  "userQueryId": 1,
  "profile": {
    "age": 23,
    "region": "부산",
    "status": "대학생, 알바 중",
    "experience": "편의점 알바 1년",
    "goal": "교통비 절감으로 학업 집중"
  }
}
```

**Response**

```json
{
  "profileId": 1
}
```

### 5.3 정책 추천 – `POST /api/recommend`

입력 텍스트 + 파싱 정보를 기반으로 정책 Top-N을 추천합니다.

**Request**

```json
{
  "message": "부산 23살, 통학 버스비 부담",
  "parsed": {
    "age": 23,
    "region": "부산",
    "concerns": ["교통비"]
  }
}
```

**처리 흐름 (하이브리드)**

1. `message`(또는 `message + parsed`) 임베딩 생성 (OpenAI)
2. `policy_embeddings`에서 코사인 유사도 Top 10 검색
3. 나이/지역/카테고리로 필터링 및 가중치 부여
4. 유사도(70%) + 룰 점수(30%) 합산 후 Top 5 반환

**Response**

```json
{
  "policies": [
    {
      "id": 1,
      "title": "청년 버스카드 교통비 지원",
      "summary": "만 19~34세 부산 거주 청년의 대중교통비 50% 할인",
      "category": "교통",
      "score": 0.92
    }
  ]
}
```

### 5.4 신청서 3버전 생성 – `POST /api/draft/generate`

선택한 정책 + 사용자 프로필을 기반으로, 다른 톤의 신청서(지원동기/활용계획) 3버전을 생성합니다.

**Request**

```json
{
  "userQueryId": 1,
  "profileId": 1,
  "policyId": 1,
  "numberOfVersions": 3
}
```

**Response**

```json
{
  "versions": [
    {
      "draftId": 1,
      "version": 1,
      "tone": "formal",
      "reason": "저는 현재 부산에서 통학 중인 23세 대학생입니다...",
      "plan": "청년 버스카드를 통해 절감된 교통비는...",
      "qualityScore": 0.89
    },
    {
      "draftId": 2,
      "version": 2,
      "tone": "passionate",
      "reason": "...",
      "plan": "...",
      "qualityScore": 0.85
    },
    {
      "draftId": 3,
      "version": 3,
      "tone": "practical",
      "reason": "...",
      "plan": "...",
      "qualityScore": 0.91
    }
  ]
}
```

### 5.5 신청서 수정/선택 – `PATCH /api/draft/:draftId`

프론트에서 수정한 내용 및 선택 여부를 저장합니다.

**Request**

```json
{
  "reason": "수정된 지원동기 내용...",
  "plan": "수정된 활용계획 내용...",
  "isSelected": true
}
```

### 5.6 정책 Q&A (Phase 2) – `POST /api/policies/:id/qa`

정책 상세 텍스트를 기반으로 RAG 질의응답 (시간 있으면 구현).

**Request**

```json
{
  "question": "이 정책은 1년에 몇 번까지 신청 가능한가요?"
}
```

**Response**

```json
{
  "answer": "공고문에 따르면 연 2회까지 지원 가능합니다."
}
```

### 5.7 정책 데이터 인입 – `POST /api/ingest/policies`

- 초기 세팅 시에만 사용
- 정책 JSON/OPEN API를 읽어서 `policies`에 저장하고, 임베딩 생성

---

## 6. 프로젝트 구조

```text
hackathon-youth-policy/
├── backend/
│   ├── server.js
│   ├── db.js
│   ├── routes/
│   │   ├── analyze.js
│   │   ├── profile.js
│   │   ├── recommend.js
│   │   ├── draft.js
│   │   ├── qa.js
│   │   └── ingest.js
│   ├── services/
│   │   ├── openai.service.js
│   │   ├── embedding.service.js
│   │   ├── recommend.service.js
│   │   ├── draft-generator.service.js
│   │   ├── quality-check.service.js
│   │   └── prompt.service.js
│   └── data/
│       └── policies.json
│
├── frontend/
│   ├── src/
│   │   ├── App.jsx
│   │   ├── components/
│   │   │   ├── ChatInput.jsx
│   │   │   ├── SummaryCard.jsx
│   │   │   ├── PolicyCard.jsx
│   │   │   ├── ProfileForm.jsx
│   │   │   ├── VersionSelector.jsx
│   │   │   └── DraftEditor.jsx
│   │   ├── api/
│   │   │   └── client.js
│   │   └── hooks/
│   │       └── useApi.js
│   └── vite.config.js
│
└── database/
    └── init.sql
```

---

## 7. 실행 방법

### 7.1 환경 변수

`backend/.env`

```bash
PORT=4000
DATABASE_URL=postgresql://user:password@localhost:5432/youth_policy
OPENAI_API_KEY=sk-...
NODE_ENV=development
```

`frontend/.env`

```bash
VITE_API_BASE_URL=http://localhost:4000/api
```

### 7.2 DB 초기화

```bash
createdb youth_policy
psql youth_policy -c "CREATE EXTENSION IF NOT EXISTS vector;"
psql youth_policy < database/init.sql
```

### 7.3 백엔드

```bash
cd backend
npm install
npm run ingest   # 정책 데이터 + 임베딩 생성 (필요 시)
npm start        # 또는 node server.js
```

### 7.4 프론트엔드

```bash
cd frontend
npm install
npm run dev
```

- 프론트 dev 서버: http://localhost:5173
- 백엔드 API 서버: http://localhost:4000

---

## 8. 핵심 구현 개요

### 8.1 pgvector 임베딩 검색 (개념)

1. 정책 텍스트를 Chunk로 나눔 (`raw_text` 또는 `summary` 기준)
2. OpenAI `text-embedding-3-small`로 벡터 생성
3. `policy_embeddings.embedding`에 저장 (vector(1536))
4. 추천 시, 입력 문장을 임베딩 → 코사인 거리로 Top-N 검색

### 8.2 하이브리드 추천 (임베딩 + 룰)

- 임베딩 점수: 내용 유사도 (0~1)
- 룰 점수:
  - 나이 범위 일치 여부
  - 지역(전국/특정 지역) 일치 여부
  - 카테고리(교통/주거/교육 등) 매칭
- 최종 점수:
  - `totalScore = similarity * 0.7 + ruleScore * 0.3`

### 8.3 3버전 신청서 생성

- tone: `formal` / `passionate` / `practical`
- 각 톤별로 System Prompt 다르게 설정
- JSON 형식으로 응답 받기:

```json
{"지원동기": "...", "활용계획": "..."}
```

- 글자 수, 키워드 포함 여부 등을 검사해서 `quality_score` 계산

---

## 9. 데모 시나리오 (예시)

### 시나리오: 교통비 지원

1. 사용자 입력  
   `"부산 23살, 통학 버스비 월 10만원 나가서 힘들어요"`
2. `/api/analyze` →  
   `age: 23, region: 부산, concerns: ["교통비"]`
3. `/api/recommend` →  
   `청년 버스카드 교통비 지원` 정책이 Top 1로 추천
4. 사용자가 정책 선택 → `/api/draft/generate`
5. 3버전 생성:
   - 진지한 버전 (0.89점)
   - 열정적인 버전 (0.85점)
   - 실용적인 버전 (0.91점) ← 선택
6. DraftEditor에서 약간 수정 후, 복사 버튼으로 텍스트 복사 → 실제 신청서 양식에 붙여넣기

---

## 10. 우선순위

### P0 (절대 포기 불가)

- `/api/analyze` 기본 파싱 (나이/지역/키워드)
- `/api/recommend` (임베딩 + 간단 룰)
- `/api/draft/generate` (3버전 생성 + quality_score 대략 적용)
- 프론트:
  - 텍스트 입력 → 요약 카드
  - 정책 카드 리스트
  - 3버전 선택 + 에디터 + 복사

### P1 (시간 허용 시)

- `/api/profile` 실제 저장
- `/api/draft/:draftId` 수정 저장
- 정책 Q&A
- 반응형 UI / UX 폴리싱

### P2 (시간 많이 남으면)

- OCR (정책 공고문 캡처 → 텍스트 추출)
- 더 정교한 품질 검증 로직
- 사용자 피드백 저장/분석

---

## 11. 발표 포인트 (3분 요약)

- **문제**  
  - 청년 정책은 많지만,  
    1) 내 상황에 맞는 정책 찾기 어렵고  
    2) 신청서 작성이 부담스러움.

- **해결**  
  1. 자연어 한 줄만 입력하면, AI가 상황 파싱 → 정책 추천  
  2. 선택한 정책에 대해 **3가지 톤의 신청서 초안** 자동 생성  
  3. 간단한 품질 점수로 가장 안정적인 버전을 추천  
  4. 사용자는 에디터에서 살짝만 수정하고 복사해서 제출

- **기술 포인트**  
  - pgvector 기반 임베딩 검색 + 룰 기반 하이브리드 매칭
  - GPT 기반 한국어 신청서 자동 생성
  - 품질 검증 로직으로 “AI 티 나는 문장” 줄이고, 글자 수/키워드 체크

- **차별점**  
  - 기존 서비스: 정책 **검색**과 **리스트 제공** 위주  
  - 본 서비스: **“추천 → 신청서 작성까지” 전 과정을 AI가 같이 가는 구조**
