# デプロイノート：Firebase 本番環境の構築とデプロイの注意点

Firebase（Hosting 2サイト / Cloud Functions 第2世代 / Firestore Enterprise / Storage / Authentication）の Web アプリを、  
何もない状態から本番にデプロイしたときに詰まった点・ハマりどころをまとめたものです。  
次回、同じ種類のデプロイを AI が行うときに最初に読むことを想定しています。開発時の注意点は `spec_01_dev_note.md` を参照してください。

- 各項目は「症状 → 原因 → 対処」の順に書いています。
- 文中の `<project-id>`（Firebase プロジェクトID）、`<db-id>`（Firestore データベースID）、`<region>`（リージョン。前回は `asia-northeast1`）、`<account>`（プロジェクト所有者の Google アカウント）、`<billing-account-id>`（請求先アカウントID）、`<web-app-id>`（Firebase ウェブアプリID）は、案件ごとに読み替えてください。
- コマンドはすべて CLI で完結します（コンソール操作は不要でした）。
- App Check（reCAPTCHA Enterprise）は当面使いません。キーの作成・CSP への追加・トークンの検証などは行わないでください。

---

## 0. 全体の手順（この順で進める）


| #   | 作業                                                        | 主なコマンド・API                                                            | 詳細  |
| --- | --------------------------------------------------------- | --------------------------------------------------------------------- | --- |
| 1   | gcloud / firebase のアカウント確認                                | `gcloud auth list`、`firebase login:list`                              | 1章  |
| 2   | プロジェクト作成                                                  | Firebase MCP `firebase_create_project` または `firebase projects:create` | 2章  |
| 3   | 請求先の紐づけ（Blaze）                                            | `gcloud billing projects link`                                        | 2章  |
| 4   | 必要な API の有効化                                              | `gcloud services enable …`                                            | 2章  |
| 5   | Web アプリ登録・Hosting サイト追加                                   | `firebase apps:create web`、`firebase hosting:sites:create`            | 2章  |
| 6   | Firestore Enterprise 作成                                   | `gcloud firestore databases create …`                                 | 3章  |
| 7   | Storage バケット作成                                            | Firebase Storage REST API                                             | 4章  |
| 8   | Authentication 初期化・メール/パスワード有効化                           | Identity Toolkit REST API                                             | 4章  |
| 9   | `.firebaserc`・`.env.production.local`・`functions/.env` 設定 | ファイル編集                                                                | 5章  |
| 10  | デプロイ                                                      | `npm run deploy`（`firebase deploy`）                                   | 5章  |
| 11  | ブラウザで動作確認（コンソールにエラーが出ていないか）                               | Playwright MCP                                                        | 6章  |
| 12  | 初期データ投入                                                   | シードスクリプト                                                              | 7章  |
| 13  | 実機相当のテスト（カメラなど）                                           | SimulatorCameraEx + Maestro                                           | 8章  |
| 14  | 予算アラート                                                    | `gcloud billing budgets create`                                       | 9章  |


**ユーザーに確認が必要な作業**：請求先アカウントの選択（課金が始まる）、既存リソースの削除、本番データを変える操作（シード・業務データの更新・画像アップロードなど）。それ以外は確認なしで進めてよいとユーザーから言われた。

---

## 1. アカウントと認証

- **gcloud の既定アカウントが、プロジェクト所有者と違うことがある。**
  - 症状: `gcloud billing accounts list` が `Reauthentication failed. cannot prompt during non-interactive execution` で失敗。
  - 原因: 既定（ACTIVE）のアカウントが別のアカウントで、その認証が切れていた。所有者のアカウントの認証は有効だった。
  - 対処: `gcloud auth login` をユーザーに頼む前に、**すべてのコマンドに** `--account=<account>` **を付けて試す**。これで通ることが多い。
- **zsh では変数に入れたオプションが分割されない。**
  - 症状: `A="--project=x --account=y"; gcloud … $A` が `The project property must be set to a valid project ID, not the project name [x --account=y]` で失敗。
  - 対処: オプションは変数に入れず、コマンドに直接書く。
