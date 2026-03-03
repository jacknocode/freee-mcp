# People

## 概要

従業員（freee工数管理）

## エンドポイント一覧

### GET /people

操作: 従業員一覧の取得

説明: このリクエストで指定したIDの事業所の従業員一覧を返します。権限・ステータス・従業員IDで取得する情報を絞り込むことができます。

### パラメータ

| 名前 | 位置 | 必須 | 型 | 説明 |
|------|------|------|-----|------|
| company_id | query | はい | integer | 事業所ID |
| role | query | いいえ | string | 役割 |
| status | query | いいえ | string | ステータス (選択肢: sent=招待中, accepted=利用中, inactive=無効) |
| person_ids[] | query | いいえ | array[integer] | 従業員ID（複数指定可） |
| limit | query | いいえ | integer | 取得レコードの件数（デフォルト：50, 最小：1, 最大：100） |
| offset | query | いいえ | integer | 取得レコードのオフセット（デフォルト：0） |

### レスポンス (200)

成功時

- meta (必須): object - ページネーション情報
  - current_offset: integer
  - next_offset: integer|null
  - prev_offset: integer|null
  - total_count: integer
- people_counts (必須): object
  - total: integer - 取得件数合計
  - by_status: object
    - sent: integer - 招待済み従業員件数
    - accepted: integer - 利用中従業員件数
    - inactive: integer - 無効従業員件数
- people (必須): array[object]
  配列の要素:
    - id: integer - 従業員ID
    - status: string - ステータス（例: accepted）
    - name: string - 氏名
    - role: string - 事業所におけるロール
    - role_display_name: string - ロールの表示名（例: システム管理者）
    - unit_cost: object - 標準単価
      - id: integer - 標準単価ID
      - name: string - 名前
    - email: string - メールアドレス
    - payroll_employee_id: integer - 人事労務側従業員ID

## 参考情報

- freee API公式ドキュメント: https://developer.freee.co.jp/docs
- OpenAPIスキーマ: [pm-api-schema.json](../../openapi/pm-api-schema.json)
