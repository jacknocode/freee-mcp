# Unit Costs

## 概要

単価マスタ（freee工数管理）

## エンドポイント一覧

### GET /unit_costs

操作: 単価マスタの取得

説明: 従業員の単価マスタを返します。プロジェクトへのメンバーアサイン時に使用する実績単価の一覧です。

### パラメータ

| 名前 | 位置 | 必須 | 型 | 説明 |
|------|------|------|-----|------|
| company_id | query | はい | integer | 事業所ID |
| limit | query | いいえ | integer | 取得レコードの件数（デフォルト：50, 最小：1, 最大：100） |
| offset | query | いいえ | integer | 取得レコードのオフセット（デフォルト：0） |

### レスポンス (200)

成功時

- unit_costs (必須): array[object]
  配列の要素:
    - id: integer - 単価マスタID
    - name: string - 単価マスタ名
    - current_cost: integer - 取得時点での適用金額（円）
    - rules: array[object] - 期間ごとの適用金額の配列
      配列の要素:
        - from_date: string(date) - 適用開始日（YYYY-MM-DD）
        - thru_date: string(date) - 適用終了日（YYYY-MM-DD）
        - cost: integer - 適用期間の金額（円）
- meta (必須): object - ページネーション情報
  - current_offset: integer
  - next_offset: integer|null
  - prev_offset: integer|null
  - total_count: integer

## 参考情報

- freee API公式ドキュメント: https://developer.freee.co.jp/docs
- OpenAPIスキーマ: [pm-api-schema.json](../../openapi/pm-api-schema.json)
