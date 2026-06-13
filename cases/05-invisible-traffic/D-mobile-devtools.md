# 事例 5-D: iOS Web Inspector / Android Studio Network Profiler

## 前提 / install

- iOS Web Inspector: [Xcode](https://apps.apple.com/jp/app/xcode/id497799835) install + macOS Safari の「開発」メニュー有効化
- Android Studio: [公式](https://developer.android.com/studio) から install、USB デバッグ有効化された Android 端末

## コード

手順（iOS、番号付き）:

1. iPhone: 設定 → Safari → 詳細 → Web インスペクタを ON
2. macOS Safari → 環境設定 → 詳細 → 「メニューバーに開発メニューを表示」を ON
3. iPhone を USB で macOS に接続（信頼ダイアログを承認）
4. Safari の「開発」メニュー → iPhone 名 → 対象 WKWebView を選択
5. DevTools が開き、Network タブで通信を観察

手順（Android、番号付き）:

1. Android Studio で対象プロジェクトを開く
2. Android 端末の 設定 → デベロッパーオプション → USB デバッグを ON
3. 端末を USB 接続、PC 側で USB デバッグ認証 prompt を承認
4. `Run → Run 'app'` で debuggable build を起動
5. `View → Tool Windows → App Inspection` を開く
6. App Inspection ペインの **「Network Inspector」** タブで OkHttp / HttpURLConnection / Java HttpClient 経由のリクエストが時系列表示

注: Android Studio Flamingo (2022.2.1) 以降、旧 Profiler の Network タブは廃止され、Network Inspector（App Inspection 配下）に分離された。Hedgehog / Iguana / Jellyfish 等の現行版では旧 Profiler の Network UI は存在しない（[Network Inspector docs](https://developer.android.com/studio/preview/features/network-inspector) / 移行前の旧 [Network Profiler docs](https://developer.android.com/studio/debug/network-profiler)）。

## 期待出力

- iOS: macOS Safari の DevTools と同じ操作感で Network タブ・コンソール・要素検査が使える
- Android: Profiler に Network チャートと individual request 一覧、payload プレビュー

## ハマりポイント

- **iOS Web Inspector は WKWebView 内の通信 + debug build 限定**。TestFlight / App Store ビルドでは Web インスペクタが無効
- **Android Network Inspector は OkHttp / HttpURLConnection / Java HttpClient 経由のみ**。生 socket / WebSocket / 独自ライブラリは見えない → [mitmproxy（A）](A-mitmproxy.md)にフォールバック
- USB デバッグの認証 prompt を承認しないと端末が認識されない
