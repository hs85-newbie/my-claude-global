# my-claude-global

Claude Code 전역 설정 중앙 관리 레포. 로컬(`~/.claude/`)과 클라우드(GitHub Actions)에서 동일한 규칙을 적용합니다.

---

## 아키텍처

```
┌─────────────────────────────────────────────────────┐
│                  Paperclip (Railway)                 │
│          태스크 관리 · 이슈 분해 · 상태 추적          │
└──────────────────────┬──────────────────────────────┘
                       │ 30분 cron (paperclip-dispatch.yml)
                       ▼
┌─────────────────────────────────────────────────────┐
│              GitHub Actions (무료)                    │
│   claude.yml · claude-dispatch.yml · claude-nightly  │
│   전역 설정 로드 → Claude Code 실행 → PR 생성         │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│              프로젝트 레포 (tms, macromalt, ...)      │
│   코드 수정 → 브랜치 push → 승인 후 PR 머지          │
└─────────────────────────────────────────────────────┘
```

## 전체 파이프라인

```
1. Paperclip에 이슈 등록: [tms:dev] 기능 수정
2. 자동 감지 → todo → in_progress (🚀 코멘트)
3. Claude Code 실행 → 코드 수정 → 브랜치 push
4. → in_review (📝 변경 파일 보고서 코멘트)
5. 사용자가 Paperclip에서 "승인" 코멘트
6. PR 자동 생성 + 대상 브랜치 머지
7. → done (🎉 PR 머지 완료 코멘트)
```

## 레포 구조

```
my-claude-global/
├── CLAUDE.md                          # 전역 개발 원칙 (슬림 코어)
├── settings.json                      # MCP 서버, 권한 설정
├── docs/                              # 상세 규칙 (@import 참조)
│   ├── comment-rules.md               # 코드 주석 규칙
│   ├── documentation-rules.md         # 문서화 규칙
│   ├── git-rules.md                   # 형상관리 규칙
│   ├── design-guide.md                # UI/UX 디자인 가이드
│   ├── error-handling.md              # 에러 처리 패턴
│   ├── api-standards.md               # API 설계 표준
│   └── testing-standards.md           # 테스트 전략
├── .github/
│   ├── dependabot.yml                 # 의존성 자동 업데이트
│   └── workflows/
│       ├── claude.yml                 # @claude 댓글 → 즉시 실행
│       ├── claude-code-review.yml     # PR 자동 리뷰
│       ├── claude-dispatch.yml        # 수동/Paperclip dispatch 실행
│       ├── claude-nightly.yml         # 야간 cron (KST 23:00)
│       ├── paperclip-dispatch.yml     # Paperclip 중앙 디스패치 (30분 cron)
│       ├── ci-quality-gate.yml        # CI 품질 게이트 (프로젝트용 템플릿)
│       ├── db-backup.yml              # OpenSpace DB 주간 백업
│       ├── sync-check.yml            # 설정 동기화 주간 체크
│       └── monthly-review.yml         # 월간 자동 점검
└── README.md
```

## 워크플로우 설명

### 핵심 워크플로우

| 워크플로우 | 트리거 | 역할 |
|---|---|---|
| `claude.yml` | 이슈/PR에 `@claude` 댓글 | Claude Code 즉시 실행 |
| `claude-code-review.yml` | PR 생성/업데이트 | 자동 코드 리뷰 |
| `claude-dispatch.yml` | 수동 UI / `repository_dispatch` | 태스크 기반 Claude 실행 |
| `claude-nightly.yml` | 매일 KST 23:00 | `claude-todo` 라벨 이슈 자동 처리 |

### 중앙 디스패치 (Paperclip 연동)

| 워크플로우 | 트리거 | 역할 |
|---|---|---|
| `paperclip-dispatch.yml` | 30분 cron | Paperclip todo 이슈 → 대상 레포 dispatch |
| | | in_review 이슈 → 승인 감지 → PR 생성 + 머지 |

### 운영 안정화

| 워크플로우 | 트리거 | 역할 |
|---|---|---|
| `db-backup.yml` | 매주 일요일 KST 00:00 | OpenSpace DB Artifact 백업 (90일) |
| `sync-check.yml` | 매주 월요일 KST 09:00 | 필수 파일/참조 검증, 문제 시 이슈 생성 |
| `monthly-review.yml` | 매월 1일 KST 09:00 | Claude가 월간 보고서 이슈 생성 |

## Paperclip 이슈 작성법

### 이슈 제목 형식

```
[레포명] 작업 내용              → main 브랜치
[레포명:브랜치] 작업 내용       → 지정 브랜치
```

### 예시

```
[tms:dev] STT 엔진 Clova로 고정
[macromalt] 코드 품질 점검 및 개선
[tms] README 작성
```

### 이슈 상태 자동 전환

```
todo → in_progress → in_review → (승인 코멘트) → done
```

| 상태 | 의미 | 자동 전환 조건 |
|---|---|---|
| `todo` | 대기 중 | 30분 cron 감지 시 dispatch |
| `in_progress` | Claude 실행 중 | dispatch 시 자동 전환 |
| `in_review` | 수정 완료, 리뷰 대기 | Claude 완료 시 자동 전환 |
| `done` | 머지 완료 | "승인" 코멘트 감지 후 PR 머지 시 |

## 로컬 ↔ 클라우드 동기화

```
my-claude-global (GitHub, 단일 진실 소스)
    │
    ├── 로컬: ~/.claude/CLAUDE.md → 심링크 → ~/my-claude-global/CLAUDE.md
    │
    └── 클라우드: Actions 실행 시 checkout → ~/.claude/CLAUDE.md 복사
```

규칙 수정 시:
```bash
vi ~/my-claude-global/CLAUDE.md
cd ~/my-claude-global && git add . && git commit -m "docs: 규칙 추가" && git push
# 끝 — 다음 Actions 실행부터 자동 적용
```

## 인프라 (Railway)

| 서비스 | URL | 역할 |
|---|---|---|
| Paperclip | `paperclip-production-7ed4.up.railway.app` | 태스크 관리, 이슈 추적 |
| Postgres | 내부 연결 | Paperclip DB |
| OpenSpace MCP | `openspace-db-production.up.railway.app` | 스킬 학습/축적 (SSE) |

## GitHub Secrets (프로젝트 레포별)

| Secret | 등록 방법 | 용도 |
|---|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | `/install-github-app` 자동 등록 | Claude 인증 |
| `GLOBAL_SETTINGS_TOKEN` | 수동 등록 (PAT, repo 스코프) | 전역 설정 레포 접근 |
| `PAPERCLIP_API_KEY` | 수동 등록 | Paperclip API 연동 |

## 새 프로젝트 시작

1. `project-template`에서 **"Use this template"** → 레포 생성
2. 레포에서 `/install-github-app` 실행
3. `GLOBAL_SETTINGS_TOKEN` + `PAPERCLIP_API_KEY` 수동 등록
4. `CLAUDE.md` 수정 (프로젝트 정보)
5. Paperclip에 `[레포명:브랜치] 작업 내용` 이슈 등록 → 자동 실행
