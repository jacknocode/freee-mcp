# freee工数管理 操作ガイド

freee工数管理 API を使ったよくある操作のサンプルと Tips。

## 概要

- `service: "pm"` を指定する
- ベースURL: `https://api.freee.co.jp/pm`
- パス形式: `/projects`, `/workloads` など（`/api/1/` プレフィックスは不要）
- `company_id` は `GET /users/me` で取得した値を使用

## 利用可能なパス

| パス | 説明 |
|------|------|
| `/users/me` | ログインユーザー情報（company_id・person_id 取得） |
| `/projects` | プロジェクト一覧・作成 |
| `/projects/{id}` | プロジェクト詳細 |
| `/workloads` | 工数一覧・登録 |
| `/workload_summaries` | 工数サマリ |
| `/teams` | チーム一覧 |
| `/people` | 従業員一覧 |
| `/unit_costs` | 単価マスタ |
| `/partners` | 取引先一覧 |

## 使用例

### ログインユーザーと事業所IDの確認

```
freee_api_get {
  "service": "pm",
  "path": "/users/me"
}
```

レスポンスの `companies[].id` が各APIで必要な `company_id` です。

### プロジェクト一覧の取得

```
freee_api_get {
  "service": "pm",
  "path": "/projects",
  "query": {
    "company_id": 12345,
    "operational_status": "in_progress",
    "limit": 50,
    "offset": 0
  }
}
```

`operational_status` の選択肢: `planning`, `awaiting_approval`, `in_progress`, `rejected`, `done`

GET /projects には `member_ids` パラメータが存在しない。
自分がアサインされているプロジェクトを探すには、レスポンスの `projects[].members[].person_id` を確認して自分の person_id が含まれるものを抽出する。
`manager_ids[]` は指定可能なので、マネージャーの場合はそちらを使用する。

### プロジェクト詳細の取得

```
freee_api_get {
  "service": "pm",
  "path": "/projects/601",
  "query": {
    "company_id": 12345
  }
}
```

詳細には収支管理情報（`balance`）が含まれます。

### プロジェクトの作成

```
freee_api_post {
  "service": "pm",
  "path": "/projects",
  "body": {
    "company_id": 12345,
    "name": "新規プロジェクト",
    "code": "PROJ-001",
    "from_date": "2026-04-01",
    "thru_date": "2026-09-30",
    "pm_budgets_cost": 500000,
    "description": "プロジェクトの概要説明",
    "publish_to_employee": true,
    "color_id": 3,
    "manager_person_id": 10,
    "members": [
      { "person_id": 11, "unit_cost_id": 3, "budgets_cost": 3000 },
      { "person_id": 12, "unit_cost_id": 3, "budgets_cost": 2500 }
    ]
  }
}
```

色ID: `1=orange, 2=blue_green, 3=green, 4=blue, 5=purple, 6=red, 7=yellow`

`manager_person_id` を省略するとログインユーザーが担当者になる。

### 工数の登録

```
freee_api_post {
  "service": "pm",
  "path": "/workloads",
  "body": {
    "company_id": 12345,
    "project_id": 601,
    "date": "2026-03-03",
    "minutes": 120,
    "memo": "機能実装"
  }
}
```

`person_id` を省略するとログインユーザーに登録される。

工数タグが必要な場合:

```
freee_api_post {
  "service": "pm",
  "path": "/workloads",
  "body": {
    "company_id": 12345,
    "project_id": 601,
    "date": "2026-03-03",
    "minutes": 60,
    "memo": "コードレビュー",
    "workload_tags": [
      { "tag_group_id": 11, "tag_id": 12 }
    ]
  }
}
```

### 工数実績の確認

```
freee_api_get {
  "service": "pm",
  "path": "/workloads",
  "query": {
    "company_id": 12345,
    "year_month": "2026-03",
    "employees_scope": "employee",
    "person_ids[]": [10, 11]
  }
}
```

全員の工数を取得する場合:

```
freee_api_get {
  "service": "pm",
  "path": "/workloads",
  "query": {
    "company_id": 12345,
    "year_month": "2026-03",
    "employees_scope": "all"
  }
}
```

### 工数サマリの確認

```
freee_api_get {
  "service": "pm",
  "path": "/workload_summaries",
  "query": {
    "company_id": 12345,
    "year_month": "2026-03",
    "employees_scope": "all"
  }
}
```

サマリには `minutes`（総工数）と `productive_minutes`（生産時間）が含まれます。

