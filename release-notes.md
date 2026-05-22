# Release Notes

## Metadata
- Generated at (UTC): 2026-05-10 18:10:17 UTC
- Release tag: auto-generated

## Summary
Detected 10 changed files in working tree.

## Changes By Area
- Docs:
  - docs/generated/README.md
  - docs/generated/adr-candidates.md
  - docs/generated/api-spec.md
  - docs/generated/application-spec.md
  - docs/generated/deployment-architecture.md
  - docs/generated/release-notes.md
  - docs/generated/sequence-flow.md
  - docs/generated/workflow-compliance.md
- Other:
  - gitignore
  - scripts/notify_webhook.py

## Breaking Changes
- Review API contract changes before release.

## Migration Notes
- Verify environment variables and queue mode configuration.

## Known Issues
- Add manual review for any generated breaking change candidates.

## Commits Included
- 75d8683 | 2026-05-11 | nkokkoyit | chore: ignore all test files
- 15e7ec1 | 2026-05-11 | nkokkoyit | chore: ignore and untrack all test files
- 99cd6a4 | 2026-05-11 | nkokkoyit | chore: ignore test result folders, db files, debug scripts
- 0a33005 | 2026-05-11 | nkokkoyit | Update marketing instruction
- 92592ca | 2026-05-08 | nkokkoyit | add health check llm
- 7bf008a | 2026-05-08 | nkokkoyit | chore: sync all files from temp branch
- 7b354de | 2026-05-07 | nkokkoyit | update config docker
- 9ff34f9 | 2026-05-07 | nkokkoyit | change postgres config
- 9e0714b | 2026-05-06 | nkokkoyit | update frontend with default user
- 037cd40 | 2026-05-06 | nkokkoyit | update crew-version
- a949380 | 2026-05-06 | nkokkoyit | new code
- 7aca9b1 | 2026-05-05 | IT Tráº§n Táº¥t Tháº¯ng | docs: add integration guide for HTTP API and queue consumers
- 789b0f0 | 2026-05-01 | IT Tráº§n Táº¥t Tháº¯ng | docs: update code review report with post-fix resolution status
- 3a84d22 | 2026-05-01 | IT Tráº§n Táº¥t Tháº¯ng | fix: address code review issues (C1, C2, I1, I3, I4)
- a873a89 | 2026-05-01 | IT Tráº§n Táº¥t Tháº¯ng | docs: document multi-LLM provider feature in README
- 7ec5060 | 2026-05-01 | IT Tráº§n Táº¥t Tháº¯ng | feat: LLMRegistry startup init + YAML loader llm/llm_provider fields
- 5a58582 | 2026-05-01 | IT Tráº§n Táº¥t Tháº¯ng | feat: admin endpoints for LLM provider CRUD management
- 4383f31 | 2026-05-01 | IT Tráº§n Táº¥t Tháº¯ng | feat: refactor llm_client to delegate to LLMRegistry + litellm
- 44b7d83 | 2026-05-01 | IT Tráº§n Táº¥t Tháº¯ng | fix: LLMRegistry thread-safety, backward compat for raw model names, DB error logging
- e05eac3 | 2026-05-01 | IT Tráº§n Táº¥t Tháº¯ng | feat: implement LLMRegistry for multi-provider resolution
