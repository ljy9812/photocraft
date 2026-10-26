# PhotoCraft for HarmonyOS (OpenHarmony) — ArkWeb shell

Runs the official `photocraft-web` build — the same static site published as
`photocraft-web-<version>.zip` on the GitHub release (see `packaging/web/`) —
inside a HarmonyOS app. An ArkTS shell wraps an ArkWeb `Web` component and
serves the site from the HAP's own `rawfile`: no network origin, no
server-side code, no Rust/wasm toolchain involved.

## How the shell works

| Concern | Mechanism |
|---|---|
| Serving | `onInterceptRequest` maps the virtual origin `https://local.photocraft/` to `entry/src/main/resources/rawfile/web/` |
| Opening files | `onShowFileSelector` → system `DocumentViewPicker`; the picked URI is handed back to the page's `<input type=file>` |
| Saving | `a[download]` clicks are intercepted by a hook injected in `onPageEnd`; the blob is fetched, base64-chunked over a `MessageChannel` port to ArkTS, and written through `DocumentSavePicker` |

## Build (Windows host, DevEco Studio toolchain)

Prerequisites: DevEco Studio (for `hvigorw`, `ohpm`, and the HarmonyOS SDK),
`hdc` on PATH, and `gh` (GitHub CLI) for the release download.

```powershell
cd packaging\ohos
.\scripts\fetch-web.ps1     # deploy the web build into rawfile (latest release, or -Tag v0.12.0)
.\scripts\build-hap.ps1     # build the Release HAP and install to the connected device
```

### Signing

The committed tree is deliberately signing-free (`"signingConfigs": []` in
`build-profile.json5`). To build a signed HAP, put your own signing fragment
at `packaging\ohos\local\signing.json` (gitignored) — one `signingConfigs`
entry as DevEco Studio writes it. `build-hap.ps1` injects it for the build
and restores the pristine file afterwards. Adjust `bundleName` in
`AppScope/app.json5` (currently the `com.example.photocraft.web` placeholder)
to match your signing material.

## Verified status and known limitations

Verified on a HarmonyOS laptop (HAD-W24, API 23, physical keyboard +
touchscreen): install & launch, first screen with full CJK UI fonts, ~60 fps
interaction (RenderService fps on the web surface), 190 s of continuous
interaction without crashes, and text input via the physical keyboard through
eframe's hidden-input focus chain.

Known issues on that device class — all specific to ArkWeb's WebGL/ANGLE
stack; the same web build runs cleanly in desktop Edge in both WebGPU and
forced-WebGL modes:

- Opening a document view (a loaded image, or a blank new image) freezes the
  canvas with repeated `GL_INVALID_OPERATION ... Source and destination
  textures of the draw are the same` errors while the app logic stays alive.
  The editor is not usable for visual work on ArkWeb until this is resolved
  (upstream renderer hardening or an ArkWeb fix).
- The save bridge round-trips, but the written file has been observed to be
  byte-identical to the source (edits not yet in the exported blob) on the
  tested web build, and later export attempts in the same session can go
  silent.
- External links inside the page 404 against the virtual origin instead of
  opening a system browser.
