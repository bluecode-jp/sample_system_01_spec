# 開発ノート：Firebase + React（TypeScript）でスマホ用バーコード Web アプリを作るときの注意点

Firebase（Hosting / Cloud Functions / Firestore Enterprise / Storage / Authentication）と React（Vite）の構成で、  
スマホのカメラでバーコードを読み取る Web アプリと管理画面を開発したときに、詰まった点・ハマりどころをまとめたものです。  
システム名やプロジェクト名に依存しない内容にしてあります。次回、同じ種類の開発を AI が行うときに最初に読むことを想定しています。

- 各項目は「症状 → 原因 → 対処」の順に書いています。
- 文中の `<system>`（システム名）、`<project-id>`（Firebase プロジェクトID）、`<scope>`（npm のスコープ名）、`<region>`（リージョン）は、案件ごとに読み替えてください。
- ファイルパスは前回の構成（下記）での例です。

```
<repo>/
├─ packages/shared/   # 型・スキーマ・共通ロジック（@<scope>/shared）
├─ packages/web/      # 両 SPA の共通コード（Firebase 初期化・API・認証・共通 UI）
├─ apps/client/       # スマホ用クライアント
├─ apps/admin/        # 管理画面
├─ functions/         # Cloud Functions（REST API）
├─ scripts/           # シード・削除スクリプト
└─ tests/             # セキュリティルール・API のテスト
```

---

## 0. 前回の仕様判断（同じ仕様なら流用できる参考例）

仕様書に書かれていない点で、前回ユーザーと確認して決めたことです。仕様が同じなら、そのまま提案に使えます。システム名などの固有の値は、毎回仕様書とユーザーに確認してください。


| 項目                | 前回の決定                                                                            |
| ----------------- | -------------------------------------------------------------------------------- |
| 仕様書の誤記の疑い         | 仕様書とディレクトリ名でシステム名が食い違っていた。推測で決めず、ユーザーに確認した                                       |
| 商品ID（独自採番・10桁）    | 接頭辞 `20` + 8桁連番。JAN のインストアコード帯（20〜29）なので既製品のコードと衝突しない。採番はカウンタのドキュメントをトランザクションで更新 |
| ダミー商品の写真          | 外部 API を使わず、スクリプトで SVG イラストを生成（API キー不要・何度でも再生成できる）                              |
| Firebase 本番プロジェクト | 後から用意。開発はエミュレータで完結させた（Blaze プランが必要）                                              |
| 商品名の LIKE 検索      | Firestore Enterprise の Pipeline（`stringContains`）を使う（→ 5章）                       |
| 在庫の変更             | 編集画面では変えられないようにし、履歴が残る「加算・減算」の API だけで変える                                        |


---

## 1. ライブラリのバージョン差

学習時点の知識より新しいメジャー版が入ることがある。**書く前に** `node_modules` **の型定義で API を確認すること。**  
以下は 2026年9月時点で入った版での例。

### MUI v9（React 用 UI 部品ライブラリ）

- **症状**: `<Stack alignItems="center">`、`<Typography fontWeight={700}>` が型エラーになる。
- **原因**: v9 ではシステムプロパティ（`alignItems`、`fontWeight`、`mt` など）が廃止された。
- **対処**: すべて `sx={{ ... }}` で指定する。
- `TextField` の `InputProps` / `inputProps` は `slotProps={{ input: …, htmlInput: … }}` を使う。
- **アイコン名の変更**: `DeleteOutline` → `DeleteOutlined`、`PeopleOutline` → `PeopleOutlined`。`ls node_modules/@mui/icons-material | grep <名前>` で実在を確認する。
- `Grid` は `size={4}` 形式（旧 Grid2 の API）。

### React Router v8

- `react-router-dom` ではなく `react-router` から import する（`createBrowserRouter`、`RouterProvider`、`useNavigate` など）。

### その他

