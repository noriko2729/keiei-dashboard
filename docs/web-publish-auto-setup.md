# Web版発行の自動化セットアップ

経営ダッシュボード（`index.html`）の「Web版発行」を、毎朝の生成タスクから**手動なしで**
公開できるようにするためのセットアップ手順と仕組みの説明です。

## 背景・現状

- 毎平日 朝7:00(JST)、プロジェクト「タスクを完了させる仕組み」のクラウドセッションが
  Google Calendar / Gmail / ChatWork の実データからダッシュボードHTMLを生成する。
- 「Web版発行」とは、生成した HTML をこのリポジトリ `noriko2729/keiei-dashboard` に
  `index.html` として反映すること。GitHub Pages（main ブランチのルート配信）が公開する。
- これまで発行だけが「手動」だった。原因は **毎朝のクラウドセッションのサンドボックスが、
  このリポジトリへの直接 push / Contents API 書き込みを許可していない**ため
  （エラー: `not in this session's authorized repository set`）。

## 解決アプローチ

push そのものを **GitHub Actions 側（リポジトリ内の権限で動く）に移す**ことで、
クラウドセッションのサンドボックス制限を回避する。

```
毎朝の生成セッション                GitHub                        閲覧者
─────────────────      ──────────────────────     ─────────
HTMLを生成
   │ gzip + base64
   │ workflow_dispatch (api.github.com へ POST 1回)
   ▼
                    Publish Dashboard ワークフロー
                       │ 入力をデコード→index.html
                       │ フッターを「自動」に更新
                       │ GITHUB_TOKEN で main に commit & push
                       ▼
                    GitHub Pages が自動デプロイ  ───────────▶  最新のダッシュボード
```

呼び出し側が必要とするのは **api.github.com への dispatch 1回だけ**。
実際の commit / push は Actions のビルトイン `GITHUB_TOKEN`（`contents: write`）で
完結するので、リポジトリ用の個人トークンを push に使う必要がなくなる。

## セットアップ手順

### 1. このブランチを main にマージする
`.github/workflows/publish-dashboard.yml` を main に取り込むと、ワークフローが有効になる。

### 2. GitHub Pages の確認（変更不要）
Settings → Pages が「Deploy from a branch: main / (root)」であること。
現在の配信方式を維持するため、本セットアップでは Pages の設定は変更しない。

### 3. 発行トークンの用意（推奨: Fine-grained PAT）
毎朝のタスクがワークフローを起動するためのトークン。対象リポジトリのみに絞った
Fine-grained personal access token を発行し、権限は以下だけを付与する。
- Repository access: `noriko2729/keiei-dashboard` のみ
- Permissions: **Actions = Read and write**, **Contents = Read**（Metadata は自動）

> ⚠️ セキュリティ: 現在スケジュールタスクの手順文（プロンプト）に
> クラシックPATが**平文で埋め込まれている**。これは漏えい時の影響が大きいので、
> 上記の Fine-grained PAT を新規発行して差し替え、旧トークンは失効（revoke）することを強く推奨。

### 4. 毎朝のタスク（手順7 = Web版発行）を差し替える
生成HTMLを保存したフォルダで、次の要領で発行する。

```bash
# 生成した index.html があるディレクトリで
PAYLOAD=$(gzip -c index.html | base64 -w0)
curl -sS -X POST \
  -H "Authorization: Bearer $PUBLISH_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/noriko2729/keiei-dashboard/actions/workflows/publish-dashboard.yml/dispatches \
  -d "{\"ref\":\"main\",\"inputs\":{\"html_gz_b64\":\"$PAYLOAD\"}}"
```

- 成功時は HTTP 204。数十秒後に Actions が index.html を更新し、Pages が再デプロイする。
- `$PUBLISH_TOKEN` はログ・応答本文に出力しないこと。

## 手動での発行（フォールバック / 動作確認）

ブラウザからも発行できる（データ差し替えは伴わず、既存 index.html の再デプロイ確認向け）。
1. GitHub → Actions → **Publish Dashboard** → **Run workflow**
2. `html_gz_b64` または `html_b64` に、発行したい HTML をエンコードして貼り付け → 実行

ローカルにHTMLがある場合の git 直接発行（authorized な環境でのみ可能）:
```bash
git clone https://github.com/noriko2729/keiei-dashboard.git
cp /path/to/generated.html keiei-dashboard/index.html
cd keiei-dashboard
git commit -am "ダッシュボード更新（手動発行）$(date +%F)"
git push
```

## 仕組みの詳細（ワークフロー）

`.github/workflows/publish-dashboard.yml`:
- トリガー: `workflow_dispatch`（入力 `html_gz_b64` または `html_b64`）
- 入力をデコードして `index.html` を作成。**空・`</html>` 無し**の壊れた入力では
  発行を中止し、既存の公開版を保護する。
- フッターの `Web版発行: <日付> 手動` を `Web版発行: <当日(JST)> 自動` に自動更新。
- 変更が無ければコミットをスキップ（無駄な発行を避ける）。
- `GITHUB_TOKEN` で main に push → Pages が自動デプロイ。

## 既知の制約

- `workflow_dispatch` の各入力は 65,535 文字まで。gzip+base64 を使えば現行HTML
  （約34KB）は十分収まる。将来HTMLが大きくなっても gzip 圧縮で余裕がある。
- 万一 api.github.com への dispatch もサンドボックスで拒否される場合は、毎朝タスクの
  実行環境に `noriko2729/keiei-dashboard` を**連携リポジトリとして許可**する必要がある
  （Cowork/Claude の環境設定側の作業）。その場合は上記スクリプトがそのまま通るようになる。
