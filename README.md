# -sanjit-fluenttalk
The folder `sanjit-fluenttalk` contains the full working app.
name: Android APK

on:
  workflow_dispatch:
    inputs:
      publish_release:
        description: Create a GitHub Release after a successful build
        required: true
        type: boolean
        default: false
      tag_name:
        description: Existing Git tag to attach the APK to (required for release)
        required: false
        type: string

permissions:
  contents: read

concurrency:
  group: android-apk-${{ github.ref }}
  cancel-in-progress: false

jobs:
  build:
    name: Build Android APK
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: sanjit-fluenttalk

    steps:
      - name: Check out source
        uses: actions/checkout@v4

      - name: Set up Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"

      - name: Set up Android SDK
        uses: android-actions/setup-android@v3

      - name: Set up Gradle
        uses: gradle/actions/setup-gradle@v4

      - name: Verify Gradle wrapper
        run: |
          test -f ./gradlew || {
            echo "::error::Gradle wrapper not found at sanjit-fluenttalk/gradlew"
            exit 1
          }
          chmod +x ./gradlew

      - name: Build debug APK
        shell: bash
        run: |
          set -o pipefail
          ./gradlew --no-daemon --stacktrace assembleDebug 2>&1 \
            | tee "$RUNNER_TEMP/gradle-build.log"

      - name: Upload APK artifact
        if: ${{ always() }}
        uses: actions/upload-artifact@v4
        with:
          name: android-apk-${{ github.run_id }}-${{ github.run_attempt }}
          path: sanjit-fluenttalk/**/build/outputs/apk/**/*.apk
          if-no-files-found: warn
          retention-days: 30

      - name: Upload build logs and reports
        if: ${{ always() }}
        uses: actions/upload-artifact@v4
        with:
          name: android-build-diagnostics-${{ github.run_id }}-${{ github.run_attempt }}
          path: |
            ${{ runner.temp }}/gradle-build.log
            sanjit-fluenttalk/**/build/reports/**
            sanjit-fluenttalk/**/build/test-results/**
          if-no-files-found: warn
          retention-days: 30

  release:
    name: Create GitHub Release
    needs: build
    if: ${{ success() && inputs.publish_release }}
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Validate release tag input
        env:
          TAG_NAME: ${{ inputs.tag_name }}
        run: |
          if [[ -z "$TAG_NAME" ]]; then
            echo "::error::Enter an existing tag name to publish a release."
            exit 1
          fi
          if [[ ! "$TAG_NAME" =~ ^[A-Za-z0-9._/-]+$ ]]; then
            echo "::error::The tag name contains unsupported characters."
            exit 1
          fi

      - name: Check out source
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Verify tag already exists
        env:
          TAG_NAME: ${{ inputs.tag_name }}
        run: |
          git ls-remote --exit-code origin "refs/tags/$TAG_NAME" >/dev/null || {
            echo "::error::Tag '$TAG_NAME' does not exist on the remote. No tag will be created."
            exit 1
          }

      - name: Download APK artifact
        uses: actions/download-artifact@v4
        with:
          name: android-apk-${{ github.run_id }}-${{ github.run_attempt }}
          path: release-apk

      - name: Publish APK to GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          tag_name: ${{ inputs.tag_name }}
          files: release-apk/**/*.apk
          fail_on_unmatched_files: true
          generate_release_notes: true