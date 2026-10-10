@AGENTS.md

# まちチケ

主催者がイベントと券種を登録し、参加者が Stripe Checkout でチケットを買うアプリ

## このプロジェクトの決まり

- 画面は Livewire 4 の単一ファイルコンポーネント（`resources/views/pages/**/⚡*.blade.php`）で作る。Volt は使わない
- UI は Flux UI の Free 版のコンポーネントだけを使う（Pro 専用は使わない）。無いものは Tailwind で作る
- 文言は日本語。日付は `isoFormat('M月D日(ddd) HH:mm')`、金額は「3,000円」の形式
- 認可は `app/Policies/` のポリシーで行い、画面側では `$this->authorize()` を使う
- 外部サービス（Stripe など）に触るコードは `app/Services/` に閉じ込める。テストではそのクラスをモックする
- 機能を追加したら `tests/Feature/` に Pest テストを追加し、`/check` が通ることを確認する
- 開発コマンドは Sail 経由（`sail artisan ...`、`sail composer ...`、`sail npm ...`）。ホストの php や composer は使わない
- ビューを追加・変更したら `sail npm run build` を実行する（Tailwind がクラスを拾うため）

## 作業の進め方

- 変更の前に、何をどのファイルに書くかを短く説明してから実装する
- 1回の指示で扱う範囲は1機能まで。大きい依頼は分割案を先に出す
- 既存のファイルの書き方に合わせる（同じ階層のファイルを先に読む）
- `.env` の値、API キー、パスワードはコードやコミットに含めない
- 失敗したコマンドは、原因の推測と対処案を1つずつ示してから再実行する

## 設計

詳細は [DB 設計](doc/design/database.md) と [画面一覧](doc/design/screens.md)。関係する実装の前に該当箇所を読む

### 用語
- イベント（events）: 主催者が登録する。`status` は `draft` / `published`。参加者に見せるのは `published` のみ
- 券種（ticket_types）: イベントごとの名前・価格・定員・販売期間
- 注文（orders）と明細（order_items）: 注文は `pending`（支払い待ち）→ `paid`（支払い済み）
- チケット（tickets）: `paid` になったとき枚数分を発行する。1枚ずつ固有の `code`（ULID）を持つ

### データの決まり
- ロールは `users.role`（customer / organizer / admin）。主催者は `events.organizer_id`、注文者は `orders.user_id`
- 金額は円の整数。注文後の金額は `order_items.unit_price`（注文時点の単価）を使い、券種の現在価格は見ない
- 残り枚数はカラムに持たず、`TicketType::remainingCount()` に集約する（`pending` と `paid` を数える）
- 削除: ユーザーと売れた券種は `restrict`、注文 → 明細 → チケットは `cascade`。購入済みイベントは削除不可（`EventPolicy` で先に判定）
- Stripe の Checkout Session ID は `orders.stripe_checkout_session_id` に保存し、Webhook で注文を特定する

### 画面の決まり
- 主催者の画面は `/organizer/events` 配下。自分のイベントだけ操作でき、他人のものは 403
- 購入者の画面は `/my/orders` 配下と支払い完了・中断ページ。注文した本人だけが見られる
- Day4 の範囲は C1〜C9 と C15。Stripe 連携・メール・マイページ・売上画面は平日課題
- 管理者画面と画像は拡張なので作らない