- **zod v4**: エラーメッセージは `{ message: '…' }` で指定する。フォームの数値は `z.coerce.number()` を使う。
- **Express 4 と** `@types/express`: 何も指定しないと型定義は v5 が入り、本体（v4）と合わない。`@types/express@4` を明示する。
- **Vite 8**: `vite.config.ts` から相対パスで `.ts` を import するときは、拡張子を付ける（例 `'../../packages/web/vite-proxy.ts'`）。付けないと警告が出る。
- **Node のバージョン**: Functions の `engines` とローカルの Node のバージョンが違うと `EBADENGINE` 警告が出るが、無害。

---

## 2. npm / モノレポ（npm workspaces）

- **npm 11 のインストールスクリプト制限**
  - `npm install` の後に `npm warn install-scripts … not yet covered by allowScripts` が出て、esbuild などの postinstall が実行されない。
  - 対処: `npm install-scripts approve esbuild @firebase/util protobufjs` → `npm rebuild <pkg>`。`re2` と `fsevents` は承認しなくても動いた。
- `npm install <pkg> -w <新しいワークスペース>` **が黙って何も入れないことがある**
  - 新しく作ったワークスペースへの初回インストールで、警告だけ出て `package.json` に追記されないことがあった。
  - 対処: インストール後に必ず `package.json` の dependencies を確認し、無ければもう一度実行する。
- **Functions をワークスペースに入れる場合**
  - デプロイ時、Cloud Build はワークスペース内の依存（`@<scope>/shared` など）を解決できない。
  - 対処: **esbuild でワークスペース内の依存を含めて1ファイルにバンドル**する（例 `functions/build.mjs`）。`external` は `functions/package.json` の dependencies にする。
  - 注意: `external` の一覧を dependencies から作る場合、サーバー SDK を直接 import するなら、そのパッケージを dependencies に直接書く。書かないとバンドルに取り込まれてしまう（例: `@google-cloud/firestore/pipelines`。firebase-admin の間接依存なので自動では入らない）。

---

## 3. Firebase エミュレータ

- `demo-` **で始まるプロジェクトID**（例 `demo-<system>`）を使うと、クラウドに接続せずにエミュレータだけで動く。本番プロジェクトが無くても開発できる。
- **Hosting エミュレータが意図せず起動する**（5000番は macOS の AirPlay が使っているため、別の番号で起動していた）。起動コマンドに `--only auth,firestore,storage,functions` を付ける。
- **データの保存（**`--export-on-exit`**）**
  - `pkill` で強制終了すると保存されない。
  - `firebase emulators:start` のプロセスに **SIGINT を送る**（`kill -INT <pid>`）と保存される。
  - 止めた後に、保存先のディレクトリに `firebase-export-metadata.json` があるかで確認する。
- **起動完了を待たずにシードを実行すると失敗する**。ログに `All emulators ready` が出るまで待つ。Firestore を Enterprise 版で起動すると 30 秒以上かかることがある。
- **Storage エミュレータで** `getDownloadURL`**（firebase-admin）が 403 になる**（`Permission denied. No READ permission`）
  - 対処: `file.getMetadata()` でダウンロードトークンを取り出し（無ければ `setMetadata` で付与し）、URL を自前で組み立てる。
  - URL の形式は `<endpoint>/v0/b/<bucket>/o/<encodeURIComponent(path)>?alt=media&token=<token>`。接続先は、エミュレータなら `http://${FIREBASE_STORAGE_EMULATOR_HOST}`、本番なら `https://firebasestorage.googleapis.com`。
- **セキュリティルールで権限を判定する書き方**: 三項演算子は避け、`request.auth.token.get('role', '') in ['admin','user']` と書く。
- **ルールのテスト**（`@firebase/rules-unit-testing`）: 開発中のエミュレータを共有して使う場合は `clearFirestore()` を呼ばない。テスト用のドキュメントIDで作成・削除する。

