name: Build Android APK

on:
  push:
    branches:
      - main
  workflow_dispatch: {}

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up JDK 11
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '11'

      - name: Install Android commandline tools and SDK components
        shell: bash
        run: |
          set -e
          sudo apt-get update
          sudo apt-get install -y wget unzip
          ANDROID_SDK_ROOT="$HOME/android-sdk"
          mkdir -p "$ANDROID_SDK_ROOT/cmdline-tools"
          cd "$HOME"
          wget -q https://dl.google.com/android/repository/commandlinetools-linux-9477386_latest.zip -O cmdline-tools.zip
          unzip -q cmdline-tools.zip -d "$ANDROID_SDK_ROOT/cmdline-tools"
          if [ -d "$ANDROID_SDK_ROOT/cmdline-tools/cmdline-tools" ]; then
            mv "$ANDROID_SDK_ROOT/cmdline-tools/cmdline-tools"/* "$ANDROID_SDK_ROOT/cmdline-tools/"
          fi
          export PATH="$ANDROID_SDK_ROOT/cmdline-tools/bin:$PATH"
          yes | sdkmanager --sdk_root="$ANDROID_SDK_ROOT" --licenses
          sdkmanager --sdk_root="$ANDROID_SDK_ROOT" "platform-tools" "platforms;android-33" "build-tools;33.0.2"
          echo "ANDROID_SDK_ROOT=$ANDROID_SDK_ROOT" >> $GITHUB_ENV
          echo "$ANDROID_SDK_ROOT/platform-tools" >> $GITHUB_PATH

      - name: Cache Gradle
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: gradle-cache-${{ runner.os }}-${{ hashFiles('**/*.gradle*', '**/gradle/wrapper/gradle-wrapper.properties') }}
          restore-keys: |
            gradle-cache-${{ runner.os }}-

      - name: Run Gradle assembleDebug
        uses: gradle/gradle-build-action@v2
        with:
          arguments: assembleDebug

      - name: Upload APK artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-apk
          path: app/build/outputs/apk/**/*.apk
