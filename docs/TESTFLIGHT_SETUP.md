# Official TestFlight setup (no personal Mac required)

This repository now has two **separate** manual GitHub Actions workflows:
- `.github/workflows/build-ipa.yml` remains the original ad-hoc SideStore/AltStore build.
- `.github/workflows/testflight.yml` builds a signed archive on GitHub's Apple Silicon `macos-26` runner, exports an App Store Connect IPA and uploads to TestFlight.

The TestFlight workflow does **not** publish its IPA as a GitHub Release. It is manually triggered, requires a **private** repository, and does not automatically add external testers or submit the game for public App Store review.

**Important:** The provided `game.wasm` is compiled to native ARM64 code and **included in the IPA**. Other users supplying their own assets later does not remove that embedded engine. Use only if you have appropriate rights to distribute the resulting executable. Internal TestFlight has no beta app review, but Apple's terms, build processing and copyright rules still apply.

## 1. Register the app with Apple

You must have an active **paid** Apple Developer Program membership.

1. At https://developer.apple.com/account/resources/identifiers/list register a unique explicit iOS App ID such as `com.example.privatefanport`. Prefer a neutral identifier you control; do **not** use a bundle ID registered to another developer.
2. In https://appstoreconnect.apple.com/apps create the corresponding iOS app using that **same bundle ID**. This is a necessary App Store Connect record; it does not submit the app for public review.
3. Copy your 10-character Apple Developer **Team ID** from the Apple Developer membership portal.
4. In App Store Connect, go to **Users and Access → Integrations → App Store Connect API** and create a **team API key** with at least App Manager access. Record its **Issuer ID**, **Key ID**, and download `AuthKey_XXXX.p8`. Store the `.p8` privately; Apple only lets you download that key once.

## 2. Create your Apple Distribution certificate and App Store profile

In the Apple Developer portal, create an **Apple Distribution** signing certificate using a certificate signing request (CSR). The certificate's **private key must be preserved** so a password-protected `.p12` can be created. A Windows PC with OpenSSL can do this without a Mac:

```powershell
openssl req -new -newkey rsa:2048 -nodes -keyout distribution.key -out distribution.csr -subj "/CN=Apple Distribution"
```

On the Apple Developer **Certificates** page choose **Apple Distribution**, upload `distribution.csr`, then download the issued `distribution.cer`. In the same directory:

```powershell
openssl x509 -inform DER -in distribution.cer -out distribution.pem
openssl pkcs12 -export -inkey distribution.key -in distribution.pem -out distribution.p12
```

Enter a strong export password when asked. The `distribution.p12` contains your **private signing key**. Never commit it, the private `distribution.key`, or the `AuthKey_*.p8` file to GitHub.

Next, create an **App Store Connect / App Store** iOS distribution provisioning profile for the registered App ID, selecting this distribution certificate, under https://developer.apple.com/account/resources/profiles/list. It must be active. A special entitlement such as Increased Memory Limit may need an explicitly authorized App ID/profile; this workflow deliberately uses `App/Free.entitlements` by default.

### If you prefer not to generate certificates manually

Apple-Actions also provides signing bootstrap scripts:
https://github.com/Apple-Actions/download-provisioning-profiles
Read their setup instructions and audit scripts before running them with your Apple credentials.

## 3. Configure GitHub Actions

Open the private repository's **Settings → Secrets and variables → Actions**.

Under **Variables**, set:

| Name | Value |
| --- | --- |
| `APP_BUNDLE_ID` | Your exact registered bundle identifier from step 1 |
| `APPLE_TEAM_ID` | Your Apple Developer team ID |
| `APPSTORE_ISSUER_ID` | Issuer ID for your App Store Connect API key |
| `APPSTORE_API_KEY_ID` | Key ID for the same API key |

Under **Secrets**, set:

| Name | Value |
| --- | --- |
| `APPSTORE_API_PRIVATE_KEY` | Full multiline contents of the downloaded `AuthKey_*.p8` |
| `APPSTORE_CERTIFICATES_FILE_BASE64` | Base64-encoded bytes of `distribution.p12` |
| `APPSTORE_CERTIFICATES_PASSWORD` | The password entered when exporting `distribution.p12` |

On Windows PowerShell, get the base64 for the GitHub secret with:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes((Resolve-Path ".\distribution.p12")))
```

Copy the output **directly into the GitHub Actions secret**, never into a committed file or public log. Delete temporary copies when safe. Keep the certificate/key backups somewhere secure.

For **my server (Tailscale)** builds, the original `TS_AUTHKEY` and `GAME_SERVER_URL` secrets are also needed; select **uploaded with the website builder** to reuse the existing `build-inputs` branch instead.

## 4. App icon (already configured)

The approved V-on-sunset icon is now committed in the Xcode asset catalog:

- `app/App/Assets.xcassets/AppIcon.appiconset/IconSource.jpg` — compressed copy for the cloud build.
- `app/App/Assets.xcassets/AppIcon.appiconset/Contents.json` — references the generated 1024x1024 PNG.

The TestFlight workflow runs `sips` on the macOS runner to create the required opaque `AppIcon-1024.png` **before** it invokes XcodeGen/Xcode. You do not need to supply icon metadata manually.

The original artwork was generated at higher resolution; the repository copy is optimized for compact transfer and will be upscaled to 1024x1024 during the build. For the sharpest possible artwork, replace `IconSource.jpg` with a high-resolution source image and adapt the conversion step, or commit a full-resolution `AppIcon-1024.png` and remove the conversion step. This icon is unofficial fan-project artwork, not licensed Rockstar Games branding.

## 5. Run your build from GitHub

1. Open https://github.com/EmilyUwU666/my-fanport-build/actions/workflows/testflight.yml
2. Choose **Run workflow** on branch `main`.
3. Use **uploaded with the website builder** if your personal inputs already exist in `build-inputs`; otherwise choose Tailscale and ensure your server is reachable.
4. Read build logs if a step fails. A signing error generally means the certificate, Team ID, bundle ID or provisioning profile does not match; a build error can require source changes for a newer Xcode/iOS SDK.
5. After successful upload, check **App Store Connect → Your app → TestFlight**. Apple's build processing is separate from CI, and the upload does **not** guarantee processing success.
6. Create an **internal testing group** and add the appropriate App Store Connect team members as testers. Internal testing is limited to 100 eligible team members. The workflow does not submit to external testing or App Review.

The workflow uses a unique `CFBundleVersion` derived from the GitHub Actions run/attempt, to avoid duplicate build numbers, while preserving the version `1.2.1`.

## Current limitations

- This workflow cannot run successfully until the signing secrets, profile, Apple app record, and icon are configured.
- No Apple upload has been attempted by the addition of the workflow.
- If Xcode 26 reports source compilation incompatibilities, fix those errors in the port itself; newer SDKs cannot be applied by changing IPA metadata alone.
- Because the wasm2c output is linked into the executable, asset-only importing doesn't avoid distribution of the compiled engine.
- Full provisioning/signing and Apple processing remain subject to Apple's rules, even for internal testing.
