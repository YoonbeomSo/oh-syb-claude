---
name: jira-api-handoff
description: 백엔드 API 변경사항을 프론트 개발자에게 넘기거나(프론트 API 핸드오프), 버그 수정을 QA 담당자에게 넘기기 위해(QA 핸드오프) Jira 이슈에 ADF 댓글을 작성/갱신한다. 신규/필드추가 색상 뱃지·필드별 타입+설명·Swagger 딥링크(프론트 모드), 원인/수정/QA확인 3블록 + 맨 아래 개발자 확인용 MR 링크 푸터(QA 모드), 공손한 말투로 ADF 댓글을 만든다. "API 변경 프론트한테 넘겨줘", "Jira에 백엔드 수정사항 댓글", "핸드오프 댓글", "QA 넘겨줘", "수정완료 댓글", "QA 핸드오프" 등에 사용.
argument-hint: "[Jira URL 또는 이슈키] [--qa | --front] (선택: 변경 API 설명 또는 수정 내용)"
allowed-tools:
  - Read
  - Glob
  - Grep
  - Bash(git diff:*)
  - Bash(git log:*)
  - Bash(curl:*)
---

# Jira API Handoff Comment

## Overview

두 가지 모드를 지원한다.

1. **프론트 API 핸드오프** (기존): 백엔드 API 변경(신규 API / 기존 응답 필드 추가)을 프론트 개발자가 보기 좋게 Jira 이슈 댓글로 정리. 색상 상태 뱃지·필드별 타입·설명·Swagger 딥링크를 갖춘 ADF 댓글.
2. **QA 핸드오프** (신규): 버그 수정 완료를 QA 담당자에게 알리는 댓글. 원인/수정/QA확인 3블록 구조 + `[수정 완료]` 상태 머리말. 배포 여부는 평문으로 밝히고, MR 링크는 맨 마지막 푸터 한 줄로 분리한다.

항상 한국어로 응답한다.

## 의존 도구

공식 Atlassian MCP(`mcp__claude_ai_Atlassian__*`)가 세션에 로드되어 있어야 한다. 미연결 시 사용자에게 연결 안내 후 중단.
- `getJiraIssue` — 이슈 존재/제목 확인, reporter accountId 조회(QA 모드 재할당 시)
- `addCommentToJiraIssue` — 댓글 작성(`commentId` 생략) 또는 갱신(`commentId` 지정), `contentFormat: "adf"`
- `getTransitionsForJiraIssue` — 이슈에서 가능한 상태 전환 목록 조회(QA 모드 완료 단계)
- `transitionJiraIssue` — 이슈 상태 전환(QA 모드 완료 단계: "해결" 전환)
- `editJiraIssue` — 이슈 필드 수정(QA 모드 완료 단계: 담당자 재할당)
- (보조) `getAccessibleAtlassianResources` — cloudId 못 찾을 때

## 입력

- `ARGS`: Jira URL(`*/browse/ISSUE-KEY`) 또는 이슈키(`[A-Z][A-Z0-9]+-[0-9]+`). 미지정 시 현재 브랜치명에서 이슈키 추출 시도, 실패하면 사용자에게 질문.
- 모드 힌트: ARGS에 "QA"/"프론트"/"front"/"qa" 포함 시 자동 결정. 명시 플래그 `--qa`/`--front`도 인식.
- 변경 내용: 사용자가 설명했거나, 현재 작업 브랜치의 diff/컨트롤러/DTO에서 자동 도출한다.

## 절차

### Step 0: 모드 선택

ARGS/대화 맥락에서 모드를 판별한다.

- ARGS에 "QA"/"qa"/"프론트"/"front" 또는 `--qa`/`--front` 포함 → 해당 모드 자동 선택.
- 판별 불가 시 AskUserQuestion:
  > 어떤 모드로 댓글을 작성할까요?
  > 1. 프론트 API 핸드오프 — API 변경사항을 프론트 개발자에게 전달
  > 2. QA 핸드오프 — 버그 수정 완료를 QA 담당자에게 전달

모드 선택 후 Step 1로 진행.

---

### Step 1: Jira 이슈 해석 (두 모드 공통)

- URL이면 호스트(`{site}.atlassian.net`)를 그대로 `cloudId`로 사용한다(별도 조회 불필요). 실패 시에만 `getAccessibleAtlassianResources`.
- `getJiraIssue(cloudId, issueIdOrKey, fields:[summary,status])`로 이슈 제목 확인 후 사용자에게 한 줄 보고.

---

## 프론트 API 핸드오프 모드

### Step F-2: 변경 API 식별 + 신규/필드추가 분류

