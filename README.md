# Notion家計簿 Worker — セットアップ手順

ブラウザ(アプリ)とNotionの間に立つ「中継役」です。Notionのトークンはこの
Workerの中だけに保管され、ブラウザやGitHubには出ません。

---

## 0. 前提
- Cloudflareアカウント（無料）
- Node.js が入っていること（`node -v` で確認）
- このフォルダ（worker.js と wrangler.toml がある場所）でコマンドを実行します

---

## 1. Notion側の準備（まだなら）
1. https://www.notion.so/my-integrations で連携を作成（Internal）
2. トークン（`ntn_…`）を控える ※後で使う
3. 取引・月・カテゴリ・口座の4つのDBそれぞれで
   「…」→「コネクト」→ 作った連携を追加（これを忘れるとアクセス不可）

---

## 2. デプロイ（このフォルダで実行）

```bash
# 初回だけ：Cloudflareにログイン（ブラウザが開きます）
npx wrangler login

# Notionトークンを secret として登録（プロンプトに貼り付け。画面には残りません）
npx wrangler secret put NOTION_TOKEN

# 公開
npx wrangler deploy
```

デプロイが成功すると、こんなURLが表示されます：
`https://kakeibo.＜あなたのサブドメイン＞.workers.dev`
このURLが、アプリから叩く「自分のサーバー」になります。

---

## 3. 動作テスト（curl）

```bash
# あなたのWorkerのURLに置き換えてください
BASE="https://kakeibo.xxxx.workers.dev"

# (a) カテゴリ・口座の一覧が取れるか（プルダウン用）
curl "$BASE/options"

# (b) ある月の集計が取れるか
curl "$BASE/summary?month=2026-06"

# (c) 取引を1件追加（テスト）
curl -X POST "$BASE/add" \
  -H "Content-Type: application/json" \
  -d '{"date":"2026-06-02","amount":300,"category":"衣服","account":"楽天カード","memo":"テスト追加"}'
```

(c)を実行したらNotionの取引DBを見てください。
- 取引が1件増え、(C)月 が「2026-06」に自動で紐づく
- 月DBに「2026-06」ページが無ければ自動で作られ、予算10万が入る

うまくいかない時は、返ってくる `error` のメッセージを見てください。
よくある原因は「DBを連携に共有し忘れ」と「カテゴリ名の表記ゆれ」です。

---

## 4. APIの仕様（アプリから使う）

### GET /options
プルダウン用。カテゴリ（収支種類つき）と口座の一覧を返す。

### GET /summary?month=YYYY-MM
その月の集計。返り値の主なフィールド：
- `income` 収入合計
- `variableExpense` 変動費（予算の対象）
- `fixedExpense` 固定費（別枠）
- `budget` / `budgetRemaining` / `budgetUsedPct` 予算と残りと使用率
- `byCategory` カテゴリ別 { 名前: { amount, kind } }
- `byAccount` 口座別 { 名前: { out, in } }
- `transactions` 取引一覧

### POST /add
取引追加。body:
```json
{ "date":"2026-06-02", "amount":300, "category":"衣服", "account":"楽天カード", "memo":"シャツ" }
```
- カテゴリの「収支種類」は自動判定（収入/支出/固定費/移動/NMD）
- 日付から月を判定し、月ページを自動作成してリレーションを貼る

---

## 補足：収支種類の扱い
- 収入 → 収入に加算
- 支出 → 変動費（予算から引く）
- 固定費 → 別枠で集計（予算には含めない）
- 移動 / NMD → 集計から除外
