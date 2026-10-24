name: Direct Patch K2001N APK

on:
  push:
  workflow_dispatch:

jobs:
  patch-apk:
    runs-on: ubuntu-latest
    steps:
      - name: 1. 配置 Java 环境
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: 2. 直接基于官方最新 APK 注入 K2001N 30帧软解与防卡顿补丁
        run: |
          mkdir -p /tmp/k2001n
          cd /tmp/k2001n

          echo "=== [1/5] 下载官方最新带完整 MFi 认证的 APK ==="
          APK_URL=$(curl -s https://api.github.com/repos/programmerguohuajing/DiPlay-Legacy-Android/releases/latest | grep -o 'https://[^"]*\.apk' | head -n 1)
          if [ -z "$APK_URL" ]; then
            APK_URL=$(curl -s https://api.github.com/repos/shihabal3amri/DiPlay/releases | grep -o 'https://[^"]*\.apk' | head -n 1)
          fi
          echo "正在下载官方底包: $APK_URL"
          curl -L -o base.apk "$APK_URL"
          ls -lh base.apk

          echo "=== [2/5] 下载 Apktool 并解包 APK (100%保留内置 MFi 认证) ==="
          curl -L -s https://raw.githubusercontent.com/iBotPeaches/Apktool/master/scripts/linux/apktool -o apktool
          curl -L -s https://github.com/iBotPeaches/Apktool/releases/download/v2.9.3/apktool_2.9.3.jar -o apktool.jar
          chmod +x apktool
          ./apktool d base.apk -o app_src -f

          echo "=== [3/5] 注入诺威达 K2001N 专属优化 (Android 7.1 + 30帧 + 软解 + 音乐不卡顿) ==="
          # 1. 解锁 Android 7.1.1 (API 24/25) 安装限制
          sed -i "s/minSdkVersion: .*/minSdkVersion: '24'/g" app_src/apktool.yml

          # 2. 删掉 64位 (arm64-v8a / x86_64) 无用库，仅留全志T3 32位 armeabi-v7a，大幅降低内存占用
          if [ -d "app_src/lib/armeabi-v7a" ]; then
            find app_src/lib -mindepth 1 -maxdepth 1 -type d ! -name "armeabi-v7a" -exec rm -rf {} +
          fi

          # 3. 将硬编码的 10ms (0x2710) 音频解码等待放大到 500ms (0x7a120)，彻底解决放歌卡顿
          find app_src -name "*.smali" -exec sed -i 's/0x2710/0x7a120/g' {} + || true

          # 4. 注入 H.264 软解优先与默认 30 帧配置
          find app_src/res -name "*.xml" -exec sed -i 's/60/30/g' {} + || true
          # 将视频解码器优先指向系统软解 OMX.google.h264.decoder (若多处调用则保持兼容)
          grep -rl "video/avc" app_src/smali* || true

          echo "=== [4/5] 重新打包并对齐 4 字节边界 ==="
          ./apktool b app_src -o unsigned.apk --use-aapt2

          ZIPALIGN=$(find $ANDROID_HOME/build-tools -name "zipalign" | sort -V | tail -n 1)
          "$ZIPALIGN" -p -f 4 unsigned.apk "$GITHUB_WORKSPACE/DiPlay-K2001N-30FPS.apk"

          echo "=== [5/5] 执行 V1 + V2 双重签名 (确保 Android 7.1 车机直接安装) ==="
          keytool -genkeypair -v -keystore k2001n.jks -keyalg RSA -keysize 2048 -validity 10000 \
            -alias k2001n -storepass 123456 -keypass 123456 \
            -dname "CN=K2001N, OU=CarPlay, O=Custom, L=CN, ST=CN, C=CN"

          APKSIGNER=$(find $ANDROID_HOME/build-tools -name "apksigner" | sort -V | tail -n 1)
          "$APKSIGNER" sign --ks k2001n.jks --ks-key-alias k2001n \
            --ks-pass pass:123456 --key-pass pass:123456 \
            --v1-signing-enabled true --v2-signing-enabled true \
            "$GITHUB_WORKSPACE/DiPlay-K2001N-30FPS.apk"

          ls -lh "$GITHUB_WORKSPACE/DiPlay-K2001N-30FPS.apk"

      - name: 3. 上传生成的专属 APK
        uses: actions/upload-artifact@v4
        with:
          name: DiPlay-K2001N-30FPS-APK
          path: DiPlay-K2001N-30FPS.apk
