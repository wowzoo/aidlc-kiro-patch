# AI-DLC v2 — Kiro 통합 harness 배포본 (patch 적용판)

순정 AI-DLC v2 **2.7.1** 배포본(`dist/kiro-ide`)에 **통합축**과 **patch** 두 층을 얹은 트리다.
**plugin 은 얹지 않았다.**

| | |
|---|---|
| 대상 harness | Kiro **IDE 1.x** + Kiro **CLI `--v3`** — 같은 `.kiro` 를 두 surface 가 그대로 쓴다 |
| 순정 버전 | **2.7.1** (`aidlc-version.ts` 의 `AIDLC_VERSION` 실측) |
| 순정 base 커밋 | **`a277af21`** — `awslabs/aidlc-workflows` 의 `main` |
| 파일 수 | **293** (순정 `dist/kiro-ide` 는 299) |
| 층 구성 | **둘** — ① 통합축(unify) → ② patch |
| `harness.json` 의 `name` | **`kiro-unified`** (순정은 `kiro-ide`) |
| plugin | **미적용** (`visual-mockups` · `code-map` 둘 다 없다) |
| 복사 시점 | 2026-09-04 KST |

## 층이 둘인 이유

```
순정 dist/kiro-ide @ a277af21  +  ① 통합축 spec  =  290파일   (상류 PR 에 내는 형태)
                              +  ② patch spec   =  293파일   (이 트리 · 우리가 쓰는 판)
```

**① 통합축**은 상류에 낼 수 있는 것만 담는다 — IDE 전용으로 갈라진 배포본을 **IDE 와 CLI 가 같은 `.kiro`
를 공유하는 형태**로 되돌리는 변경이다. 우리 자산이나 우리만의 결함 정정은 여기에 들어오지 않는다.

**② patch** 는 상류에 낼 수 없거나 아직 안 낸 것을 담는다 — 순정 산문 결함 정정, 엔진 결함 정정, 우리 자산.

## ① 통합축이 무엇을 하는가 (spec: `remove 12` · `patch 31파일` · `add 3파일`)

| 갈래 | 무엇 |
|---|---|
| **legacy hook 채널 제거** (12) | 순정 `dist/kiro-ide` 는 hook 을 **두 형식으로 중복 배포**한다 — 신세대 `.kiro/hooks/*.json` 11개와 구세대 `*.kiro.hook` 12개. 통합 트리는 `.kiro.hook` 을 **전부 걷어낸다**. 신세대 `.json` 하나가 두 surface 에 공통으로 먹기 때문이다 |
| **빠진 hook 배선 추가** (2) | 순정 `dist/kiro-ide` 에 `.json` 짝이 없는 `review-freeze` 와 `state-transition-guard` 를 신세대 형식으로 배선한다 |
| **CLI 쪽 MCP 등록 복원** (1) | 순정 `dist/kiro-ide` 에는 `.kiro/settings/mcp.json` 이 **없다**. 순정 `dist/kiro`(CLI) 의 것을 그대로 싣는다 — **byte-identical** 이므로 우리가 만든 구성이 아니다 |
| **surface 서술·배선 정합** (31) | agent `.md` 15 · protocol 4 · skill `SKILL.md` 7 · `aidlc-kiro-adapter.ts` · `aidlc-includes.ts` · `aidlc-utility.ts` · `harness.json` · `AGENTS.md` |

> `aidlc-utility.ts` patch 는 **doctor probe 두 개**를 더한다 — hook manifest 가 하나라도 있는지, 그리고
> 등록된 모든 hook command 가 실제 파일로 해석되는지. 아래 검증 절의 `40 → 42` 가 이 둘이다.

## ② patch 목록 — 2.7.1 기준 스냅샷

⚠️ **이것은 2.7.1 한 세대의 스냅샷이다.** 세대가 바뀌면 전 항목을 다시 잰다. 좌표(줄 번호)는 순정 또는
층1 기준이므로 이 트리에서 그대로 찾으면 어긋난다.

