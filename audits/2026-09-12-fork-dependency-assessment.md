# Tauri fork: security seriousness and Linux dependency assessment

Date: 2026-09-12. Scope: the six forks in this session (`tauri`, `tao`, `wry`, `muda`, `tray-icon`, `tauri-action`) at their current heads, which are byte-identical to the upstream `tauri-apps` heads on the same day.

## 1. Verdict

Maintaining a Tauri fork that is serious about security is a **small code commitment and a large ownership commitment**. The code that needs to change is contained. What makes it serious is that the change is a coordinated semver-major across five crates plus three binding crates that are not in this fork set, and every upstream release afterwards has to be re-merged over that divergence.

Four findings drive that verdict:

1. **Only one advisory is still active on the Linux stack.** All eleven "GTK3 bindings unmaintained" advisories were withdrawn in the RustSec database on 2026-08-14. The remaining one is RUSTSEC-2024-0429, the `glib::VariantStrIter` unsoundness in `glib` 0.18.5, patched in 0.20.0.
2. **The unsound code is unreachable from Tauri's own code.** `VariantStrIter` is constructed by exactly one function, `Variant::array_iter_str()`. That function has zero callers in the `glib` 0.18 sibling crates, zero in the WebKit, JavaScriptCore, Soup, AppIndicator, and GDK-X11 bindings, and zero in the source of `tao`, `wry`, `muda`, `tray-icon`, or `tauri`. An application is affected only if the application or a plugin calls `array_iter_str` itself. The maintainer's "no incidents in four years" was the wrong evidence, but the conclusion holds.
3. **The blocker is not the open `tao` pull request.** Upstream `tao` PR #1331 is a one-line Renovate bump of `gtk` to 0.19 that leaves `gdkx11-sys` and `gdkwayland-sys` at 0.18. It is not a remediation effort. The real choke point is `webkit2gtk-rs` 2.0.2 (December 2025), which pins `gtk ^0.18`, `javascriptcore-rs =1.1` (October 2023), and `soup3 ^0.5`. `wry` and `tauri` pin `webkit2gtk =2.0.2` exactly. Nothing on the webview path can leave `glib` 0.18 until those bindings are regenerated.
4. **The `tao` half is small.** A scratch copy of `tao` moved to `gtk` 0.19 resolves to a single `glib` 0.22 universe with no 0.18 leftovers, and drops the unmaintained `proc-macro-error` crate as a side effect. The first compile produced nine errors, five of which are import path changes. After fixing those, every remaining error is the removal of `glib::MainContext::channel`, used in three files. That is a contained refactor.

Upstream's process is better than its reputation suggests: every Rust repo runs `cargo audit` daily via `rustsec/audit-check`, `tauri` maintains a `cargo vet` configuration importing six third-party audit sets, and Renovate enforces a three-day minimum release age. The gap is that the audit is visible but tolerated: `tauri` explicitly ignores RUSTSEC-2024-0429 in `.cargo/audit.toml` with the note "fixed by updating to gtk4", and the other four repos surface it as a warning that does not fail CI.

## 2. What "serious" costs, by tier

| Tier | What it means | Cost | Sustainable solo? |
|---|---|---|---|
| A. Policy fork | Track upstream, make `unsound` and `unmaintained` fail CI, publish a reachability statement for each ignored advisory, port upstream security fixes within days, fix the `tauri-action` JavaScript chain | Days, then hours per month | Yes |
| B. Linux binding fork | Regenerate `webkit2gtk-rs`, `javascriptcore-rs`, `libappindicator-rs` against `gtk` 0.19 / `glib` 0.22, then bump `tao`, `wry`, `muda`, `tray-icon`, `tauri` | Weeks. Semver-major on all five crates. Every Linux plugin that touches `gtk` breaks. Permanent merge burden | Only if upstreamed |
| C. GTK4 / WebKitGTK 6.0 | New windowing and webview backends | Tauri v3 scope | No |

**Recommendation.** Do Tier A now in the fork. Do Tier B as an upstream-first pull request series, using the fork only as the staging area: upstream has shown it will accept this work (the withdrawal PR was filed by a Tauri maintainer, and Renovate is already proposing the `gtk` bump). Do not fork for Tier C.

Why Tier B cannot stay a private patch: `tauri` exposes `gtk` and `webkit2gtk` types in its public API (`Window::gtk_window()`, `Window::default_vbox()`, `Webview::inner() -> webkit2gtk::WebView`), and `tauri-runtime-wry` re-exports `tao` and `wry`. A binding bump changes the types plugins compile against, so a fork carrying it is a different ecosystem, not a patched one.

## 3. Dependency analysis

### 3.1 Advisory results per repository

`cargo-audit` 0.22.2 against the RustSec database at commit `b50980a` (2026-09-09), run on each committed `Cargo.lock` without the repo's ignore list. Yanked-crate checks were unavailable offline.

