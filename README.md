# ACTLOG（アクティビティログ）

毎日の積み上げ（ライド・筋トレ・体組成・飲酒）を記録して見える化する個人用Webアプリ。
GitHub Pages ＋ Firebase（Auth / Firestore、Sparkプラン＝無料）で動きます。

## ファイル構成

| ファイル | 役割 |
|---|---|
| index.html | アプリ本体（画面・処理すべて） |
| manifest.webmanifest / sw.js | ホーム画面に追加してアプリとして使うための設定、オフライン対応 |
| icon-192.png / icon-512.png | アプリアイコン |
| firestore.rules | Firestoreのセキュリティルール（コンソールに貼り付ける用） |

## 初期設定（1回だけ）

### 1. Firebase
1. https://console.firebase.google.com で「プロジェクトを追加」（Googleアナリティクスはオフ）
2. プロジェクトの設定 → マイアプリ →「ウェブ（</>）」でアプリを追加（Hostingのチェックは不要）
3. 表示された `firebaseConfig` の値を、index.html 上部の `firebaseConfig` に貼り付け
4. Authentication → 始める → ログイン方法
   - 「Google」を有効化
   - 「メール/パスワード」も有効化（Googleログインがうまくいかない環境用の予備）
5. Authentication → 設定 → 承認済みドメイン に `watanabe-jpg.github.io` を追加
6. Firestore Database → データベースを作成
   - ロケーション：`asia-northeast1`（東京）※後から変更不可
   - 本番環境モードで開始
7. Firestore → ルール に `firestore.rules` の中身を貼り付けて「公開」

### 2. GitHub Pages
1. このフォルダの中身をリポジトリ直下に push
2. リポジトリの Settings → Pages → Branch を `main` / `/(root)` にして Save
3. 数分後 https://watanabe-jpg.github.io/activity-log/ で開ける

### 3. スマホ
- iPhone：Safariで開く → 共有 →「ホーム画面に追加」
- Android：Chromeで開く → メニュー →「ホーム画面に追加」

## データ構造（Firestore）

```
users/{uid}/days/{YYYY-MM-DD}
  date, weight, fat, muscle（骨格筋率）, alcohol(true/false),
  sets: { squat:[15,15,15], plank:[45,60,...], ... }, updatedAt
users/{uid}/settings/main
  maxHR, zones:[Z1上限, Z2上限, Z3上限, Z4上限], exercises:[...]
```

## 段階
- 段階1（このバージョン）：ログイン、体組成・飲酒・筋トレの記録、ホーム（今日・連続記録・積み上げ・年間ヒートマップ）、設定
- 段階2：FIT取り込み・ライド詳細（地図、心拍・ケイデンス・標高・気温グラフ、ゾーン、Z2達成率、心拍ドリフト）
- 段階3：体組成の推移グラフ、飲酒と翌日体重の相関、Claude用サマリーのコピー、JSON書き出し・読み込み