### 엔진 결함 정정 — `.ts` 코드 (8건)

| ID | 무엇 | 대상 파일 |
|---|---|---|
| F-8 | adapter 의 target 열거가 자기 구현 하나를 빠뜨린다 | `hooks/aidlc-kiro-adapter.ts` |
| F-9 | 정렬 안 된 audit row 가 run floor 를 어긋내 **유닛 완료 영수증을 전량 폐기**한다 | `tools/aidlc-lib.ts` |
| F-10 | Kiro 의 **읽기 전용 도구가 mutation-capable 로 분류**돼 code-generation 중 거절된다 | `hooks/aidlc-plan-approval-guard.ts` |
| F-11 | 엔진이 스스로 발행한 `load-steering` directive 중 **계획서 쓰기까지 거절**된다 | `hooks/aidlc-plan-approval-guard.ts` |
| F-12 | 턴 경계마다 규칙 전달이 part 1 로 되감기고, 그 재발행이 **Plan Approval 증거를 삭제**한다 | `tools/aidlc-orchestrate.ts` · `tools/aidlc-lib.ts` |
| F-13 | adapter 의 마지막 opaque-mutation 분기가 **경로 없는 도구를 전부 차단**한다 | `hooks/aidlc-kiro-adapter.ts` |
| F-14 | **숫자 승인(`1`/`2`)이 기록 전에 파괴**돼 엔진이 스스로 제공하는 단축키가 도달 불가다 | `hooks/aidlc-record-human-turn.ts` |
| F-15 | review 예산 가드가 「인자 한 칸 틀림」을 「예산 소진」으로 진단하고 **리뷰를 건너뛰라고 지시**한다 | `tools/aidlc-log.ts` |

> `F-14` 를 조금 더 풀면: 순정은 사람의 답을 먼저 `JSON.parse` 하는데 `"1"` 은 유효 JSON 이라 숫자 `1` 이
> 되고, 문자열도 객체도 아니므로 빈 문자열로 지워진다. 그러면 Plan Approval 응답이 아예 기록되지 않아
> **번호로 답하면 승인이 성립하지 않는다.** 이 patch 를 얹으면 번호가 통하고, 얹지 않은 순정에서는
> 제시된 라벨(`Approve Plan`)을 그대로 입력해야만 넘어간다. `F-15` 도 같은 성격이다 — 순정에서는 게이트
> 반려 뒤 재리뷰가 「예산 소진」으로 거절되어 컨덕터가 스스로 복구하지 못한다.

### 순정 산문 결함 정정 (6건)

| ID | 무엇 | 대상 파일 |
|---|---|---|
| F-1 | hook 계수가 helper 모듈까지 센다 | `AGENTS.md` |
| F-2 | stage 파일 경로 오기 | `AGENTS.md` |
| F-3 | 없는 문서 참조 | `AGENTS.md` |
| F-4 | heading 앞 빈 줄 결손 2곳 | `AGENTS.md` |
| F-5 | 있지도 않은 seam 이 없다고 단정한다 | `.gitignore` |
| F-7 | 구현이 없는 worktree merge dispatch 를 서술하는 고아 산문 | `knowledge/aidlc-shared/audit-format.md` · `knowledge/aidlc-pipeline-deploy-agent/branching-strategies.md` |

### 우리 자산 (3건)

| ID | 무엇 | 대상 |
|---|---|---|
| L-1 | `aidlc-git-merge` 스킬 (3파일) | `.kiro/skills/aidlc-git-merge/` |
| L-2 | `## Working Language` 절 | `AGENTS.md` 의 마지막 섹션 |
| L-3 | built-in `web` 도구 부여 | `.kiro/agents/aidlc.md` · `.kiro/agents/aidlc-product-agent.md` |

### 일부러 손대지 않은 것