- **REST API を直接呼ぶときは、クォータ用のプロジェクトを指定する。**
  - `curl -H "Authorization: Bearer $(gcloud auth print-access-token --account=<account>)" -H "x-goog-user-project: <project-id>" …`
  - `x-goog-user-project` がないと、gcloud に設定された別の（削除済みなどの）プロジェクトが使われて失敗する。
  - `gcloud billing budgets` なども同じで、`--billing-project=<project-id>` を付ける（9章）。
- **ADC（アプリ用の認証、**`application_default_credentials.json`**）は、すでに有効なことがある。**
  - シードの前にユーザーに `gcloud auth application-default login` を頼む前に、次で確認する：  
  `curl -s "https://oauth2.googleapis.com/tokeninfo?access_token=$(gcloud auth application-default print-access-token)"` の `email`。
  - シード実行時は `GOOGLE_CLOUD_QUOTA_PROJECT=<project-id>` を付ける。

---

## 2. プロジェクト作成・課金・API

- **プロジェクトの表示名にアンダースコアは使えない。**
  - 症状: `project display name contains invalid characters`。
  - 対処: 表示名はハイフンにする（例 `my_app` → `my-app`）。プロジェクトID もハイフンと英小文字・数字のみ。
- **Blaze（従量課金）への切り替えは CLI でできる。**
  - `gcloud billing accounts list --account=<account>` で候補を出し、**どれに紐づけるかはユーザーに選んでもらう**（課金が始まるため）。
  - `gcloud billing projects link <project-id> --billing-account=<billing-account-id> --account=<account>`
  - 「コンソールで手動で切り替えてください」と頼まない（前回ユーザーから「コマンドでできるはず」と指摘された）。
- **有効化する API（まとめて1回で）**
  ```
  firestore.googleapis.com firebasestorage.googleapis.com storage.googleapis.com
  cloudfunctions.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com
  run.googleapis.com eventarc.googleapis.com
  identitytoolkit.googleapis.com billingbudgets.googleapis.com
  ```
- **Hosting の既定サイトは、プロジェクト作成時に** `<project-id>` **で自動作成される。** 2つ目（管理画面用など）は `firebase hosting:sites:create <project-id>-admin` で作る（`-admin` は例）。
- `.firebaserc` **は、**`projects.default` **と** `targets` **のキー（プロジェクトID）の両方を書き換える。** 開発用のダミーID（`demo-…`）が残っていると、ターゲットが解決されない。
- `.firebaserc` **の既定を本番にすると、エミュレータも本番のプロジェクトIDで起動してしまう。** クライアントはエミュレータ用に `demo-…` を使うため、食い違って動かなくなる。エミュレータの起動スクリプトに `--project demo-<system>` を明示し、デプロイのスクリプトにも `--project <project-id>` を明示する。
- **Web アプリの設定値**は `firebase apps:sdkconfig WEB <web-app-id>` で取れる。`storageBucket` は新しい形式の `<project-id>.firebasestorage.app`（旧 `appspot.com` ではない）。

---

## 3. Firestore Enterprise（最もハマった）

- **Enterprise エディションは** `(default)` **データベースを作れない。**
  - 症状: `FAILED_PRECONDITION: Firestore Enterprise requires a named database not a (default)`。
  - 対処: 名前付きデータベースにする。コード側の対応が必要：
    - 共通の定数（例 `FIRESTORE_DATABASE_ID`）を用意する。
    - クライアント: `initializeFirestore(app, settings, <db-id>)`。
    - Admin SDK（Functions・スクリプト）: `getFirestore(<db-id>)`。
    - `firebase.json` の `firestore.database` を `<db-id>` にする（ルール・インデックスのデプロイ先）。
    - **エミュレータは** `(default)` **のままにする**（`@firebase/rules-unit-testing` の `ctx.firestore()` が `(default)` 前提のため）。本番かどうかで切り替える（クライアント: エミュレータ使用フラグ、Functions: `FUNCTIONS_EMULATOR`、スクリプト: `--prod`）。