### チーム一覧の取得

```
freee_api_get {
  "service": "pm",
  "path": "/teams",
  "query": {
    "company_id": 12345
  }
}
```

チームメンバーとリーダー情報も含まれます。

### 従業員一覧の取得

```
freee_api_get {
  "service": "pm",
  "path": "/people",
  "query": {
    "company_id": 12345,
    "status": "accepted"
  }
}
```

プロジェクトのメンバーアサイン時に `person_id` を確認するために使用します。

### 単価マスタの確認

```
freee_api_get {
  "service": "pm",
  "path": "/unit_costs",
  "query": {
    "company_id": 12345
  }
}
```

プロジェクト作成時の `members[].unit_cost_id` に使用する単価ID一覧を取得できます。

### 取引先一覧の取得

```
freee_api_get {
  "service": "pm",
  "path": "/partners",
  "query": {
    "company_id": 12345
  }
}
```

プロジェクトの発注元（`orderer_ids`）・発注先（`contractor_ids`）に使用する取引先IDを確認します。

## Tips

- `year_month` は `YYYY-MM` 形式（例: `2026-03`）
- `employees_scope` と絞り込みパラメータの対応:
  - `all`: 絞り込みなし（person_ids, team_ids は無効）
  - `team`: `team_ids[]` で絞り込み
  - `employee`: `person_ids[]` で絞り込み
  - 省略: ログインユーザのみ
- 配列パラメータ（`manager_ids[]`, `person_ids[]` 等）はクエリで複数指定可能
- 工数の `minutes` は分単位（1時間=60分、半日=240分）
- GET /projects のレスポンスが大きい場合は `offset` でページネーション。`total_count` で総数を確認
- 工数登録の並列実行: 複数の POST /workloads を同時に実行しても問題ない
- person_id と employee_id は異なる: PM API では person_id、HR API では employee_id を使用する
- GET /people や GET /teams で 401 が返る場合、システム管理者ロールが必要

### よくある操作フロー

工数登録:
1. `GET /users/me` → `company_id` を確認
2. `GET /projects` → アサインされているプロジェクト一覧を確認し `project_id` を取得
3. `POST /workloads` → 工数を登録

プロジェクト作成:
1. `GET /users/me` → `company_id` を確認
2. `GET /people` → アサインする従業員の `person_id` を確認
3. `GET /unit_costs` → 使用する単価マスタの `unit_cost_id` を確認
4. `GET /partners` → 発注元・発注先の `partner_id` を確認（任意）
5. `POST /projects` → プロジェクトを作成

チーム別工数集計:
1. `GET /teams` → チームIDを確認
2. `GET /workload_summaries` with `employees_scope: "team"`, `team_ids[]: [チームID]` → チームの工数サマリを取得

## 注意点

- GET /projects のレスポンスは1プロジェクトあたり数KBになることがある（メンバー一覧、タグ、発注先/発注元情報を含む）。プロジェクト数が多い事業所では limit=100 でも数MBに達する可能性があるため、必ず `operational_status` で絞り込むこと
- 工数は API 経由での更新・削除ができない（Web UI のみ）。まとめて登録する場合は、事前に日付・プロジェクト・分数の一覧をテーブル形式で提示し、ユーザーが内容を確認してから登録を実行する
- 誤って工数を登録した場合は https://pm.freee.co.jp の工数一覧画面から修正・削除する
- 同じ日・同じプロジェクトに複数回 POST すると加算される（重複チェックなし）
- プロジェクト更新・削除・ステータス変更、チームの作成・更新・削除、単価マスタや取引先の操作は API 非対応。Web UI（https://pm.freee.co.jp）で実施する
- 401 (GET /teams, /people): システム管理者ロールが必要。`GET /users/me` で role を確認
- 402: freee工数管理の有料プランが必要
- 403: 権限不足またはレート制限。制限の場合は10分待つ

詳細: `recipes/troubleshooting.md` 参照

## リファレンス

- `recipes/pm-workload-registration.md` - 勤怠データ連携による工数登録（日次・週次・月次）
- `references/pm-workloads.md` - 工数 API 詳細
- `references/pm-projects.md` - プロジェクト API 詳細
- `references/pm-people.md` - 従業員 API 詳細
- `references/pm-teams.md` - チーム API 詳細
- `references/pm-unit-costs.md` - 単価マスタ API 詳細
- `references/pm-partners.md` - 取引先 API 詳細
