# 청년정책 올인원 – 48시간 해커톤 MVP

## MVP 범위

### 구현
- 상황 입력 → AI 파싱 (나이/지역/고민)
- 정책 추천 (룰 기반, DB 5~10개)
- **신청서 자동 생성** (지원동기, 활용계획)
- 에디터 수정 + 복사

### 제외
- OCR, pgvector, 다중 버전, Q&A, 품질 검증

---

## 기술 스택

**Backend**: Node.js + Express + PostgreSQL + OpenAI API  
**Frontend**: React 18 + Vite + Tailwind CSS  
**Database**: PostgreSQL (3개 테이블, raw query)

---

## 데이터베이스

```sql
CREATE TABLE policies (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    summary TEXT,
    category VARCHAR(50),
    target_age_min INT,
    target_age_max INT,
    target_region VARCHAR(50),
    benefit_type VARCHAR(100),
    application_url TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE user_queries (
    id SERIAL PRIMARY KEY,
    raw_text TEXT,
    parsed_json JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE drafts (
    id SERIAL PRIMARY KEY,
    user_query_id INT REFERENCES user_queries(id),
    policy_id INT REFERENCES policies(id),
    field_name VARCHAR(50),
    content TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## API 설계

### 1. POST /api/analyze

**Request**
```json
{"message": "부산 23살, 통학 버스비랑 자취방 보증금 부담"}
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

**구현**
```javascript
const prompt = `
다음 문장에서 나이, 지역, 고민을 JSON으로 추출:
"${message}"

형식: {"age": 숫자, "region": "지역명", "concerns": ["고민1", "고민2"]}
`;
```

---

### 2. POST /api/recommend

**Request**
```json
{
  "age": 23,
  "region": "부산",
  "concerns": ["교통비", "주거비"]
}
```

**Response**
```json
{
  "policies": [
    {
      "id": 1,
      "title": "청년 버스카드 교통비 지원",
      "summary": "만 19~34세 부산 거주 청년의 대중교통비 50% 할인",
      "category": "교통"
    }
  ]
}
```

**구현**
```sql
SELECT * FROM policies
WHERE target_age_min <= $1 
  AND target_age_max >= $1
  AND (target_region = $2 OR target_region = '전국')
  AND category = ANY($3)
LIMIT 5;
```

---

### 3. POST /api/draft/generate

**Request**
```json
{
  "queryId": 1,
  "policyId": 1,
  "profile": {
    "age": 23,
    "region": "부산",
    "status": "대학생, 알바 중",
    "experience": "편의점 알바 1년",
    "goal": "교통비 부담 줄이고 학업 집중"
  }
}
```

**Response**
```json
{
  "drafts": [
    {
      "fieldName": "지원동기",
      "content": "저는 현재 부산에서 대학을 다니며..."
    },
    {
      "fieldName": "활용계획",
      "content": "청년 버스카드를 통해..."
    }
  ]
}
```

**구현**
```javascript
const systemPrompt = `
청년 정책 신청서 작성 전문가.
원칙:
1. 정책 목적과 사용자 상황 연결
2. 구체적인 숫자와 경험
3. 진정성 있는 한국어 (AI스러운 표현 X)
4. 400~600자
`;

const userPrompt = `
[정책]
제목: ${policy.title}
요약: ${policy.summary}
혜택: ${policy.benefit_type}

[사용자]
나이: ${profile.age}
지역: ${profile.region}
상태: ${profile.status}
경험: ${profile.experience}
목표: ${profile.goal}

작성 항목:
1. 지원동기 (400~600자)
2. 활용계획 (400~600자)

JSON:
{"지원동기": "...", "활용계획": "..."}
`;
```

---

## 프로젝트 구조

```
hackathon-youth-policy/
├── backend/
│   ├── server.js
│   ├── db.js
│   ├── routes/
│   │   ├── analyze.js
│   │   ├── recommend.js
│   │   └── draft.js
│   ├── services/
│   │   └── openai.js
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
│   │   │   └── DraftEditor.jsx
│   │   └── api/
│   │       └── client.js
│   └── vite.config.js
│
└── database/
    └── init.sql
```

---

## 48시간 타임라인 (2인 기준)

### Day 1 오전 (0~6h)
**백엔드**: Express + DB + 정책 INSERT + /api/analyze (6h)  
**프론트**: Vite + Tailwind + ChatInput + SummaryCard (6h)