---

## 4. LAN 上のスマホで HTTPS（カメラ）を使うための構成

- **カメラの** `getUserMedia` **は安全なコンテキストでしか動かない**（HTTPS、または localhost）。LAN IP でアクセスする場合は HTTPS が必須。
- **HTTPS のページから** `http://127.0.0.1:8080`**（エミュレータ）へ接続すると、混在コンテンツとしてブロックされる。**
- **対処**: エミュレータへの通信をすべて **dev server と同じオリジンのパスで中継**する（Vite の `server.proxy`）。
  
  | パス                                                                          | 転送先                                                  |
  | --------------------------------------------------------------------------- | ---------------------------------------------------- |
  | `/api`                                                                      | Functions（`/<project-id>/<region>/<関数名>/api…` に書き換え） |
  | `/identitytoolkit.googleapis.com`、`/securetoken.googleapis.com`、`/emulator` | Auth                                                 |
  | `/google.firestore.v1.Firestore`（ws）、`/v1/projects`                         | Firestore                                            |
  | `/v0/b`                                                                     | Storage                                              |
  
  - Firestore: `initializeFirestore(app, { host: location.host, ssl: location.protocol === 'https:', experimentalForceLongPolling: true })`。`connectFirestoreEmulator` は使わない。
  - Auth: `connectAuthEmulator(auth, location.origin)`。
  - エミュレータの画像 URL（`http://127.0.0.1:9199/...`）は、表示時に `location.origin` へ置き換える。
  - 証明書は `@vitejs/plugin-basic-ssl`（自己署名）。スマホでの証明書の許可は1回で済む。
- **Express の経路**: Hosting の rewrite 経由だと `/api/...`、関数を直接呼ぶと `/...` になるため、ルーターを両方にマウントする。
- **待ち受けアドレス**
  - Vite は、`host` を指定しないと **IPv6 の** `[::1]` **だけ**で待ち受けることがある。その場合 `127.0.0.1` や LAN IP でアクセスできない（ユーザーから「アクセスできない」と指摘された）。管理画面を含め、すべての dev server に `host: true` を付ける。
  - エミュレータ UI を LAN から開くには `emulators.ui.host` を `0.0.0.0` にする。エミュレータ本体は `127.0.0.1` のままでよい（プロキシで中継するため）。
- **自己署名証明書と自動テスト**: Playwright MCP も、iOS シミュレータの Safari も証明書エラーで止まる。
  - 対処: 環境変数（例 `CLIENT_HTTP=1`）で HTTP 起動に切り替えられる dev スクリプトを用意し、`http://localhost:<port>` でテストする。localhost は安全なコンテキストなので、HTTP でもカメラが使える。
- **ログのプロキシエラー**: `http proxy error … ECONNREFUSED` が起動直後に出るのは、開いたままのブラウザタブが、エミュレータの準備完了前に通信したため。準備完了後に出なければ問題ない。

---

## 5. 文字列の LIKE 検索（Firestore Enterprise の Pipeline）

- **経緯**
  - 最初は通常のクエリ（Core operations）で実装していた。名前を1〜2文字ずつに分けて配列フィールドに保存し、`array-contains` で候補を取ってから部分一致を判定する方式。Standard エディションでも使える。
  - ユーザーの指示で、Enterprise の Pipeline に切り替えた。
- **Pipeline での方式**
  - 表記揺れを吸収した「正規化済みの名前」を別フィールド（例 `searchName`）に保存する。正規化は NFKC 変換 → 小文字化 → カタカナをひらがなに → 空白除去。Pipeline は正規化してくれないため。
  - `db.pipeline().collection('<collection>').where(field('searchName').stringContains(q)).limit(n).execute()` で検索する。
  - `field` は `@google-cloud/firestore/pipelines` から import する。firebase-admin は Pipeline を再エクスポートしていない。
  - `stringContains` はリテラルの部分一致なので、エスケープは不要。`like` や `regexContains` はワイルドカード・正規表現のエスケープが必要になる。
  - インデックスを使わない全件走査に近い検索になる。件数が増えると読み取りの費用と速度に影響する。
