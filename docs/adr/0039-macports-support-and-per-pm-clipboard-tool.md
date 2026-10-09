# ADR 0039 — MacPorts support, and a package-manager-specific clipboard tool

- **Status:** accepted
- **Date:** 2026-10-09

## Context

`install.sh`'s macOS path was Homebrew-only: `detect_os()` required `brew` on
`PATH` or failed outright, and every macOS-only provisioning step (`pkg_install`
/ `pkg_update`, the Colima suggestion in `check_docker()`, and the clipboard
global-hotkey automation in `setup_clipboard_keybind_macos()`) assumed it. A
MacPorts-only Mac — no Homebrew installed — could not run `install.sh` at all.

Two things had to be solved to lift that restriction:

1. **Generic package installs** (`git`, `keychain`) just need a second
   `pkg_install`/`pkg_update` case — MacPorts packages these under the same
   names, so no per-package translation table was needed for anything
   `install.sh` currently routes through there.
2. **The clipboard-image global hotkey** (`boxa clip` bound to Ctrl+Shift+S,
   working even in Terminal.app) was built entirely on
   [Hammerspoon](https://www.hammerspoon.org/), a full Lua automation
   framework distributed only as a Homebrew cask. Hammerspoon has **no
   MacPorts port**, and nothing resembling one — a MacPorts-only install
   needed a different tool, not a MacPorts invocation of the same one.

[skhd](https://github.com/asmvik/skhd) — a minimal hotkey daemon, no scripting
layer, just `<keys> : <command>` lines in `~/.skhdrc` — turned out to be
packaged for **both** Homebrew (via a maintainer tap,
`asmvik/formulae/skhd`, not homebrew-core) and MacPorts (`sysutils/skhd`,
confirmed on ports.macports.org), as are `terminal-notifier` and `pngpaste`,
the two supporting tools the feature also installs. That made skhd-for-both
package-managers a candidate replacement for Hammerspoon everywhere — but
skhd is a strictly thinner tool: it has no IPC/introspection mechanism
equivalent to Hammerspoon's `hs.accessibilityState()`, which `install.sh` used
to tell whether Accessibility trust was already granted and skip a redundant
prompt. Switching every macOS install to skhd would have traded that precision
away for every existing Homebrew user, to solve a problem only MacPorts users
have.

## Decision

Branch on the detected package manager (`$PM`, set by `detect_os()`) at two
points:

**Package manager detection** (`detect_os()`) — Homebrew is preferred when
both are present, so an existing Homebrew install's behavior is unchanged
bit-for-bit; MacPorts is used only when Homebrew is absent:

```sh
if has brew; then
    PM="brew"
elif has port; then
    PM="macports"
else
    error "Homebrew or MacPorts is required on macOS. Install Homebrew from https://brew.sh or MacPorts from https://www.macports.org/install.php"
fi
```

`pkg_install()` / `pkg_update()` grew a `macports)` case alongside the
existing `brew)` one (`sudo port install "$pkg"` / `sudo port selfupdate`),
used by `install_git` and `install_keychain` without any other change — both
packages share the same name on MacPorts.

**Clipboard keybind** — kept as two separate, package-manager-dedicated
implementations rather than one skhd-only path, so Homebrew users keep the
exact Hammerspoon behavior (including the accessibility-state check) they had
before this change:

```sh
setup_clipboard_keybind_macos() {
    case "$PM" in
        brew)     setup_clipboard_keybind_macos_hammerspoon ;;
        macports) setup_clipboard_keybind_macos_skhd ;;
        *)        warn "Clipboard keybind needs Homebrew or MacPorts on macOS; neither was detected." ;;
    esac
}
```

- `setup_clipboard_keybind_macos_hammerspoon()` is the pre-existing
  implementation, unchanged: `brew install --cask hammerspoon`, a managed
  block in `~/.hammerspoon/init.lua`, and — because Hammerspoon's own `hs`
  CLI can query `hs.accessibilityState()` — an Accessibility prompt shown
  only when trust isn't already granted.
- `setup_clipboard_keybind_macos_skhd()` installs `skhd` via
  `sudo port install`, writes a managed block into `~/.skhdrc` binding
  Ctrl+Shift+S to `clip-image-inject.sh`, and starts/restarts skhd as a
  launchd service. Lacking Hammerspoon's introspection, it **always** opens
  the Accessibility pane and phrases the reminder as a thing to check, not an
  assertion that permission is missing. It also surfaces a second,
  skhd-specific requirement that Hammerspoon never had: Secure Keyboard Entry
  must be off in the terminal, or skhd receives no key events at all.

**Docker install suggestions** (`check_docker()`) — the Colima suggestion is
shown as a one-line `brew install` only when `$PM` is `brew`; on MacPorts it
instead points at the Colima repo directly with a note that Colima needs
Homebrew (it has no MacPorts port either), since Docker Desktop and OrbStack
above it need neither package manager and stay the lower-friction options on
a MacPorts-only machine.

`mkcert` was evaluated and needed **no change**: `install_mkcert()` never
goes through `pkg_install` — it delegates entirely to
`scripts/ensure-mkcert.sh` / `scripts/install-mkcert.sh`, which already
install from a pinned GitHub release with a SHA-256 table, independent of
either package manager.

## Consequences

**Positive:**

- `install.sh` now runs end-to-end on a MacPorts-only Mac — one with no
  Homebrew installed at all — with no behavior change for existing Homebrew
  installs.
- The clipboard keybind, Docker suggestions, and generic package installs are
  all package-manager-symmetric: every macOS code path in `install.sh` now
  has a MacPorts branch, not just a subset.
- Homebrew users keep Hammerspoon's precise accessibility-state detection
  rather than losing it to a lowest-common-denominator skhd-only rewrite.

**Negative / limitations:**

- Two clipboard-keybind implementations (`setup_clipboard_keybind_macos_hammerspoon`,
  `setup_clipboard_keybind_macos_skhd`) must now be kept in sync for any
  future change to the managed-block logic (marker format, symlink-safe
  writes, malformed-block detection) — they share the same shape by
  construction but are not the same code.
- MacPorts users get a strictly less precise Accessibility prompt (always
  shown, never confirmed-unnecessary) than Homebrew users, and one extra
  manual step (Secure Keyboard Entry) Homebrew users never see. This is an
  honest reflection of skhd's thinner feature set, not a bug to "fix" by
  further automation — skhd has no API to close the gap.
- Colima remains unreachable via `sudo port install` on a MacPorts-only
  machine; a user who wants it still needs to install Homebrew alongside
  MacPorts, or choose Docker Desktop / OrbStack instead.

## References

- `install.sh` — `detect_os()`, `pkg_install()`, `pkg_update()`,
  `check_docker()`, `setup_clipboard_keybind_macos()`,
  `setup_clipboard_keybind_macos_hammerspoon()`,
  `setup_clipboard_keybind_macos_skhd()`.
- `docs/clipboard-images.md` — the user-facing Hammerspoon / skhd setup guide,
  updated alongside this ADR.
- [skhd](https://github.com/asmvik/skhd) — the MacPorts-packaged hotkey daemon.
- [Hammerspoon](https://www.hammerspoon.org/) — the Homebrew-cask-only
  automation framework kept for Homebrew installs.
- MacPorts port pages confirming package availability: `sysutils/skhd`,
  `sysutils/pngpaste`, `terminal-notifier` (v3.1.0) on ports.macports.org.