| Repo | Packages | Vulnerabilities (shipped path) | Vulnerabilities (dev or tooling only) | Warnings |
|---|---|---|---|---|
| `tauri` | 1090 | none | `rsa` 0.9.10 RUSTSEC-2023-0071 (via `tauri-cli` and `tauri-bundler`, build tooling) | `glib` unsound; 7 unmaintained: `difference`, `fxhash`, `paste`, `proc-macro-error`, `rustls-pemfile`, `rustybuzz`, `ttf-parser`; `rand` 0.7 unsound |
| `tao` | 338 | none | `quick-xml` 0.39.2 RUSTSEC-2026-0194 and 0195 (dev only) | `glib` unsound; `paste`, `proc-macro-error` unmaintained |
| `wry` | 419 | `time` 0.3.37 RUSTSEC-2026-0009 via `cookie` (ignored upstream, cited MSRV 1.85) | `quick-xml` 0.37.2 RUSTSEC-2026-0194 and 0195 (dev only) | `glib` unsound; `paste`, `proc-macro-error`, `ttf-parser` unmaintained |
| `tray-icon` | 518 | none | `quick-xml` 0.30.0 and 0.36.2 (dev only); `webbrowser` 1.0.3 RUSTSEC-2026-0257 (dev only) | `glib` unsound; `paste`, `proc-macro-error`, `ttf-parser` unmaintained |
| `muda` | 440 | none | none | `glib` unsound; `proc-macro-error`, `ttf-parser` unmaintained |

Ignore lists in `.cargo/audit.toml`: `tauri` ignores RUSTSEC-2023-0071, 2020-0095, 2024-0370, 2026-0097, and 2024-0429. `wry` ignores 2026-0009, 2026-0194, 2026-0195. `tao`, `tray-icon`, `muda` ignore 2026-0194 and 2026-0195.

### 3.2 The GTK3 stack: locked versus available

| Crate | In every lock | Latest on crates.io | Status |
|---|---|---|---|
| `gtk`, `gdk`, `atk`, `gdkx11`, `gtk-sys` | 0.18.2 | 0.19.0 (2026-09-08) | Upgrade available |
| `glib`, `gio`, `cairo-rs`, `pango` | 0.18.x | 0.22.9 (2026-08-30) | Upgrade available; 0.20+ fixes RUSTSEC-2024-0429 |
| `gdk-pixbuf` | 0.18.5 | 0.22.0 | Upgrade available |
| `soup3` | 0.5.0 | 0.9.0 (2026-03-08, `glib` 0.22) | Upgrade available |
| `webkit2gtk`, `webkit2gtk-sys` | 2.0.2 | 2.0.2 (2025-12-16) | **Blocker.** Requires `gtk ^0.18`, `javascriptcore-rs =1.1`, `soup3 ^0.5`. Upstream `master` unchanged since the 2.0.2 release |
| `javascriptcore-rs` | 1.1.2 | 1.1.2 (2023-10-26) | **Blocker.** Requires `glib ^0.18` |
| `libappindicator`, `libappindicator-sys` | 0.9.0 | 0.9.0 (2023-10-01) | **Blocker** for the appindicator tray backend. Requires `gtk-sys ^0.18`. `tray-icon` already ships a pure-Rust `ksni` backend as the alternative |
| `x11-dl` | 2.21.0 | 2.21.0 (2023-01-18) | Stale but no advisory |

Where `glib` 0.18 enters each lock (normal dependencies): `tauri` through 12 crates, all on the Linux target; `wry` through 11; `tao` through 7. `muda` and `tray-icon` already carry `glib` 0.22 alongside 0.18 because their `gtk4` and `ksni` backends build against the current bindings. That means the team already writes code against the fixed API; only the GTK3 path is stuck.

### 3.3 Reachability of RUSTSEC-2024-0429

The advisory: `VariantStrIter::impl_get` passed `&p` where `&mut p` was required to a variadic C out-argument, so optimised builds can drop the write and dereference null. Affected `>=0.15.0, <0.20.0`, patched `0.20.0`.

Search performed on the exact sources in every lock:

| Where | Callers of `array_iter_str` or `VariantStrIter` |
|---|---|
| `glib` 0.18.5 itself | Definition plus its own unit tests only |
| `gio`, `gdk`, `gtk`, `pango`, `atk`, `gdk-pixbuf` 0.18 | none |
| `webkit2gtk` 2.0.2, `webkit2gtk-sys`, `javascriptcore-rs` 1.1.2, `soup3` 0.5.0, `libappindicator` 0.9.0, `gdkx11` 0.18.2 | none |
| `tauri` crates, `tao`, `wry`, `muda`, `tray-icon` source | none. The only `Variant` in `tao` is `zbus`'s D-Bus type, unrelated to `glib` |

