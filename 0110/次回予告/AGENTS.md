# AGENTS.md

## Purpose

このリポジトリでは AI（Codex）を **実装・レビュー専用のエージェント**として利用する。
**プロジェクトのセットアップ（初期構築）は AI に依頼しない。**

AI の利用範囲：

- 機能実装
- リファクタリング
- テスト追加
- 設定・コードの監査
- ドキュメント整備

---

## Communication Language（厳守）

- **やりとりはすべて日本語**
- 説明・質問・要約は日本語で行う
- コードコメントは、既存コードが英語の場合のみ英語可
- 指示がない限り英語で説明しない

---

## Stack（変更不可）

- Laravel **12**
- Vue **3**（`<script setup>`）
- Inertia.js **v2**
- Ziggy（フロントエンドのURL生成は `route()` を使用）
- Tailwind CSS **v3**
- CSP 有効（nonce / hash 運用、**unsafe-inline 禁止**）
- FileMaker **Data API** + **Eloquent-FileMaker**
  - DB は FileMaker
  - **マイグレーション不可**
  - FK 制約・JOIN・トランザクション前提は禁止

---

## Global Rules（Hard）

1. **セットアップ禁止**
   - 新規プロジェクト作成手順（composer create-project / npm init 等）を提示しない
   - 必要な場合は既存ファイルを要求し、**監査のみ**行う

2. **コマンド実行ルール**
   - artisan / npm / node / php / composer などのコマンド実行が必要な場合は、
     **必ずコンテナ内の実行環境**を前提とする
   - Laravel Sail を使用している場合は、`./vendor/bin/sail` 経由での実行を前提とする
   - ホストOS（ローカルMac）での直接実行を前提にした手順やコマンドを提示しない
   - 不明な場合は、実行環境（Sail / Docker Compose サービス名）を確認するために質問する

3. **バージョン混在禁止**
   - Laravel 12 / Inertia v2 / Vue 3 以外の書き方を混ぜない
   - Inertia v1、Vue 2、Options API の使用禁止

4. **CSP 遵守**
   - inline script / inline event（`onclick` 等）禁止
   - `eval`・動的 script 注入禁止
   - nonce は Spatie\\Csp が発行する仕組みを利用する
   - CSP のロジック（Middleware / Preset / Header 付与 / nonce 運用）は、AI コーディング開始前に **手動でセットアップ済み**とする
   - Codex はその既存実装に **必ず従う**（勝手に方式変更・置き換え・最適化をしない）
   - CSP 周りの変更が必要な場合は、変更案を出す前に **現状の実装ファイル**を要求して確認する

5. **Ziggy**
   - フロントエンドのURLは `route('name', params)` を使用
   - ハードコードされたパス（`'/users/1'` 等）禁止

6. **FileMaker 制約**
   - schema 変更前提の提案をしない
   - migration / FK / RDB 的最適化を前提にしない
   - API 呼び出し回数・パフォーマンスに配慮する

---

## Communication Style

- 日本語で簡潔・技術的に回答する
- **最小変更（surgical change）**を優先
- 不明点は推測せず、必要なファイルを指定して質問する
- 変更時は以下の順で説明する：
  1. 目的
  2. 影響範囲
  3. 修正箇所

---

## What Codex SHOULD do

- Controller / Service / Model / Vue Page / Component の実装
- 安全なリファクタリング（責務分離・重複削減）
- バリデーション・エラーハンドリング改善
- テスト追加（可能な範囲で）
- 設定ファイルの**監査**（CSP / Vite / Ziggy / Inertia）
- 要求があれば diff 形式で出力

---

## What Codex MUST NOT do

- 新規セットアップ手順の提示
- 明示されていないライブラリ追加
- 勝手なバージョンアップ提案
- 英語での長文説明
- プロジェクト方針の独自解釈

---

## Review Checklist

- [ ] Laravel 12 / Inertia v2 / Vue 3 の作法に一致している
- [ ] CSP 違反がない
- [ ] Ziggy の `route()` を使用している
- [ ] FileMaker 前提を破っていない
- [ ] 不要な変更が含まれていない
- [ ] エラーケースが考慮されている
- [ ] セキュリティ上の初歩的問題がない

---

## When context is required

以下のファイルを要求してよい：

- `composer.json`
- `package.json`
- `routes/web.php`
- `resources/js/app.ts|app.js`
- `vite.config.*`
- CSP 設定（Middleware / Config）
- Ziggy 設定
- Eloquent-FileMaker の Model / Connection 定義
