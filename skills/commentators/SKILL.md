---
name: commentators
description: Spawns a 4-role team (planner / developer / security / qa) that analyzes source code from each perspective and adds role-prefixed doc comments. Use when the user wants code annotated from multiple viewpoints — planning intent, development rationale, security concerns, and QA test points — across a project. Trigger phrases include "역할별 주석", "팀으로 주석 달기", "commentators", "annotate from multiple perspectives", "4관점 리뷰 주석".
---

# commentators

역할별 관점(기획/개발/보안/QA)에서 코드를 분석하고 KDoc/Javadoc/JSDoc 등 언어별 doc 주석을 추가하는 팀 기반 스킬.

## 호출 형태

```
/commentators              # scope=all (기본)
/commentators all          # 프로젝트 소스 전체
/commentators changed      # 현재 브랜치 변경 파일만
/commentators <경로>        # 특정 파일 또는 디렉토리
/commentators roles=planner,dev,security,qa   # 역할 오버라이드 (선택)
```

## 핵심 동작 원칙

1. **멱등성**: 어떤 심볼에 이미 특정 역할의 접두어(`{기획}` 등) 주석이 있으면 해당 역할은 그 심볼을 건너뜀.
2. **순차 처리**: 한 파일 안에서 planner → developer → security → qa 순서로 한 번에 한 역할만 편집 (편집 충돌 방지).
3. **자동 커밋 금지**: 편집만 수행. 사용자가 diff를 검토해서 직접 커밋.
4. **프로젝트 관례 우선**: 프로젝트 루트의 `CLAUDE.md`에 역할 접두어 규칙이 있으면 그것을 따름. 없으면 영문 기본값.
5. **언어별 주석 문법 자동 선택**: 파일 확장자로 주석 스타일 결정.

## 실행 순서

스킬이 호출되면 아래 순서대로 진행한다.

### 1) 인자 파싱

- scope 기본값: `all`
- `all` | `changed` | `<경로>` 중 하나
- `roles=...` 이 있으면 역할 목록 오버라이드

### 2) 프로젝트 컨텍스트 해석

- `git rev-parse --show-toplevel` 로 git 루트 확인. 실패하면 cwd 사용.
- 프로젝트 이름: git 루트 basename (공백/특수문자는 `-`로 치환)
- 팀 이름: `{프로젝트명}-commentators`

### 3) 역할 접두어 결정 (우선순위)

프로젝트 루트 `CLAUDE.md` 내용을 읽어 판단:

1. **프로젝트 규칙이 있을 때**: `{...}` 형태의 역할 접두어가 정의돼 있으면 그것을 사용.
   - 예시 패턴 매칭: "접두어", "prefix", "`{기획}`", "`{Planner}`" 등의 섹션을 찾음.
   - 한국어 키워드(기획/개발/보안/QA)가 정의돼 있으면 한국어 접두어 세트 사용.
2. **없을 때 (기본값)**: 영문 접두어 사용.
   ```
   {Planner}   — planner
   {Developer} — developer
   {Security}  — security
   {QA}        — qa
   ```
3. **인자 오버라이드**: `roles=...` 이 주어지면 위 결과에 덮어씀.

결정된 접두어 맵은 이후 각 역할 에이전트에게 전달.

### 4) 팀 셋업

- `~/.claude/teams/{팀명}/config.json` 존재 여부 확인.
- 없으면 `TeamCreate` 로 팀 생성 후, 4명의 `general-purpose` 에이전트를 `Agent` 툴로 스폰:
  - `name`: `planner`, `developer`, `security`, `qa`
  - `team_name`: 위에서 결정한 팀 이름
  - `run_in_background`: true
- 이미 존재하면 재사용. (팀 config의 `members` 배열에서 4명이 모두 있는지 확인, 빠진 역할만 스폰.)

각 에이전트에게 전달할 기본 브리핑:
- 프로젝트 루트 경로
- 담당 역할과 접두어 (예: `planner` → `{기획}`)
- 주석 작성 규칙 (아래 "주석 작성 규칙" 섹션 참조)
- "team-lead(본인) 이 파일별로 작업을 할당할 테니 지시를 기다려라"

