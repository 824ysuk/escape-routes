# 事例 5-B: Charles Proxy / Proxyman

## 前提 / install

- macOS / Windows / Linux: [Charles Proxy 公式](https://www.charlesproxy.com/) から download。無料版は 30 分セッション制限あり、ライセンス $50（永久）
- macOS のみ: [Proxyman](https://proxyman.io/)（無料版あり、Premium $59 一括）

## コード

手順（番号付きで実行順）:

1. Charles を起動
2. メニュー `Proxy → macOS Proxy → ON` を選択（システム全体の通信を Charles 経由に）

    注: ON にすると Slack / VS Code / Homebrew / npm / curl / git over HTTPS 等、macOS の **全アプリの通信** が Charles に向く。Charles を意図せず落とすと「システム全死」状態になる。観察対象を絞るなら、macOS Proxy は **OFF のまま** で端末側（モバイル）の Wi-Fi proxy のみ Charles に向ける運用が安全（[Charles macOS Proxy docs](https://www.charlesproxy.com/documentation/proxying/macos-proxy/)）

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
- **使用後の後始末は 3 点セット**（戻し忘れると Charles 停止後に通信全死、または CA を残したまま [`Charles Root Certificate` の秘密鍵](https://www.charlesproxy.com/documentation/proxying/ssl-proxying/) が漏洩した場合に任意 TLS 通信を MITM される）:
    1. メニュー `Proxy → macOS Proxy → OFF`
    2. Keychain Access で `Charles Root Certificate` を削除（または「常に拒否」へ）
    3. モバイル端末側の Wi-Fi proxy 設定を OFF + 証明書削除（iOS は「証明書信頼設定で OFF」、Android は「ユーザー証明書 → mitmproxy/Charles を削除」）
