# ONE store Web SDK Skill

HTML5 웹게임에 [원스토어 H5 Game SDK](https://onestore-dev.gitbook.io/dev/tools/web-sdk)를 연동할 때 쓰는 [Agent Skill](https://agentskills.io)입니다.
AI 코딩 에이전트가 원스토어 연동 규칙에 맞춰 SDK 로드부터 결제 코드 작성, 릴리즈 전 점검까지 진행하도록 돕습니다.

[English](README.en.md)

## 설치

```sh
npx skills add ONE-store/onestore_web_sdk_skills
```

[`skills`](https://skills.sh) CLI가 프로젝트에서 쓰는 에이전트(Claude Code, Cursor, Codex, Gemini CLI, GitHub Copilot 등)를 찾아 설치합니다. 특정 에이전트에만 설치하려면 `-a claude-code`처럼 지정합니다. Node.js 22 이상이 필요합니다.

### 직접 복사

저장소를 받은 뒤 `skills/onestore-web-sdk` 폴더를 에이전트의 스킬 폴더에 넣습니다.

| 에이전트 | 프로젝트 경로 |
|---|---|
| Claude Code | `.claude/skills/` |
| Cursor, Codex, Gemini CLI, GitHub Copilot | `.agents/skills/` |
| Grok Build | `.grok/skills/` |

Node.js를 쓸 수 없는 환경에서는 `./install.sh <게임 프로젝트 경로>`로 위 세 경로에 한 번에 복사할 수 있습니다(bash 필요, Windows는 WSL 또는 Git Bash). 다시 실행하면 바뀐 파일만 갱신하고, 직접 수정한 설치본은 덮어쓰지 않습니다.

## 스킬

| 스킬 | 용도 |
|---|---|
| `onestore-web-sdk` | SDK 로드와 부팅 흐름(초기화·로딩·시작), 생명주기 처리, 인앱 결제·광고 코드 작성, iframe 임베드 요건 점검, 오류 진단, 릴리즈 전 점검 |

스킬은 요청 내용에 맞춰 자동으로 불러와집니다. "원스토어 SDK 붙여줘", "결제 붙여줘", "화면이 하얗게 나와"처럼 평소대로 요청하면 됩니다. 스킬 본문은 영어로 작성되어 있고, 에이전트는 요청한 언어로 답합니다.

작업을 바로 지정하려면 스킬 이름 뒤에 아래 단어를 붙여 부릅니다. Claude Code에서는 `/onestore-web-sdk doctor`처럼 입력합니다. 다른 에이전트에서는 그 도구의 스킬 호출 방식을 쓰거나, 같은 내용을 말로 요청하면 됩니다.

| 단어 | 하는 일 |
|---|---|
| `doctor` | 현재 연동 상태 확인 (파일은 수정하지 않음) |
| `integrate` | SDK 로드, 부팅 흐름, 생명주기 처리 |
| `iap` | 인앱 결제와 상품 정보 조회 |
| `ads` | 리워드·전면 광고 (실기기 검증 전) |
| `embed` | iframe 임베드 요건 점검 |
| `fix <오류 또는 증상>` | 오류 원인과 수정 방법 |
| `check` | 릴리즈 전 점검 |

## 준비할 것

연동에는 원스토어에서 발급·등록하는 값이 필요합니다. 값이 없어도 코드 작업은 진행되며, 비어 있는 값은 에이전트가 물어봅니다.

| 값 | 확인 위치 |
|---|---|
| SDK 버전 | [릴리즈 노트](https://onestore-dev.gitbook.io/dev/tools/web-sdk/releasenote) |
| 인앱 상품 ID | ONEconsole > Apps > (게임) > 수익화 > 인앱 상품 (관리형상품) |
| 광고 지면 ID | ONEconsole > Apps > (게임) > 수익화 > 인앱 광고에서 광고 유형별로 발급 요청 (광고를 쓰는 경우). 절차는 [ONEconsole 인앱 광고 안내](https://onestore-dev.gitbook.io/dev/docs/apps/product/monetization/iaa) 참고 |
| requestId 발급 방식 | 게임사 서버 설계 (리워드 광고를 쓰는 경우) |

원스토어 앱에서 테스트하려면 ONEconsole에 테스트 URL과 테스트 ID를 등록해야 합니다. 절차는 연동 가이드의 [07. 실행·테스트 환경](https://onestore-dev.gitbook.io/dev/tools/web-sdk/test)을 참고하세요.

## 참고 사항

- 에이전트가 작성한 코드는 반드시 검토하고 실제 단말에서 테스트한 뒤 사용하세요. 이 스킬은 연동 작업을 돕는 도구이며, 연동 결과나 심사 통과를 보장하지 않습니다.
- 아이템 지급은 게임사 서버에서 확정해야 합니다. 광고의 `rewarded` 응답이나 결제 성공 응답만으로 지급하지 마세요.
- 게임 서버는 원스토어 도메인의 iframe 임베드를 허용해야 합니다. 설정 방법은 연동 가이드의 [02. 사전준비](https://onestore-dev.gitbook.io/dev/tools/web-sdk/pre)를 확인하세요.
- 광고 연동 안내는 아직 실기기에서 검증하지 않았습니다. 광고를 쓴다면 릴리즈 전에 단말에서 꼭 확인하세요.
- 게임사 서버 구현(결제 검증, 포스트백 수신, 소비 처리)과 ONEconsole 등록 작업은 스킬 범위에 포함되지 않습니다.
- 스킬 내용과 [연동 가이드](https://onestore-dev.gitbook.io/dev/tools/web-sdk)가 다를 경우 연동 가이드를 따릅니다.

## 기준 버전

원스토어 H5 Game SDK v1.1.0, 연동 가이드 2026년 9월 18일 개정본 기준입니다.

## 문의

연동, 상품 등록, 심사 관련 문의는 원스토어 입점 담당자에게 해 주세요. 스킬 오류나 개선 제안은 [Issues](https://github.com/ONE-store/onestore_web_sdk_skills/issues)에 남겨 주세요.

## 라이선스

[LICENSE](LICENSE) 파일을 따릅니다.
