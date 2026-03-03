# Partners

## 概要

取引先（freee工数管理）

## エンドポイント一覧

### GET /partners

操作: 取引先の一覧取得

説明: 登録されている取引先の一覧を返します。プロジェクトの発注元・発注先として使用される取引先を管理します。

### パラメータ

| 名前 | 位置 | 必須 | 型 | 説明 |
|------|------|------|-----|------|
| company_id | query | はい | integer | 事業所ID |
| limit | query | いいえ | integer | 取得レコードの件数（デフォルト：50, 最小：1, 最大：100） |
| offset | query | いいえ | integer | 取得レコードのオフセット（デフォルト：0） |

### レスポンス (200)

成功時

- partners (必須): array[object]
  配列の要素:
    - id: integer - 取引先ID
    - name: string - 取引先名前
    - code: string|null - 取引先コード
    - projects_as_orderer: array[object] - 発注元として登録されているプロジェクト一覧
      配列の要素:
        - project_id: integer - プロジェクトID
        - project_name: string - プロジェクト名
    - projects_as_contractor: array[object] - 発注先として登録されているプロジェクト一覧
      配列の要素:
        - project_id: integer - プロジェクトID
        - project_name: string - プロジェクト名
- meta (必須): object - ページネーション情報
  - current_offset: integer
  - next_offset: integer|null
  - prev_offset: integer|null
  - total_count: integer

## 参考情報

- freee API公式ドキュメント: https://developer.freee.co.jp/docs
- OpenAPIスキーマ: [pm-api-schema.json](../../openapi/pm-api-schema.json)
