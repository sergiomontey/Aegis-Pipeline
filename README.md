# Aegis-Pipeline


# Aegis Pipeline — Governance‑First CI/CD for Mobile (RN + iOS + Android)

> Production‑grade delivery pipeline showcasing **GitHub Actions**, **Fastlane**, **secure key management**, **telemetry**, and **MDM readiness**.

---

## What This Repo Demonstrates
- Branching model: `main` (release) and `develop` (integration)
- CI on every push to `develop`
- Lint → Test → Build → Sign → Distribute (iOS `.ipa`, Android `.aab`/`.apk`)
- Secrets isolated in GitHub Actions **Secrets**
- Telemetry (Sentry/Datadog), HTTPS‑only API, encrypted storage
- **Compliance artifacts:** audit logs, SBOM, model cards

---

## Workflow (GitHub Actions)

`.github/workflows/ci.yml` (excerpt):

```yaml
name: CI

on:
  push:
    branches: [develop]

jobs:
  build:
    runs-on: macos-14

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with: { node-version: '20' }

      - name: Install JS deps
        run: |
          yarn install --frozen-lockfile
          yarn lint
          yarn test --ci

      - name: iOS deps
        run: |
          cd ios && pod install --repo-update && cd ..

      - name: Build iOS with Fastlane
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          APPLE_API_KEY_BASE64: ${{ secrets.APPLE_API_KEY_BASE64 }}
        run: |
          bundle install
          bundle exec fastlane ios ci

      - name: Build Android
        run: ./gradlew assembleRelease

      - name: Upload Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: mobile-builds
          path: |
            ios/*.ipa
            android/app/build/outputs/**/*.aab
            android/app/build/outputs/**/*.apk
```

---

## Fastlane (lanes)

`fastlane/Fastfile` (excerpt):
```ruby
default_platform(:ios)

platform :ios do
  desc "CI lane"
  lane :ci do
    increment_build_number
    build_app(scheme: "AuraLearn", export_method: "ad-hoc")
    upload_to_testflight(skip_waiting_for_build_processing: true)
  end
end

platform :android do
  lane :ci do
    gradle(task: "assemble", build_type: "Release")
    # supply(track: "internal", aab: "app-release.aab")
  end
end
```

**Signing & Secrets**
- iOS: Use App Store Connect API key stored as base64 in `APPLE_API_KEY_BASE64` secret.  
- Android: Keystore in GitHub Encrypted Secrets or external vault; inject via workflow at build time.  
- Never commit credentials.

---

## Security & Governance

- **HTTPS everywhere**; no hardcoded keys (use env/Secrets).  
- **Encrypted local data** (Keychain/Keystore; SQLCipher/EncryptedFile).  
- **SBOM** (CycloneDX), **SAST** (CodeQL), Dependency scan (OWASP/DC).  
- **Audit Logs** exported from app for compliance reviews.  
- **Model Cards** versioned with model artifacts.

---

## Telemetry

- Crash reporting + performance traces (Sentry or Datadog).  
- Release health tied to build numbers from CI.  
- Redact PII from breadcrumbs.

---

## MDM Readiness (iOS example)

**Info.plist** managed app configuration key (excerpt):
```xml
<key>com.apple.configuration.managed</key>
<dict>
  <key>API_BASE_URL</key>
  <string>https://mdm-configured.example.com</string>
</dict>
```

---

## Local Repro

```bash
# Lint & test
yarn lint && yarn test

# iOS
cd ios && pod install && cd ..
bundle install
bundle exec fastlane ios ci

# Android
./gradlew assembleRelease
```

---

## Roadmap
- [ ] Distribute via Firebase App Distribution in CI.
- [ ] Add notarization step for mac Catalyst targets (if used).
- [ ] Automated release notes from conventional commits.
- [ ] Provenance: sign builds with Sigstore/cosign.


## License
Copyright (c) 2025 MonteyAI LLC. All rights reserved.

This software and associated documentation files (the "Software") are proprietary 
to MonteyAI LLC. and are protected by copyright law.

VIEWING PERMITTED: This code is available for viewing, educational, and 
evaluation purposes only.

RESTRICTIONS:
- No commercial use without explicit written permission from MONTEYcodes
- No redistribution, modification, or derivative works
- No reverse engineering or decompilation
- No use in production environments without valid license
