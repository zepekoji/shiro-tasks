# shiro

開いたらメモできる、モバイルファーストのタスク管理Webアプリ。

## 現在の状態

- 端末保存版：実装済み。メモ・下書き保存、左右フリック、状態ボタン、編集、削除、検索、絞り込み、並び替え、JSON書き出し。
- 白ベースの半透明ガラスUI、起動時の全画面スケルトン、カード追加・フリック・ダイアログのアニメーション。OSの「視差効果を減らす」に対応。
- Google認証、Cloud Run API、Firestore、Workers AI：接続用コードあり。外部アカウント・プロジェクト・シークレットの設定待ち。実サービスとの結合テストは未実施。
- Sitesはフロントエンドの確認用配信先。Cloud Runを配備した状態ではない。
- GitHubリポジトリを作成済み：https://github.com/zepekoji/shiro-tasks 。Cloud Runへの配備と結合テストは未実施。

## 選定理由

| 技術 | 用途 | 選定理由・制約 |
|---|---|---|
| HTML / JavaScript ES modules | フロントエンド | 初期のJS量と依存を抑え、入力欄をすぐ表示できる。現在の画面規模ではReact等のランタイムを必須にしない。複雑な共同編集に拡張する段階ではTypeScript化を推奨。 |
| CSS backdrop-filter / Pointer Events | Liquid Glass、フリック | 標準ブラウザ機能で実装。左右の操作に加え、タッチ・マウス・キーボードで使える状態ボタンも提供。Apple固有のLiquid Glass APIではなくWebでの表現。 |
| localStorage | メモと下書きの即保存 | ネットワーク待ちを入力経路から除く。端末・ブラウザ単位の保存で、ブラウザデータ削除で消える。大量データ向けにはIndexedDBへ移行する。 |
| Node.js 22 / Express 5 | Cloud RunのAPI | 標準HTTPで扱え、コンテナを移植しやすい。フロントと同じJavaScriptで保守可能。 |
| Cloud Run | バックエンド | 指定に従い採用。APIをコンテナとして配備。最小インスタンス0ならコールドスタートの待ちがあり得るため、即メモは端末側で処理する。 |
| Firebase Authentication | Googleログイン | Googleのみを有効化。サーバーでIDトークンとプロバイダーを検証し、メールアドレスではなくuidでデータを分離。 |
| Firestore | 永続保存 | Cloud Runと同じGoogle Cloudで扱える。端末横断のタスク保存用。ブラウザの直接アクセスは禁止し、APIのみを経由。 |
| Cloudflare Workers AI REST API | メモのタスク分解 | 指定サービスをCloud RunからHTTPSで呼べる。Cloudflare Worker自体をAPIホストにする必要はない。モデル名は環境変数で差替え可能。 |
| Secret Manager | AIトークン | ブラウザ・GitHubにトークンを出さず、Cloud Runの実行環境に注入する。 |
| Docker | API配備 | 開発とCloud Runの起動方法を揃える。 |
| Sites | 確認用フロント配信 | UIの試用URLを先に提供できる。本番バックエンド指定を置き換えるものではない。 |

## 操作

1. 入力欄にメモし、右下の矢印で追加。PCはCmd/Ctrl+Enterも利用可能。
2. 右フリックで未着手→進行中→完了。左フリックで逆順。端では循環しない。
3. 丸い状態ボタンでも次の状態に移動する。
4. 本文または「…」から編集・削除。
5. 上のタブで状態を絞り込み、検索ボタンで文字を検索。
6. AIで整理は、Googleログインと外部設定後に利用可能。入力中のメモだけを送信し、最大8件のタスクを作る。ユーザー当たり1日30回まで。

## 本番への接続手順

### 1. Google Cloud / Firebase

課金が有効な既存プロジェクトを選ぶ。Firebaseに同じプロジェクトを追加し、Webアプリを登録する。AuthenticationのログインプロバイダーはGoogleのみ有効化する。公開フロントのドメインを認証の承認済みドメインに追加する。

FirestoreをNative modeで作成する。リージョンは要件に合わせて選び、Cloud Runも近い場所に配置する。`firestore.rules`はクライアントアクセスを全拒否する。Admin SDKはIAMでアクセスするため、このルールとは別にサービスアカウントの最小権限を設定する。

専用サービスアカウントにFirestoreへの必要なアクセス（通常は `roles/datastore.user`）を与える。鍵JSONを作成してリポジトリへ保存しない。Cloud RunのApplication Default Credentialsを使う。

