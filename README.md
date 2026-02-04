1. 概要  
  - 中小企業向けの売上データを、高負荷でも安定して非同期レポート化する Go 製バックエンド＋React 管理画面
  - 自社開発・中小システム会社が、自社 / お客様向けに導入する社内レポート基盤を想定
  - 技術スタック
    - 言語：Go 1.22系
    - Web：Echo
    - DB：PostgreSQL + GORM
    - インフラ：Docker compose
    - 非機能：テスト／ログ／CI
​
2. リポジトリ構成（ディレクトリ概要）  
backend/
cmd/api/ … REST API サーバ（認証・レポートジョブ受付・進捗／結果取得）
cmd/worker/ … ジョブ Worker（売上レポート生成処理）
internal/
domain/ … order, invoice, reportjob などドメインモデル
usecase/ … レポート生成ユースケース、ジョブ登録、状態更新
repository/ … PostgreSQL 向けの実装（orders, invoices, report_jobs, job_logs）
jobqueue/ … Asynq/River などのジョブキューラッパと設定
http/ … ハンドラ・ルーティング（Echo/Gin）
config/ … 設定読み込み
observability/ … ログ・メトリクス・トレース周り
frontend/
src/
pages/ … レポート一覧、レポート新規作成フォーム、詳細・進捗画面
components/ … テーブル、フィルタフォーム、進捗バー、チャートなど
api/ … fetch("/api/reports") などのクライアントラッパ
deploy/
docker-compose.yml … API＋Worker＋Postgres＋Redis＋(Prometheus/Grafana 任意)
migrations/ … DBマイグレーション
docs/
architecture.md … アーキテクチャ説明
sequence-report-generation.md … ジョブのシーケンス図
api-spec.md … API 一覧
metrics.md … 監視項目一覧

3. 機能概要（README 用の箇条書き）
ユーザ向け機能（管理画面＋API）
レポートテンプレート登録
「顧客別売上」「月次売上推移」「商品カテゴリ別売上」などのテンプレート定義。
レポート生成リクエスト
条件：期間、顧客・商品フィルタ、フォーマット（CSV / PDF）を指定して実行。
レポート進捗・状態表示
キュー待ち・処理中・完了・失敗を一覧／詳細で確認。
レポートダウンロード
完了したレポートファイルを管理画面・API からダウンロード。
通知設定（任意）
完了時にメール or Webhook で通知する設定。
運用者向け機能
ジョブ一覧・ステータス確認（API / 管理画面）
再実行／キャンセル操作（失敗ジョブのリトライ）
メトリクス確認（Prometheus エクスポート：ジョブ処理時間・失敗率など）
ヘルスチェックエンドポイント（API・Worker の稼働確認）
​
4. 主要テーブル設計（概要）
- customers
  - id, name, email, created_at, …
- products
  - id, name, category, unit_price, …
- orders
  - id, customer_id, order_date, total_amount, …
- order_items
  - id, order_id, product_id, quantity, unit_price, …
- report_jobs
  - id, user_id, report_type, status(queued/running/succeeded/failed/dead)、requested_at, started_at, finished_at, progress, error_code, error_message, output_path, output_format。
- job_logs
  - id, report_job_id, logged_at, level, message, metadata(jsonb)
- notifications（任意）
  - id, report_job_id, channel(email/webhook)、status, last_error, …

5. ジョブ状態遷移（概要）
README の図 or 箇条書き用イメージ。
QUEUED
API が report_jobs に挿入し、ジョブキューに enque。
RUNNING
Worker がジョブを取り出して処理開始。
SUCCEEDED
レポート生成成功、ファイル保存し output_path 更新。
FAILED
リトライ可能な失敗（ネットワーク等）で、規定回数までは自動リトライ。
DEAD
最大リトライ超過、または致命的エラーで手動対応が必要な状態。
progress は、
0–100（％）
「期間分割」の完了チャンク数などから算出。

6. 主な API エンドポイント（概要）
POST /api/reports
レポート生成ジョブを作成。ボディで期間・種別・フォーマットを指定。
GET /api/reports
自分が作成したレポートジョブの一覧・検索。
GET /api/reports/{id}
状態・進捗・エラー情報を取得。
GET /api/reports/{id}/download
完了済みレポートファイルのダウンロード。
POST /api/reports/{id}/retry
失敗ジョブの再実行。
POST /api/reports/{id}/cancel
実行中ジョブのキャンセル（context キャンセル＋キューキャンセル）。
GET /api/metrics（もしくは /metrics）
Prometheus 形式でメトリクス公開。
GET /api/health
API・DB・ジョブキューの簡易ヘルスチェック。

7. フロントエンド（React 管理画面）の概要
技術
React＋TypeScript＋任意のUIライブラリ（Material UI や Chakra など）

画面
レポート一覧画面
状態・種類・期間でフィルタ、進捗バー、ステータスバッジ。
レポート新規作成画面
日付範囲、レポート種別、フォーマット、通知設定を入力。
レポート詳細画面
状態、進捗、ログの抜粋、ダウンロードボタン。

8. README に書く「完成ライン」と拡張
v1 完成ライン
API＋Worker＋Postgres＋Redis／River構成
売上レポート（顧客別／月次）
ジョブ状態管理＋基本のリトライ
簡易 React 管理画面（一覧＋新規作成＋詳細＋ダウンロード）

v2 以降の拡張
メトリクス／トレースの拡充
通知チャンネル追加

新しいレポートテンプレート追加（カテゴリ別、LTV など）
