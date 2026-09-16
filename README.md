# upload_build_to_testflight

A composite GitHub Action that uploads a **pre-built iOS `.ipa`** to App Store Connect / TestFlight and finishes the post-upload setup:

1. Selects the latest installed Xcode, by exporting `DEVELOPER_DIR` — no `sudo`, and the machine's active Xcode is left alone.
2. Uploads the IPA with `xcrun altool` using an App Store Connect API key.
3. Waits for the build to finish processing and appear in App Store Connect.
4. Optionally posts TestFlight "What's New" release notes.
5. Optionally sets the build's encryption compliance flag.

> This action does **not** build or sign your app. Build and sign the IPA in your
> workflow first, then pass its path via `ipa-path`. See [the example workflow](.github/workflows/launch.yaml).

## Usage

```yaml
- name: Upload to TestFlight
  uses: crianpiro/upload_build_to_testflight@v1
  with:
    working-directory: example
    ipa-path: ${{ steps.ipa.outputs.path }}
    app-store-connect-api-key-id: ${{ secrets.ASC_API_KEY_ID }}
    app-store-connect-api-issuer-id: ${{ secrets.ASC_API_ISSUER_ID }}
    app-store-connect-api-key-base64: ${{ secrets.ASC_API_KEY_BASE64 }}
    release-notes: ${{ secrets.RELEASE_NOTES }}
    uses-non-exempt-encryption: "false"
```

The action runs only on **macOS runners**, since it relies on Xcode / `xcrun altool` — hosted (`runs-on: macos-latest`) or self-hosted. It needs no privileges: Xcode is selected through `DEVELOPER_DIR`, so a runner account without passwordless `sudo` works.

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `ipa-path` | **yes** | — | Path to the pre-built `.ipa` to upload (absolute, or relative to `working-directory`). |
| `app-store-connect-api-key-id` | **yes** | — | App Store Connect API key id (the `kid`). |
| `app-store-connect-api-issuer-id` | **yes** | — | App Store Connect API issuer id. |
| `app-store-connect-api-key-base64` | **yes** | — | Base64 of the App Store Connect API private key (`.p8` contents). |
| `working-directory` | no | `./` | Directory every step runs in, and the base for a relative `ipa-path`. The bundle id and build number come from the IPA, so this need not be a checked-out Flutter project. |
| `release-notes` | no | `""` | TestFlight "What's New" text. Empty skips this step. |
| `locale` | no | `en-US` | Locale used when creating the beta build localization. |
| `uses-non-exempt-encryption` | no | `"false"` | Encryption compliance flag. `"false"` for apps using only exempt encryption, `"true"` otherwise. **Empty skips the step** (e.g. when `ITSAppUsesNonExemptEncryption` is declared in `Info.plist`). |

This action has no outputs.

### How the build is matched

Uploading needs nothing but the file. The identifiers below are for what comes after: once App Store Connect has processed the binary, the action has to find *that* build in order to post release notes to it and set its encryption compliance.

Both are read from the IPA's own `Payload/*.app/Info.plist`:

| Read from the IPA | Used for |
| --- | --- |
| `CFBundleIdentifier` | resolving the numeric app id via `/v1/apps?filter[bundleId]=` |
| `CFBundleVersion` | picking this build out of that app's builds — confusingly, a build's `version` in the App Store Connect API *is* its build number |

Taking them from the artifact rather than the source tree matters for two reasons. The source tree describes what *would* be built rather than what was; and a job that only downloads a prebuilt IPA has no `pubspec.yaml` and no Xcode project to read, so nothing needs checking out to use this action.

The case where the difference bites: a workflow that writes the version bump into the build job's workspace and commits it only after distribution. A checkout in the upload job then holds the *previous* build number, and the poll below hunts a build that never appears — failing fifteen minutes later with a timeout rather than naming the mismatch.

It retries for up to ~15 minutes (30 × 30s) while App Store Connect processes the upload.

## Creating the App Store Connect API key

1. In [App Store Connect → Users and Access → Integrations → App Store Connect API](https://appstoreconnect.apple.com/access/integrations/api), create a key with the **App Manager** role.
2. Note the **Key ID** and **Issuer ID**, and download the `.p8` (downloadable once).
3. Base64-encode the key for use as a secret:
   ```bash
   base64 -i AuthKey_XXXXXXXXXX.p8 | pbcopy
   ```
   Store the result as `ASC_API_KEY_BASE64`, the Key ID as `ASC_API_KEY_ID`, and the Issuer ID as `ASC_API_ISSUER_ID`.

## Example workflow

[`.github/workflows/launch.yaml`](.github/workflows/launch.yaml) demonstrates the full flow against the bundled `example/` app. It builds and signs the IPA with the companion [`crianpiro/build_flutter_app`](https://github.com/marketplace/actions/build-flutter-app) action — which writes the artifact to `<working-directory>/build/ios/ipa/app-release.ipa` — then passes that path to this action.

It expects these repository **secrets**:

| Secret | Purpose |
|--------|---------|
| `P12_BASE64` | Base64 of the signing certificate (`.p12`). |
| `P12_PASSWORD` | Password for the `.p12`. |
| `PROVISIONING_PROFILE_BASE64` | Base64 of the `.mobileprovision`. |
| `RUNNER_KEYCHAIN_PASSWORD` | Any password used for the temporary keychain. |
| `EXPORT_OPTIONS` | Raw Xcode `ExportOptions.plist` contents (XML). |
| `ASC_API_KEY_ID` | App Store Connect API key id. |
| `ASC_API_ISSUER_ID` | App Store Connect API issuer id. |
| `ASC_API_KEY_BASE64` | Base64 of the App Store Connect API key (`.p8`). |
| `RELEASE_NOTES` | *(optional)* "What's New" text for the build. |

## License

BSD 3-Clause — see [LICENSE](LICENSE) © Cristian Andres Picon Rodriguez.
