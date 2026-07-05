# PLAN.md — multi_app 作業メモ

ここまでの議論・決定事項・作業ログを網羅的に記録するメモ。README.md はリポジトリの「現在の姿」を示すのに対し、このファイルは「経緯と判断理由」を残す。

## 1. リポジトリの目的

- 複数プロダクトを持つ SaaS 企業のようなリポジトリを**練習のために**作る
- 複数プロダクトは互いに関連がある（マイクロサービス構成と呼べるもの）
- フロントエンド: Next.js / バックエンド: Go（フロントとバックは分離）

## 2. アーキテクチャの決定

### 2 層構成を採用（決定）

「1 プロダクトにつき 1 API が対応する」2 層構成を採用した。

```
product-a/web ──→ product-a/api ──┐
                                  │ サービス間通信（内部ネットワーク）
product-b/web ──→ product-b/api ←─┘
```

- **規律**: フロントエンドは自プロダクトの API のみを呼ぶ。他プロダクトのデータが必要な場合は、バックエンド同士（api → api）で通信して集約する。各プロダクトの API が BFF の役割を兼ねる
- **比較した代替案**: 3 層構成（web → bff → 複数のドメインサービス）。プロダクト間で共有するドメイン（顧客・認証など）を `services/` に切り出す形で、依存が一方向に整理される利点がある。実際の複数プロダクト SaaS 企業の構成に近い
- **判断**: 3 層も検討したが（推奨もされたが）、まずはシンプルな 2 層で始めることにした。共有したいドメインが実際に見えてきたら `services/` に切り出す進化パスがある

### サービス間通信はインターネットに出ない

api 同士の通信は VPC / クラスタ内部のプライベートネットワークで完結させる（k8s の ClusterIP、docker compose のサービス名解決など）。外部公開するのはロードバランサ / API Gateway だけ。公開 URL を内部サービスから呼ぶのはアンチパターン。

## 3. 目指すディレクトリ構成（全体像）

```
multi_app/
├── apps/                        # デプロイ単位（プロダクトごと）
│   ├── product-a/
│   │   ├── web/                 # Next.js フロントエンド
│   │   └── api/                 # Go バックエンド
│   └── product-b/
│       ├── web/
│       └── api/
├── services/                    # プロダクト横断の共通サービス（auth, notification など）
├── packages/                    # フロントエンド共有コード（ui, config, api-client）
├── pkg/                         # Go の共有ライブラリ（logger, middleware, database）
├── proto/                       # API 定義（gRPC/OpenAPI）
├── infra/                       # インフラ関連（docker, k8s）
├── mise.toml                    # ツールのバージョン管理
├── go.work                      # Go workspace
├── package.json                 # npm workspace のルート
├── pnpm-workspace.yaml          # pnpm workspace 設定
└── Makefile                     # 横断的なタスクランナー
```

ディレクトリは必要になったタイミングで順次作成していく方針。
プロダクト名が product-a / product-b のままになるかは未定。

## 4. ツールチェーンの決定

### mise（決定・導入済み）

- このリポジトリで使うコマンド・ツールのバージョン管理は mise で行う
- `mise.toml` で node 24.18.0（LTS 系）/ pnpm 11.9.0 を固定
- 導入時のトラブル: mise 2026.2.8 のレジストリが pnpm 11.9 系の新しいアセット名（`pnpm-darwin-arm64.tar.gz`）に未対応でインストール失敗 → mise を 2026.7.0 に更新して解決

### pnpm workspace（決定・導入済み）

