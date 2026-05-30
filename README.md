# 石灰散布記録アプリ — Supabase + Cloudflare Pages 移行手順

## ファイル構成（移行後）

```
lime_supabase/
├── index.html   ← これだけ！（PHPファイルは不要）
└── setup.sql    ← Supabase DB初期設定（初回のみ）
```

PHPファイル（config.php / records.php / settings.php / export.php）は
**すべて不要**になります。`index.html` 単体で動作します。

---

## 全体の流れ

```
Supabase（PostgreSQL DB + REST API）
         ↑ HTTPS通信
  index.html（ブラウザで動作するSPA）
         ↑ ホスティング
Cloudflare Pages（GitHub連携で自動デプロイ）
```

---

## STEP 1 — Supabase でデータベースを作る

### 1-1. アカウント作成
1. https://supabase.com にアクセス
2. 「Start your project」→ GitHubアカウントでサインアップ（無料）

### 1-2. プロジェクト作成
1. ダッシュボードで「New project」をクリック
2. 設定項目を入力：
   - **Name**（例: `saiboku-lime`）
   - **Database Password**（メモしておく）
   - **Region**：`Northeast Asia (Tokyo)` を選択
3. 「Create new project」→ 1〜2分待つ

### 1-3. テーブル初期化
1. 左メニュー「SQL Editor」をクリック
2. 「New query」をクリック
3. `setup.sql` の内容を全文貼り付けて「Run」をクリック
4. エラーなく完了すればOK

### 1-4. 接続情報を確認
1. 左メニュー「Project Settings」→「API」をクリック
2. 以下の2つをメモ：
   - **Project URL**（例: `https://abcdefg.supabase.co`）
   - **anon public**キー（`eyJ...` から始まる長い文字列）

---

## STEP 2 — `index.html` を編集する

`index.html` の先頭付近にある2行を自分の情報に書き換えます。

```javascript
// ★★★ ここを書き換える ★★★
const SUPABASE_URL     = 'https://abcdefg.supabase.co';     // ← Project URL
const SUPABASE_ANON_KEY = 'eyJhbGci...';                    // ← anon public キー
```

---

## STEP 3 — GitHub にリポジトリを作る

1. https://github.com にログイン（アカウントがなければ作成）
2. 右上「＋」→「New repository」
3. 設定：
   - **Repository name**（例: `saiboku-lime`）
   - **Private** を選択（公開したくない場合）
   - 「Create repository」をクリック
4. `index.html` をリポジトリにアップロード（「uploading an existing file」リンクから）

---

## STEP 4 — Cloudflare Pages でホスティング

1. https://pages.cloudflare.com にアクセスし、Cloudflareアカウントを作成（無料）
2. 「Create a project」→「Connect to Git」
3. GitHubアカウントと連携し、さきほどのリポジトリを選択
4. ビルド設定：
   - **Framework preset**：`None`
   - **Build command**：（空欄のまま）
   - **Build output directory**：`/`（またはそのまま）
5. 「Save and Deploy」をクリック
6. 数十秒でデプロイ完了、`https://xxx.pages.dev` のURLが発行される

---

## STEP 5 — 動作確認

1. 発行されたURLをブラウザで開く
2. 記録入力タブで部署・実施者を選んで「記録を保存」
3. 記録一覧タブに表示されれば成功

---

## 既存データの移行（InfinityFree → Supabase）

InfinityFreeのphpMyAdminで旧データをCSV/SQLエクスポートし、
Supabaseの「Table Editor」または「SQL Editor」でインポートできます。

### 手順
1. phpMyAdmin → 各テーブル（departments / staff / records / record_staff）を選択
2. 「エクスポート」→「CSV」形式でダウンロード
3. Supabase → Table Editor → 対象テーブル → 「Import data from CSV」

> ⚠ **id の連番（シーケンス）を更新する**  
> CSVインポート後、以下のSQLをSQL Editorで実行してください：
> ```sql
> SELECT setval('departments_id_seq',  (SELECT MAX(id) FROM departments));
> SELECT setval('staff_id_seq',        (SELECT MAX(id) FROM staff));
> SELECT setval('records_id_seq',      (SELECT MAX(id) FROM records));
> SELECT setval('record_staff_id_seq', (SELECT MAX(id) FROM record_staff));
> ```

---

## 変更点まとめ

| 項目 | 移行前（InfinityFree） | 移行後（Supabase + Cloudflare） |
|------|----------------------|-------------------------------|
| ホスティング | InfinityFree（PHP対応） | Cloudflare Pages（静的） |
| データベース | MySQL | PostgreSQL（Supabase） |
| バックエンド | PHP（records.php等） | Supabase REST API（自動生成） |
| デプロイ | FTPアップロード | GitHubプッシュで自動 |
| 費用 | 無料 | 無料（Supabase 500MB、CF Pages 無制限） |
| XLSX出力 | PHPで生成 | ※廃止（PDFとCSVは継続） |
| PDF出力 | PHPでHTML生成 | ブラウザの印刷機能を使用（同等） |

---

## セキュリティ補足

現在の設定は `anon` キーで全操作を許可しています。
URLを知っていれば誰でもデータを読み書きできます。

社内限定にするには、Supabaseの**Row Level Security**を活用し、
Supabase Authで認証を追加することをお勧めします。
詳細：https://supabase.com/docs/guides/auth
