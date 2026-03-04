# Projects

## 概要

プロジェクト

## API制限事項

freee工数管理 API ではプロジェクトの更新・削除エンドポイントは提供されていません。

| 操作 | API | 備考 |
|------|-----|------|
| 作成 | POST /projects | 可能 |
| 取得 | GET /projects, GET /projects/{id} | 可能 |
| 更新 | PUT/PATCH /projects/{id} | 非対応 → Web UIで操作 |
| 削除 | DELETE /projects/{id} | 非対応 → Web UIで操作 |

## エンドポイント一覧

### GET /projects

操作: プロジェクト一覧の取得

説明: この事業所のプロジェクトの一覧情報を返します。運用ステータス、マネージャー、発注先、発注元で絞り込みできます。

### パラメータ

| 名前 | 位置 | 必須 | 型 | 説明 |
|------|------|------|-----|------|
| company_id | query | はい | integer | 事業所ID |
| operational_status | query | いいえ | string | 運用ステータス (選択肢: planning, awaiting_approval, in_progress, rejected, done) |
| manager_ids[] | query | いいえ | array[integer] | マネージャのユーザID（複数指定可） |
| orderer_ids[] | query | いいえ | array[integer] | 発注元の取引先ID（複数指定可） |
| contractor_ids[] | query | いいえ | array[integer] | 発注先の取引先ID（複数指定可） |
| limit | query | いいえ | integer | 取得レコードの件数（デフォルト：50, 最小：1, 最大：100） |
| offset | query | いいえ | integer | 取得レコードのオフセット（デフォルト：0） |

注意: メンバーによるフィルタ

GET /projects にはメンバーIDによるフィルタパラメータがない。
自分がアサインされているプロジェクトを検索するには:
1. `operational_status` で絞り込んで取得
2. レスポンスの `members[].person_id` をクライアント側で照合

また、レスポンスサイズが大きくなりやすい（メンバー情報、タグ情報を含むため）。
limit はデフォルト50、最大100。offset でページネーション可能。

### レスポンス (200)

成功時

- meta (必須): object - ページネーション情報
  - current_offset: integer - リクエストのオフセット件数
  - next_offset: integer|null - 次ページのオフセット件数
  - prev_offset: integer|null - 前ページのオフセット件数
  - total_count: integer - 全レコード件数
- projects_counts (必須): object
  - total: integer - 取得件数合計
  - by_status: object
    - planning: integer - 計画中件数
    - awaiting_approval: integer - 承認待ち件数
    - in_progress: integer - 運用中件数
    - rejected: integer - 差し戻し件数
    - done: integer - 終了件数
- projects (必須): array[object]
  配列の要素:
    - id: integer - プロジェクトID
    - name: string - プロジェクト名
    - code: string - プロジェクトコード
    - description: string|null - プロジェクト概要
    - manager: object - プロジェクトマネージャー
      - person_id: integer - ユーザID
      - person_name: string - 氏名
    - color: string - カラー（例: #c8790f）
    - from_date: string - 期間from（YYYY-MM-DD）
    - thru_date: string - 期間to（YYYY-MM-DD）
    - publish_to_employee: boolean - 従業員への公開設定
    - assignment_url_enabled: boolean - 招待リンク
    - operational_status: string - 運用ステータス
    - sales_order_status: string - 受注ステータス
    - members: array[object] - プロジェクトメンバー
      配列の要素:
        - person_id: integer - ユーザID
        - person_name: string - 氏名
    - orderers: array[object] - 発注元
    - contractors: array[object] - 発注先
    - project_tags: array[object] - プロジェクトタグ
    - workload_tag_groups: array[object] - 工数タグのグループ

---

### POST /projects

操作: プロジェクトの登録

説明: プロジェクトを登録することができます。

### リクエストボディ

Content-Type: application/json

| 名前 | 必須 | 型 | 説明 |
|------|------|-----|------|
| company_id | はい | integer | 事業所ID |
| name | はい | string | プロジェクト名 |
| code | はい | string | プロジェクトコード |
| from_date | はい | string | プロジェクト開始日（YYYY-MM-DD） |
| thru_date | はい | string | プロジェクト終了日（YYYY-MM-DD） |
| pm_budgets_cost | はい | integer | プロジェクトマネージャーのコスト(円) |
| description | いいえ | string | プロジェクト概要 |
| publish_to_employee | いいえ | boolean | 従業員への公開設定 |
| assignment_url_enabled | いいえ | boolean | 招待リンク機能設定 |
| sales_order_status_id | いいえ | integer | 受注ステータスID |
| manager_person_id | いいえ | integer\|null | プロジェクトマネージャーの従業員ID（管理者/PMのみ指定可、省略時はログインユーザ） |
| color_id | いいえ | integer | 色ID (1:orange, 2:blue_green, 3:green, 4:blue, 5:purple, 6:red, 7:yellow) |
| members | いいえ | array[object]\|null | アサインするユーザの配列 |
| members[].person_id | はい | integer | 従業員ID |
| members[].unit_cost_id | はい | integer | このプロジェクトでの実績単価ID |
| members[].budgets_cost | はい | integer | 予算計算用の単価(円) |
| members[].use_standard_unit_cost | いいえ | boolean | 標準従業員単価の利用（デフォルト：false） |
| orderer_ids | いいえ | array[integer] | 発注元として指定する取引先IDの配列 |
| contractor_ids | いいえ | array[integer] | 発注先として指定する取引先IDの配列 |

### レスポンス (200)

成功時

- project (必須): object - 登録されたプロジェクトの詳細（project_detail スキーマ）
  - id: integer - プロジェクトID
  - name: string - プロジェクト名
  - code: string - プロジェクトコード
  - description: string|null - プロジェクト概要
  - manager: object - マネージャー情報
  - color: string - カラー
  - from_date: string - 期間from
  - thru_date: string - 期間to
  - publish_to_employee: boolean
  - assignment_url_enabled: boolean
  - operational_status: string
  - sales_order_status: string
  - members: array[object]
  - orderers: array[object]
  - contractors: array[object]
  - balance: object - 収支管理詳細（予算・実績）

---

### GET /projects/{id}

操作: プロジェクト詳細の取得

説明: IDに該当するプロジェクトの詳細情報を返します。

### パラメータ

| 名前 | 位置 | 必須 | 型 | 説明 |
|------|------|------|-----|------|
| company_id | query | はい | integer | 事業所ID |
| id | path | はい | integer | プロジェクトID |

### レスポンス (200)

成功時

- project (必須): object - プロジェクト詳細（project_detail スキーマと同一）
  - id: integer
  - name: string
  - code: string
  - description: string|null
  - manager: object
  - color: string
  - from_date: string
  - thru_date: string
  - publish_to_employee: boolean
  - assignment_url_enabled: boolean
  - operational_status: string
  - sales_order_status: string
  - members: array[object]
  - orderers: array[object]
  - contractors: array[object]
  - balance: object - 収支管理詳細
    - labor_costs: object
      - total: object
        - budget: integer - 予算
        - actual_result: integer - 実績
      - monthly: array[object] - 月別
        - year_month: string - 年月（YYYY-MM）
        - budget: integer
        - actual_result: integer
      - details: array[object] - メンバー別内訳

## 参考情報

- freee API公式ドキュメント: https://developer.freee.co.jp/docs
- OpenAPIスキーマ: [pm-api-schema.json](../../openapi/pm-api-schema.json)
