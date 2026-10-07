# Chatwork認証の実装案

調査日: 2026-10-06

この文書はChatwork公式資料に基づく**実装提案**。要件定義で確定した「社内IT担当者がChatwork連携の認証設定と資格情報を管理する」という運用を前提とする。以下の投稿用アカウント、`offline_access`、送信先ルーム数の提案は、まだ確定要件ではない。

## 推奨方式

**Chatwork OAuth 2.0の認可コード方式を、サーバー側のコンフィデンシャルクライアントで使う。** 現場のスマートフォンはGoogleアカウントで本アプリにログインし、Chatworkの認証情報は持たない。ChatworkへのAPIリクエストはアプリのサーバーから行う。

Chatwork公式は認可コード方式をサポートし、コンフィデンシャルクライアントを資格情報を秘匿できるサーバー向けとしている。従来のAPIトークンは有効期限がなくフルアクセス可能と案内されているため、初期案には採用しない。[OAuth 2.0公式資料](https://developer.chatwork.com/docs/oauth) / [APIトークン公式資料](https://developer.chatwork.com/ja/docs/endpoints)

## 3点の提案

| 項目 | 推奨設定 | 公式資料との対応・理由 |
| --- | --- | --- |
| 投稿用Chatworkアカウント | 継続運用と通知のため、API投稿用に**別のChatworkアカウントを1つ**用意する案を推奨する。既存のルーム参加社員1名のアカウントを使う方法も技術的には可能。 | Chatwork公式ヘルプは、本人のアカウントからのAPI投稿は本人には既読扱いとなり、未読通知を受けたい場合はbotのような別アカウントを案内している。どちらを使う場合も対象ルームのメンバーで、投稿権限とAPI利用権限を持つことを確認する。[API投稿の未読通知](https://help.chatwork.com/hc/ja/articles/900001985863-API%E7%B5%8C%E7%94%B1%E3%81%AE%E6%8A%95%E7%A8%BF%E3%81%8C%E6%9C%AA%E8%AA%AD%E3%81%AB%E3%81%AA%E3%82%89%E3%81%AA%E3%81%84-%E6%9C%AA%E8%AA%AD%E9%80%9A%E7%9F%A5%E3%81%8C%E6%9D%A5%E3%81%AA%E3%81%84) / [API利用申請](https://help.chatwork.com/hc/ja/articles/115000169501-API%E3%81%AE%E5%88%A9%E7%94%A8%E7%94%B3%E8%AB%8B%E3%82%92%E6%89%BF%E8%AA%8D-%E5%8D%B4%E4%B8%8B%E3%81%99%E3%82%8B) |
| 長期接続 | サーバー側のOAuthコンフィデンシャルクライアントで`offline_access`を使う。リフレッシュトークンは暗号化してサーバーだけで保管し、IT担当者が接続状態と再接続を管理する。 | 公式資料は`offline_access`を、認可した人が不在でもAPIアクセスする用途向けと説明する。通常のリフレッシュトークンは14日、`offline_access`付きは認可失効まで有効。長寿命になるリスクも明記されている。現場職員がIT担当者不在時にも報告する本アプリには適合する。[OAuth 2.0公式資料](https://developer.chatwork.com/docs/oauth) |
| 送信先ルーム | 初期版では**有効な送信先を1ルーム**にする。IT担当者のみがridを登録・変更でき、設定時にルーム名を取得して確認する。現場職員には送信前にルーム名を表示し、毎回の自由な宛先選択は設けない。 | 公式APIはルーム一覧とルームIDごとの情報取得を提供しており、1ルーム制限は**本アプリの運用設計**。現在のKIIYAの報告先を固定すると誤送信を抑えやすい。複数ルームが必要になれば、IT担当者が承認したルームだけを候補にする。[ルーム一覧](https://developer.chatwork.com/reference/get-rooms) / [ルーム情報](https://developer.chatwork.com/reference/get-rooms-room_id) |

別アカウントを作る場合は別のメールアドレスが必要。組織契約の有料プランでは1ユーザーにつき1ライセンスが必要なため、追加分の契約状況を社内で確認する。[複数アカウント](https://help.chatwork.com/hc/ja/articles/203353800-%E8%A4%87%E6%95%B0%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E3%81%A7%E5%88%A9%E7%94%A8%E3%81%99%E3%82%8B) / [ユーザー数とライセンス数](https://help.chatwork.com/hc/ja/articles/203353810-%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC%E6%95%B0-%E3%83%A9%E3%82%A4%E3%82%BB%E3%83%B3%E3%82%B9%E6%95%B0%E3%81%A8%E3%81%AF)

投稿用アカウントを停止するとOAuthトークンも削除されるため、運用中は有効な状態を保つ。停止・廃止する場合は、先に新しい投稿用アカウントへ接続を切り替える。[ユーザー停止時のAPI・OAuth](https://help.chatwork.com/hc/ja/articles/4404802517529-%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC%E5%81%9C%E6%AD%A2%E6%A9%9F%E8%83%BD%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6)

### 既存のルーム参加社員を投稿用アカウントにする場合

この方法はChatwork公式のOAuth認可・API投稿手順と整合する。別アカウントを用意しない選択であり、上記の別アカウント案を採用しない理由と運用条件を確認する。

1. 本アプリでその社員のGoogleアカウントに「社内IT担当者」権限を登録する。Chatworkルームへの参加だけを根拠に本アプリのIT権限を付けない（アプリ側の権限設計）。
2. その社員本人のChatworkアカウントでOAuth同意を行い、サーバー側で取得したトークンを使う。組織契約なら、そのアカウントについてChatwork API利用承認を得る。対象ルームに参加し、投稿権限を持つことを確認する。[OAuth認可](https://developer.chatwork.com/docs/oauth) / [API利用申請](https://help.chatwork.com/hc/ja/articles/115000169501-API%E3%81%AE%E5%88%A9%E7%94%A8%E7%94%B3%E8%AB%8B%E3%82%92%E6%89%BF%E8%AA%8D-%E5%8D%B4%E4%B8%8B%E3%81%99%E3%82%8B) / [メッセージ投稿](https://developer.chatwork.com/reference/post-rooms-room_id-messages)
3. 投稿はその社員名義になる。その社員自身への未読通知は通常発生しない。メッセージ投稿APIの`self_unread=1`は本人にも未読を付ける選択肢だが、ファイルアップロードAPIには同じパラメーターが記載されていないため、PDF・`.md`添付時の本人通知をこれに依存させない。[未読通知](https://help.chatwork.com/hc/ja/articles/900001985863-API%E7%B5%8C%E7%94%B1%E3%81%AE%E6%8A%95%E7%A8%BF%E3%81%8C%E6%9C%AA%E8%AA%AD%E3%81%AB%E3%81%AA%E3%82%89%E3%81%AA%E3%81%84-%E6%9C%AA%E8%AA%AD%E9%80%9A%E7%9F%A5%E3%81%8C%E6%9D%A5%E3%81%AA%E3%81%84) / [メッセージ投稿](https://developer.chatwork.com/reference/post-rooms-room_id-messages) / [ファイルアップロード](https://developer.chatwork.com/reference/post-rooms-room_id-files)
4. 異動・退職・アカウント停止に備え、後任アカウントへの再接続手順を用意する。Chatwork公式はユーザー停止時にOAuthトークンが削除されると案内している。[ユーザー停止時のAPI・OAuth](https://help.chatwork.com/hc/ja/articles/4404802517529-%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC%E5%81%9C%E6%AD%A2%E6%A9%9F%E8%83%BD%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6)

## 実装リスト

1. **Chatwork APIの利用を準備する。** 投稿用アカウントを、別アカウント案または既存社員アカウント案から決め、対象ルームへの参加・投稿権限を確認する。組織契約では投稿用アカウントからAPI利用を組織管理者へ申請する。[利用開始方法](https://developer.chatwork.com/docs/getting-started) / [API利用申請](https://help.chatwork.com/hc/ja/articles/115000169501-API%E3%81%AE%E5%88%A9%E7%94%A8%E7%94%B3%E8%AB%8B%E3%82%92%E6%89%BF%E8%AA%8D-%E5%8D%B4%E4%B8%8B%E3%81%99%E3%82%8B)
2. **OAuthクライアントを登録する。** Chatworkにコンフィデンシャルクライアントを登録し、サーバーのHTTPSコールバックURLを設定する。クライアントIDとシークレットはサーバーの秘密情報管理に置き、リポジトリやスマートフォンへ含めない。[OAuthクライアント登録](https://developer.chatwork.com/docs/oauth)
3. **必要なスコープだけ要求する。** ルーム名の確認に`rooms.info:read`、本文投稿に`rooms.messages:write`、PDF・`.md`添付に`rooms.files:write`を使う。マスタ変更通知の結果不明時に直近メッセージで変更IDを照合する案には`rooms.messages:read`も必要。接続中のChatworkアカウント表示に`users.profile.me:read`、IT担当者不在時の接続継続に`offline_access`を加える。これらは公式スコープ表から選んだ実装案。[OAuthスコープ一覧](https://developer.chatwork.com/docs/oauth)
4. **IT担当者だけが接続を開始できるようにする。** 本アプリのGoogleログイン後、登録された社内IT担当者ロールだけに「Chatworkを接続・再接続・解除」を表示する。接続時はChatworkの同意画面へ遷移し、返却された認可コードと`state`をコールバックで検証する。コンフィデンシャルクライアントでもPKCEの`S256`を併用する案とする。[OAuth認可フロー](https://developer.chatwork.com/docs/oauth)
5. **トークンをサーバーで取得・保管・更新する。** コールバックで認可コードをアクセストークンとリフレッシュトークンに交換する。保存時は暗号化し、IT担当者のみが接続状態を管理できるようにする。アクセストークンは公式資料では30分、通常のリフレッシュトークンは14日、`offline_access`を含む場合は認可失効まで有効とされる。更新・失効時は再接続を案内する。[トークン発行・更新](https://developer.chatwork.com/docs/oauth)
6. **サーバーからBearer認証でAPIを呼ぶ。** OAuthでは`Authorization: Bearer <access_token>`を使う。従来のAPIトークン方式の`x-chatworktoken`とは使い分ける。トークン値を画面、PDF、ログ、URL、GitHubに出さない。[OAuth APIアクセス](https://developer.chatwork.com/docs/oauth) / [従来のAPIトークン方式](https://developer.chatwork.com/ja/docs/endpoints)
7. **設定画面で送信先を検証する。** IT担当者が唯一の有効なridを設定したときに`GET /rooms/{room_id}`でルーム名を取得し、設定画面にrid・ルーム名・接続中のChatworkアカウントを表示する。権限不足や未参加なら設定を有効化しない。現場職員には送信前にルーム名を表示する。[ルーム情報取得](https://developer.chatwork.com/reference/get-rooms-room_id)
8. **送信形式に応じて投稿する。** 本文記載のMarkdownは`POST /rooms/{room_id}/messages`、PDFまたは`.md`添付は`POST /rooms/{room_id}/files`を使う。ファイルアップロード上限は公式資料で5MB。送信前に宛先、本文、添付ファイル、報告IDを確認する。[メッセージ投稿](https://developer.chatwork.com/reference/post-rooms-room_id-messages) / [ファイルアップロード](https://developer.chatwork.com/reference/post-rooms-room_id-files)
9. **失敗と二重送信を扱う。** 401では接続状態を確認して更新・再接続へ誘導し、403では対象ルームへの参加・権限・スコープを確認する。429ではAPIのリセット情報を使って送信待ちにする。**清掃報告**がタイムアウトなどで成否不明なら自動再送せず、報告IDと送信状態を残す。[メッセージ投稿のエラー](https://developer.chatwork.com/reference/post-rooms-room_id-messages) / [利用回数制限](https://developer.chatwork.com/ja/docs/endpoints)
10. **マスタ変更通知は自動再通知する。** 失敗・結果不明は送信失敗扱いで各試行を履歴に残し、変更者本人へアプリ内アラートを出す。通知本文には一意の変更IDを含める。結果不明なら再通知前に`GET /rooms/{room_id}/messages?force=1`で直近メッセージを照合し、既存投稿が見つかればその投稿日時を成功日とする。見つからなければ、失敗・結果不明の判定から5分後に再通知し、初回送信に加えて最大2回まで試す。計3回とも成功せず投稿済みとも確認できなければ変更不成立として打ち切る。通知と各試行の履歴は最終結果確定から1週間保存する。公式の投稿APIに冪等キーは記載されず、取得APIで確認できる範囲も直近100件までなので、重複投稿を完全には防げない。新版は実際に成功した送信日（日本時間）の翌日から適用する。[メッセージ投稿](https://developer.chatwork.com/reference/post-rooms-room_id-messages) / [メッセージ取得](https://developer.chatwork.com/reference/get-rooms-room_id-messages)
11. **管理記録と受け入れ確認を用意する。** 接続・再接続・解除の操作をしたIT担当者、接続したChatworkアカウント、rid、送信結果を記録する。秘密情報そのものは記録しない。接続、ルーム名表示、本文投稿、PDF添付、`.md`添付、期限切れ後の更新、権限不足、オフライン後の手動送信、マスタ通知の自動再通知と適用日を確認する。

## 提案を採用する際の確認

- 投稿用アカウントを別に作るか、既存のルーム参加社員を使うか。別アカウントの場合はメールアドレスと追加ライセンス、既存社員の場合は本人通知と異動時の再接続を確認する。どちらも対象ルームへの参加・投稿権限が必要。
- `offline_access`の長寿命リフレッシュトークンを保管するサーバー環境と、接続解除・再接続を扱う社内IT担当者。
- 初期版の送信先とする1ルームのrid・ルーム名、将来複数ルームが必要になったときの登録権限。

## 公式資料

- [OAuth 2.0について](https://developer.chatwork.com/docs/oauth)
- [Chatwork APIへようこそ](https://developer.chatwork.com/docs/getting-started)
- [エンドポイントについて](https://developer.chatwork.com/ja/docs/endpoints)
- [ルーム情報取得](https://developer.chatwork.com/reference/get-rooms-room_id)
- [メッセージ投稿](https://developer.chatwork.com/reference/post-rooms-room_id-messages)
- [メッセージ取得](https://developer.chatwork.com/reference/get-rooms-room_id-messages)
- [ファイルアップロード](https://developer.chatwork.com/reference/post-rooms-room_id-files)
- [API投稿の未読通知](https://help.chatwork.com/hc/ja/articles/900001985863-API%E7%B5%8C%E7%94%B1%E3%81%AE%E6%8A%95%E7%A8%BF%E3%81%8C%E6%9C%AA%E8%AA%AD%E3%81%AB%E3%81%AA%E3%82%89%E3%81%AA%E3%81%84-%E6%9C%AA%E8%AA%AD%E9%80%9A%E7%9F%A5%E3%81%8C%E6%9D%A5%E3%81%AA%E3%81%84)
- [API利用申請](https://help.chatwork.com/hc/ja/articles/115000169501-API%E3%81%AE%E5%88%A9%E7%94%A8%E7%94%B3%E8%AB%8B%E3%82%92%E6%89%BF%E8%AA%8D-%E5%8D%B4%E4%B8%8B%E3%81%99%E3%82%8B)
- [チャット一覧](https://developer.chatwork.com/reference/get-rooms)
- [ユーザー停止時のAPI・OAuth](https://help.chatwork.com/hc/ja/articles/4404802517529-%E3%83%A6%E3%83%BC%E3%82%B6%E3%83%BC%E5%81%9C%E6%AD%A2%E6%A9%9F%E8%83%BD%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6)