- 근거: MR/브랜치 diff(`git diff <base>...HEAD`), 컨트롤러(`@GetMapping`/`@PostMapping`), DTO(`@Schema`).
- 분류:
  - **신규(green)**: 새 엔드포인트(컨트롤러에 신규 메서드).
  - **필드추가(blue)**: 기존 엔드포인트 응답 DTO에 필드만 추가.
- API별 수집: HTTP 메서드+경로, 추가/주요 응답 필드명, **각 필드의 타입(String/Long/Object/List 등)과 한국어 설명**(DTO `@Schema(description=...)` 또는 코드 맥락에서).
- 배열 필드는 항목 하위 필드를 한 단계 더 들여써서 기술.

### Step F-3: Swagger 딥링크 산출 (tag + operationId)

- **가장 정확한 방법**: 서비스가 떠 있으면 OpenAPI 스펙에서 추출.
  ```bash
  curl -s "{SWAGGER_HOST}{contextPath}/v3/api-docs" \
    | python3 -c "import sys,json; d=json.load(sys.stdin); \
      [print(p, m, o.get('tags'), o.get('operationId')) \
       for p,ms in d['paths'].items() for m,o in ms.items()]"
  ```
  - 서버 미가동 시: operationId = 컨트롤러 메서드명(기본), tag = 컨트롤러 클래스명 또는 springdoc 기본(`xxx-controller`). 불확실하면 사용자에게 확인.
- **딥링크 형식**: `{SWAGGER_HOST}{contextPath}/swagger-ui/index.html#/{tag}/{operationId}`
  - 예: `https://test-api.example.com/{context-path}/swagger-ui/index.html#/{Controller}/{operationId}`
- `SWAGGER_HOST`/`contextPath`는 사용자에게 확인하거나 프로젝트 설정(`application.yml` `server.servlet.context-path`)에서 읽는다. **localhost 링크는 프론트에게 무의미하므로 지양**하고 test 호스트를 쓴다(모르면 질문).

### Step F-4: ADF 댓글 본문 작성

`contentFormat: "adf"`, `commentBody`는 ADF JSON 문자열. 아래 패턴을 그대로 사용한다.

**전체 구조**
```
doc
├─ heading(L3): "백엔드 수정사항 · {프로젝트명}"
├─ paragraph: 한 줄 요약 + "...Swagger 링크 참고 부탁드립니다."
├─ rule
├─ [신규 API들]  heading(L4: status뱃지+이름) → paragraph(설명+Swagger링크) → bulletList(필드)
├─ rule
├─ [필드추가 API들] 동일 구조
├─ rule
└─ paragraph: "확인 후 문의사항 있으시면 편하게 말씀 부탁드립니다."
```

**상태 뱃지(색상)**
```json
{"type":"status","attrs":{"text":"신규","color":"green"}}
{"type":"status","attrs":{"text":"필드추가","color":"blue"}}
```
heading 안에 뱃지+제목을 같이 넣는다:
```json
{"type":"heading","attrs":{"level":4},"content":[
  {"type":"status","attrs":{"text":"신규","color":"green"}},
  {"type":"text","text":" API 제목"}]}
```

**Swagger 링크(굵게)**
```json
{"type":"paragraph","content":[
  {"type":"text","text":"API 설명 한 줄.  "},
  {"type":"text","text":"→ Swagger 바로가기","marks":[
    {"type":"link","attrs":{"href":"{딥링크}"}},{"type":"strong"}]}]}
```

**필드 항목(코드체 필드명 + 타입 + 설명)**
```json
{"type":"listItem","content":[{"type":"paragraph","content":[
  {"type":"text","text":"fieldName","marks":[{"type":"code"}]},
  {"type":"text","text":" (String) — 필드 설명"}]}]}
```

**배열 하위 필드(중첩 bulletList)**: listItem.content = [paragraph(목록 필드 설명), bulletList(항목 필드들)]

### Step F-5: 작성/갱신

- 신규 작성: `addCommentToJiraIssue(cloudId, issueIdOrKey, contentFormat:"adf", commentBody)`
- 갱신: 같은 핸드오프 댓글을 고칠 때는 `commentId`를 함께 전달(전체 본문 교체 방식).
- 완료 후 `https://{site}.atlassian.net/browse/{KEY}` 링크와 요약 보고.

---

## QA 핸드오프 모드

### Step Q-2: 정보 수집

아래 3블록을 구성할 정보를 수집한다. 사용자가 제공한 내용을 우선 사용하고, 부족하면 질문하거나 플레이스홀더로 표시한다.

**원인 블록**
- 근본 원인: 어떤 로직/데이터 오류인지.
- 확인 근거: ELK 로그, 재현 시나리오 등. 사용자가 fetch-elk 결과나 로그를 제공했으면 인용.