### 5) 대상 파일 목록 생성

scope별로:

- **`all`**: git 루트 하위의 소스 파일. 아래 규칙 적용.
  - 포함 확장자: `.kt .java .ts .tsx .js .jsx .py .go .rs .swift .cpp .cc .c .h .hpp .cs .rb .php .scala .m .mm`
  - 제외 디렉토리: `build/`, `.gradle/`, `node_modules/`, `dist/`, `target/`, `out/`, `.next/`, `.nuxt/`, `.venv/`, `venv/`, `__pycache__/`, `.idea/`, `.vscode/`, `vendor/`, `Pods/`, `DerivedData/`, `.git/`
  - 테스트 디렉토리(`test/`, `tests/`, `__tests__/`, `spec/`, `androidTest/`)는 기본 제외. (테스트 파일은 주석 대상이 아님)
  - 생성 파일(`*.g.kt`, `*.pb.go`, `*.generated.*`) 제외.
- **`changed`**: `git diff --name-only $(git merge-base HEAD origin/main 2>/dev/null || echo main)...HEAD` 결과 + `git status --porcelain` 의 수정/신규 파일. 위 포함/제외 규칙 재적용.
- **`<경로>`**: 인자가 파일이면 그 파일, 디렉토리면 재귀 탐색 후 위 규칙 적용.

파일 수가 많으면(>30) 사용자에게 한 번 확인받은 뒤 진행.

### 6) 언어별 주석 스타일 매핑

| 확장자 | 스타일 | 포맷 |
|---|---|---|
| `.kt` | KDoc | `/** ... */` 블록, 함수/클래스 바로 위 |
| `.java` | Javadoc | `/** ... */` 블록 |
| `.ts .tsx .js .jsx` | JSDoc | `/** ... */` 블록 |
| `.py` | docstring | 함수/클래스 본문 첫 줄 `"""..."""` |
| `.go` | line comment | 선언 바로 위 `// ...` |
| `.rs` | doc comment | `/// ...` |
| `.swift` | doc comment | `/// ...` |
| `.cpp .cc .c .h .hpp` | Doxygen | `/** ... */` |
| `.cs` | XML doc | `/// <summary>...</summary>` |
| `.rb` | YARD/RDoc | `# ...` |
| `.php` | PHPDoc | `/** ... */` |
| `.scala` | ScalaDoc | `/** ... */` |
| `.m .mm` | line/doc | `/// ...` |

### 7) TaskList 구성

- 파일별로 태스크 생성: `"[{파일명}] 4-role annotation"`
- owner는 비워두고 (파일 내에서 역할별로 순차 진행하므로 파일 단위 오너는 team-lead).

### 8) 파일별 순차 주석 작성

각 파일에 대해 아래를 순서대로 수행:

1. planner 에게 SendMessage: "이 파일에서 기획 의도를 덧붙여야 할 심볼을 찾아 접두어 `{...}` 주석을 해당 언어 doc 스타일로 추가하라. 이미 같은 접두어가 있는 심볼은 건너뛰어라. 수정 후 완료 보고."
2. planner 완료 보고 대기 → 다음 역할로.
3. developer → security → qa 순서 반복.
4. 네 역할 모두 끝나면 해당 파일 태스크 completed 처리 후 다음 파일.

각 역할 에이전트는 Read/Edit 툴로 해당 파일을 직접 수정한다. team-lead는 내용 검증은 하지 않고 흐름만 조율한다.

### 9) 주석 작성 규칙 (각 역할 에이전트 공통)

각 역할 에이전트에게 아래 규칙을 전달:

- **위치**: 함수, 클래스, 프로퍼티, 주요 블록의 **선언 바로 위** (언어 관례에 맞게)
- **형태**: 언어별 doc 주석 안에 역할 접두어로 시작하는 **한 문장 내지 두 문장**으로 관점을 서술
  - 예 (Kotlin): `/** {기획} QR 스캔 실패 시 사용자가 수동 입력으로 폴백할 수 있어야 한다는 기획 의도 반영. */`
  - 예 (TypeScript): `/** {Security} Token is stored in-memory only; persistence would expand the breach radius. */`
  - 예 (Python): `"""{QA} Edge case: empty list input should return None, not raise."""`
