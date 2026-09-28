# Local Windows build + publish (ClipWave OBS plugin)

## Prerequisites (one time)

- Visual Studio 2022 Build Tools, `VCTools` workload (any drive)
- CMake **3.30.x portable** (NOT 4.x — 4.x breaks the template quirk
  `-A x64,version=<sdk>` used by `cmake/common/buildspec_common.cmake`)
- NSIS 3 (winget `NSIS.NSIS`)
- `gh` CLI authenticated (repo + release rights on the org)

## Rules learned the hard way

- Build from a **space-free path**. `E:\Node JS\...` breaks the nested
  OBS configure (quoted `CMAKE_PREFIX_PATH`). Mirror the repo, e.g.:
  `git clone <plugin-repo> E:\cwplug` (commit first so nothing is lost).
- Always pass `-DENABLE_FRONTEND_API=ON -DENABLE_QT=ON` or the sound
  files are excluded and linking fails.
- Wipe `build/` and `.deps/obs-studio-*/build_x64` after any toolchain
  move (stale caches poison reconfigures).

## Build

```powershell
cd E:\cwplug
# 1. version first: buildspec.json "version" -> X.Y.Z (installer name embeds it)
C:\path\to\cmake-3.30.5\bin\cmake.exe -S . -B build -G "Visual Studio 17 2022" -A x64 -DENABLE_FRONTEND_API=ON -DENABLE_QT=ON
C:\path\to\cmake-3.30.5\bin\cmake.exe --build build --config RelWithDebInfo
C:\path\to\cmake-3.30.5\bin\cmake.exe --install build --prefix "release/RelWithDebInfo" --config RelWithDebInfo
$env:CI = 'true'
.\.github\scripts\Package-Windows.ps1   # needs makensis; signing warns + skips without cert
```

Outputs: `release\clipwave-obs-X.Y.Z-windows-x64-Installer.exe` + `.zip`.

## Publish

```powershell
$exe = "release\clipwave-obs-X.Y.Z-windows-x64-Installer.exe"
$zip = "release\clipwave-obs-X.Y.Z-windows-x64.zip"
$h1 = (Get-FileHash $exe -Algorithm SHA256).Hash.ToLower()
$h2 = (Get-FileHash $zip -Algorithm SHA256).Hash.ToLower()
$sha = gh api "repos/clipwaveapp/clipwave-obs-releases/commits/main" --jq '.sha'
gh api "repos/clipwaveapp/clipwave-obs-releases/git/refs" -f ref="refs/tags/X.Y.Z" -f sha="$sha" | Out-Null
# notes file must contain both checksums, then:
gh release create X.Y.Z --repo clipwaveapp/clipwave-obs-releases --draft --title "ClipWave for OBS X.Y.Z" --notes-file notes.txt
gh release upload X.Y.Z $exe $zip --repo clipwaveapp/clipwave-obs-releases --clobber
gh release view X.Y.Z --repo clipwaveapp/clipwave-obs-releases --json assets --jq '.assets[] | "\(.name)  (\(.size))"'
```

Notes: `gh release create` prints a bogus `untagged-*` URL — always confirm
with `gh release list`. The draft stays private until published; keep one
version per binary (never clobber across versions, hashes confuse users).
