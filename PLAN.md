# PLAN.md — multi_app 作業メモ

ここまでの議論・決定事項・作業ログを網羅的に記録するメモ（「経緯と判断理由」を残す）。README.md は置かない方針。ディレクトリ構成の説明は `.agents/skills/answer-directory-structure/SKILL.md` が持ち、構成を変更したらスキルも更新する。

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
- バージョン指定の方針: next / react / react-dom は完全指定（組み合わせの整合性が重要なため）、typescript / @types/\* などツール系は `^` レンジ
- `@types/node` は create-next-app 生成時の `^20` から、mise の node 24 に合わせて `^24` に変更した
- `allowBuilds`: pnpm 11 の形式で `sharp: false` / `unrs-resolver: false`（ビルド済みバイナリが配布されるためスクリプト実行不要）。create-next-app が生成した旧形式 `ignoredBuiltDependencies` は pnpm 11 では効かなかったため置き換えた

### Next.js（決定・導入済み）

- Turbopack を使う（Next.js 16 では標準）
- lint / format は ESLint / Prettier ではなく **oxlint / oxfmt** を使う（導入済み、詳細は次節）
  - oxlint: 安定版。Next.js 専用ルール（`plugins: ["nextjs"]`）を組み込みサポート
  - oxfmt: ベータ（v0.5x）だが Prettier の JS/TS 準拠テスト 100% パス。練習用途ではリスクなしと判断
  - Turbopack はバンドラ、oxlint / oxfmt は静的解析・整形でレイヤーが独立しており競合しない
  - `next lint` は Next.js 16 で削除済みのため、linter は自由に選べる

### oxlint / oxfmt（決定・導入済み）

- oxlint 1.72.0 / oxfmt 0.57.0 を**ルートの devDependencies** に導入（catalog 管理）。lint / format は「リポジトリ全体の関心事」なのでルート一元管理・ルートから実行する方針。各パッケージには置かない（pnpm の厳格な依存管理により、web/ のパッケージスクリプトからは呼べない点に注意）
- `.oxlintrc.json`: plugins は `typescript` / `react` / `nextjs` / `unicorn` / `oxc`。categories は `correctness: error` + `suspicious: warn`。`react/react-in-jsx-scope` は React 17+ の新 JSX トランスフォームでは不要な古いルールのため off。`.next/` は除外
- `.oxfmtrc.json`: `printWidth: 100`（デフォルトと同値だが明示）、`sortImports: true`（import 文の自動並び替え）、`sortTailwindcss: true`（Tailwind クラスの自動並び替え、prettier-plugin-tailwindcss 相当）。`.next/` と `pnpm-lock.yaml` は除外。その他は Prettier 互換デフォルト
- 設定ファイルは「デフォルトから変えた意図のあるものだけ書く」最小構成の方針
- ルート scripts: `lint`（oxlint）/ `format`（oxfmt . 書き換えあり）/ `format:check`（CI 用・検査のみ、ズレがあれば終了コード 1）
- typecheck: web 側に `typecheck: next typegen && tsc --noEmit`（Next.js 16 はルート型生成 `next typegen` を先に実行するのが正しい）、ルートに `typecheck: pnpm -r typecheck`（workspace 全体を再帰実行）。oxlint は型チェックの代替にはならない

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
9. ブランチ構成の修正: 空コミットのみの `main` を作成し、`feat/init-apps-web` をその上に rebase → force push → デフォルトブランチを `main` に変更 → PR #1 を作成（https://github.com/chillout2san/multi_app/pull/1）
10. oxlint / oxfmt をルート devDependencies に導入（catalog 管理）、`.oxlintrc.json` / `.oxfmtrc.json` 作成、`lint` / `format` / `format:check` scripts 整備
11. `sortImports` / `sortTailwindcss` を有効化し、リポジトリ全体に `pnpm format` を適用（4 ファイル整形）
12. `typecheck` scripts を追加（web: `next typegen && tsc --noEmit`、ルート: `pnpm -r typecheck`）。実行して型エラーなしを確認
13. tsconfig の共通化: ルートに `tsconfig.base.json`（共通 compilerOptions）を作成し、web の `tsconfig.json` は `extends` + アプリ固有設定（next plugin / paths / include）のみの薄いファイルに変更。`tsconfig.json` という名前をルートに置くとエディタが誤適用しうるため base という名前にした。`include` / `exclude` の相対パスは宣言したファイル基準で解決されるためアプリ側に残す
14. ルートの README.md を削除（README は書かない方針に変更）
15. スキル `.agents/skills/answer-directory-structure/SKILL.md` を作成: 呼び出すと現状のディレクトリ構成を説明する。目指す全体像を別ドキュメント（docs/directory.md 案）に転記する代わりにスキル化した。「ディレクトリ構成を変更したら必ずスキルも更新する」というメンテナンスルールを description に明記
16. `.claude/skills` → `../.agents/skills` の相対シンボリックリンクを作成（Claude Code にスキルを認識させるため。実体は `.agents/skills/` に一元管理し、他の AI エージェントとも共有できる）。スキルが認識されることを確認済み
17. `apps/product-b/` を作成（.gitkeep のみ）

