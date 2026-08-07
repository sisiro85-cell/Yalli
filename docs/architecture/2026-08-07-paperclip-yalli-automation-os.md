# Paperclip 기반 YALLI 통합 자동화 아키텍처

- 작성일: 2026-08-07
- 상태: 설계 백업본
- 목적: 현재 운영 중인 여러 자동화 시스템을 Paperclip 상위 Control Plane 아래 통합하기 위한 아키텍처 기준을 기록한다.
- 구현 상태: 미구현. 실제 코드 변경 전 Pilot 설계와 승인 필요.

---

## 1. 핵심 결론

Paperclip을 새로운 자동화 프로그램으로 재개발하지 않는다.

기존 자동화 프로그램은 최대한 그대로 유지하고, Paperclip을 그 위의 AI 조직 운영 및 관제 계층으로 사용한다.

```text
                         YALLI AUTOMATION OS

                              사용자
                                │
                                ▼
                         ┌──────────────┐
                         │  Paperclip   │
                         │ Control Plane│
                         └──────┬───────┘
                                │
                         CEO / Hermes
                                │
          ┌────────────┬────────┼────────┬────────────┐
          ▼            ▼        ▼        ▼            ▼
       MUSIC        SHOPPING   SNS     STORY      ANALYTICS
       Project       Project Project   Project       Project
          │            │        │        │            │
      ┌───┼───┐        │        │        │            │
      ▼   ▼   ▼        ▼        ▼        ▼            ▼
   Agent Agent Agent  Agent    Agent    Agent         Agent
      │
      ▼
 ┌─────────────────────────────────────────────┐
 │           기존 자동화 Worker               │
 │ Suno / TTS / FFmpeg / Renderer / YouTube   │
 │ Instagram / Scraper / Image / Mastering    │
 └─────────────────────────────────────────────┘
```

Paperclip의 책임:

- Company / Project 관리
- Task 생성과 상태 관리
- Agent 배정
- Routine / Scheduling
- 승인과 Review Gate
- 비용과 Budget
- 실행 로그와 감사 추적
- 실패 감지와 재실행 정책
- 조직 구조와 권한

Hermes / AI Agent의 책임:

- 목표 해석
- 작업 분해
- 조사와 판단
- 전략 수립
- 검수와 비판
- 장기 Context / Memory
- MCP / Skill 활용

기존 자동화 시스템의 책임:

- Suno 실행
- Playwright 브라우저 자동화
- TTS 생성
- FFmpeg / 영상 렌더링
- 썸네일 생성
- YouTube 업로드
- Instagram 발행
- 외부 데이터 수집
- 성과 계산
- Success / Failure DNA 계산
- 도메인별 deterministic 처리

---

## 2. Company 구조

초기에는 자동화별 Company를 여러 개 만들기보다 하나의 Company 아래 여러 Project를 두는 방식을 우선한다.

```text
YALLI AUTOMATION

├── Music Project
├── Shopping Shorts Project
├── SNS Project
├── Story Video Project
├── Blog Project
└── Analytics Project
```

이유:

- Research, Analytics, Image, TTS, Publishing, Knowledge, QA, Codex, Hermes를 여러 프로젝트가 공유한다.
- 초기부터 Company를 분리하면 Agent와 Skill 중복이 커진다.
- 실제 운영에서 데이터/권한/배포 독립성이 충분히 커진 프로젝트만 별도 Company로 분리한다.

---

## 3. 권장 Agent 조직

```text
                          사용자
                            │
                      CEO Agent
                       [Hermes]
                            │
              ┌─────────────┼─────────────┐
              │             │             │
        Strategy Agent  Production    Performance
           Hermes        Manager       Analyst
                            │
        ┌──────────┬────────┼─────────┬─────────┐
        ▼          ▼        ▼         ▼         ▼
     Research    Writer   Critic      QA     Publisher
      Agent      Agent    Agent      Agent     Agent
        │                    │         │
     Hermes/API             Local     Local
                            LLM       LLM

                     Developer Agent
                          │
                        Codex
```

Music Factory에 이미 정의된 Agent 역할은 최대한 재사용한다.

- CEO
- Market Research
- SEO
- Content Planner
- Critic
- Quality
- Knowledge
- Performance
- DNA
- Anti-Bias

새 역할을 무작정 추가하지 않고 기존 역할을 Paperclip Agent 정의로 승격하는 방향을 우선한다.

---

## 4. Music Factory 경계 재설계

현재 Music Factory는 제작 기능뿐 아니라 자체 오케스트레이션 계층도 갖고 있다.

주요 중복 영역:

- `agentOrchestratorService`
- `workflowEngineService`
- `jobQueueService`
- `workflowApprovalService`

현재 구조:

```text
Music Factory
 ├─ Workflow Engine
 ├─ Agent Manager
 ├─ Approval
 ├─ Queue
 ├─ Suno
 ├─ Renderer
 ├─ YouTube
 └─ Learning
```

목표 구조:

```text
Paperclip
 ├─ Workflow / Task
 ├─ Agent
 ├─ Approval
 ├─ Queue / Coordination
 └─ Scheduling
        │
        ▼
Music Factory Domain Engine
 ├─ Research
 ├─ Lyrics
 ├─ Suno
 ├─ Thumbnail
 ├─ Renderer
 ├─ YouTube
 └─ Performance Learning
```

주의:

- Paperclip 도입 직후 기존 Workflow Engine / Queue / Approval 코드를 삭제하지 않는다.
- Pilot 기간에는 fallback으로 유지한다.
- Paperclip 경로가 안정화된 뒤 중복 오케스트레이션만 단계적으로 축소한다.

---

## 5. Hermes 통합

기존 Music Factory의 Hermes 호환 계층은 실제 Hermes Runtime 호출보다 내부 결과 생성 스텁 성격이 강하다.

Paperclip이 제공하는 실제 Hermes Adapter를 우선 사용한다.

### Local 방식

```text
Paperclip
   ↓
hermes_local
   ↓
hermes chat
```

### Gateway 방식

```text
Paperclip
   ↓ HTTP / SSE
hermes_gateway
   ↓
Hermes Gateway
   ↓
Hermes Agent
```

장기적으로는 `hermes_gateway` 방식을 우선 검토한다.

장점:

- Paperclip과 Hermes Runtime 분리
- 기존 Hermes 환경 유지 가능
- 세션 지속 가능
- HTTP/SSE 기반 상태 추적
- 원격/Docker 실행 확장 가능

---

## 6. Music 자동화 Workflow

```text
Music Routine
    ↓
Channel / Market Analytics
    ↓
Music Director / Planner
    ↓
Lyrics Agent
    ↓
Critic / QA
    ↓
Suno Worker
    ↓
기존 Suno Automation Service
    ↓
Audio / Subtitle / Asset 처리
    ↓
Playlist Render Worker
    ↓
기존 FFmpeg Render Engine
    ↓
YouTube Draft
    ↓
Paperclip Approval
    ↓
기존 YouTube Upload Adapter
    ↓
24h / 72h / 7d Performance Routine
    ↓
Performance Learning
```

### 유지할 주요 기존 자산

- `sunoAutomationService`
- Playlist / Factory Video Render 계층
- FFmpeg `render_fast.py`
- YouTube Upload Adapter
- Thumbnail 처리
- Performance Learning
- Success / Failure DNA
- Music Knowledge / Skill 관련 도메인 로직

---

## 7. Playlist Renderer 연결

현재 Playlist 렌더 구조는 Paperclip Worker와 궁합이 좋다.

```text
Paperclip Task
   ↓
Playlist Render Worker
   ↓
Render Job JSON
   ↓
Python render_fast.py
   ↓
FFmpeg
   ↓
MP4
   ↓
Paperclip Work Product / Artifact
```

렌더 엔진을 Paperclip 내부로 재작성하지 않는다.

---

## 8. YouTube Upload 연결

기존 업로드 상태 모델을 유지한다.

```text
draft
 ↓
approved
 ↓
uploaded
```

권장 흐름:

```text
영상 완료
   ↓
YouTube Draft 생성
   ↓
Paperclip Review / Approval
   ↓
사용자 승인
   ↓
Publisher Agent
   ↓
기존 YouTube Upload Adapter
```

Paperclip의 승인과 기존 업로드 모듈의 안전장치를 중복 제거하기 전까지 이중 안전장치로 유지한다.

---

## 9. Performance Learning

성과 계산 알고리즘은 Paperclip로 옮기지 않는다.

Music Factory 도메인에 유지한다.

예상 데이터:

- Views
- Likes
- Comments
- CTR
- Retention
- Top / Bottom 콘텐츠
- Insight
- Success DNA
- Failure DNA
- Opportunity Score Candidate

권장 학습 흐름:

```text
Performance Routine
        ↓
Performance Worker
        ↓
후보 Insight / DNA 생성
        ↓
Critic
        ↓
Anti-Bias
        ↓
CEO Review
        ↓
사용자 승인
        ↓
Verified Knowledge / Company DNA 승격
```

자동 적용보다 `candidate → 검증 → 승인 → 승격` 방식을 유지한다.

---

## 10. SNS / Instagram 자동화

기존 Instagram Publisher의 CLI 구조를 Paperclip Pilot에 활용한다.

현재 형태:

```text
validate
 ↓
dry-run
 ↓
publish --yes-publish
```

Paperclip 연결:

```text
Content Agent
     ↓
Instagram Content JSON
     ↓
Validate Worker
     ↓
Dry Run
     ↓
QA Agent
     ↓
Paperclip Approval
     ↓
Publish Worker
     ↓
기존 Meta Instagram API Publisher
```

