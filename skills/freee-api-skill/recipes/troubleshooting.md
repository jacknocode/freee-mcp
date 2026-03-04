# トラブルシューティング

freee API スキル使用時の一般的な問題と解決方法。

## 認証関連

### 問題: "401 Unauthorized"

原因: 認証トークンの有効期限切れ

解決方法:

```
freee_authenticate
```

再認証後、再度操作を実行してください。

### 問題: "403 Forbidden"

原因: 必要な権限がない、またはレートリミット

解決方法:

1. freee 開発者ポータルでアプリケーションの権限を確認
2. 必要な権限が有効化されているか確認
3. 権限を追加した場合は再認証が必要

```
freee_clear_auth
freee_authenticate
```

レートリミットの場合は数分待ってから再試行してください。

### 問題: OAuth 認証画面が表示されない

原因: ブラウザがブロックしている可能性

解決方法:

1. ポップアップブロックを一時的に無効化
2. 手動でコールバック URL にアクセス
3. 別のブラウザを試す

## 事業所関連

### 問題: "Company not found"

原因: 指定した事業所 ID が存在しないか、アクセス権限がない

解決方法:

```
# 利用可能な事業所を確認
freee_list_companies

# 正しい事業所IDを設定
freee_set_current_company { "company_id": 12345 }
```

### 問題: 事業所を切り替えたい

解決方法:

```
freee_list_companies
freee_set_current_company { "company_id": 12345 }
freee_get_current_company  # 切り替わったことを確認
```

### 問題: 複数事業所がある場合どれを選ぶべきか

解決方法:

- 操作したい事業所を選択
- 不明な場合は経理部門に確認
- `freee_list_companies` で事業所の説明を確認

### 問題: "company_id の不整合"

原因: リクエストに含まれる `company_id` と、現在設定されている事業所が異なる

解決方法:

```
# 現在の事業所を確認
freee_get_current_company

# 方法1: 事業所を切り替える
freee_set_current_company { "company_id": 12345 }

# 方法2: リクエストの company_id を現在の事業所に合わせる
```

注意: company_id を含むリクエストは、必ず現在の事業所と一致している必要があります。

## 経費申請作成時のエラー

### 問題: "expense_application_line_template_id が無効"

原因: 指定した経費科目 ID が存在しない、または事業所で無効化されている

解決方法:

```
# 有効な経費科目IDを確認
freee_api_get {
  "service": "accounting",
  "path": "/api/1/expense_application_line_templates"
}
```

詳細: 各事業所で利用可能な経費科目は異なります。必ず事前に確認してください。

### 問題: "amount must be positive"

原因: 金額が 0 以下または数値でない

解決方法: 正の整数を指定

```json
"amount": 5000     // 正しい
"amount": -100     // 負の数 - NG
"amount": 0        // ゼロ - NG
"amount": "5000"   // 文字列 - NG（JSONでは整数で指定）
"amount": 5000.5   // 小数 - NG（整数のみ）
```

### 問題: "Invalid date format"

原因: 日付形式が不正

解決方法: "yyyy-mm-dd" 形式を使用

```json
"transaction_date": "2025-10-19"           // 正しい
"transaction_date": "10/19/2025"           // NG - スラッシュ区切り
"transaction_date": "2025-10-19T00:00:00Z" // NG - 時刻部分は不要
"transaction_date": "20251019"             // NG - ハイフンなし
```

### 問題: "issue_date must be after transaction_date"

原因: 申請日が発生日より前

解決方法: 申請日を発生日以降に設定

```json
"transaction_date": "2025-10-15",  // 発生日
"issue_date": "2025-10-19"         // 申請日は発生日以降
```

注意: 通常、申請日は「今日」または発生日以降の日付を指定します。

### 問題: "title is required"

原因: 申請タイトルが空または未設定

解決方法: わかりやすいタイトルを設定

```json
"title": "2025年10月 東京出張経費"  // 具体的で推奨
"title": "経費申請"                  // 抽象的だが可
"title": ""                         // 空文字 - NG
```

### 問題: "expense_application_lines is required"

原因: 経費明細が空または未設定

解決方法: 少なくとも 1 つの経費明細を含める

```json
{
  "expense_application_lines": [
    {
      "expense_application_line_template_id": 1001,
      "amount": 5000,
      "transaction_date": "2025-10-19"
    }
  ]
}
```

### 問題: "section_id が無効"

原因: 指定した部門 ID が存在しない

解決方法:

```
# 部門一覧を確認
freee_api_get {
  "service": "accounting",
  "path": "/api/1/sections"
}
```

### 問題: 申請作成後に内容を確認したい

解決方法:

```
# 最近の申請を確認
freee_api_get {
  "service": "accounting",
  "path": "/api/1/expense_applications",
  "query": { "limit": 10 }
}
```

Web画面での確認: `https://secure.freee.co.jp/expense_applications/{id}`

## データ取得時の問題

### 問題: 経費科目一覧が取得できない

原因:

- company_id が未設定または不正
- 権限がない

解決方法:

```
# 現在の事業所を確認
freee_get_current_company

# 経費科目を取得
freee_api_get {
  "service": "accounting",
  "path": "/api/1/expense_application_line_templates"
}
```

### 問題: "経費科目が多すぎてどれを選べばいいかわからない"

解決方法:

1. 経費科目の `name` と `description` を確認
2. 一般的な科目:
   - 交通費: 電車、バス、タクシー
   - 宿泊費: ホテル、旅館
   - 接待交際費: 会食、接待
   - 消耗品費: 文房具、備品
3. 不明な場合は経理部門に確認

## PM（工数管理）関連

### 問題: GET /teams や GET /people で 401 エラー

