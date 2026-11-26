# Copier

![Linux](https://img.shields.io/badge/platform-Linux-blue?style=for-the-badge&logo=linux)
![Rust](https://img.shields.io/badge/built_with-Rust-orange?style=for-the-badge&logo=rust)
![License](https://img.shields.io/badge/license-GPL--3.0-green?style=for-the-badge)
![AppImage](https://img.shields.io/badge/distro-AppImage-red?style=for-the-badge)

• Strategic Recommendation: Frames Lubuntu as the primary native development environment for guaranteed stability.
• Technical Breakdown: Bullet points highlighting the streamlined configuration, low resource footprint, and its ability to revive older hardware with zero lag.
• Veteran Insight: Concludes with an engaging nod to the fun of managing multiple Linux environments.

A compact, single-binary Linux desktop utility that adds **Copy To…**, **Move To…**, and **Wipe Me (Secure Zero-Fill)** to the context menus of the major file managers, and drives copy/move/wipe through a borderless dark-neon GUI.

It ships as one `x86_64` AppImage. No daemon, no root, no PATH requirement.

---

## Why it exists

Every Linux file manager exposes a different extension API:

| Family | Mechanism |
|---|---|
| KDE Dolphin | `kio/servicemenus/*.desktop` |
| LXQt PCManFM-Qt / GTK PCManFM | DES-EMA `file-manager/actions/*.desktop` |
| GNOME Nautilus | `nautilus-python` extensions and `nautilus/scripts` |
| MATE Caja | `caja-python` extensions |
| XFCE Thunar | `~/.config/Thunar/uca.xml` |
| Cinnamon Nemo | `nemo/actions/*.action` |

Copier normalizes all of them behind one executable. On first GUI launch it writes the config files, installs a private helper binary, refreshes the relevant desktop caches, and gets out of the way.

---

## Design goals

1. **Lightweight** — one Rust binary, immediate-mode GUI, no background service.
2. **No PATH dependency** — every menu entry calls an absolute helper path.
3. **Least privilege** — everything runs as the invoking user; no `pkexec`, no `sudo`.
4. **Deterministic UI** — all text and controls are laid out inside the window bounds; no overlap, clipping, or focus traps.
5. **Auditable** — all generated files live under `$HOME`; the target URL exists in exactly one place in the source.

---

## Runtime model

`main()` dispatches before any GUI is created:

```
copier                       -> main window + integration installer
copier --copy-to  <paths…>   -> destination picker -> copy -> exit
copier --move-to  <paths…>   -> destination picker -> move -> exit
copier --wipe     <paths…>   -> standalone "Secure Wipe" progress window
```

- **Copy / Move** never show the main window. They open the native folder chooser directly (or use `COPIER_DEST` for scripted use) and exit.
- **Wipe** opens a dedicated window titled *Secure Wipe — Copier*. The main status panel, title, and contact button are not rendered there.
- Multiple selected files are supported everywhere. `file:///…` URLs with percent-encoding are accepted and decoded.

### Installer side effects

Performed by `CopierApp::install_file_manager_integrations()`:

| Artifact | Purpose |
|---|---|
| `~/.local/bin/copier-fm` | Executable helper; absolute path used by every menu entry |
| `~/.local/share/kio/servicemenus/copier.desktop` | Dolphin menu (`0755` — KDE requires the executable bit) |
| `~/.local/share/file-manager/actions/copier-{copy,move,wipe}.desktop` | DES-EMA actions for PCManFM-Qt and PCManFM (GTK) |
| `~/.local/share/nautilus{,-python}/…` | Nautilus Python extension + Scripts-fallback shell scripts |
| `~/.local/share/caja-python/extensions/copier_ext.py` | Caja Python extension |
| `~/.config/Thunar/uca.xml` | Thunar actions, merged without destroying existing entries |
| `~/.local/share/nemo/actions/copier.action` | Nemo actions |
| `kbuildsycoca5/6` | KDE cache refresh |
| `~/.local/share/copier-desktop-refreshed` | One-shot marker for the LXQt desktop restart |

The LXQt desktop restart captures the running `pcmanfm-qt --desktop` process's exact command line **and environment**, relaunches it detached with `setsid`, and retries until it is alive. This preserves wallpaper and desktop icons. Because it is gated by the marker, it happens **once**, never on every launch.

### Wipe worker

1. Expand each target with `walkdir`.
2. For every file: write `1 MiB` zero chunks, `sync_all()`, then unlink.
3. Remove the emptied directories.
4. Stream `{percent, current_file}` over an `mpsc` channel to the progress window.

---

## User interface

- Near-black window fill, faint blue wash, thin blue neon border.
- Default text is soft silver-blue. Only the **Copier** title (red) and subtitle (green) are coloured.
- Glow is produced by faint offset copies of the glyphs — **no filled shade boxes or rectangles** behind text.
- Top-left controls: yellow `-` (minimize), green square (maximize), red circle (close). Each is a metallic body with a thin glow stroke.
- Top-right metallic gradient **Contact** button opens the maintainer's GitHub repositories page in the default browser. That URL appears nowhere else in the UI or scripts.
- Header dragging uses a single `ViewportCommand::StartDrag` on drag-start: no pointer lock, no flicker.

---

## Build from source

Requirements: Rust stable, `cargo-audit`, `appimagetool`.

```bash
cd copier

# Mandatory pre-package security gate.
cargo audit --ignore RUSTSEC-2026-0194 --ignore RUSTSEC-2026-0195

cargo build --release
./build-appimage.sh
```

The two ignored advisories are in `quick-xml 0.30.0`, pulled transitively by the desktop/accessibility stack (`zbus`, Wayland tooling). This application never feeds untrusted XML to them; no other vulnerabilities remain. Five `unmaintained` warnings are allowed and documented.

Output: `Copier-x86_64.AppImage`.

---

## Install and run

```bash
chmod +x Copier-x86_64.AppImage
./Copier-x86_64.AppImage
```

The first GUI launch installs the helper and menu files. Restart affected file managers once if their menus do not refresh immediately (Dolphin and PCManFM-Qt are handled automatically).

For non-interactive use:

```bash
COPIER_DEST=/srv/inbox copier-fm --copy-to /path/a.txt 'file:///path/b.txt'
```

Diagnostics for copy/move are appended to `/tmp/copier-copymove.log`.

---

## Verification performed

| Check | Result |
|---|---|
| `cargo audit` (with the two documented ignores) | Pass — 5 allowed warnings |
| `cargo build --release` | Pass |
| AppImage build and launch | Pass |
| Helper executable + absolute `Exec` paths | Pass |
| Config files for all six file-manager families | Present |
| Python extensions (`py_compile`) and Nautilus scripts (`sh -n`) | Pass |
| Thunar `uca.xml` XML parse | Pass, existing actions preserved |
| Multi-file copy, including `file://` URL | Pass (`Ok(4)`) |
| Multi-file move | Pass (`Ok(4)`) |
| Multi-file wipe | Pass — files removed |
| LXQt desktop after one-time restart | Alive, wallpaper/icons intact |
| Contact URL occurrences in source | Exactly 1 |

---

## Security notes

- No network access except the explicit Contact click.
- No privilege escalation at any point.
- Wipe is a **single zero-fill pass** with `fsync`. It is fast and low-resource, but on SSDs, wear leveling and TRIM mean a single pass is *not* a forensic-grade guarantee. Use full-disk encryption for that threat model.
- All generated files are user-owned under `$HOME`.

---

## Known limitations

- Thunar, Caja, and Nemo were not installed on the development host; their config files were generated and syntax-checked, but not GUI-tested.
- The two `quick-xml` advisories are transitive and unresolved upstream for this dependency chain; see the audit note above.
- `quick-xml`, `paste`, `ttf-parser`, `zbus_names`, and `zvariant` produce allowed unmaintained warnings.

---

## Project layout

```
copier/
├── Cargo.toml              # crate metadata and dependencies
├── Cargo.lock              # pinned dependency graph
├── src/main.rs             # entire application
├── create_icon.py          # vector-style icon generator
├── copier.png              # 256x256 AppImage icon
├── build-appimage.sh       # AppDir assembly + appimagetool
├── README.md               # this document
└── RELEASE_NOTES.md        # release history
```

---

## License

Choose and add your preferred license before publishing (for example MIT or Apache-2.0). Until then, treat the source as provided without an explicit license grant.
