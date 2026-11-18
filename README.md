# Claude Desktop for Fedora 45

Install **Claude Desktop on Fedora 45** as a native RPM, built from Anthropic's **official Linux x86_64 build**. Unmodified upstream payload, Chromium sandbox on, Claude Code and Cowork working. Supports Fedora 45 (GNOME 51, KDE Plasma 6.7, RPM 6.1); tested on Fedora 44 with KDE Plasma (Wayland) and GNOME (Wayland); should work on Fedora 43 and later. See [Fedora 45 notes](#fedora-45-notes) for what changed.

**TL;DR:** `./build-official.sh && sudo dnf install build/official/claude-desktop-*.rpm`

> Unofficial packaging. Anthropic ships the app; this repo only rewraps it for `dnf`. If you hit a packaging bug, open an issue here, not with Anthropic.

## Install Claude Desktop on Fedora 45

```bash
sudo dnf install rpm-build binutils file xz curl gnupg2

git clone https://github.com/dewzor/claude-desktop-fedora.git
cd claude-desktop-fedora
./build-official.sh
sudo dnf install build/official/claude-desktop-*.rpm
```

Install with `dnf`, not `rpm -i`: Fedora 45 makes `rpm` refuse unsigned packages, and this one is built on your machine, so it isn't signed. `dnf` installs it and prints `Warning: skipped OpenPGP checks for 1 package from repository: @commandline`. That warning is expected; see [Fedora 45 notes](#fedora-45-notes).

Launch **Claude** from your application menu, or run `claude-desktop`. The launcher also ships two desktop actions: *New Chat* and *New Claude Code Session*.

The build runs entirely as your own user; root is only needed for the final `dnf install`.

## Auto-updates via COPR

```bash
sudo dnf copr enable dewzor/claude-desktop
sudo dnf install claude-desktop-builder
```

`claude-desktop-builder` is a small MIT-licensed package. It does not contain Claude Desktop, and it never will. Claude Desktop is proprietary, so no repository is allowed to ship it. What the package ships is the build itself: `build-official.sh`, a `claude-desktop-update` command, and a systemd timer.

The timer runs once a day. It asks Anthropic's signed apt index which Linux build is current, and checks the signature against Anthropic's release key. If that build is already installed and was built here from that exact signed `.deb`, it stops there and doesn't download the app. If a newer build has shipped, it fetches Anthropic's own `.deb` on your machine, repackages the payload as an RPM, and installs it with `dnf`. You get exactly the package `build-official.sh` produces by hand, without having to remember to run it.

Run it yourself any time:

```bash
sudo claude-desktop-update --check   # says whether a new build exists
sudo claude-desktop-update           # rebuilds and installs if there is one
```

Removing the package leaves Claude Desktop alone:

```bash
sudo dnf remove claude-desktop-builder
```

## What works on Fedora 45

| Feature | Status |
| --- | --- |
| Fedora 45, GNOME 51 (Wayland) | Supported |
| Fedora 45, KDE Plasma 6.7 (Wayland) | Supported |
| Fedora 44, KDE Plasma (Wayland) | Tested |
| Fedora 44, GNOME (Wayland) | Tested |
| Claude Code tab (integrated terminal) | Working |
| Cowork | Working |
| MCP (Model Context Protocol) | Working |
| Chrome extension MCP bridge | Working |
| Native window frame | Yes, no patching needed |
| Chromium sandbox | Enabled (`chrome-sandbox` setuid 4755, no `--no-sandbox`) |
| System tray | Working |
| GNOME Activities search ("Ask Claude") | Working |
| Open `.mcpb`, `.dxt` and `.skill` files in Claude | Working |
| Auto-download latest Claude | Yes, from Anthropic's signed apt repository |
| Architecture | x86_64 only (see FAQ) |
| Auto-update | Optional, via the `claude-desktop-builder` COPR package |

## How the Fedora RPM is built

Anthropic publishes an official Linux build of Claude Desktop, but only as a `.deb`. The payload inside is ordinary, self-contained Linux content under `/usr/lib/claude-desktop`. Nothing in it is Debian-specific except the wrapper and dependency names.

`build-official.sh`:

1. Fetches the `InRelease` index of Anthropic's apt repository and checks its signature against Anthropic's release key. The script pins the key by fingerprint, `31DD DE24 DDFA B679 F42D 7BD2 BAA9 29FF 1A7E CACE`, the one in the Claude Code setup docs. It refuses an index past its `Valid-Until` date, so an old index can't be replayed once it expires (as long as the system clock is right)
2. Checks the `Packages` index against the SHA256 in the signed index and picks the newest `claude-desktop` entry
3. Downloads that `.deb`, about 175 MB, and checks it against the SHA256 in the signed index
4. Unpacks it and checks that the control file's version matches the signed index
5. Verifies that `@ant/claude-native` and `node-pty` are real ELF x86-64 objects, and fails loudly if not
6. Maps the Debian dependencies to Fedora package names
7. Builds an RPM with `rpmbuild`, preserving `chrome-sandbox` setuid via `%attr(4755,root,root)`
8. Confirms the setuid bit survived into the package metadata

The payload is **unmodified upstream content**. No `app.asar` patching, no stubs, no injected CSS, no launcher wrapper.

At install time the upstream `postinst` also copies three things out of `resources/`. It has to, because electron-installer-debian, which builds the `.deb`, can't ship extra files. The RPM packages them as ordinary files, so `dnf remove` cleans them up:

- the GNOME Shell search provider, `com.anthropic.Claude.search-provider.ini` plus its D-Bus service, which adds an "Ask Claude" result to Activities search
- the MIME types for `.mcpb`/`.dxt` desktop extensions and `.skill` files, so they open in Claude
- the `/usr/bin/ccd` launcher link, when the build ships `resources/bin/ccd`

Two Debian-specific `postinst` behaviours are intentionally dropped:

- The AppArmor `flags=(unconfined)` userns profile. That one is only needed on Ubuntu 24.04+, which restricts unprivileged user namespaces. Fedora leaves them enabled.
- apt repository registration.

### Options

```
--deb PATH     Use an already-downloaded .deb instead of fetching it. Skips verification.
--outdir DIR   Where to write the finished RPM (default: ./build/official).
               Earlier claude-desktop-*.x86_64.rpm builds there are removed.
--release N    RPM release field (default: 1).
--keep         Keep the build tree for inspection.
--print-latest Verify the signed index, print "VERSION SHA256 FILENAME" for the
               newest build, and exit without downloading it.
```

## Updating Claude Desktop on Fedora

The RPM does not auto-update. To pick up a new release:

```bash
./build-official.sh
sudo dnf install build/official/claude-desktop-*.rpm
```

The spec carries `Obsoletes: claude-desktop < %{version}-%{release}`, so a newer build cleanly replaces any earlier locally-built package, including ones from the legacy script.

## Fedora 45 notes

Fedora 45 changed three things that touch a locally built package like this one. The scripts already handle them; this is what to expect.

**RPM refuses unsigned packages.** RPM 6.1 in Fedora 45 sets `%_pkgverify_level` to `all`, so `rpm -i` and `rpm -U` reject any package without a signature from a key in the rpm keyring (`package ... does not verify: no signature`). The RPM built here is unsigned: it's made on your machine, and its trust comes from Anthropic's signed apt index, which `build-official.sh` verifies before it builds anything. `dnf install ./file.rpm` still installs local files without an OpenPGP check by default, so the normal install command works and prints a `skipped OpenPGP checks` warning. `claude-desktop-update` also passes `--setopt=localpkg_gpgcheck=0` for its own install, so the daily update keeps working if you've turned that check on. Repository packages pulled in as dependencies are still checked.

If you'd rather have a signed package, set up rpmbuild autosigning once and `build-official.sh` picks it up:

```bash
/usr/lib/rpm/rpm-setup-autosign          # as your own user; prints the key to import
sudo rpmkeys --import <the .asc file it printed>
```

After that, every build is signed and `rpm -i` accepts it too. This applies to manual builds. The daily timer builds as root, so it doesn't use your key.

**oo7 replaced GNOME Keyring.** Fedora 45 uses `oo7-daemon` as its Secret Service provider, which Electron uses to keep you signed in. The RPM used to recommend `gnome-keyring`, which pulled a second, competing provider onto a stock Fedora 45 desktop. It now recommends `(oo7-daemon or gnome-keyring or kf6-kwallet)`, so whichever one you already have satisfies it.

**DNF5 doesn't change vendors.** Fedora 45's DNF5 won't silently switch an installed package to a different `Vendor`. The RPM has never set a vendor and still doesn't, so upgrades from earlier builds replace it in place.

## MCP setup on Fedora

MCP servers are configured at:

```
~/.config/Claude/claude_desktop_config.json
```

Same format as macOS and Windows Claude Desktop. Drop in your `mcpServers` block and restart the app.

Logs live in `~/.config/Claude/logs/` (`main.log`, `mcp.log`, `ssh.log`, …).

## Why not the Windows repack any more?

