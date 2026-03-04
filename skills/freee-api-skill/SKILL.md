---
name: freee-api-skill
description: "freee API を MCP 経由で操作するスキル。会計・人事労務・請求書・工数管理・販売の詳細APIリファレンスと使い方ガイドを提供。"
---

# freee API スキル

## 概要

[freee-mcp](https://www.npmjs.com/package/freee-mcp) (MCP サーバー) を通じて freee API と連携。

このスキルの役割:

- freee API の詳細リファレンスを提供
- freee-mcp 使用ガイドと API 呼び出し例を提供

注意: OAuth 認証はユーザー自身が自分の環境で実行する必要があります。

## セットアップ

### 1. OAuth 認証（あなたのターミナルで実行）

```bash
npx freee-mcp configure
```

ブラウザで freee にログインし、事業所を選択します。設定は `~/.config/freee-mcp/config.json` に保存されます。

### 2. プラグインをインストール

- Claude Code: コマンドパレット → "Claude: Install Plugin" → このリポジトリのパス
- Claude Desktop: 設定 → Plugins → Add Plugin → このリポジトリのパス

### 3. 再起動して確認

Claude を再起動後、`freee_auth_status` ツールで認証状態を確認。

## リファレンス

API リファレンスが `references/` に含まれます。各リファレンスにはパラメータ、リクエストボディ、レスポンスの詳細情報があります。

目的のAPIを探すには、`references/` ディレクトリ内のファイルをキーワード検索してください。

主なリファレンス:

- `accounting-deals.md` - 取引
- `accounting-expense-applications.md` - 経費申請
- `hr-employees.md` - 従業員情報
- `hr-attendances.md` - 勤怠
- `invoice-invoices.md` - 請求書
- `pm-projects.md` - プロジェクト
- `pm-workloads.md` - 工数実績
- `pm-users.md` - ログインユーザー（person_id、ロール確認）
- `pm-people.md` - 従業員（工数管理）
- `pm-teams.md` - チーム
- `pm-partners.md` - 取引先（工数管理）
- `pm-unit-costs.md` - 単価マスタ

## 使い方

### MCP ツール

認証・事業所管理:

- `freee_authenticate` - OAuth 認証
- `freee_auth_status` - 認証状態確認
- `freee_clear_auth` - 認証情報クリア
- `freee_current_user` - ログインユーザー情報取得
- `freee_list_companies` - 事業所一覧
- `freee_set_current_company` - 事業所切り替え
- `freee_get_current_company` - 現在の事業所取得

API 呼び出し:

- `freee_api_get` - GET リクエスト
- `freee_api_post` - POST リクエスト
- `freee_api_put` - PUT リクエスト
- `freee_api_delete` - DELETE リクエスト
- `freee_api_patch` - PATCH リクエスト
- `freee_api_list_paths` - 利用可能なAPIパス一覧

serviceパラメータ (必須):

| service | 説明 | パス例 |
|---------|------|--------|
| `accounting` | freee会計 (取引、勘定科目、取引先など) | `/api/1/deals` |
| `hr` | freee人事労務 (従業員、勤怠など) | `/api/v1/employees` |
| `invoice` | freee請求書 (請求書、見積書、納品書) | `/invoices` |
| `pm` | freee工数管理 (プロジェクト、工数など) | `/projects` |
| `sm` | freee販売 (見積、受注、売上など) | `/api/1/...` |

### company_id について

リクエストに `company_id` を含める場合、現在設定されている事業所（`freee_get_current_company` で確認可能）と一致している必要があります。不一致の場合はエラーになります。

- 事業所を変更する場合: 先に `freee_set_current_company` で切り替えてからリクエストを実行
- company_id を含まない API（例: `/api/1/companies`）: そのまま実行可能

### 基本ワークフロー

1. レシピを確認: `recipes/` 内の該当レシピを読む
2. リファレンスを検索: 必要に応じて `references/` を参照
3. API を呼び出す: `freee_api_*` ツールを使用

### レシピ

よくある操作のユースケースサンプルとTipsは以下を参照:

- `recipes/expense-application-operations.md` - 経費申請
- `recipes/deal-operations.md` - 取引（収入・支出）
- `recipes/hr-employee-operations.md` - 人事労務（従業員・給与）
- `recipes/hr-attendance-operations.md` - 勤怠（出退勤・打刻・休憩の登録）
- `recipes/invoice-operations.md` - 請求書・見積書・納品書
- `recipes/pm-operations.md` - 工数管理（プロジェクト・工数登録）
- `recipes/pm-workload-registration.md` - 工数登録（勤怠データ連携・クロスサービス）

### クロスサービスワークフロー

複数のfreeeサービスを組み合わせるワークフロー:

- 工数登録（PM + HR）: 人事労務の勤怠データを基に工数を登録
  → `recipes/pm-workload-registration.md`

クロスサービスで注意すべき点:
- 各サービスのIDは独立している（PM の person_id と HR の employee_id は異なる値）
- company_id はサービス間で共通とは限らない
- 各サービスでのロール権限が必要（例: PM API の GET /teams, GET /people はシステム管理者ロールが必要）

## エラー対応

- 認証エラー: `freee_auth_status` で確認 → `freee_clear_auth` → `freee_authenticate`
- 事業所エラー: `freee_list_companies` → `freee_set_current_company`
- PM API エラー: GET /teams, GET /people で 401 の場合はシステム管理者ロールが必要。`recipes/troubleshooting.md` 参照
- 詳細: `recipes/troubleshooting.md` 参照

## 対応 API

| service | ベースURL | パス形式 |
|---------|-----------|----------|
| `accounting` | `https://api.freee.co.jp` | `/api/1/...` |
| `hr` | `https://api.freee.co.jp/hr` | `/api/v1/...` |
| `invoice` | `https://api.freee.co.jp/iv` | `/invoices`, `/quotations`, `/delivery_slips` |
| `pm` | `https://api.freee.co.jp/pm` | `/projects`, `/workloads`, `/teams`, etc. |
| `sm` | `https://api.freee.co.jp/sm` | `/api/1/...` |

### 請求書 API について

請求書・見積書・納品書の操作については `recipes/invoice-operations.md` を参照してください。

注意: 会計 API の `/api/1/invoices` は過去の API であり、現在は請求書 API (`service: "invoice"`) を使用してください。

## 関連リンク

- [freee-mcp](https://www.npmjs.com/package/freee-mcp)
- [freee API ドキュメント](https://developer.freee.co.jp/docs)