- **データベースIDは 4〜63 文字。** `app` のような3文字は `database_id should be 4-63 characters` で失敗した。
- **作成時に必ず次の2つのフラグを付ける。付けないと MongoDB 互換モードで作られる。**
  ```
  gcloud firestore databases create --database=<db-id> --location=<region> \
    --edition=enterprise --enable-firestore-data-access --enable-realtime-updates \
    --project=<project-id> --account=<account>
  ```
  - 症状: シードで `9 FAILED_PRECONDITION: Access to this database via the Firestore in Native mode API is disabled`。
  - 原因: gcloud の既定では `firestoreDataAccessMode: DISABLED` / `mongodbCompatibleDataAccessMode: ENABLED` / `realtimeUpdatesMode: DISABLED` になる。**この設定は作成後に変えられない。**
  - `--enable-realtime-updates` がないと `onSnapshot`（リアルタイム購読）が使えない。
  - 作成後に `gcloud firestore databases describe` で3つのモードを確認する。
- **作り直すと、同じIDは約5分使えない。**
  - 症状: `Database ID '<db-id>' is not available in project … Please retry in 297 seconds`。
  - 対処: `run_in_background` で待ってから作り直す（前景の長い `sleep` はブロックされる）。
  - 作り直したら、ルールとインデックスを `firebase deploy --only firestore` で再デプロイする。
- データベースの削除はユーザーの確認が必要な操作。前回は「自分が作った空のデータベース」だったので、理由を説明して作り直した。

---

## 4. Storage・Authentication の初期化（CLI で）

- **Storage の既定バケット**（コンソールの「始める」に相当）
  ```
  curl -X POST -H "Authorization: Bearer $T" -H "x-goog-user-project: <project-id>" -H "Content-Type: application/json" \
    "https://firebasestorage.googleapis.com/v1alpha/projects/<project-id>/defaultBucket" -d '{"location":"<region>"}'
  ```
- **Authentication の初期化とメール/パスワードの有効化**
  ```
  curl -X POST … "https://identitytoolkit.googleapis.com/v2/projects/<project-id>/identityPlatform:initializeAuth"
  curl -X PATCH … "https://identitytoolkit.googleapis.com/admin/v2/projects/<project-id>/config?updateMask=signIn.email.enabled,signIn.email.passwordRequired" \
    -d '{"signIn":{"email":{"enabled":true,"passwordRequired":true}}}'
  ```

---

## 5. 環境変数とデプロイ

- `.env.production.local`（`apps/*/`）: `VITE_FIREBASE_*`、そのほかアプリ固有の値（例: 管理画面から案内するクライアントの URL）。git 管理外（`.gitignore` の `.env*.local`）。
- `functions/.env`: Functions の環境変数。秘密情報でなければ git 管理でよい。
- `functions/.env` **を消しても、デプロイ済みの環境変数は消えない。**
  - 例として、動作モードを自前の環境変数 `FEATURE_MODE`（`test` / `live`）で切り替えていた場合。
  - 症状: `FEATURE_MODE=test` を外すために `.env` を削除して再デプロイしたが、動作が変わらなかった。
  - 確認: `gcloud run services describe <function> --region=<region> --format='yaml(spec.template.spec.containers[0].env)'`
  - 対処: 値を明示的に書き換える（`FEATURE_MODE=live`）。
- **初回デプロイは** `npm run deploy -- --force` で、Artifact Registry のクリーンアップポリシーの確認を自動で通す。Functions の初回作成は数分かかる。
  - デプロイのログに `No cleanup policy detected` の警告が出ても、`--force` で「1日より古いイメージを削除」のポリシーは作られていた。変更する前に `firebase functions:artifacts:setpolicy` の表示（`update an existing policy`）で確認できる。
- **ルールがどのデータベースに入ったか**は、Rules API の releases で確認できる（`cloud.firestore/<db-id>` と出れば名前付き DB に適用されている）。
  - `curl -H "Authorization: Bearer $T" -H "x-goog-user-project: <project-id>" https://firebaserules.googleapis.com/v1/projects/<project-id>/releases`
- Functions 第2世代の実体は Cloud Run。ログは `resource.type="cloud_run_revision"`、`resource.labels.service_name="<function>"` で絞る。

---

## 6. デプロイ後の確認