Pilot 1순위 후보로 사용한다.

이유:

- 입력/출력 계약이 작다.
- 실제 외부 발행 전 dry-run이 있다.
- 승인 경계를 검증하기 쉽다.
- 실패 시 영향 범위가 Music Factory보다 작다.

---

## 11. Shopping Shorts 자동화

```text
Shopping Project

Product Discovery Routine
       ↓
Product Research
       ↓
Source Collector
       ↓
Video Analyzer
       ↓
Script Agent
       ↓
Critic
       ↓
TTS Worker
       ↓
Subtitle / Cleanup Worker
       ↓
Video Renderer
       ↓
QA
       ↓
Approval
       ↓
Publisher
       ↓
Performance Tracking
```

원칙:

- 영상 수집기 재작성 금지
- TTS 엔진 재작성 금지
- 자막 제거/처리 엔진 재작성 금지
- 기존 렌더러 우선 재사용
- Paperclip에는 명령, 상태, 결과, Artifact만 연결

---

## 12. Story / 야담 영상 자동화

```text
Story Project

Topic Routine
       ↓
Research
       ↓
Source Verification
       ↓
Story Planner
       ↓
Writer
       ↓
Critic
       ↓
Scene Director
       ↓
Image / Video Worker
       ↓
TTS Worker
       ↓
Video Worker
       ↓
QA
       ↓
Approval
       ↓
YouTube Publisher
```

AI Agent는 판단/기획/검증을 담당하고, 영상·음성·파일 처리는 deterministic Worker를 우선한다.

---

## 13. Analytics 공통 계층

각 자동화 프로젝트마다 YouTube Analytics를 중복 구현하지 않는다.

```text
                     Analytics Project
                           │
       ┌───────────────────┼───────────────────┐
       ▼                   ▼                   ▼
     Music              Shopping             Story
       │                   │                   │
       └───────────────────┼───────────────────┘
                           ▼
                  Performance Knowledge
```

권장 측정 Routine:

- 24h
- 72h
- 7d
- 30d

성과 데이터를 각 프로젝트의 Performance Learning과 연결한다.

---

## 14. 책임 분리표

| 기능 | Paperclip | Hermes / AI | 기존 시스템 |
|---|---|---|---|
| Company / Project | O |  |  |
| Task | O |  |  |
| Routine | O |  |  |
| Agent 배정 | O |  |  |
| 비용 / Budget | O |  |  |
| Approval | O |  | 보조 안전장치 유지 |
| 전체 실행 추적 | O |  |  |
| 조직 운영 | O | O |  |
| Task 분해 |  | O |  |
| 장기 Context / Memory |  | O |  |
| MCP / Skills |  | O |  |
| 개발 |  | Codex |  |
| Suno |  |  | O |
| FFmpeg |  |  | O |
| Playlist Render |  |  | O |
| TTS |  |  | O |
| Instagram Publish |  |  | O |
| YouTube Upload |  |  | O |
| 콘텐츠 성과 알고리즘 |  |  | O |
| Success / Failure DNA |  |  | O |
| 이미지/미디어 처리 |  |  | O |

---

## 15. Adapter 전략

초기에는 Paperclip Plugin으로 모든 시스템을 다시 만들지 않는다.

권장 구조:

```text
Paperclip
     │
     │ Task / Runtime Context
     ▼
Yalli Worker Adapter
     │
     ├── CLI
     ├── Local HTTP
     └── Process
             │
             ▼
       기존 프로그램
```

Adapter가 담당할 공통 계약 예시:

### Input

- taskId
- projectId
- operation
- payload
- workspace
- correlation/run ID

### Output

- status
- summary
- structured result
- artifact paths
- logs
- error code
- retryable 여부

비밀값은 Task 본문에 넣지 않고 환경변수/Secret으로 주입한다.

---

## 16. Paperclip Plugin 사용 전략

초기 도입에서는 Plugin보다 CLI / Local HTTP Adapter를 우선한다.

Plugin 승격 조건:

- 2개 이상의 프로젝트가 동일 Worker를 반복 사용
- 인터페이스가 안정화됨
- Paperclip UI에 전용 관리 화면이 실제로 필요함
- Capability / 권한 경계를 명확히 정의할 수 있음
- 기존 CLI/HTTP 방식보다 유지보수 비용이 낮음

처음부터 모든 기능을 Plugin으로 재작성하지 않는다.

---

## 17. 구현 순서

### Phase 0. 구조 동결 및 백업

- 현재 시스템 백업
- 실행 흐름 기록
- Secret / 로컬 프로필 제외 확인
- 기존 Worker 인터페이스 목록화

### Phase 1. Instagram Pilot

목표:

```text
Paperclip Task
 → Worker 호출
 → Validate
 → Dry Run
 → Approval
 → Publish
 → Result 기록
```

검증 항목:

- Task 전달
- 상태 업데이트
- 로그 수집
- 승인 차단
- 실패 처리
- Secret 노출 방지
- 재실행

### Phase 2. Hermes Gateway Pilot

- Paperclip → Hermes Gateway
- 실제 Agent 실행 확인
- Session persistence
- Task / comment wake 확인
- 비용 추적

### Phase 3. Music Factory 연결

우선 연결 Worker:

1. Research
2. Lyrics / Planning
3. Suno
4. Playlist Render
5. Thumbnail
6. YouTube Draft
7. Approval
8. YouTube Upload
9. Performance Learning

### Phase 4. 중복 오케스트레이션 축소

Paperclip 안정화 전에는 삭제하지 않는다.

검증 완료 후 단계적으로:

- 내부 Agent Orchestrator 역할 축소
- Workflow Engine 역할 축소
- Approval 상위 책임 Paperclip 이관
- Queue 상위 책임 Paperclip 이관

도메인 내부의 작은 Job Queue는 필요한 경우 유지한다.

### Phase 5. Shopping Shorts

- Source Collector Adapter
- Script Agent
- TTS Worker
- Media Worker
- Render Worker
- Publisher

### Phase 6. Story Automation

- Research
- Fact Verification
- Script
- Scene Plan
- Image/TTS/Video
- QA
- Publish

### Phase 7. Shared Analytics

- YouTube Analytics 공통화
- 24h / 72h / 7d / 30d Routine
- Performance Learning 연결

### Phase 8. Organizational Learning

```text
성과
 ↓
Candidate Insight
 ↓
Evidence
 ↓
Critic
 ↓
Anti-Bias
 ↓
CEO
 ↓
사용자 승인
 ↓
Verified Knowledge
 ↓
Skill / DNA 후보
 ↓
다음 생산에 적용
```

---

## 18. 하지 않을 것

초기 단계에서 금지:

- Paperclip 전체 포크 후 대규모 개조
- 기존 자동화 시스템 전체 재작성
- 모든 기존 Workflow 삭제
- 모든 Worker를 AI Agent로 변환
- 모든 연결을 Paperclip Plugin으로 구현
- 검증 전 자동 Strategy 변경
- 검증 전 Success DNA 자동 승격
- Secret을 Prompt / Task 본문에 저장
- 기존 fallback 제거

---

## 19. 리스크

### 이중 오케스트레이션

Paperclip과 기존 Workflow Engine이 동시에 같은 업무를 관리하면 상태가 분리될 수 있다.

대응:

- Pilot에서는 책임 영역 명시
- Paperclip을 상위 상태의 Source of Truth로 단계적 승격
- 기존 Workflow는 fallback으로만 유지 후 축소

### 중복 실행

동일 Task가 Paperclip과 내부 Queue에서 이중 실행될 수 있다.

대응:

- correlation / run ID 도입
- Worker idempotency 검토
- 외부 발행 작업은 승인 + idempotency 필수

### Secret

Hermes, YouTube, Meta 등의 Credential이 Task/로그에 노출될 수 있다.

대응:

- Secret Manager 또는 환경변수 사용
- Prompt에 원문 키 금지
- 로그 redaction
- Agent별 최소 권한

### Plugin 성숙도

Paperclip Plugin Runtime은 초기 단계에서 과도하게 의존하지 않는다.

대응:

- CLI / HTTP 우선
- 안정화 후 Plugin 승격

---

## 20. 최종 목표

현재:

```text
각 프로그램
 ├─ 자기 Workflow
 ├─ 자기 Agent
 ├─ 자기 Queue
 ├─ 자기 Approval
 └─ 자기 Scheduler
```

목표:

```text
              Paperclip
 ├─ Workflow / Task
 ├─ Agent Organization
 ├─ Queue Coordination
 ├─ Approval
 ├─ Budget
 ├─ Routine
 └─ Audit / Observability
        │
        ▼
각 Yalli 프로그램
 └─ 자기 전문 기능만 담당
```

즉 여러 자동화 프로그램을 각각 독립적으로 움직이는 시스템에서 하나의 `YALLI AUTOMATION OS` 아래 운영되는 AI 조직으로 통합한다.

---

## 21. 다음 승인 대상

다음 구현 승인 전 작성할 산출물:

1. Instagram Pilot PRD
2. Paperclip ↔ Worker Adapter Contract
3. Hermes Gateway 연결 SDD
4. Pilot Test Case
5. Rollback Plan
6. 수정 파일 목록
7. 신규 파일 필요성 검토
8. ADR: Paperclip을 상위 Control Plane으로 채택할지 결정

실제 구현은 별도 승인 후 진행한다.
