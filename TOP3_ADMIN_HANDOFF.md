# TOP3 管理画面 — Cloudflare 接続・公開引き継ぎ v1

更新: 2026-10-19

## 役割
- フロントエンド: `top3-admin-preview.html` (GitHub Pages 上の公開・非接続プレビュー)
- バックエンド: ヒロト担当の新規管理専用 Worker `top3-admin-api`
- DB: 既存 D1 `top3_orders_db` の `orders` テーブル
- 既存決済 Worker `cpaw-towel-checkout` と Stripe Webhook は変更しない
- 既存の注文2件を維持する

## 最重要: 公開・認証
- 公開 GitHub Pages の `top3-admin-preview.html` は UI デザイン確認専用。顧客データを読み込む JS、API 接続、秘密情報は追加しない。
- 正式な管理画面 HTML と API は Cloudflare Access 保護下の **同一オリジン** で配信する。
- 管理画面の HTML・静的ファイルと `/admin/api/*` の双方が Access 保護対象に含まれるよう構成する。
- Access ポリシーで TOP3 の許可された3人のメールアドレスに限定する。
- API の各リクエストで Access JWT の署名、issuer、audience、期限、許可メールをサーバー側で検証する。クライアント側のログイン表示だけで認証済みと判断しない。
- JWT 秘密情報、Cloudflare API token、Service Token、Stripe Secret Key はブラウザに渡さない。
- 個人情報を含むレスポンスは `Cache-Control: no-store`、必要に応じ `Pragma: no-cache`。
- Access 認証画面や未認証時のリダイレクトが API fetch に返る可能性を考慮し、フロント側は JSON 以外のレスポンスを安全に扱う。
- 同一オリジン構成のため、フロントは相対パス `/admin/api/...` を使う。CORS を不要にする。
- 個人情報をブラウザの localStorage/sessionStorage、URL、ログ、分析ツールに保存しない。
- ブラウザのキャッシュ・ブラウザ戻る時の個人情報表示も考慮する。

## API (ヒロト側実装)
- `GET /admin/api/me`: 管理者情報、権限確認
- `GET /admin/api/orders?page=1&limit=20`: 注文一覧、ページネーション
- `GET /admin/api/orders/{id}`: 注文詳細
- `PATCH /admin/api/orders/{id}/shipping`: `{"shipping_status":"shipped"}`
- `PATCH /admin/api/orders/{id}/tracking`: `{"carrier":"Japan Post","tracking_number":"TEST123456789"}`

一覧例:
```json
{
  "orders": [
    {
      "id": 1,
      "created_at": "2026-10-16 17:22:32",
      "product_name": null,
      "amount_total": 3080,
      "currency": "jpy",
      "payment_status": "paid",
      "shipping_status": "pending"
    }
  ],
  "pagination": {"page":1,"limit":20,"total":2,"total_pages":1}
}
```

注: `product_name`, `product_code`, `quantity`, `shipped_at` は既存15カラムにはない。Stripeの元データと照合できるまで推測で埋めない。フロントは null を「未登録」と表示する。

エラー形式: `{"error":{"code":"ORDER_NOT_FOUND","message":"Order not found"}}`。
HTTP: 400, 401, 403, 404, 409, 500。更新成功は 200 + 更新後データ。

## フロント接続の前提
1. ヒロトが正式な管理画面URL、Accessアプリ設定、APIレスポンス実例（個人情報は伏せる）を共有
2. ヒロト側で認証テスト（未ログイン401/Access遮断、許可外403、許可済200）と注文APIの動作確認
3. 管理画面を Cloudflare 側に配置。**公開プレビューとは別の配信物**として管理
4. フロントで `GET /me` → `GET /orders` → 詳細 → PATCH を段階的に接続
5. 画面からの発送操作は確認ダイアログ・二重送信防止・失敗時の再取得を実装
6. 更新競合 409 のときは最新データを再取得し、上書きを防止

## ヒロトから共有してほしい情報
- 正式な管理画面URL（ホスト名・配信パス）
- Access 保護範囲（画面と API 両方）
- `GET /admin/api/me` の成功レスポンス
- `GET /admin/api/orders` と詳細の個人情報を伏せた実レスポンス
- 更新APIの成功レスポンス、発送ステータスの許可遷移
- 追跡番号の検証ルール
- デプロイ方法（Worker静的アセット/Pages + Workerルーティングなど）
- Access ログアウト方法

## 安全な確認事項
- 公開 GitHub Pages から管理 API を呼ばない
- 認証済みでも管理者以外のメールでは注文データを返さない
- API は Stripe webhook と別ルートに置き、既存決済に影響させない
- D1 スキーマ変更前にはバックアップ/移行方針を確認
- 注文詳細の氏名・住所は一覧には出さず、詳細画面で必要時のみ取得
