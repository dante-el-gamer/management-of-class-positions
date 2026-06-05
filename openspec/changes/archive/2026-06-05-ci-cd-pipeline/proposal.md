# Proposal: CI/CD Pipeline

## Intent

Automate building and publishing Tauri 2 installers for Windows and Linux on every tag push. Currently, releases are manual — no CI exists. This pipeline ensures consistent, reproducible builds and publishes to GitHub Releases.

## Scope

### In Scope
- GitHub Actions matrix build (`ubuntu-22.04`, `windows-latest`)
- Linux installers: `.deb` (Ubuntu/Debian) + `.AppImage` (Fedora/universal)
- Windows installers: `.msi` / `.exe` via NSIS
- Aggressive caching: `swatinem/rust-cache@v2` + `actions/setup-node` cache
- GitHub Release on tag push (`v*`)
- Single workflow file with matrix strategy

### Out of Scope
- macOS builds (deferred)
- Code signing (future)
- Test reporting dashboards
- Split test/build workflows (user preference recorded, not blocking)
- Docker-based cross-compilation

## Capabilities

### New Capabilities
- `ci-cd-pipeline`: GitHub Actions workflow for building Tauri installers and publishing to GitHub Releases

### Modified Capabilities
- None — CI/CD is infrastructure, no user-facing spec changes

## Approach

Single GitHub Actions workflow with a matrix strategy:
- **Matrix**: `os: [ubuntu-22.04, windows-latest]`
- **Trigger**: push to `main` + tag `v*` (release only on tag)
- **Steps per OS**: install system deps → setup Node → npm ci → Rust cache → build with `tauri-apps/tauri-action@vX`
- **Release**: `tauri-action` publishes artifacts to GitHub Releases automatically when triggered by a tag
- **GOOGLE_CLIENT_ID**: passed as `${{ secrets.GOOGLE_CLIENT_ID }}`, defaulting to a placeholder

## Affected Areas

| Area | Impact | Description |
|------|--------|-------------|
| `.github/workflows/release.yml` | New | Main CI/CD workflow definition |
| `tauri.conf.json` | Maybe modified | Verify `bundle.targets` includes `deb`, `appimage`, `msi` (currently `"all"`) |
| `.github/` directory | New | Entire CI structure |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Linux system deps drift | Medium | Pin Ubuntu version (`ubuntu-22.04`), install deps explicitly |
| Long build times (first run) | High | Rust + npm caching; subsequent runs are fast |
| Secret exposure in logs | Low | Use `${{ secrets.* }}` only; no debug logging in production |
| WebKit2GTK version mismatch | Medium | `ubuntu-22.04` ships a known-good version from Tauri docs |

## Rollback Plan

1. Delete `.github/workflows/release.yml`
2. Revert `tauri.conf.json` if `bundle.targets` was changed
3. Remove any GitHub Secrets if added (user-managed)
4. — Pipeline never runs, builds return to manual

## Dependencies

- GitHub repository with Actions enabled (already is)
- `GOOGLE_CLIENT_ID` configured as a GitHub secret (will use placeholder until configured)
- `GITHUB_TOKEN` automatically available (no extra config)

## Success Criteria

- [ ] Linux workflow produces `.deb` and `.AppImage` artifacts
- [ ] Windows workflow produces `.msi` and/or `.exe` installer
- [ ] All artifacts attached to a GitHub Release when a `v*` tag is pushed
- [ ] Cached runs complete in under 10 minutes