Earlier versions of this repo (`build-fedora.sh`, still in the tree) rebuilt Claude Desktop from the **Windows installer**: extract `app.asar`, swap the native module for a JS stub, patch the titlebar constants, bundle Electron 37. That worked through the 0.14.x era. Current releases can't be served that way:

- **`@ant/claude-native`** is now hard-required for filesystem containment, and the app explicitly refuses to degrade without it:
  `@ant/claude-native is required for safe-fs containment but failed to load; refusing to fall back to a path-based open (CC-2885)`.
  A JS stub can't satisfy that, and the module also moved from `claude-native` to `@ant/claude-native`, so the old stub path is never even loaded. In a Windows repack the real module is a PE32+ DLL: `ERR_DLOPEN_FAILED: invalid ELF header`.
- **`node-pty`**: Claude Code's terminal needs `prebuilds/linux-x64/pty.node`. The Windows package only ships `prebuilds/win32-x64/conpty.node`.

Net effect: the app starts, but **the Claude Code tab does not work**. The official Linux build ships both modules as real ELF x86-64 objects, and its native frame is already correct, so none of the old patching is needed.

## Troubleshooting on Fedora 45

Exact error strings and what they mean. All of these are fixed by building with `build-official.sh`.

### `@ant/claude-native is required for safe-fs containment but failed to load` / `CC-2885`
You're running a Windows-repack build (`build-fedora.sh` or any of the older Fedora scripts). Current Claude Desktop hard-requires the real native module and refuses to start Claude Code without it. Rebuild from the official Linux build.

### `ERR_DLOPEN_FAILED: invalid ELF header` / `Failed to load Claude Native`
The `claude-native-binding.node` in your install is a Windows PE32+ DLL, not a Linux ELF object. Same cause as above, a Windows repack. Rebuild with `build-official.sh`; it verifies both native modules are ELF x86-64 before packaging.

### Claude Code tab not working / blank / terminal never appears
Claude Code's terminal needs `node-pty`'s `prebuilds/linux-x64/pty.node`. Windows packages only ship `win32-x64/conpty.node`. Rebuild from the official Linux build and check `~/.config/Claude/logs/main.log` for `[CCD] Initialized`.

### `⚠ titleBarStyle pattern not found` during `build-fedora.sh`
The legacy script's titlebar `sed` no longer matches current releases. It's harmless but it's also a sign you're on the wrong path. The official build already draws a correct native frame. Use `build-official.sh`.

### `InRelease is not signed by Anthropic's release key` / `signature check failed`
The apt index didn't verify against the pinned key, or it was signed with an expired or revoked key. Don't work around this with `--deb`. Check https://code.claude.com/docs/en/desktop-linux and the key at https://downloads.claude.ai/claude-desktop/key.asc for a key change, and open an issue.

### The daily update stopped working
Run `journalctl -u claude-desktop-update.service` to see the last error. Every check fails closed: nothing is installed and the old version keeps working until the next run succeeds.

### `the signed index expired`
The index is past its `Valid-Until` date, which Anthropic sets a week after signing it. Either the repository hasn't been re-signed in that time, or your system clock is wrong. Check the clock first.

