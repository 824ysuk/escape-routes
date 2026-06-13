# 事例 5-D: iOS Web Inspector / Android Studio Network Inspector

> **対象範囲**: 自社が開発したアプリの debug build に限る。公式ツールは Production / TestFlight / App Store / Google Play 配布 build では使えない設計になっている。

## 前提 / install

- iOS Web Inspector: [Xcode](https://apps.apple.com/jp/app/xcode/id497799835) install + macOS Safari の「開発」メニュー有効化
- Android Studio: [公式](https://developer.android.com/studio) から install、USB デバッグ有効化された Android 端末
    - **Android Studio Hedgehog (2023.1.1) 以降**: 旧 Profiler の Network タブは廃止、**App Inspection → Network Inspector** に移行
    - Android Studio Dolphin (2021.3.1) 以降の名称変更を反映

## コード

### iOS Safari Web Inspector

1. iPhone: 設定 → Safari → 詳細 → Web インスペクタを ON
2. macOS Safari → 環境設定 → 詳細 → 「メニューバーに開発メニューを表示」を ON
3. iPhone を USB で macOS に接続（信頼ダイアログを承認）
4. Safari の「開発」メニュー → iPhone 名 → 対象 WKWebView を選択
5. DevTools が開き、Network タブで通信を観察

### Android Studio Network Inspector (Hedgehog 2023.1.1+)

1. Android Studio で対象プロジェクトを開く
2. Android 端末の 設定 → デベロッパーオプション → USB デバッグを ON
3. 端末を USB 接続、PC 側で USB デバッグ認証 prompt を承認
4. Android Studio で `Run → Run 'app'` でデバッグビルドを起動
5. **画面下部の `App Inspection` タブを開く** → `Network Inspector` を選択 (旧 Profiler → Network は廃止)
6. すべての OkHttp / HttpURLConnection 経由通信が時系列で表示

旧 Android Studio (2021 以前) を使っている場合は `View → Tool Windows → Profiler → Network` だが、**新規導入は Hedgehog+ を推奨**。

## 期待出力

- iOS: macOS Safari の DevTools と同じ操作感で Network タブ・コンソール・要素検査が使える
- Android: App Inspection → Network Inspector に Network チャートと individual request 一覧、payload プレビュー

## ハマりポイント

### iOS Web Inspector の対象範囲

- **対象は WKWebView 内の JS / リソース読み込みのみ**
- native Swift コードの `URLSession.shared.dataTask(...)` は **観察できない**
- `SFSafariViewController` も別プロセスで動作するため対象外
- `URLSessionWebSocketTask` の native WebSocket も対象外
- ハイブリッドアプリで「native API の通信が見えない」と混乱する筆頭 → [mitmproxy (A)](A-mitmproxy.md) にフォールバック
- TestFlight / App Store ビルドでは Web インスペクタが無効

### Android Network Inspector の対象範囲

- **OkHttp / HttpURLConnection / Java 標準 HTTP クライアント経由のみ**
- gRPC / 生 socket / 独自 HTTP ライブラリ / WebSocket の一部は見えない → [mitmproxy (A)](A-mitmproxy.md) にフォールバック
- App Inspection は debug build 専用 (release build には attach できない)

### USB / 接続

- USB デバッグの認証 prompt を承認しないと端末が認識されない
- iOS の場合は最初の USB 接続で「このコンピュータを信頼」を tap
- Android で `adb devices` に `unauthorized` と出たら再接続して prompt を承認

### 参考

- [Apple Developer: Safari Web Inspector](https://developer.apple.com/documentation/safari-developer-tools/inspecting-iphone-or-ipad-apps)
- [Android Developers blog: Inspect network traffic with the Network Inspector](https://android-developers.googleblog.com/2022/10/inspect-network-traffic-with-network.html)
- [Android Studio release notes (Hedgehog)](https://developer.android.com/studio/releases#hedgehog)