### 2. Workers AI

CloudflareでAccount IDとWorkers AIの実行権限を持つAPIトークンを用意する。トークン値はSecret Managerに `shiro-workers-ai-token` という名前で登録し、Cloud Runのサービスアカウントへ当該シークレットのみのSecret Accessorを付与する。

初期モデルは `@cf/meta/llama-3.1-8b-instruct`。日本語での分解品質は接続後に実データで評価し、必要ならモデルを変更する。AIは出力をJSONとして検証するが、意図しないタスクが含まれる可能性は残る。

### 3. Cloud Run

以下はGoogle Cloud CLIで実行する例。プレースホルダーを置換する。API自体はネットワーク到達可能にし、各APIリクエストをFirebaseトークンで保護する。CORSは認可の代わりにはしない。

```bash
gcloud run deploy shiro-api \
  --source ./backend \
  --project YOUR_PROJECT_ID \
  --region asia-northeast1 \
  --service-account shiro-api@YOUR_PROJECT_ID.iam.gserviceaccount.com \
  --allow-unauthenticated \
  --min-instances 0 \
  --max-instances 3 \
  --set-env-vars 'ALLOWED_ORIGINS=https://YOUR_FRONTEND_DOMAIN,CLOUDFLARE_ACCOUNT_ID=YOUR_ACCOUNT_ID' \
  --set-secrets CLOUDFLARE_API_TOKEN=shiro-workers-ai-token:latest
```

ビルドに必要なAPI・IAM権限はプロジェクト構成に応じて別途有効化する。予算アラートも設定する。上記は本セッションで実行していない。

### 4. フロント設定

`dist/config.js` を次のように設定して再配信する。Firebase Web設定は公開用のプロジェクト識別子であり、Admin秘密鍵・Cloudflareトークンとは別物。

```js
window.SHIRO_CONFIG = {
  apiBase: 'https://YOUR_API_SERVICE.run.app',
  firebase: {
    apiKey: 'YOUR_FIREBASE_WEB_API_KEY',
    authDomain: 'YOUR_PROJECT.firebaseapp.com',
    projectId: 'YOUR_PROJECT_ID',
    appId: 'YOUR_FIREBASE_APP_ID'
  }
};
```

### 5. GitHub

GitHubの認証が済んだアカウントで空のリポジトリ `shiro-tasks` を作成する。個人用アプリの初期値はprivateを推奨。公開ソースにするなら含まれる設定を確認してpublicにする。サイトをインターネットへ公開することと、ソースを公開することは別。

Sitesの既存リモートを保ったままGitHubを追加する例：

```bash
git remote add github https://github.com/YOUR_ACCOUNT/shiro-tasks.git
git push github main
```

トークン・秘密鍵・`.env`はコミットしない。Cloud Runへの継続配備は、初回手動配備を検証した後、Workload Identity Federationなどの短期認証で接続する。

## 実施した検証

2026-10-21：ブラウザで新規メモ追加、状態ボタン、右フリックで完了、左フリックで進行中へ戻す、編集、検索、再読込での状態保持を確認。JavaScript構文チェックを実施。

未検証：実機iPhone/Androidのタッチ感、Google OAuthの実ログイン、Firestoreの実読み書き・権限分離、複数端末競合、Cloud Runの配備、Workers AI実推論。すべて合格とは扱わない。

## 同期仕様と残課題

ログイン前の端末メモとログイン後のアカウントメモは別保存。ログイン前メモはクラウドへ自動送信しない。必要ならログイン前にJSONを書き出す。クラウド同期は更新時刻の新しい内容を採用する。失敗時は端末に保持し、オンライン復帰時に再送する。共同編集・サーバー時刻による競合制御・大量データのページングは今後の改善対象。

APIにはGoogle以外の認証拒否、uid毎の保存先分離、入力検証、AIの日次回数制限を実装。本番に出す前に上記の未検証項目を実アカウントで通す。

## 参照した公式ドキュメント

- https://docs.cloud.google.com/run/docs/authenticating/end-users
- https://firebase.google.com/docs/auth/admin/verify-id-tokens
- https://developers.cloudflare.com/workers-ai/get-started/rest-api/

## ディレクトリ

`dist/`: 配信するフロント。`backend/`: Cloud Run向けAPI。`firestore.rules`: Firestoreのブラウザアクセス制限。`.openai/hosting.json`: Sitesの配信設定。
