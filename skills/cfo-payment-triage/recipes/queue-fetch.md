# 承認待ちキュー 一括取得レシピ

3種の申請種別（支払依頼・経費精算・各種申請）を並列で取得し、統合優先度一覧を生成する。

## 対象エンドポイント

| 種別 | エンドポイント | キー | 備考 |
|------|-------------|------|------|
| 支払依頼 | `GET /api/1/payment_requests` | `payment_requests[]` | partner_id・payment_date あり |
| 経費精算 | `GET /api/1/expense_applications` | `expense_applications[]` | purchase_lines・領収書 |
| 各種申請 | `GET /api/1/approval_requests` | `approval_requests[]` | フォーム定義依存・金額任意 |

`approver_id` を指定すると3種とも `status: in_progress` のみ返る。並列取得で問題ない。

## Step 1: 自分のユーザーIDを確認

```
freee_current_user
```

`id` を approver_id として使用する。

## Step 2: 3種を並列取得

以下3つを同時に実行する:

```
freee_api_get {
  "service": "accounting",
  "path": "/api/1/payment_requests",
  "query": {
    "approver_id": <自分のユーザーID>,
    "limit": 100
  }
}
```

```
freee_api_get {
  "service": "accounting",
  "path": "/api/1/expense_applications",
  "query": {
    "approver_id": <自分のユーザーID>,
    "limit": 100
  }
}
```

```
freee_api_get {
  "service": "accounting",
  "path": "/api/1/approval_requests",
  "query": {
    "approver_id": <自分のユーザーID>,
    "limit": 100
  }
}
```

## Step 3: 統合ノーマライズ

3種のレスポンスを共通フォーマットに変換する。

### 共通フォーマット

| フィールド | payment_requests | expense_applications | approval_requests |
|-----------|-----------------|---------------------|------------------|
| `type` | `"payment"` | `"expense"` | `"approval"` |
| `id` | `id` | `id` | `id` |
| `title` | `title` | `title` | `title` |
| `total_amount` | `total_amount` | `total_amount` (任意) | `request_items[]` の `type: amount` の合計値。なければ `null` |
| `application_date` | `application_date` | `issue_date` | `application_date` |
| `due_date` | `payment_date` | `null` | `null` |
| `applicant_id` | `applicant_id` | `applicant_id` | `applicant_id` |
| `partner_name` | `partner_name` | `null` | `request_items[]` の `type: partner` の値。なければ `null` |
| `current_round` | `current_round` | `current_round` (任意) | `current_round` |
| `application_number` | `application_number` | `application_number` | `application_number` |

### approval_requests の金額取得

`request_items[]` を走査し、`type: "amount"` のエントリの `value` を数値変換して合計する:

```
approval_total_amount = sum(
  item.value as integer
  for item in approval_request.request_items
  if item.type == "amount"
)
```

`request_items[]` に `type: "amount"` がない場合は `total_amount: null` とし、リスクスコア算出から金額要素を除外する。

### approval_requests の取引先取得

`request_items[]` を走査し、`type: "partner"` のエントリの `value` を使用する。なければ `null`。

## Step 4: 優先度スコア算出

### 緊急度スコア（urgency_score）

全種共通:

| 条件 | 加点 |
|------|------|
| `due_date` まで3日以内 | +30 |
| `due_date` まで7日以内 | +20 |
| `due_date` まで14日以内 | +10 |
| `due_date` が null | +0（due_date なし種別） |
| 再申請（`current_round > 0`） | +15 |
| 申請から7日以上待機 | +10 |

支払依頼のみ `due_date = payment_date` が存在する。経費精算・各種申請は `due_date` がないため、申請待機日数で代替する。

### リスクスコア（risk_score）

| 条件 | 加点 | 適用種別 |
|------|------|---------|
| `total_amount >= 5,000,000` | +35 | 全種 |
| `total_amount >= 1,000,000` | +25 | 全種 |
| `total_amount >= 300,000` | +10 | 全種 |
| `total_amount: null` かつ approval_request | +10 | 各種申請 |
| `partner_name` あり（支払依頼・各種申請） | +5 | 支払依頼・各種申請 |
| タイトルに値引き・特例シグナル語 | +20 | 全種 |
| タイトルに契約シグナル語 | +20 | 全種 |
| 再申請 | +15 | 全種 |

シグナル語はタイトルのみで暫定スコア。詳細分析は各種別レシピで実施。

### 優先度バンド

| バンド | urgency_min | risk_min |
|--------|-------------|----------|
| critical | ≥ 30 | ≥ 40 |
| high | ≥ 20 | ≥ 25 |
| medium | ≥ 10 | ≥ 10 |
| low | — | — |

## Step 5: 統合一覧を表示

優先度バンド → urgency_score 降順 → risk_score 降順でソートして表示する。

```
# 承認待ち一覧（合計 X件 / 支払依頼 N件 / 経費精算 M件 / 各種申請 L件）

| 優先度 | 種別 | No. | タイトル | 申請者ID | 金額 | 支払期限 | 申請日 | 推奨アクション |
|--------|------|-----|---------|---------|------|---------|--------|--------------|
| CRITICAL | 支払依頼 | 2341 | システム保守費（株式会社ABC） | 101 | ¥3,500,000 | 3日後 | 2026-03-10 | 詳細確認→詳細分析 |
| HIGH | 経費精算 | 892 | 3月大阪出張精算 | 45 | ¥85,000 | - | 2026-03-08 | 詳細確認 |
| HIGH | 各種申請 | 501 | Q1予算追加申請（稟議） | 22 | ¥2,000,000 | - | 2026-03-09 | 詳細確認 |
| MEDIUM | 支払依頼 | 2340 | 広告費請求（DEF社） | 78 | ¥450,000 | 10日後 | 2026-03-12 | 通常確認 |
```

## Step 6: 詳細分析への分岐

一覧表示後、個別案件の詳細分析を行う場合は種別に応じて分岐する:

| 種別 | 詳細分析レシピ |
|------|--------------|
| 支払依頼 | `recipes/payment-request-triage.md` |
| 経費精算 | `recipes/expense-application-triage.md` |
| 各種申請 | `recipes/approval-request-triage.md` |

複数件を処理する場合は priority_band: critical → high の順に処理する。

## 注意点

- `approver_id` 指定時、各 API は自分の承認ステップに到達した案件のみ返す（他の承認ステップ待ちは含まない）
- 件数が多い場合（各種 50件超）は `offset` を使ってページング取得する
- 3種の合計件数が多い場合は `priority_band: critical / high` のみ詳細分析し、`medium / low` は一覧確認にとどめることを推奨する
