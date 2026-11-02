# Handoff: build + refresh OrionTV (tvOS) without a Mac

Give this document to the agent. It is self-contained.

## Environment (already set up, do not repeat)

- Apple TV 4K 3rd gen (`AppleTV14,1`), tvOS **27.2**, LAN IP `192.168.18.145`,
  name 主卧, hardware **UDID `00008110-00145DAE3C06401E`**.
- Host: Linux box `192.168.18.153`. Docker container `atvloadly`
  (`bitxeno/atvloadly:latest`) on **http://127.0.0.1:5533**, data in
  `~/.local/share/atvloadly`, host `avahi-daemon` socket mounted.
- Apple TV is already **paired** (`status: paired`, tunnel port 49152).
- Apple ID already signed in: `hbczljr@163.com` (free account, team `4FA2CAWDDN`).
- Installed app: `com.oriontv` 1.3.14, atvloadly app id **1**.
- **No Mac exists.** Compilation happens on GitHub Actions `macos-14`.
- Build repo: **https://github.com/hbczljr/oriontv-tvos-build** (public → free
  macOS minutes). Auth via `gh` CLI (already logged in as `hbczljr`).
- Upstream source: **https://github.com/orion-lib/OrionTV** (branch `master`),
  Expo 51 + React Native TVOS 0.74. Releases upstream ship **Android APK only** —
  there is no official `.ipa`, building it is the whole point.

## Build: what must be done, and why

`macos-14` is required (Xcode 15.4 / tvOS 17.5 SDK). Do **not** use
`macos-latest`/Xcode 16+: RN 0.74 fails to compile there.

1. `actions/checkout` of `orion-lib/OrionTV@master`.
2. Node 20, `corepack enable`, `yarn install` (yarn 1.22.22 per `packageManager`).
3. `EXPO_TV=1 npx expo prebuild --clean -p ios --no-install`
   → generates `ios/platform :tvos, '13.4'`, `SDKROOT = appletvos`.
4. Write `react-native.config.js` at repo root **before** `pod install`:

```js
module.exports = {
  dependencies: {
    'react-native-blob-util': { platforms: { ios: null } },
    'react-native-file-viewer': { platforms: { ios: null } },
    '@react-native-cookies/cookies': { platforms: { ios: null } },
  },
};
```

   All three are dependencies of OrionTV but **never imported** by its code, and
   their Objective-C uses tvOS-unavailable APIs (`UIDocumentInteractionController`,
   `AssetsLibrary`, `QLPreviewController`, `WKHTTPCookieStore`). Autolinking them
   breaks the build. Unused ⇒ zero runtime risk.
5. `cd ios && pod install --repo-update`
   (`react-native-blob-util` also declares `s.platforms = { :ios => "11.0" }`
   only; if it is ever needed on tvOS, patch the podspec to add `:tvos => "13.4"`.)
6. Discover the scheme: **take the workspace basename**
   (`OrionTV.xcworkspace` → `OrionTV`). Never take `schemes[0]` or "first
   non-`Pods-` scheme": every pod (`boost`, `fmt`, `CocoaAsyncSocket`, …) is
   listed as a scheme, and picking one produces a 2-minute "successful" build
   with no `.app`.
7. Build:

```bash
xcodebuild -workspace OrionTV.xcworkspace -scheme OrionTV \
  -configuration Release -sdk appletvos \
  -destination 'generic/platform=tvOS' -derivedDataPath "$PWD/build" \
  CODE_SIGNING_ALLOWED=NO CODE_SIGNING_REQUIRED=NO CODE_SIGN_IDENTITY="" \
  NODE_BINARY="$(which node)" build
```

8. Package (no signing here — atvloadly signs later):

```bash
mkdir -p out/Payload
cp -R ios/build/Build/Products/Release-appletvos/*.app out/Payload/
cd out && zip -qry OrionTV-tvos.ipa Payload
```

9. Upload as artifact. A healthy IPA is ~12 MB, `Mach-O 64-bit arm64`,
   `CFBundleSupportedPlatforms = ['AppleTVOS']`, `UIDeviceFamily = [3]`, and
   **contains `main.jsbundle` (~5.7 MB)**. No `main.jsbundle` ⇒ debug build that
   needs a running Metro server (this is what causes the upstream white-screen
   reports in issue #74).

### Platform gotchas already hit

| symptom | cause | fix |
|---|---|---|
| `sed: extra characters at the end of n command` | BSD `sed -i` requires an argument on macOS | use `perl -pi -e` |
| `pod install` says target not compatible | podspec declares iOS only | exclude pod or add `:tvos` to its podspec |
| build "succeeds" in ~2 min, no `.app` | wrong scheme (a pod scheme) | scheme = workspace basename |
| white screen on launch | debug build without Metro | build `-configuration Release` |
| `UIDocumentInteractionController is unavailable: not available on tvOS` | iOS-only lib | drop from tvOS autolinking |

## Install (sign + push to device)

`POST http://127.0.0.1:5533/api/install` as multipart:

```bash
curl -sS -X POST http://127.0.0.1:5533/api/install \
  -F "file=@OrionTV-tvos.ipa" \
  -F "device_id=<id from GET /api/devices where status==paired>" \
  -F "account=hbczljr@163.com" \
  -F "signing_mode=apple_id"
```

Progress: `GET /api/apps/installing`. Done when it returns `[]`.
Success log line: `Installing ipa success: OrionTV`.

`GET /api/accounts` returns an **object keyed by email**, e.g.
`{"hbczljr@163.com": {...}}` — not a list.

## Refresh (re-sign before the 7-day expiry)

A free Apple ID certificate lasts **7 days**. atvloadly stores the IPA in
`/data/ipa/<id>/app.ipa` and re-signs it:

- `POST /api/apps/1/refresh` — re-sign + reinstall now
- `POST /api/apps/1/reinstall` — full reinstall
- Automatic: the container's scheduled task (config `task.mode`, cron
  `0,30 3-6 * * *`) refreshes apps ahead of expiry, but **only while the
  container runs**.

Verify: `GET /api/apps` → `refreshed_result: true`, `expiration_date` ~7 days out.

Constraints of the free account: max **3 active sideloaded apps** (installing a
4th silently disables an earlier one) and 10 app IDs.

## Debugging on device

- Screenshot: `POST /api/devices/<id>/screenshot` → JSON `data.data` is base64
  (data-URI prefix possible); decode to JPEG, 1920×1080.
- Pairing is required again after a tvOS update: on the Apple TV open
  **Settings → Remotes and Devices → Remote App and Devices** (scroll to the
  bottom; the empty list *is* pairing mode), then pair over `/ws/pair`:
  `{"t":1,"d":"<device id>"}` → TV shows a 6-digit PIN → `{"t":2,"d":"<pin>"}`.
  **Keep the websocket open until `SUCCESS`** — closing it kills the pairing
  process server-side (`signal: killed`).
- Pairing over **Ethernet is broken** on tvOS 26.4+; use Wi-Fi.
- A VPN / local proxy on the host (e.g. `http_proxy=127.0.0.1:7890`) makes
  `http://192.168.18.153:5533` return **400 Bad Request** — use
  `http://127.0.0.1:5533` or bypass `192.168.18.0/24`.

## First-run on the TV

OrionTV is only a player. It opens to a config screen asking for the **MoonTV
server URL**, which must be addressable from the Apple TV (e.g.
`http://192.168.18.153:<port>`, never `localhost`).