Conclusion: the unsound path is reachable only from application or plugin code that calls `glib::Variant::array_iter_str()` directly. That is the documented reachability assessment the ignore entry should cite.

### 3.4 Compile experiment: `tao` on `gtk` 0.19

Scratch copy of `tao`, `Cargo.toml` changed to `gtk = "0.19"`, `gdkx11-sys = "0.19"`, `gdkwayland-sys = "0.19"`, then `cargo update` on those three crates.

- Lock resolves to a single set: `gtk` family 0.19.0, `glib` family 0.22.9, `gdk-pixbuf` 0.22.0. No 0.18 crates remain. `proc-macro-error` 1.0.4 (unmaintained) and `proc-macro-crate` 1.x drop out.
- First `cargo check`: 9 errors. `glib::Sender` no longer exists (4 sites), `Cast` and `IsA` moved to `gtk::prelude` (3 sites), `gtk::traits` became private (2 sites).
- After five one-line import fixes: 7 errors, all from `glib::MainContext::channel` being removed in `glib` 0.19. Sites: `platform_impl/linux/portal.rs`, `device.rs`, `event_loop.rs`, with the `Sender` type threaded through `window.rs`. Eleven usages in total. The replacement is `async-channel` plus `MainContext::spawn_local`, the pattern `muda`'s `gtk4` backend already uses.

No comparable experiment is possible for `wry` or `tauri` until `webkit2gtk-rs` is regenerated. Those bindings are 29,576 lines of `gir` output plus hand-written glue, so the regeneration is mechanical but must be verified against the `gir` version that produced `gtk3-rs` 0.19.

### 3.5 `tauri-action` (JavaScript)

`pnpm audit --prod` reports 6 vulnerabilities, all through `@actions/artifact`:

| Package | Installed | Patched | Path |
|---|---|---|---|
| `undici` (3 moderate: CRLF injection, cookie attribute injection, response desynchronisation) | 6.27.0 | ≥6.28.0 | `@actions/artifact` → `@actions/http-client` → `undici` |
| `brace-expansion` (3 high: exponential-time and unbounded expansion) | 2.1.0 | ≥2.1.4 | `@actions/artifact` → `archiver` → `archiver-utils` → `glob` → `minimatch` → `brace-expansion` |

This is release-pipeline code that handles upload of the built binaries. It is the one place in the set where a fix is cheap and entirely within the fork's control: bump `@actions/artifact` or add `pnpm.overrides`, rebuild `dist/`.

## 4. Concrete next steps in the fork

Tier A, in this order:

1. `tauri-action`: bump `@actions/artifact` and `undici` and `brace-expansion` via overrides, regenerate `dist/`.
2. All five Rust repos: switch the audit workflow to fail on `unsound` and `unmaintained` (`denyWarnings: true` on `rustsec/audit-check`, or `cargo audit --deny warnings`), and require every entry in `.cargo/audit.toml` to carry a reachability note and an expiry.
3. `tauri`: replace the "fixed by updating to gtk4" note on RUSTSEC-2024-0429 with the reachability finding in section 3.3 and a link to the tracking issue.
4. `wry`: decide `time` 0.3.47 versus MSRV 1.85 explicitly rather than ignoring it.
5. `tray-icon`: consider making `ksni` the default Linux backend so the stale `libappindicator` bindings become opt-in.

Tier B, upstream-first:

1. Regenerate `webkit2gtk-rs` and `javascriptcore-rs` against `gtk3-rs` 0.19 in forks of those repos (not in this set; add `tauri-apps/webkit2gtk-rs`, `tauri-apps/javascriptcore-rs`, `tauri-apps/libappindicator-rs`).
2. Land the `tao` refactor from section 3.4 as a real PR replacing #1331, with `gdkx11-sys` and `gdkwayland-sys` bumped together.
3. Bump `wry`, `muda`, `tray-icon`, then `tauri`, as one coordinated major.

## 5. Method and limits

- Sources read: the six forks at their current heads; the committed `Cargo.lock` of each; crates.io metadata for every GTK-family crate; the RustSec advisory database cloned at `b50980a`; upstream `tao` PR #1331 fetched by ref; upstream `webkit2gtk-rs` master cloned.
- Not reachable from this environment: rustsec.org pages and the GitHub API for `tauri-apps` repositories. Advisory status comes from the cloned database instead.
- Not assessed: WebKitGTK system library patch delivery on target distributions; runtime behaviour; Windows, macOS, Android, iOS dependency paths; Tauri's IPC and capability model beyond noting the May 2026 origin-confusion advisory.
- The compile experiment covers `tao` only and stops at the point where a real refactor is required. It measures the size of the gap; it is not a patch.
