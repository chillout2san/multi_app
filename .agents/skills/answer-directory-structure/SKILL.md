---
name: answer-directory-structure
description: このリポジトリ（multi_app）のディレクトリ構成を説明するスキル。このスキルは現状のディレクトリ構成を記述したものなので、ディレクトリ構成を変更したら必ずこのスキルも更新すること。
---

# リポジトリの説明

このリポジトリは「複数プロダクトを持つ SaaS 企業」を模した練習用モノレポである。

## 現在のディレクトリ構成

```
multi_app/
├── .agents/
│   └── skills/                  # AI エージェント用スキル（このファイルもここにある）
│       └── answer-directory-structure/
│
├── .claude/
│   └── skills -> ../.agents/skills   # シンボリックリンク（Claude Code にスキルを認識させるため）
│
├── .github/
│   └── PULL_REQUEST_TEMPLATE.md # PR テンプレート（Summary / TestPlan の構成）
│
├── apps/                        # デプロイ単位（プロダクトごと）
│   ├── product-a/
│   │   ├── web/                 # Next.js 16 フロントエンド（TypeScript / App Router / src dir / Tailwind / Turbopack）
│   │   └── api/                 # Go バックエンド（未実装、README のみ）
│   └── product-b/               # 2 つ目のプロダクト（ディレクトリのみ、中身は未着手）
│
├── mise.toml                    # ツールのバージョン管理（node 24 / pnpm 11）
├── package.json                 # pnpm workspace のルート（lint / format / typecheck の scripts、oxlint / oxfmt の devDependencies）
├── pnpm-workspace.yaml          # workspace 定義・依存バージョンの catalog・allowBuilds
├── pnpm-lock.yaml               # ロックファイル（ルートに一元化）
├── tsconfig.base.json           # TypeScript の共通設定（各パッケージが extends する）
├── .oxlintrc.json               # oxlint（linter）設定
└── .oxfmtrc.json                # oxfmt（formatter）設定
```

## 設計方針（説明時に添えること）

- **2 層構成**: 1 プロダクトにつき 1 API が対応する。フロントエンド（web）は自プロダクトの API のみを呼び、他プロダクトのデータが必要な場合はバックエンド同士（api → api）が内部ネットワークで通信して集約する
- **apps/ はプロダクト単位**で切る。言語単位（frontend/ backend/）では切らない
