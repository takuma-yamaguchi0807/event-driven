# イベント駆動学習環境 — SNS + SQS + Lambda 統合構成

## 概要

AWS の SNS・SQS・Lambda を組み合わせ、**Queue** と **Pub/Sub** の両方を同一構成で体感するための学習環境。

## 構成

```
API Gateway (POST /publish)
        ↓
    SNS Topic
    ↙         ↘
SQS Queue    Lambda B（直接サブスク）
  ↓   ↘DLQ        ↓
Lambda A        ログ出力
  ↓
ログ出力
```

詳細は [ARCHITECTURE.md](./ARCHITECTURE.md) を参照。

## 各コンポーネントの役割

| コンポーネント | 役割 |
|---|---|
| API Gateway | 受け口。`POST /publish` に JSON を投げると処理が始まる |
| SNS Topic | イベント配信元。サブスクライバー全員に同時配信（Pub/Sub の核心） |
| SQS Queue | SNS のサブスクライバー。メッセージを一時的に貯めてバッファリング（Queue の核心） |
| DLQ | Lambda A が一定回数失敗したメッセージを退避させる別の SQS キュー |
| Lambda A | SQS からメッセージを受け取ってログ出力。SQS がトリガー（ポーリング） |
| Lambda B | SNS から直接メッセージを受け取ってログ出力。SNS がトリガー（プッシュ） |

## Queue vs Pub/Sub の対比

| | Lambda A（Queue 経由） | Lambda B（直接） |
|---|---|---|
| トリガー | SQS（ポーリング） | SNS（プッシュ） |
| 失敗時 | DLQ に退避、リトライ可 | SNS がリトライ |
| 順序保証 | あり（FIFO 選択可） | なし |
| バッファリング | あり | なし |

## データの流れ（1 リクエスト）

1. `POST /publish` に `{"message": "hello"}` を投げる
2. API Gateway が SNS に Publish
3. SNS が **同時に** SQS と Lambda B に配信
4. Lambda B → 即時起動、ログ出力
5. SQS → Lambda A をポーリングで起動、ログ出力
6. Lambda A が失敗したら → DLQ にメッセージが移動

## 学習ポイント

- **Pub/Sub**：SNS → Lambda B で「1 発信→複数受信」が体感できる
- **Queue**：SQS → Lambda A で「バッファ・リトライ・DLQ」が体感できる
- **Fan-out**：両方が同時に動くのを CloudWatch ログで確認できる

## 今後追加予定の改善

1. **DLQ 失敗シナリオ**：`"fail": true` を受け取ったら意図的に例外を投げる処理を Lambda A に追加し、失敗→リトライ→DLQ 移動→手動リドライブのフルサイクルを体験する
2. **SNS Message Filtering**：メッセージ属性でサブスクライバーを分岐させ、受信側が条件を定義する Pub/Sub の本質を体感する
