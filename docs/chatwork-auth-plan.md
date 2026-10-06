# Chatwork認証の実装案

調査日: 2026-10-06

この文書はChatwork公式資料に基づく**実装提案**。要件定義で確定した「社内IT担当者がChatwork連携の認証設定と資格情報を管理する」という運用を前提とする。OAuth方式の採用と投稿用Chatworkアカウントは、まだ最終決定していない。

## 推奨方式

**Chatwork OAuth 2.0の認可コード方式を、サーバー側のコンフィデンシャルクライアントで使う。** 現場のスマートフォンはGoogleアカウントで本アプリにログインし、Chatworkの認証情報は持たない。ChatworkへのAPIリクエストはアプリのサーバーから行う。

Chatwork公式は認可コード方式をサポートし、コンフィデンシャルクライアントを資格情報を秘匿できるサーバー向けとしている。従来のAPIトークンは有効期限がなくフルアクセス可能と案内されているため、初期案には採用しない。[OAuth 2.0公式資料](https://developer.chatwork.com/docs/oauth) / [APIトークン公式資料](https://developer.chatwork.com/ja/docs/endpoints)

## 実装リスト

1. **Chatwork APIの利用を準備する。** 社内IT担当者が、投稿に使うChatworkアカウントを決め、そのアカウントの対象ルーム参加・投稿権限を確認する。パーソナルプラン以外ではChatwork API利用を組織管理者へ申請する。[利用開始方法](https://developer.chatwork.com/docs/getting-started)
2. **OAuthクライアントを登録する。** Chatworkにコンフィデンシャルクライアントを登録し、サーバーのHTTPSコールバックURLを設定する。クライアントIDとシークレットはサーバーの秘密情報管理に置き、リポジトリやスマートフォンへ含めない。[OAuthクライアント登録](https://developer.chatwork.com/docs/oauth)
3. **必要なスコープだけ要求する。** ルーム名の確認に`rooms.info:read`、本文投稿に`rooms.messages:write`、PDF・`.md`添付に`rooms.files:write`を使う。接続中のChatworkアカウント表示には`users.profile.me:read`を加える。IT担当者が不在でも現場から報告できるようにする案では`offline_access`も使う。これらは公式スコープ表から選んだ実装案。[OAuthスコープ一覧](https://developer.chatwork.com/docs/oauth)
4. **IT担当者だけが接続を開始できるようにする。** 本アプリのGoogleログイン後、登録された社内IT担当者ロールだけに「Chatworkを接続・再接続・解除」を表示する。接続時はChatworkの同意画面へ遷移し、返却された認可コードと`state`をコールバックで検証する。コンフィデンシャルクライアントでもPKCEの`S256`を併用する案とする。[OAuth認可フロー](https://developer.chatwork.com/docs/oauth)
5. **トークンをサーバーで取得・保管・更新する。** コールバックで認可コードをアクセストークンとリフレッシュトークンに交換する。保存時は暗号化し、IT担当者のみが接続状態を管理できるようにする。アクセストークンは公式資料では30分、通常のリフレッシュトークンは14日、`offline_access`を含む場合は認可失効まで有効とされる。更新・失効時は再接続を案内する。[トークン発行・更新](https://developer.chatwork.com/docs/oauth)
6. **サーバーからBearer認証でAPIを呼ぶ。** OAuthでは`Authorization: Bearer <access_token>`を使う。従来のAPIトークン方式の`x-chatworktoken`とは使い分ける。トークン値を画面、PDF、ログ、URL、GitHubに出さない。[OAuth APIアクセス](https://developer.chatwork.com/docs/oauth) / [従来のAPIトークン方式](https://developer.chatwork.com/ja/docs/endpoints)
7. **設定画面で送信先を検証する。** IT担当者がridを設定したときに`GET /rooms/{room_id}`でルーム名を取得し、設定画面にrid・ルーム名・接続中のChatworkアカウントを表示する。権限不足や未参加なら設定を有効化しない。[ルーム情報取得](https://developer.chatwork.com/reference/get-rooms-room_id)
8. **送信形式に応じて投稿する。** 本文記載のMarkdownは`POST /rooms/{room_id}/messages`、PDFまたは`.md`添付は`POST /rooms/{room_id}/files`を使う。ファイルアップロード上限は公式資料で5MB。送信前に宛先、本文、添付ファイル、報告IDを確認する。[メッセージ投稿](https://developer.chatwork.com/reference/post-rooms-room_id-messages) / [ファイルアップロード](https://developer.chatwork.com/reference/post-rooms-room_id-files)
9. **失敗と二重送信を扱う。** 401では接続状態を確認して更新・再接続へ誘導し、403では対象ルームへの参加・権限・スコープを確認する。429ではAPIのリセット情報を使って送信待ちにする。タイムアウトなどで成否不明なら自動再送せず、報告IDと送信状態を残す。[メッセージ投稿のエラー](https://developer.chatwork.com/reference/post-rooms-room_id-messages) / [利用回数制限](https://developer.chatwork.com/ja/docs/endpoints)
10. **管理記録と受け入れ確認を用意する。** 接続・再接続・解除の操作をしたIT担当者、接続したChatworkアカウント、rid、送信結果を記録する。秘密情報そのものは記録しない。接続、ルーム名表示、本文投稿、PDF添付、`.md`添付、期限切れ後の更新、権限不足、オフライン後の手動送信を確認する。

## 実装前に決めること

- **投稿用アカウント**: IT担当者自身のChatworkアカウントを使うか、運用用に指定した別アカウントを使うか。どちらも対象ルームへの参加・投稿権限が必要。
- **長期接続**: `offline_access`で長期接続するか、使わずにリフレッシュトークン失効時にIT担当者が再認可する運用にするか。無期限のリフレッシュトークンを使う場合は、その保管・失効時の対応を厳格にする。
- **接続先の範囲**: ridを1つに固定するか、複数ルームをIT担当者が登録できるようにするか。

## 公式資料

- [OAuth 2.0について](https://developer.chatwork.com/docs/oauth)
- [Chatwork APIへようこそ](https://developer.chatwork.com/docs/getting-started)
- [エンドポイントについて](https://developer.chatwork.com/ja/docs/endpoints)
- [ルーム情報取得](https://developer.chatwork.com/reference/get-rooms-room_id)
- [メッセージ投稿](https://developer.chatwork.com/reference/post-rooms-room_id-messages)
- [ファイルアップロード](https://developer.chatwork.com/reference/post-rooms-room_id-files)