- ルートの `package.json` は `private: true` のオーケストレーション用。各パッケージ（`apps/*/web`, `packages/*`）が自分の `package.json` を持つ
- ロックファイルはルートの `pnpm-lock.yaml` に一元化
- **catalog 機能を採用**: 依存バージョンは `pnpm-workspace.yaml` の `catalog:` に一元定義し、各 package.json は `"catalog:"` で参照する。複数パッケージ間でバージョンが自動的に揃う
- バージョン指定の方針: next / react / react-dom は完全指定（組み合わせの整合性が重要なため）、typescript / @types/* などツール系は `^` レンジ
- `@types/node` は create-next-app 生成時の `^20` から、mise の node 24 に合わせて `^24` に変更した
- `allowBuilds`: pnpm 11 の形式で `sharp: false` / `unrs-resolver: false`（ビルド済みバイナリが配布されるためスクリプト実行不要）。create-next-app が生成した旧形式 `ignoredBuiltDependencies` は pnpm 11 では効かなかったため置き換えた

### Next.js（決定・導入済み）

- Turbopack を使う（Next.js 16 では標準）
- lint / format は ESLint / Prettier ではなく **oxlint / oxfmt** を使う（未導入・次のステップ）
  - oxlint: 安定版。Next.js 専用ルール（`plugins: ["nextjs"]`）を組み込みサポート
  - oxfmt: ベータ（v0.5x）だが Prettier の JS/TS 準拠テスト 100% パス。練習用途ではリスクなしと判断
  - Turbopack はバンドラ、oxlint / oxfmt は静的解析・整形でレイヤーが独立しており競合しない
  - `next lint` は Next.js 16 で削除済みのため、linter は自由に選べる

### create-next-app の実行内容（実施済み）

```sh
mise exec -- pnpm create next-app@latest apps/product-a/web \
  --ts --app --src-dir --tailwind --no-eslint --turbopack \
  --import-alias "@/*" --use-pnpm --skip-install --disable-git --yes
```

- 採用オプション: TypeScript / App Router / src ディレクトリあり / Tailwind CSS あり / ESLint なし / Turbopack / インポートエイリアス `@/*`
- `--skip-install` にして、ルートでの `pnpm install` でロックファイルを一元化した
- 生成後の後処理:
  - `web/pnpm-workspace.yaml`（ネスト）は中身をルートに統合して削除（残すと web/ が独立 workspace と誤認される）
  - パッケージ名を `web` → `@product-a/web` に変更（将来の product-b/web との名前衝突を回避）
  - 依存バージョンを catalog 参照に書き換え
- 事前にプレースホルダの `web/README.md` を削除した（create-next-app は対象ディレクトリに許可リスト外のファイルがあると失敗するため）

## 5. 作業ログ（時系列）

1. `apps/product-a/web` / `apps/product-a/api` のディレクトリ作成（プレースホルダ README 付き）
2. ルート README.md 作成（全体構成と進捗チェックリストを記載）
3. `mise.toml` 作成、node 24.18.0 / pnpm 11.9.0 をインストール（mise 更新のトラブルシュート含む）
4. ルート `package.json` / `pnpm-workspace.yaml` 作成
5. create-next-app で `apps/product-a/web` をスキャフォールド、後処理（上記）
6. catalog 化、`@types/node` を `^24` へ
7. `pnpm install` 成功。workspace 2 プロジェクト（ルート + @product-a/web)認識、ロックファイルはルートのみ
8. dev サーバ起動確認（Next.js 16.2.10 / Turbopack、Ready in 192ms、ブラウザで表示確認済み)

## 6. 次のステップ

- [ ] `apps/product-a/web` に oxlint / oxfmt を導入
  - devDependencies はルートの package.json に置く想定
  - `.oxlintrc.json` に `plugins: ["nextjs"]` を設定
  - `lint` / `format` scripts の整備
- [ ] `apps/product-a/api` の Go モジュール初期化
  - モジュールパスは `github.com/chillout2san/multi_app/apps/product-a/api` のような形を想定（要確認）
  - `go.work` の設置（マルチモジュール + go workspace 方式）
- [ ] その後の候補: Makefile（または mise tasks）整備、docker compose での起動、product-b の追加、`services/` / `packages/` / `proto/` の整備

## 7. 学んだこと・議論したことのメモ

- **スキャフォールド**: プロジェクトの初期ファイル一式を自動生成すること。`pnpm create next-app` の `create` がそれ。Go には公式ジェネレータがないので api 側は手で作る
- **pnpm の node_modules 構造**: ルートの `node_modules/.pnpm`（virtual store）に全パッケージの実体、各パッケージの `node_modules` には宣言した依存へのシンボリックリンクのみ。幽霊依存（宣言していない依存の import）を構造的に防ぐ
- **semver レンジ**: `16.2.10` はピン留め、`^4` はメジャーを跨がない範囲で許容。実際に入るバージョンは `pnpm-lock.yaml` が確定させる。レンジ内で上がるのは `pnpm update` 実行時のみ
- **pnpm のビルドスクリプト制御**: pnpm v10+ は依存のライフサイクルスクリプトをデフォルト実行しない（サプライチェーン攻撃対策）。pnpm 11 では `allowBuilds` でパッケージごとに true / false を明示する
- **AGENTS.md / CLAUDE.md**: 最近の create-next-app が生成する AI エージェント向け指示ファイル。「Next.js 16 は学習データより新しい可能性があるので `node_modules/next/dist/docs/` を読め」という内容。残しておく
