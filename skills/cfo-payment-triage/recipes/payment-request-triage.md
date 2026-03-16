# 支払依頼トリアージ レシピ

CFO向け高額支払依頼の承認判断下準備ワークフロー。8ステップで実行する。

## フロー全体像

```
Step 1: 承認待ちキュー取得
Step 2: 対象案件の詳細取得
Step 3: 根拠・シグナル抽出（内部処理）
Step 4: 参照URL解決（重要度の高いもののみ）
Step 5: 過去履歴・類似案件取得
Step 6: Gate / Policy / Heuristic 評価（内部処理）
Step 7: 判断ブリーフ生成
Step 8: 人間が承認/差し戻し/保留を実行
```

## Step 1: 承認待ちキュー取得

自分が承認者として関与している申請中の支払依頼を取得する。

```
freee_api_get {
  "service": "accounting",
  "path": "/api/1/payment_requests",
  "query": {
    "approver_id": <自分のユーザーID>,
    "status": "in_progress",
    "limit": 50
  }
}
```

自分のユーザーIDが不明な場合は先に取得する:

```
freee_current_user
```

レスポンスの `payment_requests[]` から以下を抽出して一覧を構築する:

| フィールド | 意味 | トリアージ用途 |
|-----------|------|--------------|
| `id` | 支払依頼ID | 詳細取得のキー |
| `title` | 申請タイトル | 概要把握 |
| `total_amount` | 合計金額 | 優先度スコア計算 |
| `payment_date` | 支払期限 | 緊急度スコア計算 |
| `application_date` | 申請日 | 待機日数計算 |
| `current_round` | 差し戻しラウンド | 再申請検出（> 0 なら再申請） |
| `partner_name` | 取引先名 | ベンダー特定 |
| `applicant_id` | 申請者ID | 申請者パターン把握 |
| `status` | ステータス | in_progress のみ対象 |

### 優先度スコア算出（一覧ソート用）

緊急度スコア（urgency_score）:

- 支払期限まで3日以内: +30
- 支払期限まで7日以内: +20
- 支払期限まで14日以内: +10
- 再申請（current_round > 0）: +15
- 申請から7日以上待機中: +10

リスクスコア（risk_score）:

- total_amount >= 5,000,000: +35
- total_amount >= 1,000,000: +25
- total_amount >= 300,000: +10
- タイトル・備考に値引き・特例シグナル語: +20（Step 3で再評価）
- 契約関連シグナル語: +20（Step 3で再評価）

優先度バンド:

- critical: urgency >= 30 かつ risk >= 40
- high: urgency >= 20 かつ risk >= 25
- medium: urgency >= 10 かつ risk >= 10
- low: それ以外

### 一覧表示フォーマット

```
# 承認待ち 支払依頼一覧（X件）

| 優先度 | ID | タイトル | 取引先 | 金額 | 支払期限 | 理由 | 推奨 |
|--------|----|---------|--------|------|---------|------|------|
| CRITICAL | 12345 | システム保守費 | 株式会社ABC | ¥3,500,000 | 3日後 | 高額・期限切迫 | 詳細確認 |
| HIGH | 12346 | 広告費（特別割引） | DEF社 | ¥1,200,000 | 7日後 | 値引きシグナル | 根拠確認 |
```

## Step 2: 対象案件の詳細取得

対象IDの支払依頼詳細を取得する（コメント・承認履歴・明細含む）。

```
freee_api_get {
  "service": "accounting",
  "path": "/api/1/payment_requests/<id>"
}
```

取得する主要フィールド（field-mapping.md 参照）:

- `description` — 備考（URLやシグナル語が含まれることが多い）
- `payment_request_lines[]` — 明細（description, amount, account_item）
- `comments[]` — コメント履歴（往復回数・URLが重要）
- `approval_flow_logs[]` — 承認履歴（差し戻し理由・前回ラウンド情報）
- `receipt_ids[]` — 証憑ファイルID一覧
- `current_step_id`, `current_round` — 承認アクション実行時に必要

## Step 3: 根拠・シグナル抽出（内部処理）

APIコールなし。Step 2のレスポンスから以下を抽出する。

### URL抽出対象フィールド

- `description`（備考）
- `comments[].body`（各コメント本文）
- `payment_request_lines[].description`（明細内容）