**수정 블록**
- 수정 내용: 무엇을 어떻게 고쳤는지(diff/브랜치에서 도출 또는 사용자 설명).
- 배포 여부: "테스트 서버 배포 완료되었습니다" 정도의 **평문 한 문장**으로만 언급한다. **Jenkins 잡명·빌드번호·MR 번호를 이 블록에 넣지 않는다.**

**QA 확인 블록**
- 재현/검증 절차: 어떤 액션 → 어떤 화면/결과.
- 정상 판정 기준: 기대 동작(안내문구 노출 등) + "정상" 판정 조건.

**MR 링크 (푸터)**
- 댓글 **가장 마지막 줄**에 `개발자 확인용 MR : ` + MR 번호를 둔다. 번호에는 반드시 **하이퍼링크(link mark)** 를 건다.
- MR이 여러 개면 쉼표로 나열하고 각각 링크를 건다.
- MR URL은 `glab mr view`/`glab api` 또는 사용자가 준 링크에서 얻는다. URL을 모르면 사용자에게 묻고, 추측한 URL로 링크를 걸지 않는다.

### Step Q-3: ADF 댓글 본문 작성

`contentFormat: "adf"`. 아래 구조를 그대로 사용한다.

**전체 구조**
```
doc
├─ paragraph: [수정 완료] 상태 머리말 (status 뱃지)
├─ rule
├─ paragraph(strong "원인:") + 원인 내용
├─ paragraph(strong "수정:") + 수정 내용 + 배포 여부(평문)
├─ paragraph(strong "QA 확인:") + 검증 절차 + 정상 판정
├─ [선택] paragraph(strong "참고:") + 부가 유의사항
├─ rule
├─ paragraph: 마무리 인사
└─ paragraph(strong "개발자 확인용 MR :") + MR 번호(하이퍼링크)   ← 항상 맨 마지막
```

**`[수정 완료]` 상태 머리말**
```json
{"type":"paragraph","content":[
  {"type":"status","attrs":{"text":"수정 완료","color":"green"}}]}
```

**원인/수정/QA 확인 블록 (strong 라벨 + 본문)**
```json
{"type":"paragraph","content":[
  {"type":"text","text":"원인: ","marks":[{"type":"strong"}]},
  {"type":"text","text":"근본 원인 설명 (ELK 확인)."}]}
```

```json
{"type":"paragraph","content":[
  {"type":"text","text":"수정: ","marks":[{"type":"strong"}]},
  {"type":"text","text":"수정 내용. 테스트 서버 배포 완료되었습니다."}]}
```

> 이 블록에 `(Jenkins {JOB} #{빌드번호}, MR !{번호})` 같은 괄호 덩어리를 넣지 않는다. QA·기획이 읽는 본문에 내부 빌드 정보가 섞이면 읽기 흐름이 끊긴다. 개발자용 정보는 아래 푸터로 분리한다.

```json
{"type":"paragraph","content":[
  {"type":"text","text":"QA 확인: ","marks":[{"type":"strong"}]},
  {"type":"text","text":"{액션} → \"{안내문구}\" 노출 + {조건}이면 정상."}]}
```

**마무리 인사**
```json
{"type":"paragraph","content":[
  {"type":"text","text":"QA 확인 부탁드립니다."}]}
```

**개발자 확인용 MR 푸터 (댓글 맨 마지막 — 필수)**
```json
{"type":"paragraph","content":[
  {"type":"text","text":"개발자 확인용 MR : ","marks":[{"type":"strong"}]},
  {"type":"text","text":"!{번호}","marks":[{"type":"link","attrs":{"href":"{MR URL}"}}]}]}
```

MR 2건 이상일 때(작업 MR + 배포 MR 등):
```json
{"type":"paragraph","content":[
  {"type":"text","text":"개발자 확인용 MR : ","marks":[{"type":"strong"}]},
  {"type":"text","text":"!484","marks":[{"type":"link","attrs":{"href":"{MR1 URL}"}}]},
  {"type":"text","text":", "},
  {"type":"text","text":"!485","marks":[{"type":"link","attrs":{"href":"{MR2 URL}"}}]}]}
```

### Step Q-4: 작성/갱신

- 신규 작성: `addCommentToJiraIssue(cloudId, issueIdOrKey, contentFormat:"adf", commentBody)`
- 갱신(재배포 등): `commentId`를 함께 전달(전체 본문 교체).
- 게시 전 본문을 사용자에게 보여주고 확인받는 것을 권장한다(운영 이슈 외부 노출 행위).
- 완료 후 `https://{site}.atlassian.net/browse/{KEY}` 링크와 요약 보고.

### Step Q-5: 완료 단계 — 상태 전환 + 담당자 재할당

댓글 작성(Step Q-4) 완료 후, 사용자에게 **한 번에** 확인한다.

> 다음 작업을 진행할까요?
> - 상태를 "해결"로 변경
> - 담당자를 보고자(`{reporter 표시이름}`)로 재할당
> (각각 스킵 가능합니다.)

