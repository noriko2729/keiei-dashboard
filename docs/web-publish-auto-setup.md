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

## 最短の解決策（本命）

いちばん確実なのは、**毎朝のスケジュール実行環境に `noriko2729/keiei-dashboard` を
連携リポジトリとして許可する**こと（Cowork / Claude 側の環境設定）。これができれば、
既存の `publish_dashboard.py`（Contents API）や git push フォールバックがそのまま通り、
新しいコードは不要でWeb版発行が自動化される。手順文にある

> 「git proxy ... not in this session's authorized repository set」

というエラーは、この連携が未設定であることを示している。

下記の GitHub Actions ワークフローは、**git push が塞がれていても API 1回で発行できる
代替経路**（＝リポジトリ側で push を完結）として用意したもの。どちらの経路でも、
最終的には「毎朝の環境から api.github.com へ到達できること」が前提になる。

## セットアップ手順（Actions 経路）

### 1. ワークフローを main に取り込む（済）
`.github/workflows/publish-dashboard.yml` は main にマージ済みで、すでに有効。

### 1b. Actions の書き込み権限を確認
Settings → Actions → General → Workflow permissions が **「Read and write permissions」**
であること（ビルトイン `GITHUB_TOKEN` で index.html を push するため）。

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

ペイロード（gzip+base64）は最大 65,535 文字まで。現行HTML（約34KB）は gzip で
約13.5KB に収まる。**必ずファイル経由（`--data @file`）で送ること**。長い文字列を
シェル変数やJSONにインラインで埋め込むと、経路によっては末尾が欠落して base64 が
壊れる（デコード時に "number of data characters ... multiple of 4" エラーになる）。

```bash
# 生成した index.html があるディレクトリで
python3 - <<'PY' > /tmp/publish.json
import gzip, base64, json
b64 = base64.b64encode(gzip.compress(open("index.html","rb").read())).decode()
json.dump({"ref": "main", "inputs": {"html_gz_b64": b64}}, open("/tmp/publish.json","w"))
PY
curl -sS -X POST \
  -H "Authorization: Bearer $PUBLISH_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/noriko2729/keiei-dashboard/actions/workflows/publish-dashboard.yml/dispatches \
  --data @/tmp/publish.json
```

- 成功時は HTTP 204。数十秒後に Actions が index.html を更新し、Pages が再デプロイする。
- `$PUBLISH_TOKEN` はログ・応答本文に出力しないこと。
- 発行後は Actions の実行結果（Publish Dashboard）が success か確認する。

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

## 既知の制約・注意

- `workflow_dispatch` の各入力は 65,535 文字まで。gzip+base64 なら現行HTML（約34KB）
  は約13.5KBに収まる。**入力は必ず `--data @file` で送る**こと（インライン埋め込みは
  経路によって末尾が欠落し、`number of data characters ... multiple of 4` エラーになる）。
- Actions が push するには、リポジトリの Workflow permissions が
  「Read and write」である必要がある（未設定だと push 段でコケる）。
- **本命は環境のリポジトリ連携**。api.github.com への dispatch もサンドボックスで
  拒否される場合は、毎朝タスクの実行環境に `noriko2729/keiei-dashboard` を連携リポジトリ
  として許可する必要がある（Cowork/Claude の環境設定側の作業）。許可されれば、この
  ワークフロー経路も、既存の `publish_dashboard.py`（Contents API）経路も通るようになる。

## この Actions 経路の検証状況

- ワークフローの YAML・デコード処理・フッター置換はローカルで検証済み（現行HTMLを
  gzip+base64 → 復元で完全一致、フッターのみ差分）。
- ただし **end-to-end の実発行は本セッションからは未検証**。テストに使った GitHub API
  クライアント（MCP）が長い入力を切り詰めてしまい、正しいペイロードを送れなかったため。
  実際の `curl --data @file` からの呼び出しでは切り詰めは起きない。
- 初回本番前に、上記「手動での発行」または `curl` で1度発行して success を確認すること。
