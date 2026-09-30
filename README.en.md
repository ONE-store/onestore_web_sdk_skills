# ONE store Web SDK Skill

An [Agent Skill](https://agentskills.io) for integrating the [ONE store H5 Game SDK](https://onestore-dev.gitbook.io/dev/eng/tools/web-sdk) into HTML5 web games.
It helps AI coding agents follow ONE store's integration rules, from loading the SDK to writing purchase code and running a pre-release check.

[한국어](README.md)

## Installation

```sh
npx skills add ONE-store/onestore_web_sdk_skills
```

The [`skills`](https://skills.sh) CLI detects the agents used in your project (Claude Code, Cursor, Codex, Gemini CLI, GitHub Copilot, and others) and installs the skill for them. To target a specific agent, pass `-a claude-code`, for example. Requires Node.js 22 or later.

### Manual installation

Clone this repository and copy the `skills/onestore-web-sdk` folder into your agent's skills directory.

| Agent | Project path |
|---|---|
| Claude Code | `.claude/skills/` |
| Cursor, Codex, Gemini CLI, GitHub Copilot | `.agents/skills/` |
| Grok Build | `.grok/skills/` |

If you can't use Node.js, `./install.sh <path to game project>` copies the skill into all three paths at once (requires bash; on Windows, use WSL or Git Bash). Running it again updates only changed files and leaves copies you have edited untouched. The script's messages are in Korean; run `./install.sh --help` for the options.

## Skills

| Skill | Use for |
|---|---|
| `onestore-web-sdk` | Loading the SDK and wiring the boot flow (init, loading, start), lifecycle handling, in-app purchase and ad code, iframe embedding checks, error diagnosis, pre-release checks |

The skill loads automatically when a request matches, so you can ask as usual: "add the ONE store SDK", "add in-app purchase", "the screen is blank". The agent answers in the language you use.

To pick a task directly, add one of these words after the skill name. In Claude Code, type `/onestore-web-sdk doctor`, for example. In other agents, use that tool's way of invoking a skill, or ask for the same thing in plain language.

| Word | What it does |
|---|---|
| `doctor` | Checks the current integration status (does not modify files) |
| `integrate` | SDK load, boot flow, lifecycle handling |
| `iap` | In-app purchase and product lookup |
| `ads` | Rewarded and interstitial ads (not yet verified on a device) |
| `embed` | iframe embedding checks |
| `fix <error or symptom>` | Cause and fix for an error |
| `check` | Pre-release check |

## Before you start

The integration needs values issued or registered on the ONE store side. The agent can place the code without them and will ask for anything missing.

| Value | Where to find it |
|---|---|
| SDK version | [Release notes](https://onestore-dev.gitbook.io/dev/eng/tools/web-sdk/releasenote) |
| In-app product ID | ONEconsole > Apps > (your game) > Monetization > In-App Products (managed products) |
| Ad placement ID | ONEconsole > Apps > (your game) > Monetization > In-App Ads, one per ad type (if you use ads). See [ONEconsole: In-App Ads](https://onestore-dev.gitbook.io/dev/eng/docs/apps/product/monetization/iaa) |
| How `requestId` is issued | Your server design (if you use rewarded ads) |

To test inside the ONE store app, register a test URL and a test ID in ONEconsole. See [07. Runtime and Test Environment](https://onestore-dev.gitbook.io/dev/eng/tools/web-sdk/test) in the integration guide.

## Notes

- Review any code the agent writes and test it on a real device before shipping. This skill assists with integration work; it does not guarantee the result or that your submission passes review.
- Grant items on your server. Do not grant them based only on a `rewarded` ad result or a successful purchase response.
- Your game server must allow iframe embedding from ONE store domains. See [02. Pre Preparations](https://onestore-dev.gitbook.io/dev/eng/tools/web-sdk/pre) in the integration guide.
- The ad integration guidance has not been verified on a device yet. If you use ads, test them on a device before release.
- Server-side work (purchase verification, postback handling, consumption) and ONEconsole registration are outside the skill's scope.
- If the skill and the [integration guide](https://onestore-dev.gitbook.io/dev/eng/tools/web-sdk) disagree, follow the guide.

## Baseline

ONE store H5 Game SDK v1.1.0, integration guide revised September 18, 2026.

## Support

For integration, product registration, or review questions, contact your ONE store partner manager. Report skill bugs or suggestions in [Issues](https://github.com/ONE-store/onestore_web_sdk_skills/issues).

## License

See [LICENSE](LICENSE).
