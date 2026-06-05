# Tasks: CI/CD Pipeline

## Review Workload Forecast

| Field | Value |
|-------|-------|
| Estimated changed lines | ~80 |
| 400-line budget risk | Low |
| Chained PRs recommended | No |
| Suggested split | Single PR |
| Delivery strategy | ask-on-risk |
| Chain strategy | pending |

Decision needed before apply: No
Chained PRs recommended: No
Chain strategy: pending
400-line budget risk: Low

### Suggested Work Units

| Unit | Goal | Likely PR | Notes |
|------|------|-----------|-------|
| 1 | CI/CD workflow + secret | PR 1 | main; single ~80-line file |

## Phase 1: Foundation

- [x] 1.1 Create `.github/workflows/` directory structure
- [x] 1.2 Add `GOOGLE_CLIENT_ID` as GitHub repository secret (placeholder value — documented in workflow YAML comment)

## Phase 2: Core Implementation

- [x] 2.1 Create `.github/workflows/release.yml` with trigger config (`push: main`, `tags: v*`, `workflow_dispatch`) and OS matrix (`ubuntu-22.04`, `windows-latest`)
- [x] 2.2 Add Linux system dependencies step: `libwebkit2gtk-4.1-dev`, `libgtk-3-dev`, `libayatana-appindicator3-dev`, `librsvg2-dev`, `libsoup-3.0-dev`, `libjavascriptcoregtk-4.1-dev`
- [x] 2.3 Add Node setup with caching: `actions/setup-node@v4`, `npm ci`, `npm run build`
- [x] 2.4 Add Rust caching: `swatinem/rust-cache@v2`
- [x] 2.5 Add `tauri-apps/tauri-action@v2` with release config, passing `GOOGLE_CLIENT_ID` from secrets

## Phase 3: Testing & Verification

- [ ] 3.1 Trigger `workflow_dispatch` on `main` to verify both Linux and Windows builds succeed
- [ ] 3.2 Push test tag `v0.1.0-test` to verify GitHub Release creation with artifacts attached
- [ ] 3.3 Download and smoke-test installers: `.deb`/`.AppImage` on Linux, `.msi`/`.exe` on Windows
- [ ] 3.4 Push second tag with no dependency changes to verify cache reduces build time under 10 min
- [ ] 3.5 Clean up: delete test tag, test release, and test artifacts
