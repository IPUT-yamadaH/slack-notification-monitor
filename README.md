# Slack Notification Monitor (Google Apps Script)

複数のSlackチャンネルを定期確認し、すべての投稿・キーワード一致・Gemini AIによる判定のいずれかで選んだ投稿をWebhookへ転送するGoogle Apps Scriptです。監視先やキーワードを設定すれば、特定の授業や組織に依存せず利用できます。

## 機能

- 複数のSlackチャンネルを5分ごとに確認
- チャンネルごとに `all`、`keyword`、`ai_filter` の通知方式を設定
- Gemini AIによる任意の投稿フィルター
- 添付ファイルと元投稿へのリンクを通知に付加
- チャンネルごとに処理済み時刻を記録し、初回の過去投稿一括通知を回避
- 秘密情報や監視対象をソースコードに含めない設定方式

## 必要なもの

- GoogleアカウントとGoogle Apps Scriptプロジェクト
- 監視したいチャンネルへのアクセス権を持つSlack OAuthトークン
- 転送先のSlack Workflow webhook、またはIncoming Webhook
- `ai_filter`を使う場合はGoogle AI StudioのGemini APIキーと利用可能なモデル名

Slackトークンには、監視対象に応じた `conversations.history` の読み取り権限と `auth.test` の権限が必要です。プライベートチャンネルやDMは、トークンの種類・権限・アプリが会話に参加しているかどうかで読み取れない場合があります。

## セットアップ

1. [Google Apps Script](https://script.google.com/)で新しいプロジェクトを作成します。
2. エディタにあるコードを消し、GitHubにアップロードした `Code.gs` の内容を貼り付けて保存します。
3. Slackで監視したいチャンネルを開き、チャンネル詳細からチャンネルIDをコピーします。IDはチャンネル名とは別の英数字の値です。監視用Slackアプリ／トークンがそのチャンネルを読めることも確認します。
4. Apps Scriptの **プロジェクトの設定 → スクリプト プロパティ**を開き、以下のプロパティを登録します。値の例はこのREADMEの表とJSON例を参照してください。
5. `WATCH_TARGETS_JSON`のID・表示名・通知モードを実際の監視先に合わせ、`keyword`モードを使うなら `KEYWORDS_JSON` にキーワードを登録します。
6. エディタで `checkSlackAuth` を選んで実行し、Googleの認可を進めます。実行ログでSlack認証が成功したことを確認します。
7. `monitorSlack` を一度手動実行します。各チャンネルの基準時刻を記録するため、この初回は過去投稿を通知しません。
8. `setupTrigger` を一度実行して認可します。以降は5分ごとに新しい投稿を確認します。

### 通知モード

- `all`: 新しい投稿をすべて通知します。
- `keyword`: `KEYWORDS_JSON`のいずれかを含む投稿を通知します。
- `ai_filter`: Geminiに投稿を判定させます。このモードを使う対象がある場合だけ `GEMINI_API_KEY` と `GEMINI_MODEL` も設定します。

### 必須プロパティ

| プロパティ | 値 |
|---|---|
| `SLACK_TOKEN` | Slack OAuthトークン。秘密として扱い、コードやREADMEに書かない |
| `SLACK_WEBHOOK_URL` | 通知先Webhook URL。秘密として扱い、コードやREADMEに書かない |
| `WATCH_TARGETS_JSON` | 監視対象のJSON配列。下記例を参照 |
| `KEYWORDS_JSON` | `keyword`モードを使う場合に必要なキーワード配列。ほかのモードだけなら `[]` |

### 任意プロパティ

| プロパティ | 用途 |
|---|---|
| `SLACK_WEBHOOK_FIELD` | Webhookが受け取るJSONフィールド名。Slack Workflow webhookは通常、Workflow側で定義したフィールド名（既定値 `message_content`）。Incoming Webhookの場合は `text` にする |
| `SLACK_WORKSPACE_DOMAIN` | 元投稿リンクを作るワークスペースのサブドメイン（`https://`や`.slack.com`を含めない） |
| `GEMINI_API_KEY` | `ai_filter`モードに必要なGemini APIキー |
| `GEMINI_MODEL` | `ai_filter`モードに必要なモデル名。利用可能なモデル名をGoogle AI Studioで確認 |
| `AI_INTERESTS` | AI判定で重視する分野・内容を説明する文 |

`WATCH_TARGETS_JSON`の例（`CHANNEL_ID_1`などを実際のチャンネルIDに置き換えてください。使わない対象は行ごと削除できます）:

```json
[
  {"name":"お知らせ","id":"CHANNEL_ID_1","mode":"keyword"},
  {"name":"一般","id":"CHANNEL_ID_2","mode":"all"},
  {"name":"質問","id":"CHANNEL_ID_3","mode":"ai_filter"}
]
```

`KEYWORDS_JSON`の例:

```json
["deadline", "schedule", "meeting"]
```

## Webhookの形式

コードは既定で次のJSONをWebhookへ送ります。Slack Workflow webhookの受信フィールド名が異なる場合は `SLACK_WEBHOOK_FIELD` を合わせてください。Slack Incoming Webhookを使う場合は `SLACK_WEBHOOK_FIELD=text` とします。

```json
{"message_content":"転送する投稿本文"}
```

## 注意事項

- このリポジトリにトークン、Webhook URL、APIキー、実際のチャンネルIDやDM情報をコミットしないでください。スクリプトプロパティにのみ登録してください。
- このスクリプトはSlack投稿本文を通知先Webhookへ転送します。`ai_filter`では、判定のため投稿本文をGemini APIに送信します。対象データの取り扱いルールを確認してから利用してください。
- AI判定が失敗した場合は、重要な投稿を落とさないよう通知する設計です。
- Slack APIの権限、チャンネル参加状態、Webhookの受信形式によっては調整が必要です。
- `Code.gs`には特定の授業の問題・教材・監視チャンネルは含めていません。監視先とキーワードは利用者が設定します。