- **관점 구체성**: 코드가 *무엇을* 하는지 반복하지 말고, 역할의 *왜/어떻게 검증할지*를 적는다.
  - planner: 사용자 시나리오, 기능 존재 이유, 우선순위 근거
  - developer: 아키텍처·패턴 선택 이유, 기술적 트레이드오프
  - security: 위협 모델, 보안 전제, 잠재 취약점·방어 전략
  - qa: 테스트 포인트, 엣지 케이스, 회귀 위험
- **이미 주석 블록이 있는 심볼**:
  - 같은 접두어 주석이 이미 있으면 **건너뜀**.
  - 다른 접두어 주석만 있으면, 해당 doc 블록 안에 새 줄로 본인 접두어 라인을 **추가**.
- **주석이 불필요한 경우는 추가하지 않는다**: 게터/세터, 오버라이드된 toString, 단순 DTO 필드 등 관점이 평범할 때는 생략. 무리해서 모든 심볼에 달지 말 것.
- **기존 코드 수정 금지**: 주석만 추가/확장. 로직, 시그니처, 임포트, 포매팅 변경 금지.

### 10) 빌드 검증 (선택)

scope가 `all`이 아니고 파일 수가 적으면 건너뛰어도 됨.

- Kotlin/Java 프로젝트면 `./gradlew compileDebugKotlin` 또는 `./gradlew compileKotlin` 시도.
- TS 프로젝트면 `tsc --noEmit` 시도 (프로젝트에 tsconfig 있을 때).
- 실행 실패/미지원은 무시하고 리포트에 기록.

### 11) 최종 리포트

team-lead가 사용자에게 보고:
- 처리한 파일 수
- 역할별로 추가된 주석 개수 (각 에이전트 보고 기반 추정)
- 건너뛴 파일/심볼 요약
- 빌드 검증 결과 (수행했다면)
- **다음 단계 안내**: "자동 커밋은 하지 않았습니다. `git diff` 로 검토 후 커밋하세요."

## 에지 케이스 처리

- **git 리포가 아닌 경우**: cwd 기준으로 `scope=<경로>` 만 허용. `changed` 는 에러.
- **대상 파일 0개**: 사용자에게 알리고 종료.
- **CLAUDE.md 없음**: 영문 기본 접두어 사용.
- **지원 안 하는 확장자만 있음**: 지원 파일이 없다고 리포트 후 종료.
- **에이전트 편집 실패**: 해당 역할은 건너뛰고 다음 역할로. 리포트에 실패 기록.
- **기존 팀의 접두어 규칙이 이전 실행과 다름**: 새 규칙으로 새 팀 스폰(팀 이름에 해시 접미사) 하거나, 사용자에게 기존 팀을 재사용할지 물음.

## 주의 사항

- **역할 에이전트는 Read + Edit 만 쓰도록 지시**: Bash로 git 조작·빌드·파일 생성 금지. team-lead가 조율.
- **역할 에이전트 스폰 시 반드시 `team_name` 지정**: 스킬 호출자(= team-lead)가 이 팀의 리더가 되도록 유지.
- **서브밋/푸시/PR 생성 금지**: 이 스킬은 편집까지만.
- **민감 파일 스킵**: `.env`, `secrets.*`, 키 파일 등 주석 대상이 될 리 없는 파일은 확장자 필터에서 자동 제외되지만, 경로에 `secret`/`credential` 이 포함된 파일은 명시적으로 건너뜀.

## 종료 조건

- 모든 대상 파일에 대해 4역할 루프가 완료됨.
- 또는 사용자가 중단 요청.
- 또는 대상 파일 0개.

종료 시 팀은 **유지**한다 (다음 호출에서 재사용). 사용자가 명시적으로 "팀 정리해" 라고 하면 SendMessage로 shutdown_request 를 보내서 정리한다.