### `package claude-desktop-... does not verify: no signature`
You installed with `rpm -i` or `rpm -U` on Fedora 45, which refuses unsigned packages. Use `sudo dnf install build/official/claude-desktop-*.rpm` instead, or set up autosigning as described in [Fedora 45 notes](#fedora-45-notes). Don't fix it with `--nosignature` or by setting `%_pkgverify_level` back to `digest`; that turns the check off for every package.

### `cannot install both claude-desktop-X from @commandline and claude-desktop-Y from @commandline`
`build/official/` still holds an RPM from an earlier build, so `claude-desktop-*.rpm` matched two packages. Install the file the build printed after `Built:`. Builds from v9.4 on remove the earlier RPM from the output directory, so the wildcard matches only the new one.

### `Warning: skipped OpenPGP checks for 1 package from repository: @commandline`
Expected when you install the unsigned local build with `dnf`. The build was checked against Anthropic's signed apt index before it was packaged.

### Signed out of Claude after upgrading to Fedora 45
Fedora 45 moved saved secrets from GNOME Keyring or KWallet to oo7 and migrates them on first login. If Claude asks you to sign in again, check that `systemctl --user status oo7-daemon` shows it running, then sign in again so the new token is stored there.

### `'--ozone-platform=wayland' is not compatible with Vulkan`
Chromium noise on Wayland sessions. Cosmetic; ignore.

### Cowork says the VM sandbox is unavailable
Install the optional VM stack: `sudo dnf install qemu-system-x86 edk2-ovmf virtiofsd`. The RPM lists these as `Suggests:`.

### Package conflicts with `claude-desktop-unofficial`
That's [aaddrick/claude-desktop-debian](https://github.com/aaddrick/claude-desktop-debian)'s RPM, which adds a launcher wrapper and a `--doctor`. Both packages share `~/.config/Claude`, so run one or the other, not both.

## Legacy: `build-fedora.sh`

Kept for anyone who needs a 0.14.x-era build. It is no longer the recommended path and is not maintained against current releases.

```bash
sudo dnf install rpm-build p7zip nodejs npm
sudo ./build-fedora.sh
sudo dnf install build/electron-app/$(uname -m)/claude-desktop-*.rpm
```

Known limits on current releases: the Claude Code tab doesn't work (see above), and the `titleBarStyle` sed no longer matches (`⚠ titleBarStyle pattern not found`). History of what it did and why is in [CHANGELOG.md](CHANGELOG.md).

## FAQ

### Does Claude Desktop work on Fedora 45?
Yes. The payload is self-contained, every runtime dependency still exists under the same name in Fedora 45, and the two Fedora 45 changes that touch this package are handled (see [Fedora 45 notes](#fedora-45-notes)). Install with `dnf`, not `rpm -i`.

### Does it work on Fedora 44?
Yes, tested on Fedora 44 with KDE Plasma (Wayland) and GNOME (Wayland).

### Does it work on Fedora 43 or older?
It should. The payload is self-contained (bundled Electron), so the only real constraint is the runtime dependency list, which resolves on 43. Not actively tested. If you try it, open an issue with what you see.

### Does the Claude Code tab work?
Yes. That's the whole point of the official-build path. Look for `[CCD] Initialized` in `~/.config/Claude/logs/main.log`.

### Does Cowork work?
Yes. Cowork's optional full-VM sandbox needs `qemu-system-x86`, `edk2-ovmf` and `virtiofsd`; they're listed as `Suggests:` in the RPM and are harmless to omit.

### Does MCP work?
Yes. Config lives at `~/.config/Claude/claude_desktop_config.json`.

### Why x86_64 only?
The script only builds for x86_64 for now. An `arm64` build exists in Anthropic's apt repo; if you need it, fetch that `.deb` yourself and pass it in with `--deb PATH` (the script currently refuses non-x86_64 hosts, patches welcome).

### I see a Wayland/Vulkan warning in the logs
`'--ozone-platform=wayland' is not compatible with Vulkan` is Chromium noise. It's cosmetic.

### How do I uninstall Claude Desktop from Fedora?
```bash
sudo dnf remove claude-desktop
```

## Credits

- [@bsneed](https://github.com/bsneed) (Brandon Sneed): the original project this one grew out of.
- [@boujuan](https://github.com/boujuan) (Juan Boullosa): the Fedora 43 title bar fixes and automatic download of the latest version in the legacy script (v6 to v8).
- [@anilsrml](https://github.com/anilsrml) (Anıl Sürmeli): found and fixed the locale bug that broke builds on Turkish and other systems ([#8](https://github.com/dewzor/claude-desktop-fedora/pull/8)).
- [@arcanistzed](https://github.com/arcanistzed): download verification against Anthropic's signed apt repo, GNOME search and file associations ([#6](https://github.com/dewzor/claude-desktop-fedora/pull/6)).
- [@itsnitinr](https://github.com/itsnitinr): multithreaded zstd payload compression ([#5](https://github.com/dewzor/claude-desktop-fedora/pull/5)).
- [@sharpandpearl](https://github.com/sharpandpearl): `build-official.sh` and the analysis of why the Windows repack stopped working ([#4](https://github.com/dewzor/claude-desktop-fedora/pull/4)).
- [@randy-johnson](https://github.com/randy-johnson): `titleBarOverlay` regex fix on the legacy script ([#3](https://github.com/dewzor/claude-desktop-fedora/pull/3)).
- [@gordonmessmer](https://github.com/gordonmessmer): the [com.anthropic.Claude](https://github.com/gordonmessmer/com.anthropic.Claude) Flatpak manifest that our earlier Flatpak was based on.

## License

Dual-licensed under MIT and Apache 2.0. See [LICENSE-MIT](LICENSE-MIT) and [LICENSE-APACHE](LICENSE-APACHE).

The Claude Desktop application is not included in this repository and is covered by [Anthropic's Consumer Terms](https://www.anthropic.com/legal/consumer-terms).
