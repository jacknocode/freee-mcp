# Teams

## 概要

チーム（freee工数管理）

## エンドポイント一覧

### GET /teams

操作: チームの一覧取得

説明: 登録されているチームの一覧を返します。

### パラメータ

| 名前 | 位置 | 必須 | 型 | 説明 |
|------|------|------|-----|------|
| company_id | query | はい | integer | 事業所ID |
| limit | query | いいえ | integer | 取得レコードの件数（デフォルト：50, 最小：1, 最大：100） |
| offset | query | いいえ | integer | 取得レコードのオフセット（デフォルト：0） |

### レスポンス (200)

成功時

- teams (必須): array[object]
  配列の要素:
    - id (必須): integer - チームID
    - name (必須): string - チーム名
    - memo: string|null - メモ
    - member_count (必須): integer - メンバー数
    - members: array[object] - チームに登録されているメンバーの配列
      配列の要素:
        - person_id (必須): integer - チームに所属している従業員ID
        - person_name (必須): string - チームに所属している従業員名
        - is_leader (必須): boolean - リーダフラグ（リーダならばtrue）
- meta (必須): object - ページネーション情報
  - current_offset: integer
  - next_offset: integer|null
  - prev_offset: integer|null
  - total_count: integer

## 参考情報

- freee API公式ドキュメント: https://developer.freee.co.jp/docs
- OpenAPIスキーマ: [pm-api-schema.json](../../openapi/pm-api-schema.json)