- **エミュレータ**
  - Standard 版のエミュレータでは `ExecutePipeline requires the Database Edition to be 'enterprise'` になる。
  - `firebase.json` の `emulators.firestore.edition: "enterprise"` で Enterprise として起動する。起動ログに `started in enterprise edition` と出る。
  - 本番用の `firestore` 設定にも `edition: "enterprise"`、`location: "<region>"` を書いておく（両方が食い違うと警告が出る）。
- **Pipeline は onSnapshot に対応しない**
  - 対処: 検索（Pipeline）でドキュメントIDを得て、画面側はその ID を `where(documentId(), 'in', 30件ずつ)` で `onSnapshot` 購読し直す。
  - Enterprise のエミュレータでも、Core のクエリと onSnapshot は動作した。
- **データの移行**: 検索用フィールドを変えた場合は、既存データに無いのでシードで入れ直すか、移行スクリプトを用意する。

---

## 6. バーコードスキャン（`@zxing/browser`）のタイミング不具合

ユーザーから「スキャンと表示（リセット）のタイミングがおかしい」と指摘され、SimulatorCameraEx で再現した。  
1回読み取ったら停止する方式のスキャナで起きた問題です。

1. **2回目のスキャンで「スキャン中」のまま固まる**
   - 原因: バーコードが最初から映っていると、`decodeFromConstraints()` の Promise が返る前に読み取りのコールバックが発火する。コールバックで停止して `idle` にした後、遅れて返った処理が「停止済みの制御オブジェクト」を保存し、状態を `scanning` に戻していた。
   - 対処: 開始ごとにセッション番号を振る。読み取り済み・停止済み・古いセッションの制御オブジェクトは、返ってきた直後に `stop()` して捨てる。
2. **再開しても前回の結果が残る**（同じ商品を続けて読んでも画面が変わらない）
   - 対処: スキャン開始時に、前回の結果（URL のパラメータ・入力欄・エラー表示）をリセットする。
3. **再開直後に前回の映像を読む**
   - 対処: 最初のコマから 500ms は読み取り結果を無視する（ウォームアップ）。
   - 実機でも「同じ商品にカメラを向けたまま次のスキャンを押すと即座に同じ商品を読む」ので、その対策も兼ねている。
4. **読取枠の外側を暗くする** `boxShadow: 0 0 0 9999px` が、映像エリアの外（下の「スキャンを停止」ボタン）まで暗くしていた。
   - 対処: 映像エリアに `overflow: 'hidden'` を付ける。

---

## 7. SimulatorCameraEx + Maestro での実機相当テスト

- **ツール**
  - SimulatorCameraEx の CLI `simcamctl`（`/Applications/SimulatorCameraEx.app/Contents/MacOS/simcamctl`）で、iOS シミュレータのカメラ映像を切り替える。
  - 手順は SimulatorCameraEx リポジトリの `docs/AUTOMATION.md` にある。
  - なお、環境によってはSimulatorCameraExが無い場合もあるので注意（その場合は代替手段を用いる）。
- **Safari でも動く**
  - Safari の `getUserMedia` もシム（`SimCamWebShim.js`）で差し替えられる。カメラの許可ダイアログは出ない。
  - 状態は `printf '{"command":"status"}\n' | nc -w 2 127.0.0.1 47848` で確認できる。
- **映像の切り替え**
  - `set-source --code128 "<コード>"`、`--pattern`（何も映さない）。
  - 同じコードを続けて読ませるときは、間に `--pattern` を挟む。
- **シム側の既知の挙動（前回時点で未修正）**
  - カメラ映像がすべて閉じられた後も canvas に最後のコマが残り、次にカメラを開いた直後の1コマ目として返される。そのため、再開直後に古いコードを読む（6章の3の原因）。
