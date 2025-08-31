# design-v1.md

## 1. 概要

本ドキュメントは、要件定義書に基づき、投票アプリケーション「WeVote」のシステム設計を定義するものである。

## 2. システム構成

本アプリケーションは、モダンなWebアプリケーション構成を採用する。

-   **フロントエンド**: シングルページアプリケーション（SPA）として実装し、レスポンシブデザインによりPC・スマートフォンの両デバイスに対応する。
-   **バックエンド**: フロントエンドからのリクエストを処理するAPIサーバーとして実装する。
-   **データベース**: システムのデータを永続化するためのリレーショナルデータベース（RDBMS）を使用する。
-   **認証**: GoogleのOAuth認証を利用する。

## 3. データベース設計 (ER図)

エンティティ間の関連を以下に示す。

```

[users] 1--\* [topics] 1--\* [votes] *--1 [users]
|
\`--* [votes]

```

### テーブル定義

#### 3.1. users (ユーザー)

| カラム名 | データ型 | 説明 | 制約 |
| :--- | :--- | :--- | :--- |
| `id` | `BIGINT` | ユーザーID | 主キー, 自動採番 |
| `google_id` | `VARCHAR(255)` | Googleアカウントの一意なID | NOT NULL, UNIQUE |
| `name` | `VARCHAR(30)` | ユーザー名（30文字以内）| NOT NULL |
| `icon_image_url` | `VARCHAR(255)` | アイコン画像のURL | |
| `is_suspended` | `BOOLEAN` | ログイン拒否フラグ（管理者用）| NOT NULL, DEFAULT `false` |
| `created_at` | `DATETIME` | 作成日時 | NOT NULL |
| `updated_at` | `DATETIME` | 更新日時 | NOT NULL |

#### 3.2. topics (議題)

| カラム名 | データ型 | 説明 | 制約 |
| :--- | :--- | :--- | :--- |
| `id` | `BIGINT` | 議題ID | 主キー, 自動採番 |
| `user_id` | `BIGINT` | 作成したユーザーのID | NOT NULL, 外部キー(users.id) |
| `title` | `VARCHAR(255)` | 議題のタイトル| NOT NULL |
| `description` | `TEXT` | 議題の説明文|
| `category` | `VARCHAR(50)` | カテゴリ（"恋愛", "政治"）| NOT NULL |
| `image_url_1` | `VARCHAR(255)` | 議題に添付する画像のURL1 |
| `image_url_2` | `VARCHAR(255)` | 議題に添付する画像のURL2 |
| `expires_at` | `DATETIME` | 投票受付終了日時（作成から半年後）| NOT NULL |
| `created_at` | `DATETIME` | 作成日時 | NOT NULL |
| `updated_at` | `DATETIME` | 更新日時 | NOT NULL |

#### 3.3. votes (投票)

| カラム名 | データ型 | 説明 | 制約 |
| :--- | :--- | :--- | :--- |
| `id` | `BIGINT` | 投票ID | 主キー, 自動採番 |
| `user_id` | `BIGINT` | 投票したユーザーのID | NOT NULL, 外部キー(users.id) |
| `topic_id` | `BIGINT` | 投票された議題のID | NOT NULL, 外部キー(topics.id) |
| `choice` | `BOOLEAN` | 選択（例: `true`=賛成, `false`=反対） | NOT NULL |
| `created_at` | `DATETIME` | 作成日時 | NOT NULL |
| `updated_at` | `DATETIME` | 更新日時 | NOT NULL |
| | | | **UNIQUE (`user_id`, `topic_id`)** |

*補足: `votes` テーブルの `UNIQUE` 制約により、ユーザーは1つの議題に1回しか投票記録を持てない。投票の変更は、このレコードの `choice` を更新することで実現する。*

## 4. API設計

### 4.1. ユーザー認証・管理 (`/api/users`)

-   **`GET /api/auth/google`**
    -   **説明**: Google認証ページへリダイレクトする。
