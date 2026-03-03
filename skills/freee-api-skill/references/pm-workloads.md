# Workloads

## 概要

工数実績

## エンドポイント一覧

### POST /workloads

操作: 工数登録

説明: 工数を登録することが出来ます。

### リクエストボディ

Content-Type: application/json

| 名前 | 必須 | 型 | 説明 |
|------|------|-----|------|
| company_id | はい | integer | 事業所ID |
| project_id | はい | integer | 対象プロジェクトID |
| date | はい | string(date) | 対象日（YYYY-MM-DD） |
| minutes | はい | integer | 記録時間（分）（最小：1） |
| person_id | いいえ | integer | 対象従業員ID（管理者/チームリーダーのみ指定可、省略時はログインユーザ） |
| memo | いいえ | string | 業務内容（最大255文字） |
| workload_tags | いいえ | array[object] | 工数タグ |
| workload_tags[].tag_group_id | はい | integer | 工数タググループID |
| workload_tags[].tag_id | はい | integer | 工数タグID |

### レスポンス (200)

成功時

- workload (必須): object
  - id: integer - 工数実績ID
  - person_id: integer - 対象従業員のユーザーID
  - person_name: string - 対象従業員氏名
  - date: string(date) - 工数登録日（YYYY-MM-DD）
  - project_id: integer - 対象プロジェクトID
  - project_name: string - プロジェクト名
  - project_code: string - プロジェクトコード
  - memo: string - 業務内容
  - minutes: integer - 工数実績（分）
  - workload_tags: array[object] - 工数タグ
    配列の要素:
      - tag_group_id: integer - タググループのID
      - tag_group_name: string - タググループ名
      - tag_id: integer|null - タグのID
      - tag_name: string|null - タグ名

---

### GET /workloads

操作: 工数詳細の取得

説明: 取得対象の従業員の工数実績の詳細を返します。取得対象従業員と年月の取得範囲で絞り込みできます。

### パラメータ

| 名前 | 位置 | 必須 | 型 | 説明 |
|------|------|------|-----|------|
| company_id | query | はい | integer | 事業所ID |
| year_month | query | はい | string | 取得対象範囲（YYYY-MM） |
| employees_scope | query | いいえ | string | 取得対象従業員の検索スコープ (選択肢: all, team, employee)。allは全従業員、teamはチーム単位、employeeはperson_ids指定。省略時はログインユーザのみ |
| person_ids[] | query | いいえ | array[integer] | 取得対象従業員のユーザID（employees_scope=employee のときのみ有効） |
| team_ids[] | query | いいえ | array[integer] | 取得対象のチームID（employees_scope=team のときのみ有効） |
| limit | query | いいえ | integer | 取得レコードの件数（デフォルト：50, 最小：1, 最大：100） |
| offset | query | いいえ | integer | 取得レコードのオフセット（デフォルト：0） |

注意: employees_scope の組み合わせ
- `all`: person_ids, team_ids での絞り込み不可
- `team`: team_ids で絞り込み可、person_ids は不可
- `employee`: person_ids で絞り込み可、team_ids は不可
- 省略: ログインユーザの情報のみ

### レスポンス (200)

成功時

- workloads (必須): array[object]
  配列の要素（workload スキーマと同一）:
    - id: integer - 工数実績ID
    - person_id: integer - 対象従業員のユーザーID
    - person_name: string - 対象従業員氏名
    - date: string(date) - 工数登録日（YYYY-MM-DD）
    - project_id: integer - 対象プロジェクトID
    - project_name: string - プロジェクト名
    - project_code: string - プロジェクトコード
    - memo: string - 業務内容
    - minutes: integer - 工数実績（分）
    - workload_tags: array[object] - 工数タグ
- meta (必須): object - ページネーション情報
  - current_offset: integer
  - next_offset: integer|null
  - prev_offset: integer|null
  - total_count: integer

---

### GET /workload_summaries

操作: 工数実績の取得

説明: 取得対象の従業員の工数実績のサマリを返します。取得対象従業員と年月の取得範囲で絞り込みできます。

### パラメータ

| 名前 | 位置 | 必須 | 型 | 説明 |
|------|------|------|-----|------|
| company_id | query | はい | integer | 事業所ID |
| year_month | query | はい | string | 取得対象範囲（YYYY-MM） |
| employees_scope | query | いいえ | string | 取得対象従業員の検索スコープ (選択肢: all, team, employee) |
| person_ids[] | query | いいえ | array[integer] | 取得対象従業員のユーザID |
| team_ids[] | query | いいえ | array[integer] | 取得対象のチームID |
| limit | query | いいえ | integer | 取得レコードの件数（デフォルト：50, 最小：1, 最大：100） |
| offset | query | いいえ | integer | 取得レコードのオフセット（デフォルト：0） |

### レスポンス (200)

成功時

- workload_summaries (必須): array[object]
  配列の要素:
    - person_id: integer - 対象従業員のユーザーID
    - person_name: string - 対象従業員氏名
    - from_date: string - 工数登録期間from（YYYY-MM-DD）
    - thru_date: string - 工数登録期間to（YYYY-MM-DD）
    - minutes: integer - 工数実績（分）
    - productive_minutes: integer - 生産時間実績（分）
- meta (必須): object - ページネーション情報

## 参考情報

- freee API公式ドキュメント: https://developer.freee.co.jp/docs
- OpenAPIスキーマ: [pm-api-schema.json](../../openapi/pm-api-schema.json)