| ID | 무엇 | 이유 |
|---|---|---|
| H-1 | 배포본의 bash rc 주장 | 전제가 미확인이라 HOLD |
| H-2 | `continue-workflow.json` 의 IDE 한정 서술 | HOLD |
| N-1 · N-2 | `AGENTS.md` 의 Plugins 절 · hook 계수 | 재측정에서 판정이 뒤집혀 순정이 맞다고 결론했다 (`NOT-A-DEFECT`) |

## 지원 surface

| surface | 지원 | 비고 |
|---|---|---|
| Kiro IDE **1.x** | ✅ | 이 트리를 그대로 쓴다 |
| Kiro CLI **`--v3`** | ✅ | 같은 트리를 그대로 복사해 쓴다 |
| Kiro IDE **0.x** | ❌ | agent 가 JSON-only 세대라 이 트리의 `.md` 자산이 돌지 않는다 |
| Kiro CLI **2.x classic / v2** | ❌ | 통합 형태가 그 엔진 세대에 서지 않는다 |

legacy 두 surface 를 써야 한다면 이 트리가 아니라 **순정 배포본을 그대로** 쓴다 — 통합축과 patch 는 그
세대를 대상으로 잰 것이 아니다.

## 쓰는 법

**전제** — `bun` 이 `PATH` 에 있어야 한다. 엔진 도구 전부가 `bun <tool>.ts` 로 돈다.

**설치** — 이 디렉터리의 내용을 프로젝트 루트에 그대로 둔다. 네 항목이다.

```
.kiro/        엔진 (tools · hooks · agents · skills · knowledge · settings · steering)
aidlc/        워크플로 데이터가 쌓이는 셸 (씨앗만 들어 있다)
AGENTS.md     harness 규약 문서
.gitignore
```

**진입점** — `/aidlc <하고 싶은 것>`.

## 검증

```bash
bun .kiro/tools/aidlc-utility.ts doctor
```

**42 passed, 0 failed** 이 나온다.

🔴 판정 기준은 이 절대값이 아니라 **등식**이다 — 같은 base 의 순정 `dist/kiro-ide` 는 **40 passed**이고,
차이 2가 통합축의 `aidlc-utility.ts` patch 가 더한 probe 두 개다(실측: 그 두 probe 를 제외한 우리 델타는
검사 수를 바꾸지 않는다 — 우리가 더한 hook 등록 2개를 지워도 42 그대로였다). 상류가 검사를 더하면 숫자는
그날 움직이니, 다음 세대에는 순정을 다시 재서 대조한다.

⚠️ **`doctor` 가 묻지 않는 것이 있다** — 우리 델타가 트리에 실제로 들어 있는지, hook 이 실제로 발화하는지,
통합축이 일관한지. 이 트리는 배포 전에 그 항목까지 전수 검증했다 — 2026-09-04 기준 두 층 모두 **PASS**.

## 주의

- **plugin 은 별도 작업이다.** `visual-mockups`(시각 mockup) 와 `code-map`(외부 코드맵) 은 이 트리에 없다.
  얹으면 MCP 서버가 5 → 7 로 늘고 `.aidlc-plugin/` 표시가 생기므로, 적용 여부는 그 둘로 판별한다.
- **배포된 hook body 가 전부 등록돼 있는 것은 아니다.** `ls .kiro/hooks/*.json` 이 live view 이고, 왜 일부가
  미등록인지는 이 트리의 `AGENTS.md` 가 설명한다(`aidlc-session-end.ts`·`aidlc-fold-usage.ts` 는 Claude Code
  전용 producer 다). 이름만 보고 누락이라 판정하지 않는다.
- 실런 데이터가 쌓이기 시작하면 `aidlc/` 아래가 그 프로젝트의 것이 된다. 다음 세대로 올릴 때 `aidlc/` 를
  덮어쓰지 않도록 주의한다.
