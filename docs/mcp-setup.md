# MCP Setup

5종 MCP(GitHub / Slack / Notion / Google Drive / HubSpot) 인증 절차. 각 사용자별 1회.

## 사전 조건

- `claude /plugin install claude-code-toolkit` 완료 (README §설치 참조)
- (선택) 보안 baseline 선행 적용 — `enableAllProjectMcpServers: false` 환경에서도 plugin이 정의한 `.mcp.json`은 plugin 컨텍스트에서 자동 활성화됨

## 1. GitHub PAT (stdio)

1. https://github.com/settings/personal-access-tokens/new 에서 **Fine-grained PAT** 발급
2. Repository access: 사용 repo만 명시 (`All repositories` 금지)
3. Permissions:
   - Contents: read
   - Pull requests: read+write (pr-pattern-auditor PR comment 게시용)
   - Issues: read (pr-storybook PR 메타 fetch용)
   - Metadata: read
4. shell 환경변수 export:
   ```
   # ~/.zshrc 또는 ~/.bashrc
   export CLAUDE_CODE_TOOLKIT_GITHUB_PAT="github_pat_..."
   ```
5. shell 재시작 후 `claude /mcp` 명령으로 `mcp__github__*` 도구 노출 확인

## 2. Slack OAuth (http)

1. Slack workspace admin이 plugin OAuth 앱 등록 — scope:
   - `chat:write` (pr-storybook 채널 포스팅)
   - `channels:history`, `groups:history` (메시지 읽기)
   - `channels:read` (채널 목록)
   - `search:read` (decision-log-keeper, deal-context-loader)
   - `users:read` (메시지 작성자 display_name 조회)
2. 사용자 셀프: `claude /mcp slack` 명령 → OAuth 브라우저 흐름 → Slack workspace 접근 승인
3. 인증 후 `mcp__slack__*` 도구 노출 확인

## 3. Notion 통합 (http)

1. Notion workspace 관리자가 plugin Internal Integration 등록 — capability:
   - Read content
   - Update content (decision-log-keeper, spec-from-thread DB row/page 추가)
   - Insert content
2. 통합이 접근할 페이지/DB 명시 권한 부여:
   - "Decision Log" DB (decision-log-keeper)
   - "Spec" 부모 페이지 (spec-from-thread)
   - 프로젝트 페이지 (notion-scout)
3. 사용자 셀프: `claude /mcp notion` 명령 → OAuth → 워크스페이스 선택
4. 인증 후 `mcp__notion__*` 도구 노출 확인

## 4. Google Drive (선택, **기본 비활성화**)

> **현재 상태: `.mcp.json`에서 분리됨.** Composio 3rd-party 브로커 경유 risk로 보안 검토 전까지 활성화 금지.

활성화 절차 (검토 후만):
1. 보안 검토 — Composio (`mcp.composio.dev`) 브로커 경유로 OAuth refresh token이 외부 인프라에 위탁되는 risk 검토 + 승인
2. `.mcp.google-drive.optional.json`의 `mcpServers.google-drive` 블록을 `.mcp.json`의 `mcpServers`에 머지
3. `claude /plugin` 재시작 후 `claude /mcp google-drive` → 브라우저 OAuth 승인
4. 인증 후 `mcp__google-drive__*` 도구 노출 확인
5. **사용 제한:** 중요 문서(계약서, 내부 OKR 등)는 Notion/Slack 우선. 외부 공유 자료 읽기 위주
6. **사용 종료 시 토큰 revoke 필수:** https://composio.dev → connected accounts → Google Drive → revoke
7. Anthropic-native GDrive MCP GA 시 즉시 교체 + Composio 측 토큰 revoke

## 5. HubSpot OAuth (http, **Drift H**)

1. **GA 신생 (2026-04-13):** 도구 ID drift 가능성 — 첫 호출 전 dry-run으로 노출 확인 필수
2. HubSpot 계정 admin이 plugin OAuth 앱 승인
3. 사용자 셀프: `claude /mcp hubspot` 명령 → OAuth → HubSpot portal 접근 승인
4. 인증 후 `mcp__hubspot__*` 도구 노출 확인 — `mcp__hubspot__get_deal`, `mcp__hubspot__search_companies` 등 실제 도구 ID 캡처
5. battle-card-builder / deal-context-loader 사용 전, 도구명이 SKILL.md 가정과 일치하는지 1회 점검 (drift 시 plugin patch release)

## 검증

5종 인증 후:

```
claude /mcp
```

다음 도구 prefix가 모두 노출되어야 함:
- `mcp__github__*`
- `mcp__slack__*`
- `mcp__notion__*`
- `mcp__google-drive__*`
- `mcp__hubspot__*`

prefix 누락 시 README §트러블슈팅 참조.

## OAuth 토큰 저장 위치

Claude Code는 OAuth 토큰을 macOS Keychain (또는 Linux Secret Service)에 저장. 사용자별 격리 — 다른 사용자 셸에서 접근 불가. 토큰 회전 시 `claude /mcp <server> --re-auth`.

## 보안 책임 매트릭스

| 항목 | 보안 담당 | 사용자 |
|---|---|---|
| GitHub PAT scope 정책 | 정의 | 발급/관리 |
| Slack/Notion/HubSpot OAuth 앱 등록 | 1회 등록 | 셀프 OAuth |
| Google Drive Composio trust 검토 | 1회 검토 + 갱신 주기 정의 | 셀프 OAuth |
| Notion 페이지/DB 권한 | 분배 | 사용 |