-   **`GET /api/auth/google/callback`**
    -   **説明**: Googleからのコールバックを受け、認証処理を行い、セッションを確立する。
-   **`GET /api/users/me`**
    -   **説明**: 現在ログインしているユーザーの情報を取得する。
-   **`PUT /api/users/me`**
    -   **説明**: ログインユーザーのプロフィール（名前、アイコン画像）を更新する。アイコン画像は4MBまでのJPEG, PNG形式のみ許容。
    -   **リクエストボディ**: `{ "name": "新しい名前", "icon_image": (multipart/form-data) }`
-   **`DELETE /api/users/me`**
    -   **説明**: サービスから退会する。
-   **`POST /api/logout`**
    -   **説明**: ログアウトする。

### 4.2. 議題 (`/api/topics`)

-   **`GET /api/topics`**
    -   **説明**: 議題の一覧を取得する。
    -   **クエリパラメータ**:
        -   `category`: "恋愛" or "政治"
        -   `sort`: "new" (投稿日順) or "votes" (投票総数順)
        -   `page`: ページネーションのためのページ番号
-   **`POST /api/topics`**
    -   **説明**: 新しい議題を作成する。添付画像はそれぞれ4MBまでのJPEG, PNG形式のみ許容。
    -   **リクエストボディ**: `{ "title": "...", "description": "...", "category": "...", "option_yes_image": (multipart/form-data), "option_no_image": (multipart/form-data) }`
-   **`PUT /api/topics/:id`**
    -   **説明**: 議題を編集する。投稿後5分以内かつ投票数が0の場合のみ成功する。
-   **`DELETE /api/topics/:id`**
    -   **説明**: 自分が作成した議題を削除する。

### 4.3. 投票・結果 (`/api/topics/:id/`)

-   **`POST /api/topics/:id/vote`**
    -   **説明**: 議題に投票する。すでに投票済みの場合は、選択内容を更新する。
    -   **リクエストボディ**: `{ "choice": true }` (true: 賛成, false: 反対)
-   **`GET /api/topics/:id/result`**
    -   **説明**: 議題の投票結果を取得する。未投票のユーザーは、投票期限が切れるまでアクセスできない。
    -   **レスポンス例**: `{ "yes_votes": 100, "no_votes": 50, "yes_percentage": 66.7, "no_percentage": 33.3 }`

### 4.4. マイページ (`/api/mypage`)

-   **`GET /api/mypage/topics`**
    -   **説明**: 自分が作成した議題の一覧を取得する。
-   **`GET /api/mypage/votes`**
    -   **説明**: 自分が投票した議題の一覧を取得する。

### 4.5. 管理者機能 (`/api/admin`)

-   **`DELETE /api/admin/topics/:id`**
    -   **説明**: (管理者) 議題を強制的に削除する。
-   **`PUT /api/admin/users/:id/suspend`**
    -   **説明**: (管理者) 特定のユーザーをログイン拒否状態にする。

## 5. 画面設計

※議題詳細ページとコメント投稿機能は初期リリースでは実装しない。

-   **トップページ (議題一覧画面)**
    -   機能: 議題一覧表示、カテゴリでのフィルタリング、表示順序の切り替え、スクロールによる追加読み込み、議題作成ボタン（ログイン時）。
-   **議題作成画面**
    -   機能: タイトル・説明文の入力、画像添付、カテゴリ選択。
-   **マイページ**
    -   機能: プロフィール情報表示、プロフィール編集画面への導線、作成した議題一覧、投票した議題一覧のタブ切り替え。
-   **プロフィール編集画面**
    -   機能: ユーザー名とアイコン画像の編集、退会処理。

## 6. バッチ処理

-   **議題期限切れ処理バッチ**
    -   **実行タイミング**: 1日1回、深夜に実行。
    -   **処理内容**: `topics` テーブルをスキャンし、`expires_at` が現在時刻を過ぎている議題を検出する。該当する議題は、全ユーザーが結果を閲覧できる状態に更新する。

```