# KIIYA見落としチェッカー サーバー環境・秘密情報・権限設計

## 1. 英語の名称が指すもの

| 名称 | 日本語で指すもの |
| --- | --- |
| `Cloud Run` | アプリのAPIと非同期処理を動かすGoogle Cloudのサーバー実行環境。 |
| `Cloud Tasks` | マスタ変更通知の5分後の再通知を予約するGoogle Cloudのタスクキュー。 |
| `Secret Manager` | Chatwork連携の秘密値を保管するGoogle Cloudのサービス。 |
| `service account` | サーバー処理に与えるGoogle Cloud上の機械用の身分。アプリ利用者のGoogleアカウントとは別。 |
| `rid` | ChatworkルームID。検証用と本番用を分ける。 |
| `Next.js` | スマートフォンから開くWeb画面を作るフレームワーク。 |
| `React Native for Web` | React Native形式の画面部品をブラウザーで表示する仕組み。 |
| `PWA` | ブラウザーからホーム画面へ追加できるWebアプリ。 |
| `Vercel` | 今回のフロントエンドを公開する場所。APIサーバーとは分ける。 |

## 2. 選定した構成〔決定〕

- サーバー環境は**Google Cloud Run**、秘密情報管理サービスは**Google Cloud Secret Manager**を採用する。社内にAWS環境はない。
- 検証環境と本番環境を**別のGoogle Cloudプロジェクト**に置く。プロジェクトID、請求先、運用担当者は環境構築時に登録する。検証用ridはローカル設定にあり、本番の投稿用社員とridは本番手動テスト時に設定する。各環境の有効な送信先は1ルームとする。
- API用のCloud Runと、マスタ通知の再通知を実行する非公開のCloud Runワーカーを分ける。両方を`asia-northeast1`（東京）に配置する。5分後の再通知は同リージョンのCloud Tasksへ予約する。通知の最大2回という業務上限はアプリの状態・試行回数で判定し、タスク再配信時も同じ試行を重ねないよう照合する。ただしChatworkへの投稿結果不明時には重複投稿の可能性が残る。[Cloud Runのリージョン](https://docs.cloud.google.com/run/docs/locations) / [Cloud Tasksのリージョン](https://docs.cloud.google.com/tasks/docs/locations) / [予約実行](https://docs.cloud.google.com/tasks/docs/create-tasks) / [重複実行の注意](https://docs.cloud.google.com/tasks/docs/common-pitfalls)
- 初期版では**グローバルのSecret Manager**を使う。Cloud Run標準のシークレット連携は地域指定のシークレットをサポートしていないため。秘密値の日本国内保存が要件になった場合は、東京の地域指定Secret ManagerをAPIから直接使う設計へ変更する。[Cloud Runのシークレット制約](https://docs.cloud.google.com/run/docs/configuring/services/secrets) / [Secret Managerの地域](https://docs.cloud.google.com/secret-manager/docs/locations)
- 作業記録・マスタ・通知履歴を保存するデータベース製品は、この選定の対象外とし、詳細な技術設計で決める。

### 担当者確認用フロントエンド〔2026-10-08決定〕

- **Next.js + React Native + React Native for Web + TypeScript**で、スマートフォン向けの**PWA**を作る。React Native形式の画面部品をReact Native for Webでブラウザーへ表示する。デプロイ先は**Vercel**。ネイティブアプリの配布は今回の試作に含めない。[React Native for Webの説明](https://necolas.github.io/react-native-web/docs/installation/) / [Next.jsのPWAガイド](https://nextjs.org/docs/app/guides/progressive-web-apps) / [VercelのNext.js対応](https://vercel.com/docs/frameworks/full-stack/nextjs)
- Webの画面遷移には**Next.js Pages Router**を使う。`react-native`を`react-native-web`へ読み替える設定を置き、開発・ビルドは`next dev --webpack`／`next build --webpack`でそろえる。ExpoのNext.jsアダプターにはApp Router非対応の制約があるため、この試作では採用しない。[React Native for Webの設定](https://necolas.github.io/react-native-web/docs/setup/) / [ExpoのNext.js連携資料](https://docs.expo.dev/guides/using-nextjs/) / [Next.jsの開発・ビルドコマンド](https://nextjs.org/docs/app/getting-started/installation)
- フロントエンドのコードはこのリポジトリの`web/`配下に置く。Node.jsは**24.x**、パッケージ管理は**npm**とし、`package-lock.json`をコミットする。Vercel側もNode.js 24.xと`web/`をRoot Directoryに設定する。[VercelのNode.js対応版](https://vercel.com/docs/functions/runtimes/node-js/node-js-versions) / [Root Directory設定](https://vercel.com/docs/monorepos)
- PWAとしてアプリ名、アイコン、Webアプリマニフェストを用意し、VercelのHTTPS上でホーム画面への追加を確認する。**オフライン入力・同期とサービスワーカーによるデータ保存は今回のUI試作の対象外**とし、実装済みと誤認させない。[Next.jsのPWAガイド](https://nextjs.org/docs/app/guides/progressive-web-apps)
- GitHubリポジトリからVercelプロジェクトを作成し、作業ブランチの**Preview**で担当者に画面を見せる。確認後に`main`を**Production**へ反映する。VercelはGit連携でPreviewとProductionを分けられる。[VercelのGit連携](https://vercel.com/docs/git)
- 試作は架空データだけを使い、VercelにはChatworkの資格情報や実際のridを置かない。既に決めた**Cloud Run・Cloud Tasks・Secret Managerは後続のサーバー実装で維持**し、Vercel上の画面から秘密情報を直接扱わない。

## 3. 秘密情報の置き場所〔決定〕

| 情報 | 保存先・扱い |
| --- | --- |
| Chatwork OAuthクライアントシークレット | 環境ごとのSecret Managerシークレット。アプリ画面や通常ログに表示しない。 |
| `offline_access`付きリフレッシュトークン | 環境ごとの別のSecret Managerシークレット。トークン再発行時に新しい値が返れば、新しい版として保存する。 |
| 短寿命のアクセストークン | サーバー処理内で使用する。スマートフォン・通常ログ・GitHubには渡さない。 |
| OAuthクライアントID、連携アカウントID、rid、ルーム名 | アプリ設定データ。秘密値そのものは含めない。 |

サービスアカウントの鍵ファイルを発行・配置せず、Cloud Runに割り当てたサービスIDでGoogle Cloud APIを呼ぶ。秘密値をリポジトリ、端末、報告書、Chatwork通知本文に保存しない。[Cloud RunのサービスID](https://docs.cloud.google.com/run/docs/securing/service-identity) / [Chatwork OAuth](https://developer.chatwork.com/docs/oauth)

## 4. 権限設定〔決定〕

| 主体 | 与える権限・操作 |
| --- | --- |
| 一般職員 | Googleアカウントでアプリにログインし、許可された清掃作業を行う。Chatwork連携設定と秘密値にはアクセスできない。 |
| アプリ内の社内IT担当者 | アプリ上でChatwork接続・再接続・解除と、唯一の有効なridの設定・確認を行う。秘密値そのものは閲覧できない。Google Cloudの管理者権限は自動付与しない。 |
| API用Cloud Runの専用サービスアカウント | 2つのChatworkシークレットへの`Secret Accessor`、リフレッシュトークンのシークレットにだけ`Secret Version Adder`、指定したCloud Tasksキューへの`Cloud Tasks Enqueuer`を与える。タスク呼び出し用サービスアカウントを指定するために必要な`Service Account User`も、そのアカウントに限定する。 |
| 非公開ワーカー用Cloud Runの専用サービスアカウント | API用と同じ2つのChatworkシークレットへの読み取り、リフレッシュトークン版の追加、指定キューへのタスク登録、タスク呼び出し用サービスアカウントを指定するための権限だけを与える。 |
| Cloud Tasksの呼び出し用サービスアカウント | 非公開ワーカーのCloud Runサービスに対する`Cloud Run Invoker`だけを与える。タスクはOIDCトークンを付けてワーカーを呼ぶ。 |
| Google Cloud環境の管理担当者 | プロジェクト、Cloud Run、キュー、シークレットの権限と配備を管理する。アプリ内IT担当者とは別の権限として登録し、日常のアプリ操作には広いGoogle Cloud IAM権限を使わない。 |

Secret Managerの権限はプロジェクト全体ではなく対象シークレット単位で付ける。APIとワーカーは別のサービスアカウントとし、既定の広い権限を持つサービスアカウントを使わない。ユーザーから公開APIへの呼び出しではGoogleログインをサーバー側で検証し、IT担当者ロールの判定も画面表示だけでなくサーバー側で行う。[Secret ManagerのIAM](https://docs.cloud.google.com/secret-manager/docs/access-control) / [Cloud RunのサービスID](https://docs.cloud.google.com/run/docs/configuring/services/service-identity) / [Cloud TasksからCloud Runへの認証](https://docs.cloud.google.com/run/docs/triggering/using-tasks)

## 5. 環境構築時に登録する値

- 検証用・本番用のGoogle CloudプロジェクトIDと請求先。
- 投稿に使う既存社員のChatworkアカウント、各環境のOAuthクライアントとコールバックURL。
- 本番手動テストで確認した本番ridとルーム名。検証用ridの値は公開リポジトリに含めない。
- Google Cloud環境の管理担当者と、アプリ内の社内IT担当者。

関連文書: [要件定義](requirements.md)、[論理データ設計](data-design.md)、[Chatwork認証の実装案](chatwork-auth-plan.md)
