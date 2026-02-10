# oh-syb-claude

Claude Code에서 사용할 수 있는 커스텀 commands와 agents 모음입니다.

## 설치 방법

### 전체 설치
```bash
# commands 폴더에 클론
git clone https://github.com/YoonbeomSo/oh-syb-claude.git ~/.claude/commands

# agents 폴더로 심볼릭 링크 생성
ln -s ~/.claude/commands/agents ~/.claude/agents
```

### 개별 파일 다운로드

```bash
# commands 폴더로 다운로드
curl -o ~/.claude/commands/<filename>.md <raw-url>

# agents 폴더로 다운로드
curl -o ~/.claude/agents/<filename>.md <raw-url>
```

---

## Commands

> 출처: [LOOPERS](https://www.loopers.im/)

| 파일 | 설명 | 다운로드 |
|------|------|----------|
| [requirements-analysis.md](./commands/requirements-analysis.md) | 요구사항 분석 도구. 애매한 요구사항을 질문/답변을 통해 명확히 하고, Mermaid 다이어그램(시퀀스, 클래스, ERD)으로 정리합니다. | [Raw](https://raw.githubusercontent.com/YoonbeomSo/oh-syb-claude/main/commands/requirements-analysis.md) |
| [worktree.md](./commands/worktree.md) | Git worktree 자동화 도구. 독립된 작업 디렉토리에서 기능 개발을 할 수 있도록 `/worktree create`, `/worktree list`, `/worktree remove`, `/worktree done`, `/worktree switch` 명령어를 제공합니다. | [Raw](https://raw.githubusercontent.com/YoonbeomSo/oh-syb-claude/main/commands/worktree.md) |

---

## Agents

> 출처: [Playwright Test Agents](https://playwright.dev/docs/test-agents)

| 파일 | 설명 | 다운로드 |
|------|------|----------|
| [playwright-test-planner.md](./agents/playwright-test-planner.md) | 웹 애플리케이션의 포괄적인 테스트 계획을 생성합니다. 브라우저를 탐색하며 사용자 흐름을 분석하고 테스트 시나리오를 설계합니다. | [Raw](https://raw.githubusercontent.com/YoonbeomSo/oh-syb-claude/main/agents/playwright-test-planner.md) |
| [playwright-test-generator.md](./agents/playwright-test-generator.md) | Playwright를 사용한 자동화된 브라우저 테스트를 생성합니다. 테스트 계획을 기반으로 실제 테스트 코드를 작성합니다. | [Raw](https://raw.githubusercontent.com/YoonbeomSo/oh-syb-claude/main/agents/playwright-test-generator.md) |
| [playwright-test-healer.md](./agents/playwright-test-healer.md) | 실패하는 Playwright 테스트를 디버깅하고 수정합니다. 셀렉터 변경, 타이밍 이슈, assertion 실패 등을 분석하고 해결합니다. | [Raw](https://raw.githubusercontent.com/YoonbeomSo/oh-syb-claude/main/agents/playwright-test-healer.md) |

---

## 사용법

### Commands
Claude Code에서 `/` 명령어로 실행:
```
/requirements-analysis
/worktree create feature-name
/worktree list
```

### Agents
Claude Code가 자동으로 적절한 상황에서 agent를 사용하거나, 직접 호출할 수 있습니다.