- **必ずブラウザで開いて、コンソールとネットワークを見る**（Playwright MCP）。`curl` の 200 だけでは、CSP などブラウザ側の問題は見つからない。
  - `browser_console_messages`（level=warning）が0件であること。
  - `browser_network_requests` で、失敗しているリクエスト（4xx / 5xx）がないこと。
- API: `/api/health` が `{"ok":true}`、未ログインの `/api/<resource>` が 401。
- ログイン後の画面（一覧・検索・画像表示）まで確認する。**最初の表示は関数のコールドスタートで数秒かかる**（待ってから判断する）。
- `.playwright-mcp/` はスクリーンショットの置き場。`.gitignore` に入れておく。

---

## 7. 初期データの投入（シード）

- **ルートの** `package.json` **からワークスペースのスクリプトに引数を渡すと、途中で消える。**
  - 症状: `npm run seed -- --prod --project <project-id>` が「接続先: エミュレータ」になった（`--prod` が消えた）。
  - 原因: ルートのスクリプトが `npm run seed -w scripts` で、後ろに付いた `--prod` を内側の npm がオプションとして食べた。
  - 対処: ルートのスクリプトを `npm run seed -w scripts --` にする（末尾の `--`）。
- 本番操作の確認プロンプトは `echo yes |` で渡す。
- シードの管理者アカウントは推測しやすいパスワード（開発用）。**公開サイトに入れる場合は、実際の管理者を作ったあと無効化するよう、ユーザーに伝える。**

---

## 8. 実機相当のテスト（SimulatorCameraEx + Maestro）

手順の詳細は `spec_01_dev_note.md` の7章。デプロイ後の追加の注意点：

- **ユーザーが見ている端末でテストする。** 前回、画面に表示されていない別のシミュレータでテストし、ユーザーから「ヘッドレスでやりましたか？」と指摘された。
  - Orca などの開発環境の右パネルに iOS シミュレータが表示されていることがある。`screencapture -x` でデスクトップを撮って、**表示中の端末名を確認してから UDID を選ぶ**。
  - 起動中の端末が複数あるときは、どれを使うかを確かめる。
  - Xcode 27 には `Simulator.app` がない（`Xcode.app/Contents/Applications` にあるのは `DeviceHub.app` など）。表示が必要なら、ユーザーの表示環境を使う。
- **テスト中にシミュレータが止まっていたら、勝手に起動しない。** ユーザーが止めた可能性がある。
- **「最初からバーコードが映った状態で開始」のテストでは、停止ボタンの表示を待たない。** 読み取りが一瞬で終わり、ボタンが消えるため、待つ手順が失敗する（アプリの不具合ではない）。開始ボタンをタップしたら、すぐに結果の表示を待つ。
- ユーザーが目で追えるように、各結果の表示後に数秒止める。
- 本番でテストする場合、データを変える操作（数量の増減・登録・削除など）はしない。

---

## 9. 予算アラート

```
gcloud billing budgets create --billing-account=<billing-account-id> --display-name="<name>" \
  --budget-amount=<金額>JPY --filter-projects=projects/<project-id> --calendar-period=month \
  --threshold-rule=percent=0.5 --threshold-rule=percent=0.9 --threshold-rule=percent=1.0 \
  --threshold-rule=percent=1.0,basis=forecasted-spend \
  --account=<account> --billing-project=<project-id>
```

- 通貨は請求先アカウントに合わせる（`gcloud billing accounts describe` の `currencyCode`）。
- `--billing-project` がないと、gcloud に設定された別のプロジェクトで `USER_PROJECT_DENIED` になる。
- 成功しても何も出力されないことがある。`gcloud billing budgets list` で確認する。
- 通知先は、既定では請求先アカウントの管理者のメール。**アラートは通知だけで、費用は止まらない**ことをユーザーに伝える。

---

## 10. 報告のしかた

- 作成したリソース（プロジェクトID、URL、データベースID、キー、予算）は、報告に具体的に書く。
- 途中で方針を変えた点（データベース名の変更、作り直しなど）は、理由とあわせて報告する。
- 確認していないこと（データを変える操作、反映待ちの項目）は、「確認していない」と明記する。
- 手順書（`README.md`）に、見つかった落とし穴を反映する。