## 6. 次のステップ

- [x] `apps/product-a/web` に oxlint / oxfmt を導入（作業ログ 10〜12 参照）
- [ ] CI（GitHub Actions）の整備: PR ごとに `pnpm lint` / `pnpm format:check` / `pnpm typecheck` / `next build` を実行。CI がないと format:check などの「弾く」系の守りが機能しない
- [ ] `apps/product-a/api` の Go モジュール初期化
  - モジュールパスは `github.com/chillout2san/multi_app/apps/product-a/api` のような形を想定（要確認）
  - `go.work` の設置（マルチモジュール + go workspace 方式）
- [ ] main ブランチの Ruleset 設定（Require a pull request / Block force pushes / Restrict deletions）
- [ ] その後の候補: Makefile（または mise tasks）整備、docker compose での起動、product-b の追加、`services/` / `packages/` / `proto/` の整備

## 7. 学んだこと・議論したことのメモ

- **スキャフォールド**: プロジェクトの初期ファイル一式を自動生成すること。`pnpm create next-app` の `create` がそれ。Go には公式ジェネレータがないので api 側は手で作る
- **pnpm の node_modules 構造**: ルートの `node_modules/.pnpm`（virtual store）に全パッケージの実体、各パッケージの `node_modules` には宣言した依存へのシンボリックリンクのみ。幽霊依存（宣言していない依存の import）を構造的に防ぐ
- **semver レンジ**: `16.2.10` はピン留め、`^4` はメジャーを跨がない範囲で許容。実際に入るバージョンは `pnpm-lock.yaml` が確定させる。レンジ内で上がるのは `pnpm update` 実行時のみ
- **pnpm のビルドスクリプト制御**: pnpm v10+ は依存のライフサイクルスクリプトをデフォルト実行しない（サプライチェーン攻撃対策）。pnpm 11 では `allowBuilds` でパッケージごとに true / false を明示する
- **AGENTS.md / CLAUDE.md**: 最近の create-next-app が生成する AI エージェント向け指示ファイル。「Next.js 16 は学習データより新しい可能性があるので `node_modules/next/dist/docs/` を読め」という内容。残しておく
- **GitHub の PR には共通祖先が必要**: 履歴が完全に別のブランチ同士では PR を作れない。「空の main への PR」は、空コミットを作って feature ブランチをその上に rebase することで実現した
- **デフォルトブランチ**: 最初に push されたブランチが自動でデフォルトになる。変更できるのはリポジトリの Admin のみ。個人リポジトリではコラボレーターは常に Write 相当で、設定変更（デフォルトブランチ・Ruleset など）は Owner にしかできない
- **GitHub Ruleset**: main 保護の基本は Require a pull request / Block force pushes / Restrict deletions。Restrict creations は `release/*` などパターン対象で生きる機能で、既存の main 保護には実質関係ない
- **oxlint / oxfmt はパス自動検知**: 引数なし（または `.`）でカレント以下の対応拡張子ファイルを再帰的に走査する。`.gitignore` を尊重し、`ignorePatterns` で追加除外できる。ファイルリストのメンテは不要
- **printWidth**: 1 行の最大幅の目標値。oxfmt のデフォルトは 100（Prettier は 80）。収まる行はまとめ、超える行は折り返される