原因: GET /teams および GET /people はシステム管理者ロールが必要。
プロジェクトマネージャーやメンバーロールではアクセスできない。

freee工数管理のデフォルトロール:
- メンバー: 基本的な工数登録・閲覧
- プロジェクトマネージャー: プロジェクト管理、GET /projects 等は利用可能
- システム管理者: 全エンドポイントにアクセス可能（GET /teams, GET /people を含む）

解決方法:

- `GET /users/me` のレスポンスで `companies[].person_me.role` を確認し、現在のロールを把握する
- システム管理者でない場合、これらのエンドポイントは利用できない
- システム管理者への昇格が必要な場合は、既存のシステム管理者に依頼する

注意: GET /users/me や GET /projects はプロジェクトマネージャーでも動作するが、
GET /teams や GET /people はシステム管理者ロールが必要。

### 問題: GET /projects のレスポンスが大きすぎる

原因: プロジェクトデータにはメンバー一覧、タグ、発注先情報など多くの情報が含まれるため、
プロジェクト数が多い事業所では数MBに達することがある

解決方法:

- `operational_status: "in_progress"` で運用中のみに絞る
- `limit` と `offset` でページネーション
- `manager_ids[]` でマネージャーが自分のプロジェクトのみ取得

### 問題: 工数を誤って登録してしまった

原因: POST /workloads は成功すると取り消せない（API に更新・削除エンドポイントがない）

解決方法:

Web UI (https://pm.freee.co.jp) の工数一覧画面から修正・削除する。
API では対応できないため、登録前に必ず内容を確認すること。

### 問題: 同じ日・同じプロジェクトに工数が二重登録された

原因: POST /workloads には重複チェックがなく、同じ条件で複数回 POST すると加算される

解決方法:

Web UI (https://pm.freee.co.jp) で余分な工数を削除する。
再実行する前に GET /workloads で現在の登録状況を確認すること。

### 問題: work_record_summaries で期待と異なる月のデータが返る

原因: work_record_summaries の year/month パラメータは「給与支払い月」を指定する。
翌月払いの場合、実際の勤怠月と給与支払い月がずれる。

解決方法:

- レスポンスの `start_date` と `end_date` を確認し、実際にどの期間のデータかを検証
- ずれている場合は month をずらして再取得
- 確実な方法: 個別の日付で `GET /employees/{id}/work_records/{date}` を使用する

### 問題: プロジェクトにメンバーでフィルタできない

原因: GET /projects には `member_ids` パラメータが存在しない。
`manager_ids[]` のみがフィルタパラメータとして提供されている。

解決方法:

1. `GET /projects?operational_status=in_progress` で取得
2. レスポンスの `projects[].members[].person_id` をクライアント側でフィルタ
3. 自分の person_id は `GET /users/me` で事前に取得しておく

### 問題: PM の person_id と HR の employee_id の対応がわからない

原因: PM API と HR API では異なるID体系を使用する

解決方法:

- PM: `GET /users/me` → `companies[].person_me.id` が person_id
- HR: `GET /api/v1/users/me` → `companies[].employee_id` が employee_id
- PM の `GET /people` レスポンスには `payroll_employee_id` フィールドがあり、
  これが HR 側の employee_id に対応する

## パフォーマンス関連

### 問題: レスポンスが遅い

原因:

- 大量のデータを取得している
- ネットワークの問題

解決方法:

- `limit` パラメータで取得件数を制限
- 必要なデータのみ取得するよう条件を絞る

### 問題: PM API のレスポンスが大きい

原因: GET /projects はプロジェクトあたりのデータ量が多い（メンバー、タグ情報を含む）

解決方法:

- `operational_status` で絞り込む
- ページネーション（`limit` + `offset`）で分割取得

## よくある質問

### Q: 経費申請を下書き保存できますか？

A: freee API では、申請作成時に自動的に申請されます。下書き保存したい場合は、ローカルでデータを保存しておき、後で API を実行してください。

### Q: 領収書画像を添付できますか？

A: ファイルボックス API (`/api/1/receipts`) を使用して証憑をアップロードし、`receipt_ids` で経費申請や取引に紐づけることができます。

### Q: 作成した申請を修正できますか？

A: 下書き・差戻し状態の経費申請は PUT API で更新できます。申請中・承認済みの場合は freee Web UI を使用してください。

### Q: 複数の経費をまとめて申請すべきですか？

A: 以下を考慮してください:

- 同じ出張: まとめる（例: 交通費+宿泊費）
- 同じ月の交通費: まとめることが多い
- 異なる種類の経費: 別々に申請することを推奨
- 会社の方針に従ってください

### Q: 工数の登録を修正・削除できますか？

A: API 経由では修正・削除できません。Web UI (https://pm.freee.co.jp) から操作してください。

### Q: 勤怠データと工数を連動させるには？

A: HR API の勤怠記録で出勤日と勤務時間を取得し、それを基に PM API で工数を登録します。
詳細な手順は `recipes/pm-workload-registration.md` を参照してください。

### Q: 工数をまとめて登録できますか？

A: POST /workloads に一括登録エンドポイントはありませんが、
複数の POST リクエストを並列で実行することで効率的に登録できます。
ただし重複チェックがないため、再実行には注意が必要です。

## サポートが必要な場合

### freee API 公式ドキュメント

https://developer.freee.co.jp/docs

### GitHub Issues

https://github.com/freee/freee-mcp/issues

freee-mcp に関する不具合や要望は、上記リポジトリの Issue で報告してください。freee サポートでは freee-mcp に関するお問い合わせは受け付けておりません。

### 問い合わせ前に確認すること

1. `freee_auth_status` で認証を確認しましたか？
2. `freee_get_current_company` で事業所を確認しましたか？
3. エラーメッセージを正確にコピーしましたか？
