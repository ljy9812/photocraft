# PhotoCraft for HarmonyOS (OpenHarmony) — ArkWeb shell

Runs the official `photocraft-web` build — the same static site published as
`photocraft-web-<version>.zip` on the GitHub release — inside a HarmonyOS app. An
ArkTS shell wraps an ArkWeb `Web` component and serves the site from
the HAP's own `rawfile`: no network origin, no server-side code.

This branch (`ohos-webshell`) is the integrated form: the ArkWeb shell
packaging under `packaging/ohos/` plus the wasm-side CJK font change
merged from the `cjk-web-face` branch (which remains the clean
single-purpose form for an upstream PR). Build either variant from this
one checkout.

## How the shell works

| Concern | Mechanism |
|---|---|
| Serving | `onInterceptRequest` maps the virtual origin `https://local.photocraft/` to `entry/src/main/resources/rawfile/web/` |
| Opening files | `onShowFileSelector` → system `DocumentViewPicker`; the picked URI is handed back to the page's `<input type=file>` |
| Saving | `a[download]` clicks on attached anchors are intercepted by a hook installed in `onPageEnd`; the blob is fetched, base64-chunked over a `MessageChannel` port to ArkTS, and written through `DocumentSavePicker` |
| External links | dual path: `onLoadIntercept` for main-frame http(s) navigations + `multiWindowAccess(true)`/`onWindowNew` for `window.open` popups — both route to the system browser |

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
interaction without crashes, text input via the physical keyboard through
eframe's hidden-input focus chain, and — after the save-bridge fixes of the
follow-up rounds — a full open→edit→save→reopen round-trip (brush strokes
byte- and pixel-verified, five consecutive exports, picker-based reopen).

Known issues on that device class — all specific to ArkWeb's WebGL/ANGLE
stack; the same web build runs cleanly in desktop Edge in both WebGPU and
forced-WebGL modes:

- Opening a document view (a loaded image, or a blank new image) triggers a
  `GL_INVALID_OPERATION ... Source and destination textures of the draw are
  the same` error storm and can freeze the canvas while the app logic stays
  alive. Not fixable from the ArkTS shell; needs upstream renderer
  hardening or an ArkWeb fix.
- The interface has no CJK font issue (runtime font channel from the zip's
  `fonts/` directory); earlier releases wrote exports byte-identical to the
  source on the tested build — retest against the release you fetch.
