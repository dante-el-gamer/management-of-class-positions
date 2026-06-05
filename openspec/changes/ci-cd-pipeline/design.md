# Design: CI/CD Pipeline

## Technical Approach

Single GitHub Actions workflow with an OS matrix strategy. On tag push (`v*`), both Linux and Windows runners install system dependencies, cache Rust + npm artifacts, build the frontend, and run `tauri-apps/tauri-action@v2` to produce platform installers. The action attaches artifacts to a GitHub Release automatically.

No changes to `tauri.conf.json` — `"targets": "all"` already covers `.deb`, `.AppImage` (Linux) and NSIS (Windows).

## Architecture Decisions

| Decision | Choice | Alternatives | Rationale |
|----------|--------|-------------|-----------|
| Build strategy | Matrix (native OS runners) | Cross-compile from Linux | Matrix produces native binaries matching the target OS; cross-compiling Tauri (WebKit2GTK, WinUI) is brittle and unsupported |
| Linux runner | `ubuntu-22.04` | `ubuntu-24.04` | Ubuntu 22.04 is the Tauri-recommended version with known-good WebKit2GTK. 24.04 ships a newer WebKit that may break bundled Tauri 2 |
| Rust caching | `swatinem/rust-cache@v2` | Manual `~/.cargo` caching | Zero-config, picks up `Cargo.lock` changes automatically, shares cache across matrix jobs |
| Node caching | `actions/setup-node` built-in cache | Manual `~/.npm` caching | Single config line, saves `node_modules` install time on cache hit |
| Release trigger | Tag push (`v*`) + release creation | Manual workflow_dispatch only | Tag triggers are deterministic and standard for semantic releases; `workflow_dispatch` will also be available for testing |
| tauri-action version | Pinned (`@v2`) | Floating (`@v2`) or latest | Pinning prevents surprise breakage from action updates; use Dependabot for version bumps |
| bundle targets | Leave `"all"` in tauri.conf.json | Pin specific targets per OS | `"all"` already produces correct formats per platform (deb + appimage on Linux, NSIS on Windows). No functional change needed |

## Data Flow

```text
git push tag v*
      │
      ▼
GitHub Actions trigger (on: push: tags: v*)
      │
      ├── ubuntu-22.04 runner          ├── windows-latest runner
      │                                 │
      ├── Install deps                  ├── No system deps needed
      │   (libwebkit2gtk, libgtk, etc)  │
      ├── actions/setup-node + cache    ├── actions/setup-node + cache
      ├── npm ci                        ├── npm ci
      ├── npm run build (Vite)          ├── npm run build (Vite)
      ├── swatinem/rust-cache@v2        ├── swatinem/rust-cache@v2
      └── tauri-apps/tauri-action@v2    └── tauri-apps/tauri-action@v2
                │                                    │
                └────── GitHub Release ──────────────┘
                       Artifacts: .deb, .AppImage, .msi/.exe
```

## File Changes

| File | Action | Description |
|------|--------|-------------|
| `.github/workflows/release.yml` | Create | Main CI/CD workflow — matrix build + release on tag push |
| `src-tauri/tauri.conf.json` | None | `"targets": "all"` already covers all required formats |

## Interfaces / Contracts

### GitHub Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `GOOGLE_CLIENT_ID` | Yes | OAuth client ID for Google Drive integration; set `your-client-id.apps.googleusercontent.com` as placeholder |

The `GITHUB_TOKEN` is automatically available and has permission to create releases.

### Environment Variables (workflow)

| Variable | Source | Used By |
|----------|--------|---------|
| `GOOGLE_CLIENT_ID` | `${{ secrets.GOOGLE_CLIENT_ID }}` | Tauri app via env at build time |
| `GITHUB_TOKEN` | `${{ github.token }}` | `tauri-action` for release creation |

## Testing Strategy

| Layer | What to Test | Approach |
|-------|-------------|----------|
| CI workflow | Syntax validity | `act` (local) or push a branch and run via `workflow_dispatch` |
| Integration | Full build cycle | Run `workflow_dispatch` manually, verify artifacts attach to draft release |
| Verification | Installer smoke test | Download `.deb` / `.AppImage` / `.exe`, install on clean OS, verify app launches |
| Cache | Second-run speed | Push a second tag without dep changes — should complete under 10 min |

## Migration / Rollout

1. Create `.github/workflows/release.yml`
2. Add `GOOGLE_CLIENT_ID` as repository secret (placeholder value)
3. Run via `workflow_dispatch` on `main` to verify Linux + Windows both build
4. Push a test tag (`v0.1.0-test`) to verify release creation
5. Delete the test tag and release

**Rollback**: Delete the workflow file and remove the secret. No code changes needed anywhere else.

## Open Questions

None.
