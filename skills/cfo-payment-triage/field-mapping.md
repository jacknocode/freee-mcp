# フィールドマッピング: freee API → キャノニカルスキーマ

## GET /api/1/payment_requests/{id} レスポンス → application

| キャノニカルフィールド | freee APIフィールド | 型 | 備考 |
|---------------------|-------------------|-----|------|
| `application.id` | `payment_request.id` | integer → string | 文字列化して使用 |
| `application.title` | `payment_request.title` | string | |
| `application.description` | `payment_request.description` | string | URLやシグナル語の抽出元 |
| `application.applicant_id` | `payment_request.applicant_id` | integer → string | |
| `application.total_amount` | `payment_request.total_amount` | integer | 円単位 |
| `application.application_date` | `payment_request.application_date` | string (yyyy-mm-dd) | |
| `application.due_date` | `payment_request.payment_date` | string (yyyy-mm-dd) | 支払期限 |
| `application.current_step` | `payment_request.current_step_id` | integer | 承認アクション時に target_step_id として使用 |
| `application.current_round` | `payment_request.current_round` | integer | > 0 なら再申請 |
| `application.approvers` | `payment_request.approvers` | array | 承認者状態の確認 |
| `application.payment_request_lines` | `payment_request.payment_request_lines` | array | 明細行（下記参照） |
| `application.comments` | `payment_request.comments` | array | コメント履歴（下記参照） |
| `application.approval_flow_logs` | `payment_request.approval_flow_logs` | array | 承認履歴（下記参照） |
| `application.receipt_ids` | `payment_request.receipt_ids` | array[integer] | 証憑ファイルID一覧 |

## applicant_name の取得

`payment_request.applicant_id` からユーザー名は直接取得できない。
承認履歴 `approval_flow_logs[].user_name` または コメント `comments[].user_name` から
申請者のIDに対応する名前を探す。見つからない場合は「ユーザーID: {applicant_id}」と表示する。

## payment_request_lines[] → application.payment_request_lines[]

| キャノニカルフィールド | freee APIフィールド | 備考 |
|---------------------|-------------------|------|
| `line_id` | `id` | |
| `description` | `description` | URLやシグナル語の抽出元 |
| `amount` | `amount` | 円単位 |
| `account_item` | `account_item_id` | IDのみ（名称はマスタ参照が必要） |
| `tax_code` | `tax_code` | |
| `segment_1` | `segment_1_tag_id` | IDのみ |
| `segment_2` | `segment_2_tag_id` | IDのみ |
| `segment_3` | `segment_3_tag_id` | IDのみ |
| `memo` | `description` | description と同一フィールド |

明細合計の検証:

```
sum(payment_request_lines[].amount) vs total_amount
```

`line_type` の解釈:
- `deal_line`: 通常取引行（加算）
- `negative_line`: 控除・マイナス行（減算）→ 値引きシグナルとして検出
- `withholding_tax`: 源泉所得税行

## comments[] → application.comments[]

| キャノニカルフィールド | freee APIフィールド | 備考 |
|---------------------|-------------------|------|
| `comment_id` | `id` | |
| `user_name` | `user_name` | |
| `user_id` | `user_id` | |
| `created_at` | `created_at` | |
| `body` | `body` | URLやシグナル語の抽出元 |

コメント往復数の計算:

異なる user_id 間で交互にコメントが続く回数をカウントする。
`comments[]` を `created_at` 昇順でソートし、前後のコメントの user_id が異なる場合に往復+1。

## approval_flow_logs[] → application.approval_flow_logs[]

| キャノニカルフィールド | freee APIフィールド | 備考 |
|---------------------|-------------------|------|
| `log_id` | `id` | |
| `action` | `action` | approve/feedback/reject など |
| `user_name` | `user_name` | |
| `user_id` | `user_id` | |
| `created_at` | `created_at` | |
| `body` | `body` | 差し戻し理由など（P005で参照） |

再申請の前回理由取得:

`current_round > 0` の場合、`approval_flow_logs[]` から `action: "feedback"` のエントリを
`created_at` 降順でソートし、直近の `body` を前回差し戻し理由として使用する。

## vendor_key の生成

freee API には vendor_key という概念はない。以下の優先順位でベンダーを特定する:

1. `payment_request.partner_id`（取引先IDで過去案件を絞り込み）
2. `payment_request.partner_name`（表示用ラベル）
3. `payment_request.partner_code`（コードが存在する場合）

過去案件の取得は `partner_id` を使用する（名前は表記ゆれがあるため）。

## status の解釈

| freee ステータス | 意味 | トリアージ上の扱い |
|----------------|------|-----------------|
| `draft` | 下書き | 承認待ちキュー対象外 |
| `in_progress` | 申請中 | 承認待ちキュー対象 |
| `approved` | 承認済 | 過去案件（履歴取得用） |
| `rejected` | 却下 | 過去案件（履歴取得用） |
| `feedback` | 差し戻し | 申請者が修正中（再申請前） |

## 再申請検出

```
is_resubmission = payment_request.current_round > 0
```

`current_round` は差し戻し等により申請がステップの最初からやり直しになると増加する。
1以上であれば何らかの形で一度差し戻しが発生したと判断する。

## 承認アクション実行に必要なフィールド

POST /api/1/payment_requests/{id}/actions で必要:

| パラメータ | 取得元 | 備考 |
|-----------|--------|------|
| `company_id` | `payment_request.company_id` | |
| `target_step_id` | `payment_request.current_step_id` | 最新の詳細取得値を使用 |
| `target_round` | `payment_request.current_round` | 最新の詳細取得値を使用 |
| `approval_action` | 人間が選択 | approve/feedback/reject など |

Step 8 実行前に必ず最新の詳細を再取得することを推奨する（承認ステップが変わっている可能性）。

## 過去案件取得クエリのパラメータマッピング

| 取得目的 | クエリパラメータ | 値の計算方法 |
|---------|---------------|------------|
| 同一取引先の履歴 | `partner_id` | `payment_request.partner_id` |
| 申請者のパターン | `applicant_id` | `payment_request.applicant_id` |
| 金額バンド絞り込み | `min_amount`, `max_amount` | `total_amount * 0.5`, `total_amount * 2.0` |
| 期間絞り込み | `start_application_date` | 今日から365日前（ベースライン）/ 180日前（申請者パターン） |
| 承認済みのみ | `status` | `approved`（ベースライン算出時） |

## 日付計算リファレンス

| 目的 | 計算式 |
|------|--------|
| 支払期限まで残り日数 | `payment_date - today` |
| 申請からの待機日数 | `today - application_date` |
| ベースライン期間開始 | `today - 365日` |
| 申請者パターン期間開始 | `today - 180日` |
| 参照資料の鮮度チェック | `today - document_updated_at > 90日` で stale |