### シグナル語検出

値引きシグナル（discount_signal）:
- 割引・値引き・ディスカウント・discount・特別価格・調整・控除・マイナス

例外シグナル（exception_signal）:
- 例外・特例・特別対応・一時的・今回のみ・special case・先方都合

契約シグナル（contract_signal）:
- 契約・更新・締結・合意・SLA・NDA・保守・ライセンス・サブスクリプション

緊急シグナル（urgency_signal）:
- 至急・緊急・urgent・ASAP・本日中・今週中・期限厳守

再申請シグナル（resubmission_signal）:
- `current_round > 0`（APIフィールドから直接判定）
- 差し戻し・再申請・修正後・前回との違い

### 抽象的説明の検出（H002ヒューリスティック）

以下のようなパターンが説明の大半を占める場合は警告:
- 「調整済み」「いつも通り」「先方都合」「例年通り」「合意済み」「確認済み」

### コメント往復数カウント（H001ヒューリスティック）

`comments[]` の件数と、異なるユーザー間の往復回数をカウントする。往復が5回以上あれば未解決論点の可能性。

### 明細合計の検証（G002ゲート）

`sum(payment_request_lines[].amount)` と `total_amount` を比較する。
10%以上乖離し、かつ `description` や `comments` に説明がない場合は blocking。

## Step 4: 参照URL解決（重要度の高いもののみ）

Step 3で抽出したURLのうち、以下の優先ルールに基づいて実際にアクセスする:

### アクセスする対象

- 値引き・調整理由の根拠に直結する（discount_signal があり参照URLがある）
- 契約条件や価格条件の正本候補（タイトルに「見積書」「契約書」「稟議」を含む）
- blocking なゲート解消に必要（G003対象URL）
- 再申請の差分説明に直結する

### アクセスしない対象

- 背景説明のみで承認判断に不要
- 正本性が低い参考リンク集
- URLが5件を超える場合は上位3件のみ

### 各URLの評価項目

- `accessible`: 権限不足・リンク切れでないか（G003ゲート判定用）
- `title`: 文書タイトルから正本候補語を確認
- `authoritative`: 見積書・契約書・稟議など正本候補語を含むか
- `latest_likely`: 更新日時が申請日に対して十分新しいか
- `relevance_score`: 承認判断への直接的な関連度（1-5）
- `evidence_type`: pricing / contract / discount_reason / approval_context / background / unknown

Slack パーマリンク (`slack://` や `app.slack.com/archives/`) は概要説明のみ取得し、条件変更の正本確認はしない（H003ヒューリスティック対象）。

## Step 5: 過去履歴・類似案件取得

同一取引先の過去12ヶ月分の支払依頼を取得してベースライン算出。

### 同一取引先の履歴

```
freee_api_get {
  "service": "accounting",
  "path": "/api/1/payment_requests",
  "query": {
    "partner_id": <partner_id>,
    "start_application_date": "<今日から365日前>",
    "limit": 100
  }
}
```

算出するベースライン指標:

- `avg_amount_6m`: 過去6ヶ月の平均金額
- `median_amount_6m`: 過去6ヶ月の中央値
- `last_approved_amount`: 直近承認金額
- `case_count_6m`: 過去6ヶ月の件数
- `baseline_deviation_rate`: `(今回金額 - avg_amount_6m) / avg_amount_6m`

ベースライン逸脱判定（P006ポリシー）:
- `baseline_deviation_rate > 0.3`（30%超） かつ 件数3件以上: high
- 過去件数3件未満: P003（初回または疎な履歴）を適用

### 類似案件の取得（同一取引先かつ金額バンド）

```
freee_api_get {
  "service": "accounting",
  "path": "/api/1/payment_requests",
  "query": {
    "partner_id": <partner_id>,
    "min_amount": <今回金額の50%>,
    "max_amount": <今回金額の200%>,
    "limit": 10
  }
}
```

各類似案件から確認する:

- `status`: approved / feedback / rejected
- `approval_flow_logs[].body`: 差し戻し・却下理由
- `comments[]`: 過去のやりとりパターン
- `current_round`: 過去案件も再申請が多いか

### 申請者パターン

申請者IDで絞り込み、過去6ヶ月の傾向を把握:

```
freee_api_get {
  "service": "accounting",
  "path": "/api/1/payment_requests",
  "query": {
    "applicant_id": <applicant_id>,
    "start_application_date": "<今日から180日前>",
    "limit": 50
  }
}
```

算出:
- `avg_amount_6m`: 申請者の平均金額
- `return_rate_6m`: feedback ステータスの比率
- `resubmission_rate_6m`: current_round > 0 の比率

## Step 6: Gate / Policy / Heuristic 評価（内部処理）

以下の順で評価する。blocking が1件でもある場合は return_candidate を優先する。

### Gate ルール（blocking / 通過のみ判定）

#### G001: 金額根拠不足

条件: 以下のいずれも存在しない
- `description` に金額根拠の説明（金額の記載、見積番号等）
- `payment_request_lines[].description` に内訳説明
- `payment_request_lines[].amount` の合計が明示的に説明されている
- 参照URLが `accessible: true` かつ `evidence_type: pricing | contract`

結果: blocking → return_candidate

#### G002: 明細合計と申請金額の乖離

条件:
- `|sum(lines.amount) - total_amount| / total_amount > 0.1` かつ
- `description` と `comments[]` に説明なし

結果: blocking → return_candidate

#### G003: 参照資料がアクセス不能

条件:
- `description` または `comments[]` で「詳細は参照」「資料参照」等があり
- 参照URLが `accessible: false`（権限不足・リンク切れ）

結果: blocking → return_candidate

#### G004: 契約案件の根拠欠如

条件:
- contract_signal が検出された かつ
- `accessible: true` な `evidence_type: contract` の参照がない かつ
- `description` に契約条件の記述がない

結果: blocking → return_candidate

### Policy ルール（severity: high / medium）

#### P001: CFO確認対象金額

条件: `total_amount >= 1,000,000`
結果: high — CFO確認対象

#### P002: 値引きに理由が必要

条件:
- discount_signal あり かつ
- `baseline_deviation_rate` が大きい（30%以上安い or 説明のある例外）かつ
- 値引き理由の説明が抽象的（H002候補）

結果: high — 値引き理由と一時対応か恒久条件かを確認

#### P003: 初回または疎な取引先

条件: `case_count_6m < 3`
結果: medium — 取引背景と承認根拠を確認

#### P004: 契約変更・特別条項の示唆

条件: contract_signal あり かつ 過去の類似案件と条件が異なる兆候
結果: high — 契約変更の承認経緯を確認

#### P005: 再申請の差分未確認

条件: `current_round > 0`
結果: medium — 前回差し戻し理由が解消されたか確認（`approval_flow_logs[]` から前回理由を取得）

#### P006: ベースライン逸脱

条件: `baseline_deviation_rate > 0.3` かつ `case_count_6m >= 3`
結果: high — 今回だけ高い/安い理由を確認

### Heuristic ルール（severity: high / medium）

#### H001: コメント往復多数

条件: 異なるユーザー間の往復が5回以上
結果: medium — 未解決論点が残っていないか確認

#### H002: 説明が抽象的

条件: description・comments の説明が「調整済み」「いつも通り」等に留まる
結果: high — 具体的理由の追記を求める

#### H003: Slack断片のみで条件変更

条件: 重要な条件変更・値引き理由が Slack パーマリンクにしか存在しない
結果: high — 正本資料への反映有無を確認

#### H004: 参照資料が古い

条件: `latest_likely: false`（更新日時が申請日から90日以上前）
結果: medium — 最新版確認を求める

#### H005: 勘定科目が通常と異なる可能性

条件: 同取引先・同種案件の過去案件で使われた勘定科目と異なる
結果: medium — 科目選択理由を確認

### 推奨アクション決定ロジック

```
if blocking gate findings > 0:
  recommended_action = "return_candidate"

elif high severity policy/heuristic findings >= 3 OR
     (escalation_condition: amount >= 5,000,000 AND exception_signal):
  recommended_action = "escalate"

elif gate findings = 0 AND
     high severity findings = 0 AND
     unresolved_questions <= 1:
  recommended_action = "approve_candidate"

else:
  recommended_action = "approve_after_check"
```

信頼度（confidence）:

- high: ゲート通過 かつ 参照資料が確認済み かつ 過去類似案件が3件以上
- medium: 一部参照不能 または 類似案件が少ない
- low: 参照資料の多くが未確認 または 初回取引