사용자가 확인하면 아래 순서대로 실행. 거부하면 Step Q-4 결과 링크만 보고하고 종료.

#### 상태 → 해결

1. `getTransitionsForJiraIssue(cloudId, issueIdOrKey)`로 가능한 전환 목록을 조회한다.
2. 목록에서 이름이 "해결", "Resolved", "Done" 등 해결 계열인 전환의 `id`를 찾는다. **전환 id를 하드코딩하지 않는다.**
3. 매칭 전환 발견 시: `transitionJiraIssue(cloudId, issueIdOrKey, transition:{id:"<전환id>"})`
4. 매칭 전환이 없거나 여러 개로 모호한 경우: 가능한 전환 목록(이름+id)을 사용자에게 보여주고 어떤 것으로 할지 묻는다.
5. 엣지케이스:
   - 이미 "해결" 상태인 경우 → "이미 해결 상태입니다"로 안내하고 스킵.
   - 권한 없어 실패 시 → 오류 내용 보고하고 담당자 재할당 단계는 계속 진행.

#### 담당자 → 보고자

1. `getJiraIssue(cloudId, issueIdOrKey, fields:["reporter"])`로 reporter의 `accountId`를 읽는다.
2. reporter가 있으면: `editJiraIssue(cloudId, issueIdOrKey, fields:{assignee:{accountId:"<reporter accountId>"}})`
3. 엣지케이스:
   - reporter가 없는 경우 → "보고자 정보가 없어 재할당을 건너뜁니다"로 안내.
   - reporter == 현재 담당자인 경우 → "이미 보고자가 담당자입니다"로 안내하고 스킵.
   - 권한 없어 실패 시 → 오류 내용 보고.

#### 결과 보고

변경된 항목(상태, 담당자)을 요약하고 이슈 링크(`https://{site}.atlassian.net/browse/{KEY}`)를 출력한다.

---

## 말투 규칙 (두 모드 공통)

- **공손한 존댓말**: "참고 부탁드립니다", "~추가되었습니다", "~반환합니다". 명령형("참고하세요") 금지.
- **이모지 금지.** 시각 강조는 상태 뱃지(색상)와 `→`, 구분선, 코드체로 처리.
- 마무리 인사:
  - 프론트 모드: "확인 후 문의사항 있으시면 편하게 말씀 부탁드립니다."
  - QA 모드: "QA 확인 부탁드립니다."

### 내부정보 포함 여부 (모드별 분기)

| 항목 | 프론트 API 모드 | QA 모드 |
|------|----------------|---------|
| MR 번호 | **제외** (내부 정보) | **맨 마지막 푸터에만** (하이퍼링크) |
| Jenkins 잡명·빌드번호 | **제외** (내부 정보) | **제외** |
| "배포 완료" 문구 | **제외** | **포함 필수** (평문, 수정 블록) |
| Swagger 딥링크 | **포함 필수** | 불필요(생략) |

프론트 모드에서는 MR 번호·배포 사정 등 내부 정보를 댓글에 적지 않는다(사용자가 명시 요청하면 예외).

QA 모드에서는 배포 여부는 본문에 밝히되, **MR 번호는 본문이 아니라 맨 마지막 푸터 한 줄로 분리**하고 Jenkins 잡명·빌드번호는 적지 않는다. 젠킨스 정보는 QA가 쓸 일이 없고 본문 가독성만 해친다.

## 작성 규칙

1. 신규 API는 주요 **응답 필드**를, 필드추가 API는 **추가된 필드**를 빠짐없이 나열한다(프론트 모드).
2. 모든 필드에 **타입을 표기**한다(신규/필드추가 일관). 배열은 `(List)` + 항목 하위필드.
3. 헤더·요청 파라미터 등 Swagger에 이미 있는 디테일은 과하게 적지 않는다(링크로 대체).
4. 신규/필드추가를 그룹으로 묶고 구분선(rule)으로 나눈다(프론트 모드).
5. 같은 이슈에 핸드오프 댓글이 이미 있으면 새로 달지 말고 `commentId`로 갱신한다.

## 주의사항

- cloudId는 사이트 호스트(`xxx.atlassian.net`)를 그대로 넘기면 대부분 동작한다.
- ADF JSON은 유효성이 엄격하다 — status/link/code 마크 구조를 위 패턴대로 유지.
- Swagger 딥링크의 `tag`는 springdoc 기본값이 컨트롤러마다 다를 수 있다(`{Controller}` vs `{controller-kebab}`). 반드시 `/v3/api-docs`로 실제 값을 확인한다.
- 운영 이슈에 댓글을 다는 것은 외부 노출 행위이므로, 본문을 사용자에게 보여주고 확인받은 뒤 게시하는 것을 권장한다.
