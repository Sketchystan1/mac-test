# mv2-mem-patch-mac

Turns Manifest V2 extensions back on in macOS Chrome. The Mac version of
`mv2-mem-patch-win/`.

A small **dylib** Chrome loads at startup, patching Chrome's MV2 switch **in memory**.
The *Google Chrome Framework* on disk stays byte-for-byte stock, so Widevine DRM still
sees Google's bytes — verified against castLabs' VMP lab (no `PLATFORM_TAMPERED`).
Re-applies every launch.

The catch: to load the dylib, `install.sh` re-signs Chrome's main program ad-hoc (the
framework and helpers stay stock). Confirmed on **Chrome 153.0.8010.37 (arm64)**: 6 MV2
gates flipped, MV2 back on, uBlock Origin **force-installs from enterprise policy**, DRM
intact.

## Build (on a Mac)

```sh
./build.sh      # needs Xcode Command Line Tools; builds a universal arm64 + x86_64 dylib
```

## Install

Two ways to make Chrome load the dylib — choose with `MV2_INJECT`:

```sh
# dyld  — recommended, and REQUIRED on current Chrome (152+)
MV2_INJECT=dyld ./install.sh "/Applications/Google Chrome.app"

# loadcmd — the default; only works on older Chrome with Mach-O header room
./install.sh "/Applications/Google Chrome.app"
```

- **dyld** sets `DYLD_INSERT_LIBRARIES=@executable_path/mv` in the bundle's `Info.plist`
  `LSEnvironment`, so a normal Finder/Spotlight launch injects the dylib. Needs no Mach-O
  header room — the only method that works on current Chrome, whose main-exe stub is too small
  to add a load command to.
- **loadcmd** injects an `LC_LOAD_DYLIB` into the main exe, dropping a few metadata-only
  load commands (source-version / function-starts / data-in-code — never `LC_UUID`, which
  dyld requires) to make room. Infeasible on Chrome 152+; kept for older builds.

Both keep the framework stock and re-sign only the main exe (ad-hoc, non-hardened).

Other modes:

```sh
./install.sh --offline "/Applications/Google Chrome.app"   # just check signatures.json coverage
./install.sh --restore "/Applications/Google Chrome.app"   # undo
```

**After a Chrome auto-update, re-run `install.sh`** — an update drops a fresh stock main
exe with no injection, so MV2 turns off again.

## Install uBlock Origin (MV2) via policy

Re-enabling MV2 doesn't install the extension, and on branded Chrome `--load-extension`
is refused. Force-install uBO through Chrome's **mandatory** policy. macOS has no registry;
the equivalent of Windows `HKLM\Software\Policies\Google\Chrome\ExtensionSettings` is the
managed-preferences domain `cfprefsd` serves. That's what MDM / configuration profiles
populate — but you can write it directly, no MDM needed (confirmed served as a *forced*
policy via `CFPreferencesAppValueIsForced`):

```sh
UBO=fkgkibajhfbepljeaefdnfnegdcjomkh
UPD=https://github.com/gorhill/uBlock/raw/refs/heads/master/dist/chromium/update.xml
sudo defaults write "/Library/Managed Preferences/com.google.Chrome" \
  ExtensionInstallForcelist -array "$UBO;$UPD"
sudo killall cfprefsd          # make Chrome re-read the managed policy
```

Launch Chrome normally; uBO force-installs on startup — confirmed on **cold start, no relaunch
needed**. Verify at `chrome://policy` that `ExtensionInstallForcelist` shows **source: Platform**,
and at `chrome://extensions` that uBlock Origin is **installed by policy**. On disk it lands in
`~/Library/Application Support/Google/Chrome/Default/Extensions/<id>`.

Off-store force-install specifically depends on the **`LoadChromePolicy` gate** — the `cbz`-kind
site in `signatures.json` that makes Chrome honor off-store `ExtensionSettings` on unmanaged
Chrome. The other four `bcond` gates only cover manual enable / Load-unpacked, so without this one
MV2 turns on but a policy-forced off-store extension is silently skipped. The core implements the
`cbz` kind, so it works out of the box.

> User-domain `defaults write com.google.Chrome …` is *recommended*-only — it does **not**
> force-install. It has to be the `/Library/Managed Preferences` path above.
> `ExtensionSettings` (the exact `HKLM` analogue) works too but is a nested dict; the
> `ExtensionInstallForcelist` array is the simplest reliable force-install from the CLI.

## Notes

- No SIP change, but editing `/Applications` needs **Full Disk Access** for your terminal
  (System Settings → Privacy & Security).
- The dylib installs as `Contents/MacOS/mv`; `signatures.json` lives in
  `Contents/Resources/mv2/` (a data file under `Contents/Frameworks/` would break the
  codesign seal). `--restore` undoes everything.
- Debug: `MV2_MEMPATCH_DEBUG=1 "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"`
  and watch stderr for `[mv2patch] … MV2 extensions ENABLED`.
- DRM: the framework is left stock precisely to preserve Widevine's VMP hash. If a tier
  ever drops, the ad-hoc re-sign of the main exe is the only suspect — `--restore` reverts
  to Google's signature.
- A CI testbed that builds + installs this on a fresh Chrome and hands you a VNC desktop
  to verify it live is in `.github/workflows/mv2-mem-patch-mac.yml`.
