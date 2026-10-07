# KIIYA見落としチェッカー API設計（第1稿）

## 1. 英語名の意味

| 名称 | 意味 |
| --- | --- |
| `WorkRecord` | 1回の清掃作業の入力記録。 |
| `MasterRevision` | 客室・設備・清掃項目の固定された版。 |
| `ReportVersion` | 送信確定時の作業内容と送信先を固定した報告の版。 |
| `MasterChange` | 承認取得を申告して登録するマスタ変更候補。 |
| `NotificationAttempt` | マスタ変更通知の1回のChatwork投稿試行。 |
| `Idempotency-Key` | 通信再試行や連打による同一操作の重複を防ぐキー。 |
| `revision` | 作業記録の更新番号。競合更新の検出に使う。 |

機械可読の契約は[OpenAPI定義](../api/openapi.yaml)に置く。この文書とOpenAPIは実装前の設計であり、APIサーバーはまだ存在しない。

## 2. 認証・権限・時刻

- 公開APIはGoogleログインのIDトークンを受け、サーバーで署名、発行元、audience、有効期限を検証する。`/v1/me`でアプリ内権限を返す。
- Chatwork設定とOAuth接続操作は登録済みIT担当者のみ。OAuthクライアントシークレットとリフレッシュトークンはレスポンスへ出さず、Google Cloud Secret Managerで扱う。
- 作業記録への閲覧・編集権限の細部は要確認。少なくとも未認証者を拒否し、APIで所有者または許可された担当者か検査する。
- 報告結果不明の確認は、その作業を作成して「当日担当職員」を入力したログイン利用者、またはIT担当者に許可する。自由入力の氏名だけで本人確認せず、サーバーが記録した作成者IDで照合する。
- OAuthコールバックは`state`とPKCEに加え、接続を開始したIT担当者のログインセッションに結び付けて検証する。
- 業務日、過去日付判定、マスタ適用日は`Asia/Tokyo`。APIの時刻はUTCオフセット付きRFC 3339、作業日は`YYYY-MM-DD`。
- 開始済み作業の`masterRevisionId`は変えない。新しいマスタ版の有効日はChatwork通知が成功した日本時間の日付の翌日。

## 3. 主なAPI操作

| 操作 | 用途・要点 |
| --- | --- |
| `GET /v1/me`、`GET /v1/name-suggestions` | 利用者情報と全員共有の氏名候補を取得。候補は無期限保存。 |
| `GET /v1/master-revisions/current?workDate=...` | 作業日に有効なマスタを取得。作業開始後の自動切替なし。 |
| `POST /v1/work-records` | 作業を開始し、マスタ版を固定する。人数は1〜5人。 |
| `GET/PATCH /v1/work-records/{workId}` | 入力中の作業を取得・保存。`revision`一致で更新。対象から外した箇所のチェックも編集用データには保持。 |
| `GET /v1/work-records/{workId}/history` | 過去記録画面用。対象から外した箇所とチェックは含めない。 |
| `GET /v1/work-records?archived=true` | 設定画面のリンク先で表示する過去記録一覧。日付変更で過去扱い。 |
| `GET /v1/work-records/{workId}/readiness` | 現在対象の必須項目、担当者、人数、休憩回答を検査。「いいえ」も回答済み。 |
| `POST /v1/work-records/{workId}/report-preview` | 結果画面用の一時プレビュー。押下時刻や報告版を作らず、PDF・Markdownの表示内容を取得する。 |
| `POST /v1/work-records/{workId}/report-button-press` | 結果画面下部の「報告」で呼ぶ。初回のみ押下時刻を記録し、確認ダイアログを開く。キャンセルは追加API不要。 |
| `POST /v1/work-records/{workId}/report-versions` | ダイアログの「完了」で呼ぶ。再検証後、固定された新版を作りChatwork投稿を開始。過去日付は明示的な確認値を必要とする。 |
| `GET /v1/work-records/{workId}/report-versions`、`GET /v1/report-versions/{reportId}` | 版と送信結果を取得。送信結果不明時の自動再送はしない。 |
| `POST /v1/report-versions/{reportId}/verification` | 不明結果の確認結果を登録する。当日担当職員またはIT担当者が確認できる。確認者・日時・根拠を残す。 |
| `GET/PUT /v1/settings/chatwork` | IT担当者が接続状態、唯一のルームID・取得したルーム名、Markdown送信方法を確認・更新。 |
| `POST /v1/settings/chatwork/oauth/start`、`GET /v1/settings/chatwork/oauth/callback` | Chatwork OAuth認可コード方式、`offline_access`で接続。stateとPKCEを検証。 |
| `POST /v1/master-changes`、`POST /v1/master-changes/{changeId}/submit` | 変更候補と承認取得の自己申告・相手2名の氏名を記録し、通知を開始。 |
| `GET /v1/master-changes/{changeId}`、`GET /v1/alerts` | 変更状態・通知試行結果と本人宛アラートを表示。 |

## 4. 共通規則

1. `PATCH`は`revision`を必須とし、現在値と違う場合は`409`で最新値を返す。上書きは行わない。基本は1端末だが、通信再試行にも必要。
2. 作成・報告確定・マスタ変更確定は`Idempotency-Key`を必須とする。同一利用者・同一操作・同一キー・同一本文なら同じ結果を返し、本文が異なれば`409`。保持期間は実装時に決める。
3. レスポンスの`202`は処理受付を表し、Chatwork投稿成功を意味しない。クライアントは状態取得で`Sent`を確認して初めて完了表示する。
4. 報告ボタンの初回押下時刻は再押下や再送で更新しない。押下済みで未送信の作業と未送信本文は7日後に削除し、結果不明になった事実だけを残す。未押下下書きと送信済み作業は無期限保存。
5. 対象箇所を外してもチェックは削除しない。報告版・履歴表示には送信確定時または現在の対象箇所だけを含める。対象変更の備考記入は利用者に促し、必須化の是非は要確認。
6. マスタ変更の通知失敗・結果不明は本人だけにアラートを表示する。5分間隔で最大2回再通知し、計3回成功しなければ変更不成立。結果不明の再通知前に変更IDで投稿済み照合を行う。通知試行履歴は最終結果から7日保存。
7. マスタ変更と承認取得の記録は確定時から1年保存する。アラートは確認後30日で削除し、未確認でも作成から1年を上限とする。
   1年の起算点は、通知成功または変更不成立の最終結果が確定した時刻とする設計案。
8. 投稿先は環境ごとに有効な1ルーム。検証用の具体的なridは公開ドキュメントとAPI例に書かない。本番の投稿用社員とridは本番手動テスト時に設定する。
9. オフラインでは端末内に入力を保持し、オンライン復帰後に同期する。オフライン中に「完了」は送信開始と見なさず、利用者の再確認後にAPIを呼ぶ。端末保存・復旧方式は別途技術設計で確定する。

## 5. API実装前に確定する項目

| 項目 | 現状 | APIへの影響 |
| --- | --- | --- |
| 作業記録の共同閲覧・編集範囲 | 要確認 | 所有者以外への`403`条件。 |
| 表示用の報告版番号 | 要確認 | APIではまず単調増加する`sequence`を採用する設計案。 |
| 端末紛失・例外的な複数端末利用 | 要確認 | 同期・復旧UX。APIは競合検出を先に備える。 |

関連文書: [要件定義](requirements.md)、[状態遷移](state-transitions.md)、[論理データ設計](data-design.md)、[テストケース](test-cases.md)