## Step 7: 判断ブリーフ生成

人間が読める形式で出力する。

### ブリーフ構造

```
## [優先度バンド] 申請No.XXXX: {title}

### なぜ今見るべきか
{urgency の理由を1-2文で。期限・金額・再申請・待機日数など}

### この申請は何か
申請者: {applicant_name}
取引先: {partner_name}
金額: ¥{total_amount:,}（{payment_method}）
支払期限: {payment_date}
概要: {description の要点を3文以内}

### 金額と過去比較
過去6ヶ月平均（同取引先）: ¥{avg_amount_6m:,}
今回との差: {+/-X%} {↑大幅増 / ↓大幅減 / →概ね同水準}
過去の承認金額レンジ: ¥{min} 〜 ¥{max}

### 値引き/例外の有無と理由
{discount_signal や exception_signal がある場合のみ記載。なければ「特記なし」}

### 確認した根拠資料
{resolved_references のサマリ。accessible: false のものは「アクセス不能」と明記}

### 未解決論点（最大3件）
1. {最重要論点}
2. {次点}
3. {三点目（あれば）}

### ゲート評価
{blocking がある場合のみ表示。なければ省略}
- [BLOCKING] G001: {message}

### 推奨アクション
{approve_candidate / approve_after_check / return_candidate / escalate}
確信度: {high / medium / low}

{推奨理由を2-3文で}

---
最終操作は必ず人間が行ってください。
承認する場合: POST /api/1/payment_requests/{id}/actions（approve）
差し戻しの場合: POST /api/1/payment_requests/{id}/actions（feedback）+ コメント
```

### 出力ルール

- 断定は根拠がある場合のみ。推定は「〜と推定される」と明記する
- 重要論点は最大3件
- 長い説明より判断に必要な差分を優先する
- 参照資料が読めていないのに内容を決めつけない
- Slack断片だけで正式条件変更を断定しない

## Step 8: 人間が承認/差し戻し/保留を実行

このスキルは承認操作を自動実行しない。

承認操作が必要な場合は以下を提示し、人間の確認を待つ:

```
承認する場合のコマンド（実行前に確認してください）:

freee_api_post {
  "service": "accounting",
  "path": "/api/1/payment_requests/<id>/actions",
  "body": {
    "company_id": <company_id>,
    "approval_action": "approve",
    "target_step_id": <current_step_id>,
    "target_round": <current_round>
  }
}

差し戻しの場合（差し戻しコメントを必ず添える）:

freee_api_post {
  "service": "accounting",
  "path": "/api/1/payment_requests/<id>/actions",
  "body": {
    "company_id": <company_id>,
    "approval_action": "feedback",
    "target_step_id": <current_step_id>,
    "target_round": <current_round>
  }
}
```

approval_action の選択肢:
- `approve`: 承認する
- `force_approve`: 特権承認する
- `reject`: 却下する
- `feedback`: 申請者へ差し戻す
- `cancel`: 申請を取り消す

差し戻しの際は `target_step_id` と `target_round` が必要。これらは Step 2 で取得した詳細レスポンスの `current_step_id` と `current_round` を使用する。

## 典型的なケースパターン

### ケース1: 高額・根拠充足

- total_amount >= 1,000,000
- 参照URLが accessible かつ evidence_type: pricing
- ベースライン逸脱なし
- 推奨: approve_after_check（P001のみ）

### ケース2: 値引きあり・根拠抽象的

- discount_signal 検出
- description: 「先方と調整済み」のみ
- 過去比較で30%以上乖離
- 推奨: return_candidate（G001またはP002）

### ケース3: 再申請・差し戻し理由未解消

- current_round > 0
- approval_flow_logs に前回差し戻し理由あり
- comments に差分説明なし
- 推奨: return_candidate（P005）

### ケース4: 契約更新・大型案件

- contract_signal 検出
- total_amount >= 3,000,000
- 参照URLが契約書（authoritative: true）
- exception_signal あり
- 推奨: escalate

### ケース5: 問題なし

- ゲート全通過
- ベースライン逸脱なし
- 参照資料確認済み
- unresolved_questions: 0
- 推奨: approve_candidate（確信度: high）
