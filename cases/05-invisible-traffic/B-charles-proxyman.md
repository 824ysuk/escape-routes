# 事例 5-B: Charles Proxy / Proxyman

## 前提 / install

- macOS / Windows / Linux: [Charles Proxy 公式](https://www.charlesproxy.com/) から download。無料版は 30 分セッション制限あり、ライセンス $50（永久）
- macOS のみ: [Proxyman](https://proxyman.io/)（無料版あり、Premium $59 一括）

## コード

手順（番号付きで実行順）:

1. Charles を起動
2. メニュー `Proxy → macOS Proxy → ON` を選択（システム全体の通信を Charles 経由に）
3. メニュー `Help → SSL Proxying → Install Charles Root Certificate` を選択
4. macOS Keychain Access で当該証明書を選択 → 情報を見る → 「信頼」→「この証明書を使用するとき」→「常に信頼」に変更
5. メニュー `Proxy → SSL Proxying Settings` を開く → "Enable SSL Proxying" を ON
6. `Add` をクリック → Host に `*.example.com`、Port に `443` を入力（ワイルドカード可）
7. 対象アプリを操作 → Charles 画面左に全リクエストがツリー表示される
8. 個別 request を選択 → 右ペインで Request / Response / Contents を確認
9. Request 右クリック → Repeat / Edit / Compose で再送・改変

モバイル設定:

1. メニュー `Help → SSL Proxying → Install Charles Root Certificate on a Mobile Device or Remote Browser` を選択
2. 表示される URL（例: `chls.pro/ssl`）をモバイルブラウザで開く
3. 証明書を install
4. モバイル端末の Wi-Fi proxy を Mac の IP に向ける

## 期待出力

- Charles のメイン画面に Structure（ホスト別ツリー）/ Sequence（時系列）ビュー
- SSL Proxying 対象ホストは body の中身まで復号して見える
- 右ペインで Contents タブを開くと JSON が整形表示される

## ハマりポイント

- Charles 無料版は 30 分でセッション切れ。長時間観察するなら有料ライセンス必須
- SSL Proxying Settings に追加していないホストは「Unknown」と表示されて中身が見えない
- Proxyman は macOS のみ（Windows / Linux 版はない）
