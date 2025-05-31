# n8n 非同期ワークフロー例

このリポジトリには、同期的な HTTP 応答と背後で実行される長時間処理を分離する n8n のサンプルワークフロー（Workflow A と Workflow B）が含まれています。目的は、webhook などからのリクエストに即座に応答しつつ、重い処理（API 呼び出しなど）をバックグラウンドで実行し、タイムアウトを回避する方法を示すことです。

---

## コンテンツ

* **Workflow A (workflowA.json)**：HTTP リクエスト（例：`curl` コマンドや任意の webhook）を受け取り、即時に確認メッセージを返し、n8n の **Execute Workflow** ノードを使って Workflow B をバックグラウンドで起動します。

* **Workflow B (workflowB.json)**：Workflow A から転送されたデータを受け取り、短い遅延（Delay）を挟んだあと、Slack の Incoming Webhook にメッセージを投稿します。必要に応じて、この最終ステップを OpenAI や GitHub API 呼び出しなどに置き換えることができます。

* **README.md**：このファイル。これらのワークフローを n8n にインポートして実行する手順を解説します。

---

## 前提条件

* 動作する n8n 環境（セルフホストまたは n8n.cloud）。
* Slack の Incoming Webhook URL（または他の HTTP エンドポイント）を用意。
* n8n の操作（JSON インポート、ワークフローの有効化、実行ログの確認）に慣れていること。

---

## Workflow A：即時応答＋Workflow B の起動

### 概要

1. **Webhook ノード** (`/webhook/trigger`)
   * 任意の HTTP POST リクエスト（例：`curl`）を受け取る。
   * 2 つの出力ポートを持ち、
     * **1st 出力** → **Respond to Webhook**：クライアントに即座に JSON 確認メッセージを返します（例：`{ "status": "ok", "msg": "リクエストを受け取りました。バックグラウンド処理を開始します。" }`）。
     * **2nd 出力** → **Execute Workflow (Workflow B)**：同じ JSON ペイロードを Workflow B に引き渡し、非同期に実行させます。

### インポートと有効化手順

1. n8n エディタで **Workflows → Import** をクリック。
2. `workflowA.json` の内容を貼り付けて **Import** をクリック。
3. インポートされた **"Workflow A: Sync + Trigger B"** を開く。
4. **Webhook** ノードのパスが `/trigger` になっていることを確認（必要に応じて変更）。
5. **Execute Workflow** ノードの **Workflow ID** を Workflow B の ID に設定（後述の手順で B の ID をコピーしてください）。
6. 画面右上の **Active** トグルを「ON」にしてワークフローを有効化。

### テスト例

以下を実行して動作確認できます：

```
curl -X POST https://<あなたの-n8nドメイン>/webhook/trigger \
     -H "Content-Type: application/json" \
     -d '{ "foo": "bar" }'
```

* 即時に以下のような応答が返ってきます：

```json
{ "status": "ok", "msg": "リクエストを受け取りました。バックグラウンド処理を開始します。" }
```

* n8n の **Executions** パネルで Workflow A の Webhook → Respond → Execute Workflow の実行が確認できます。

---

## Workflow B：バックグラウンドで遅延実行＋Slack 通知

### 概要

1. **Execute Workflow Trigger ノード**
   * Workflow A の **Execute Workflow** ノードによって起動され、A から転送された JSON ペイロードを受け取ります。

2. **Wait ノード**（5 秒）
   * 一定時間（例：5 秒）待機します（ここが長時間処理の模擬）。

3. **HTTP Request (Slack Incoming Webhook)**
   * Slack の Incoming Webhook URL に POST を行い、メッセージを投稿します。
   * ボディは以下のような形式です：
     ```json
     { "text": "<メッセージ本文>" }
     ```
   * この例では、A から転送されたペイロードに含まれる `foo` の値（`$json["body"]["foo"]`）を利用して Slack に送信します。

### インポートと有効化手順

1. n8n エディタで **Workflows → Import** をクリック。
2. `workflowB.json` の内容を貼り付けて **Import** をクリック。
3. インポートされた **"Workflow B: Async → Slack"**（名称は任意ですがわかりやすい名前に変更してもOK）を開く。
4. **HTTP Request** ノードを編集：
   * **Request Method** を `POST` に設定。
   * **URL** にあなたの **Slack Incoming Webhook** URL を貼り付け。
   * **Response Format** を `String` に変更（Slack がプレーンテキスト `ok` を返すため）。
   * **JSON/RAW Parameters** を ON にし、**Body Parameters** に次のように入力：
     ```jsonc
     { "text": "{{ $json['body']['foo'] }}" }
     ```
   * `$json['body']['foo']` で、Workflow A から渡ってきた `foo` の値（例：`bar` など）を取得します。

5. 画面右上の **Active** トグルを「ON」にしてワークフローを有効化。

### Workflow B 単独テスト（オプション）

Workflow A を経由せずに B をテストしたい場合、B の **Execute Workflow Trigger**（または Webhook トリガー）に対して直接以下のように実行できます：

```
curl -X POST https://<あなたの-n8nドメイン>/webhook/workflowB-trigger-path \
     -H "Content-Type: application/json" \
     -d '{ "body": { "foo": "baz" } }'
```

* 即座に 200 OK が返り、5 秒後に Slack に "baz" を含むメッセージが投稿されます。

---

## A と B が連携して動く仕組み

1. **クライアント (curl)** → Workflow A の `/webhook/trigger` に POST。
2. **Workflow A**：
   * **Respond to Webhook**：クライアントに即座に `{ "status": "ok" }` を返却。
   * **Execute Workflow**：同じ JSON を Workflow B に転送し、B を非同期起動。

3. **Workflow B**：
   * 受け取った JSON を `Execute Workflow Trigger` で受信。
   * **Wait (5秒)** を実行（またはここに AI → GitHub などの処理を追加）。
   * **HTTP Request (Slack)** で Slack に `{ "text": "<転送された foo の値>" }` を送信。

これにより、
* 元の HTTP リクエストはタイムアウトせずにすぐ応答され、
* 重い処理（AI 処理や GitHub Issue 作成など）をバックグラウンドで安全に行えます。

---

## カスタマイズのヒント

* **Wait ノードの代わりに、任意の処理を挿入**：
  * OpenAI ノード → 応答生成や要約機能
  * GitHub ノード → Issue の自動作成や取得
  * Notion、Google Sheets、任意の REST/GraphQL API

* **Slack 通知のメッセージを変更**：
  * `{{ $json['body']['foo'] }}` の部分を別のフィールド、または複数の値を組み合わせたメッセージに変更可能。
  * `JSON.stringify($json)` を用いて、ペイロード全体を文字列化したものを送信しても OK。

* **トリガー方法を変更**：
  * Webhook ではなく定期スケジュールをトリガーにして、一定時間ごとに同期的な応答とバックグラウンド処理を実行する。
  * その場合、Workflow A は「Cron トリガー → Execute Workflow B → Respond」（※ Cron トリガーでも HTTP レスポンスは不要）という流れにすれば同様の非同期分離が実現できます。

---

<img width="829" alt="image" src="https://github.com/user-attachments/assets/db42ad2a-6e16-4f77-a699-fbcc6611b41f" />
<img width="926" alt="image" src="https://github.com/user-attachments/assets/81d03289-3472-455c-b625-f96b781b96da" />


## ライセンス

このサンプルコードは MIT ライセンスの下で提供されています。

自由に変更・拡張してご利用ください。Happy automating!

