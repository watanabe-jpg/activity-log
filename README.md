# ACTLOG（アクティビティログ）

毎日の積み上げ（ライド・筋トレ・体組成・血圧・散歩・飲酒・食事）を記録して見える化する個人用Webアプリ。
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

### 3. AI（食事の写真判定）の初期設定
食事タブの写真判定は Firebase AI Logic（Gemini Developer API の無料枠）を使います。Sparkプラン（無料）のまま使えます。

1. reCAPTCHA のキーを作る：https://www.google.com/recaptcha/admin/create
   - 種類：「スコアベース（v3）」
   - ドメイン：`watanabe-jpg.github.io`
   - 作成後に表示される「サイトキー」と「シークレットキー」を控える
2. Firebaseコンソール → セキュリティ（Security）→ App Check → アプリ → ウェブアプリ →「reCAPTCHA」
   - 1のシークレットキーを貼り付けて保存
3. Firebaseコンソール → AIサービス（AI Services）→ AI Logic →「始める（Get started）」
   - Gemini API プロバイダは「Gemini Developer API」を選ぶ（請求先アカウントの登録は不要）
4. index.html 上部の `RECAPTCHA_SITE_KEY` に1のサイトキーを貼り付けて push

メモ
- 写真はAIに送ったあと、長辺1024pxの画像は捨て、200pxのサムネイルだけ Firestore に保存します（Cloud Storage は使わない＝Sparkのまま）
- Gemini の無料枠では、送った内容がGoogleの製品改善に使われます。人の顔や書類が写り込まないようにしてください
- 使うモデルは index.html の `AI_MODELS`（左から順に試し、無料枠の上限などで失敗したら次へ）

### 4. スマホ
- iPhone：Safariで開く → 共有 →「ホーム画面に追加」
- Android：Chromeで開く → メニュー →「ホーム画面に追加」

## データ構造（Firestore）

```
users/{uid}/days/{YYYY-MM-DD}
  date, weight, fat, muscle（骨格筋率）, alcohol(true/false),
  sets: { squat:[15,15,15], plank:[45,60,...], ... }, updatedAt
  walks:[{slot,min,dogs}], bp:{am:{sys,dia},pm:{...}}, rides:[rideId...],
  meal:{ kcal,p,f,c,salt,alcG,n, groups:{主食:n,...}, veg:[...] }  ← その日の食事の合計（食事を保存するたびに再計算）
users/{uid}/meals/{id}
  date, slot(b朝/l昼/s間食/d夜), at, src(photo/text/again), note, thumb(サムネイルのdataURL),
  items:[{ name, kind, amount, group, vegKinds, base:{kcal,p,f,c,salt}, factor(量の倍率), alc:{type,ml,abv,other}|null, manual }],
  total:{ kcal,p,f,c,salt,alcG }
users/{uid}/settings/main
  maxHR, zones:[Z1上限, Z2上限, Z3上限, Z4上限], exercises:[...], dogs, bpRef, privacyM,
  sections:{ body,bp,walk,alcohol,strength,ride,meal }（表示する項目）, saltTarget（塩分の目標量g）
```

お酒のカロリーは「純アルコール量(g) = ml × 度数 × 0.8」に 7kcal/g を掛け、糖質などの分（AIの推定値 kcal/100ml）を足して計算します。

## 段階
- 段階1：ログイン、体組成・飲酒・筋トレの記録、ホーム（今日・連続記録・積み上げ・年間ヒートマップ）、設定
- 段階2：FIT取り込み・ライド詳細（地図、心拍・ケイデンス・標高・気温グラフ、ゾーン、Z2達成率、心拍ドリフト）
- 段階3：体組成の推移グラフ、飲酒と翌日体重の相関、Claude用サマリーのコピー、JSON書き出し・読み込み

### 食事（カロリー・栄養バランス）
- 食事 第1段階（v1.4.0）：写真 → AI判定 → 量・サイズ・濃さの確認 → 当日合計（PFC・塩分・純アルコール・食品の区分）、もう1杯/もう1つ、写真なしの入力、表示する項目のオン/オフ
- 食事 第2段階：消費カロリー（体組成からの推定＋Garminの合計kcal）、摂取との差、目標設定
- 食事 第3段階：AIの総評（口調は設定で切替）、PCでの履歴・グラフ、推定の自動補正
