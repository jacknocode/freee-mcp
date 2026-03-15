# 経費精算トリアージ レシピ

`expense_applications` の詳細分析。支払依頼と異なり取引先概念がなく、
領収書添付・費用カテゴリ・申請者の経費パターンが主要チェックポイント。

## payment_requests との主な差異

| 項目 | payment_requests | expense_applications |
|------|-----------------|---------------------|
| 取引先 | partner_id / partner_name | なし |
| 支払期限 | payment_date | なし |
| 明細 | payment_request_lines[] | purchase_lines[] 内の expense_application_lines[] |
| 領収書 | receipt_ids[]（申請全体） | purchase_lines[].receipt_id（行ごと） |
| 給与連携 | なし | payroll_attached フラグ |
| 勘定科目 | account_item_id（行単位） | expense_application_lines[].account_item_id |

## Step 1: 詳細取得

```
freee_api_get {
  "service": "accounting",
  "path": "/api/1/expense_applications/<id>"
}
```

主要フィールド:

| フィールド | 意味 | 用途 |
|-----------|------|------|
| `title` | 申請タイトル | 種別判断（出張・購買・交際等） |
| `description` | 備考 | URLやシグナル語の抽出元 |
| `total_amount` | 合計金額 | リスクスコア |
| `issue_date` | 申請日 | 待機日数計算 |
| `applicant_id` | 申請者ID | 申請者パターン取得 |
| `section_id` | 部門ID | 費用カテゴリの整合確認 |
| `payroll_attached` | 給与連携フラグ | 精算方法の確認 |
| `purchase_lines[]` | 購買行一覧 | 領収書確認・明細分析 |
| `comments[]` | コメント | 往復数・URL抽出 |
| `approval_flow_logs[]` | 承認履歴 | 差し戻し理由 |
| `current_step_id` | 現在ステップID | 承認アクション用 |
| `current_round` | ラウンド | 再申請検出 |

## Step 2: purchase_lines の分析

各 `purchase_lines[]` エントリは「購買日（transaction_date）」単位の行で、
その中に `expense_application_lines[]` として費目明細が入る。

### 構造

```
purchase_lines[]: [
  {
    "receipt_id": 123,           // 領収書ファイルID（null の場合は未添付）
    "transaction_date": "2026-03-10",
    "expense_application_lines": [
      {
        "account_item_id": 55,
        "amount": 15000,
        "description": "タクシー代（得意先訪問）"
      }
    ]
  }
]
```

### チェック項目

#### EA-G001: 領収書未添付の高額行

条件:
- `purchase_lines[].receipt_id == null` かつ
- その行の `expense_application_lines[].amount` の合計 > 10,000円

結果: blocking — 領収書の添付または紛失理由の説明を求める

#### EA-G002: 明細合計と申請合計の乖離

条件:
- `sum(expense_application_lines[].amount for all purchase_lines)` と
  `total_amount` の差が 10% 超 かつ説明なし

結果: blocking — 金額整合の確認を求める

#### EA-P001: 高額精算

条件: `total_amount >= 100,000`（経費精算の CFO 確認基準は支払依頼より低め）
結果: high — CFO確認対象

#### EA-P002: 単一費目集中

条件:
- 特定の `account_item_id` に金額が集中しており、
  その費目が通常の経費カテゴリ（交通費・宿泊費等）でない場合
- 例: 「雑費」「消耗品費」に大半が集中

結果: medium — 費目選択の確認

#### EA-P003: 長期間にわたる精算

条件:
- `purchase_lines[]` の `transaction_date` の幅が 60日超

結果: medium — 長期分の一括精算であることを確認

#### EA-H001: 交際費・接待費の人数・目的記載

条件:
- `account_item_id` が交際費・接待費カテゴリ かつ
- `expense_application_lines[].description` に人数・目的の記載がない

結果: medium — 接待目的と参加者の記載を確認

#### EA-H002: 給与連携あり

条件: `payroll_attached == true`
結果: low — 給与精算として処理されることを確認

## Step 3: 申請者の経費パターン取得

```
freee_api_get {
  "service": "accounting",
  "path": "/api/1/expense_applications",
  "query": {
    "applicant_id": <applicant_id>,
    "start_issue_date": "<今日から180日前>",
    "limit": 50
  }
}
```

算出指標:

- `avg_amount_6m`: 申請者の過去6ヶ月平均額
- `return_rate_6m`: feedback ステータスの比率
- `max_amount_6m`: 過去6ヶ月の最高額
- `missing_receipt_rate`: 過去案件で領収書未添付だった行の比率（高い場合は傾向として記載）

ベースライン逸脱（EA-P004）:
- 今回金額が `avg_amount_6m * 2.0` 超 かつ 件数3件以上

結果: high — 通常より大きな精算の理由確認

## Step 4: 評価まとめと推奨アクション

### blocking ゲートがある場合

推奨: `return_candidate`

差し戻しコメント例:
- EA-G001: 「〇〇行（¥XX,XXX）に領収書が添付されていません。領収書添付または紛失理由（金額・日時・目的・支払先）の記載をお願いします。」
- EA-G002: 「明細合計（¥XX,XXX）と申請合計（¥XX,XXX）に差異があります。修正または理由のご説明をお願いします。」

### blocking なし・確認事項あり

推奨: `approve_after_check`

確認事項（最大3件）を提示する。

### 問題なし

条件:
- 全行に領収書添付あり
- 明細合計一致
- ベースライン逸脱なし
- 説明が具体的

推奨: `approve_candidate`

## Step 5: ブリーフ出力

```
## [優先度] 経費精算 No.{application_number}: {title}

### なぜ今見るべきか
{待機日数・金額・再申請の有無}

### この申請は何か
申請者ID: {applicant_id}
合計金額: ¥{total_amount:,}
申請日: {issue_date}
明細行数: {purchase_lines の件数}件（{合計expense_application_lines数}費目）
給与連携: {payroll_attached}

### 領収書添付状況
全 {N} 行中:
- 添付あり: {X} 行
- 未添付: {Y} 行 {Y > 0 なら: → EA-G001 blocking}

### 主要費目内訳
{account_item 別の合計額（上位3件）}

### 申請者の過去パターン（過去6ヶ月）
平均精算額: ¥{avg_amount_6m:,}
今回との差: {+/-X%}
差し戻し率: {return_rate_6m}%

### ゲート評価
{blocking がある場合のみ}

### 未解決論点（最大3件）
1. ...

### 推奨アクション
{approve_candidate / approve_after_check / return_candidate}

---
最終操作は必ず人間が行ってください。
承認: POST /api/1/expense_applications/{id}/actions（approve）
差し戻し: POST /api/1/expense_applications/{id}/actions（feedback）
```

## 承認アクション

`expense_applications` の承認操作は `payment_requests` と同じエンドポイントパターンを使用する:

```
freee_api_post {
  "service": "accounting",
  "path": "/api/1/expense_applications/<id>/actions",
  "body": {
    "company_id": <company_id>,
    "approval_action": "approve",
    "target_step_id": <current_step_id>,
    "target_round": <current_round>
  }
}
```

`approval_action` の選択肢: `approve` / `force_approve` / `reject` / `feedback` / `cancel`