### Day 1 오후 (6~12h)
**백엔드**: /api/recommend + ProfileForm 엔드포인트 (6h)  
**프론트**: PolicyCard + ProfileForm + API 연동 (6h)

### Day 1 밤 (12~18h)
**백엔드**: /api/draft/generate (OpenAI 프롬프트 설계 및 구현) (6h)  
**프론트**: DraftEditor + 복사 기능 + 전체 플로우 연결 (6h)

### Day 2 새벽 (18~24h)
**통합 테스트 + 버그 픽스 + UI 개선 (6h)**

### Day 2 오전 (24~36h)
**백엔드**: 프롬프트 튜닝 + 정책 데이터 추가 (6h)  
**프론트**: UI/UX 개선 + 에러 핸들링 (6h)

### Day 2 오후 (36~48h)
**발표 PPT (3h) + 데모 리허설 (2h) + 코드 정리 (2h) + 휴식 (1h)**

---

## 환경 변수

```bash
# backend/.env
PORT=4000
DATABASE_URL=postgresql://user:password@localhost:5432/youth_policy
OPENAI_API_KEY=sk-...
```

```bash
# frontend/.env
VITE_API_BASE_URL=http://localhost:4000/api
```

---

## 실행

```bash
# DB
createdb youth_policy
psql youth_policy < database/init.sql

# Backend
cd backend
npm install
node server.js

# Frontend
cd frontend
npm install
npm run dev
```

---

## 핵심 프롬프트 엔지니어링

### System Prompt
```
청년 정책 신청서 작성 전문가입니다.

작성 원칙:
1. 정책 목적과 지원자 상황을 자연스럽게 연결
2. 구체적인 숫자, 사례, 경험 포함
   - 나쁜 예: "저는 열심히 공부하고 있습니다."
   - 좋은 예: "매일 새벽 5시에 2시간씩 공부하며 3개월간 점수를 550점에서 680점으로 올렸습니다."
3. 지원자 배경과 정책 혜택의 시너지 논리적 서술
4. 진정성 있는 한국어 (AI스러운 표현 지양)
   - 지양: "~에 대해 깊이 고민하게 되었습니다", "~의 기회를 통해"
   - 지향: "~를 경험하며 느꼈습니다", "~를 활용해서"
5. 글자 수 준수 (400~600자)
6. 과도한 겸손 X, 적절한 자신감
7. 구체적이고 실현 가능한 미래 계획
   - 나쁜 예: "열심히 노력하여 좋은 결과를 얻겠습니다."
   - 좋은 예: "6개월 내 토익 750점 달성, 2025년 하반기 대기업 신입 공채 지원하겠습니다."
```

### 톤앤매너 (시간 남으면 다중 버전)
- Formal: 공식적, 정중, 논리적, "~입니다", "~하고자 합니다"
- Passionate: 열정, 의지, 강조 표현, "반드시", "꼭", "절실히"
- Practical: 구체적 계획, 숫자/기한/목표, 감정보다 사실 중심

---

## 데모 시나리오

### 시나리오 1: 교통비
```
입력: "부산 23살, 통학 버스비 월 10만원"
→ 파싱: 23세, 부산, 교통비
→ 추천: 청년 버스카드
→ 프로필: 대학생, 편의점 알바, 교통비 절감으로 학업 집중
→ 생성: 지원동기 + 활용계획
→ 복사 → 제출
```

### 시나리오 2: 어학시험
```
입력: "대구 25살, 토익 시험비 부담"
→ 추천: 어학시험 응시료 지원
→ 생성 → 제출
```

---

## 우선순위

1. **신청서 생성** (포기 불가, 프롬프트 품질에 최소 3시간)
2. **기본 플로우** (입력→파싱→추천→생성→복사)
3. **UI/UX** (작동만 하면 됨)
4. **추가 기능** (시간 남으면)

---

## 발표 (3분)

```
청년들은 매년 수백 개의 정책 혜택을 놓칩니다.
1) 어떤 정책이 있는지 모름
2) 복잡한 신청서 작성 부담

AI로 해결:
1. 자연어 입력 → AI 자동 파싱
2. 맞춤 정책 추천
3. 신청서 자동 생성
4. 복사해서 바로 제출

기존 서비스는 정책 '검색'만.
우리는 '신청서 작성'까지 자동화.
```
