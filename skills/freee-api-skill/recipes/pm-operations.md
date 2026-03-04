# freee工数管理 操作ガイド

freee工数管理 API を使ったよくある操作のサンプルと Tips。

## 前提

- `service: "pm"` を指定する
- ベースURL: `https://api.freee.co.jp/pm`
- パス形式: `/projects`, `/workloads` など（`/api/1/` プレフィックスは不要）
- `company_id` は `GET /users/me` で取得した値を使用

## 1. ログインユーザーと事業所IDの確認

```
freee_api_get
  service: "pm"
  path: "/users/me"
```

レスポンスの `companies[].id` が各APIで必要な `company_id` です。

## 2. プロジェクト一覧の取得

```
freee_api_get
  service: "pm"
  path: "/projects"
  query:
    company_id: 12345
    operational_status: "in_progress"   # 運用中のみ絞り込み（省略可）
    limit: 50
    offset: 0
```

`operational_status` の選択肢: `planning`, `awaiting_approval`, `in_progress`, `rejected`, `done`

## 3. プロジェクト詳細の取得

```
freee_api_get
  service: "pm"
  path: "/projects/601"    # {id} にプロジェクトIDを指定
  query:
    company_id: 12345
```

詳細には収支管理情報（`balance`）が含まれます。

## 4. プロジェクトの作成

```
freee_api_post
  service: "pm"
  path: "/projects"
  body:
    company_id: 12345
    name: "新規プロジェクト"
    code: "PROJ-001"
    from_date: "2026-04-01"
    thru_date: "2026-09-30"
    pm_budgets_cost: 500000
    description: "プロジェクトの概要説明"
    publish_to_employee: true
    color_id: 3               # 3=green
    manager_person_id: 10     # 省略時はログインユーザ
    members:
      - person_id: 11
        unit_cost_id: 3
        budgets_cost: 3000
      - person_id: 12
        unit_cost_id: 3
        budgets_cost: 2500
```

色ID: `1=orange, 2=blue_green, 3=green, 4=blue, 5=purple, 6=red, 7=yellow`

## 5. 工数の登録

```
freee_api_post
  service: "pm"
  path: "/workloads"
  body:
    company_id: 12345
    project_id: 601
    date: "2026-03-03"
    minutes: 120              # 2時間 = 120分
    memo: "機能実装"
    # person_id は省略するとログインユーザに登録
```

工数タグが必要な場合:

```
freee_api_post
  service: "pm"
  path: "/workloads"
  body:
    company_id: 12345
    project_id: 601
    date: "2026-03-03"
    minutes: 60
    memo: "コードレビュー"
    workload_tags:
      - tag_group_id: 11
        tag_id: 12
```

注意: プロジェクトに工数タグが必須設定されている場合は `workload_tags` が必要です。

## 6. 工数実績の確認（詳細）

```
freee_api_get
  service: "pm"
  path: "/workloads"
  query:
    company_id: 12345
    year_month: "2026-03"
    employees_scope: "employee"
    person_ids[]: [10, 11]    # 特定メンバーを指定
```

全員の工数を取得する場合:

```
freee_api_get
  service: "pm"
  path: "/workloads"
  query:
    company_id: 12345
    year_month: "2026-03"
    employees_scope: "all"
```

## 7. 工数サマリの確認

```
freee_api_get
  service: "pm"
  path: "/workload_summaries"
  query:
    company_id: 12345
    year_month: "2026-03"
    employees_scope: "all"
```

サマリには `minutes`（総工数）と `productive_minutes`（生産時間）が含まれます。

## 8. チーム一覧の取得

```
freee_api_get
  service: "pm"
  path: "/teams"
  query:
    company_id: 12345
```

チームメンバーとリーダー情報も含まれます。

## 9. 従業員一覧の取得

```
freee_api_get
  service: "pm"
  path: "/people"
  query:
    company_id: 12345
    status: "accepted"    # 利用中のみ
```

プロジェクトのメンバーアサイン時に `person_id` を確認するために使用します。

## 10. 単価マスタの確認

```
freee_api_get
  service: "pm"
  path: "/unit_costs"
  query:
    company_id: 12345
```

プロジェクト作成時の `members[].unit_cost_id` に使用する単価ID一覧を取得できます。

## 11. 取引先一覧の取得

```
freee_api_get
  service: "pm"
  path: "/partners"
  query:
    company_id: 12345
```

プロジェクトの発注元（`orderer_ids`）・発注先（`contractor_ids`）に使用する取引先IDを確認します。

## API非対応操作（Web UIで実施）

freee工数管理 API では以下の操作は提供されていません。Web UIから手動で操作してください。

| リソース | 非対応操作 |
|---------|-----------|
| プロジェクト | 更新（名前/期間/メンバー変更）、削除、ステータス変更 |
| 工数 | 修正、削除 |
| チーム | 作成、更新、削除 |
| 従業員 | 作成、更新、削除（freee人事労務APIを使用） |
| 単価マスタ | 作成、更新、削除 |
| 取引先 | 作成、更新、削除 |

これらの操作をリクエストされた場合は、Web UI（https://pm.freee.co.jp）で操作するよう案内してください。

## よくある操作フロー

### 工数登録フロー

1. `GET /users/me` → `company_id` を確認
2. `GET /projects` → アサインされているプロジェクト一覧を確認し `project_id` を取得
3. `POST /workloads` → 工数を登録

### プロジェクト作成フロー

1. `GET /users/me` → `company_id` を確認
2. `GET /people` → アサインする従業員の `person_id` を確認
3. `GET /unit_costs` → 使用する単価マスタの `unit_cost_id` を確認
4. `GET /partners` → 発注元・発注先の `partner_id` を確認（任意）
5. `POST /projects` → プロジェクトを作成

### チーム別工数集計フロー

1. `GET /teams` → チームIDを確認
2. `GET /workload_summaries` with `employees_scope: "team"`, `team_ids[]: [チームID]` → チームの工数サマリを取得

## Tips

- `year_month` は `YYYY-MM` 形式（例: `2026-03`）
- `employees_scope` と絞り込みパラメータの対応:
  - `all`: 絞り込みなし（person_ids, team_ids は無効）
  - `team`: `team_ids[]` で絞り込み
  - `employee`: `person_ids[]` で絞り込み
  - 省略: ログインユーザのみ
- 配列パラメータ（`manager_ids[]`, `person_ids[]` 等）はクエリで複数指定可能
- 工数の `minutes` は分単位（1時間=60分、半日=240分）

## エラー対応

| ステータス | 原因 | 対処 |
|-----------|------|------|
| 400 | リクエスト不正 | パラメータを確認（必須項目、型、値の範囲） |
| 401 | 認証エラー | `freee_auth_status` で認証状態確認 |
| 402 | 有料プランが必要 | freee工数管理のプランを確認 |
| 403 | 権限不足またはレート制限 | ロールを確認。制限の場合は10分待つ |
| 404 | リソースが見つからない | IDを確認 |

詳細: `recipes/troubleshooting.md` 参照
