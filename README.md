# multi_app

複数プロダクトを持つ SaaS 企業を模した練習用モノレポ。

- フロントエンド: Next.js (TypeScript)
- バックエンド: Go
- 構成: 1 プロダクトにつき 1 API が対応する 2 層構成。プロダクト間の連携はバックエンド同士の内部通信で行い、フロントエンドは自プロダクトの API のみを呼ぶ。

## ディレクトリ構成

```
multi_app/
├── apps/                        # デプロイ単位（プロダクトごと）
│   ├── product-a/
│   │   ├── web/                 # Next.js フロントエンド
│   │   └── api/                 # Go バックエンド
│   └── product-b/
│       ├── web/
│       └── api/
│
├── services/                    # プロダクト横断の共通サービス
│   ├── auth/                    # 認証・認可
│   └── notification/            # 通知基盤など
│
├── packages/                    # フロントエンド共有コード（TypeScript）
│   ├── ui/                      # 共通UIコンポーネント
│   ├── config/                  # ESLint / tsconfig の共有設定
│   └── api-client/              # 生成されたAPIクライアント
│
├── pkg/                         # Go の共有ライブラリ
│   ├── logger/
│   ├── middleware/
│   └── database/
│
├── proto/                       # API定義（gRPC/OpenAPI）
│
├── infra/                       # インフラ関連
│   ├── docker/
│   └── k8s/
│
├── mise.toml                    # ツールのバージョン管理（node / pnpm など）
├── go.work                      # Go workspace（複数モジュールを束ねる）
├── package.json                 # npm workspace のルート
├── pnpm-workspace.yaml          # pnpm workspace 設定
└── Makefile                     # 横断的なタスクランナー
```

※ 上記は目指す全体像であり、ディレクトリは必要になったタイミングで順次作成していく。

進捗状況や経緯・判断理由は [PLAN.md](PLAN.md) を参照。