- **Maestro で Safari の Web ページを操作するときの注意**
  - 画面を開くのは `xcrun simctl openurl <UDID> "http://localhost:<port>/<path>"`。
  - `hideKeyboard` は失敗する。代わりに、何もない見出しの文字などをタップしてキーボードを閉じる。
  - ログイン後に「パスワードを保存しますか？」が出る。`tapOn: { text: "このウェブサイトでは保存しない", optional: true }` で閉じる。
  - **MUI のアイコン付きボタンは、アクセシビリティ上の文字の先頭にゼロ幅スペースが入る**（例 `'​ スキャンを停止'`）。文字指定のタップは `text: ".*スキャンを停止"` のように正規表現にする。
  - 座標でタップするときは、縮小したスクリーンショットから位置を推定しない（ずれた）。`maestro --device <UDID> hierarchy` の `bounds`（pt 単位）から計算する。
  - スクリーンショットは `xcrun simctl io <UDID> screenshot x.png`。確認用には `sips -Z 700` で縮小する。
- **デバッグ表示の落とし穴**: 一時的なデバッグ表示を `?debug=1` で出し分けていたが、読み取り時に URL のパラメータを置き換えたため `debug=1` が消え、ログが止まった。フラグはページ読み込み時に1回だけ評価する。

---

## 8. その他の細かい点

- **CSV**
  - `Papa.unparse`（papaparse）の出力は最後に改行が付かない。そのまま行を追記すると最終行に連結されてしまうので、`\r\n` を付ける。
  - 出力は BOM 付きの UTF-8。取込は、UTF-8 として読めなければ `TextDecoder('shift_jis')` で読む（Excel の既定の保存形式が Shift\_JIS のため）。
- **在庫などカウンタ値の同時更新**
  - トランザクションで処理する。エミュレータ上で、同じドキュメントに10件並行で更新しても数が合うことをテストで確認した。
  - 同じドキュメントへの並行更新は、競合が起きても約12件/秒だった。
- `<meta name="apple-mobile-web-app-capable">` **は非推奨**。`mobile-web-app-capable` を使う。
- **画像の表示**: ダウンロードトークン付きの URL を Firestore に保存しておくと、ページの `<img>` から認証ヘッダーなしで表示できる。
- **Storage のルール**: アップロードは PNG / JPEG / WebP のみ許可する（SVG は XSS 対策で不可）。シードで入れる SVG は Admin SDK で書き込むので、ルールの対象外。
- `npm audit`: firebase-admin の間接依存（uuid）に中程度の脆弱性が出た。バッファを渡す使い方をしなければ該当しないため、上流の更新待ちとした。
- **フロントのプログラムサイズ**: Firebase + MUI + zxing で約500KB（gzip 後）になった。プロトタイプとしては許容範囲とした。

---

## 9. 次回の進め方（おすすめの順序）

1. 仕様書と本ノートを読む。0章の判断を流用するかどうか、システム名などの固有の値は、ユーザーに確認する。
2. 最初に次の設定を入れておく。
   - `firebase.json` の `emulators.firestore.edition: "enterprise"`（Pipeline を使う場合）、エミュレータの起動コマンドの `--only`、エミュレータ UI の `host`。
   - dev server の同一オリジン中継（4章）と、すべての dev server の `host: true`。
3. MUI・React Router などのメジャー版は、型定義で API を確認してから書く。
4. スキャン画面は、6章の対策（セッションの管理・開始時のリセット・ウォームアップ・`overflow: hidden`）を最初から入れる。
5. 検証
   - 画面: Playwright MCP（HTTP で起動した dev server に対して）。
   - カメラ: SimulatorCameraEx + Maestro（iOS シミュレータの Safari）。
   - 権限: セキュリティルールと API の権限テスト（エミュレータ上）。

