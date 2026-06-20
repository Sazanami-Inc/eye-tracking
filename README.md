# eye-tracking

小学生が iPad で数学問題を解く際の視線データを収集・分析し、認知パターン特化型モデルの構築を目指すプロジェクト。

## ドキュメント

- [設計指針](docs/DESIGN.md) — アーキテクチャ・データ設計・実装ロードマップ

## クイックスタート

```bash
# 依存インストール
npm install

# 開発サーバー起動（iPad Safari でアクセス）
npm run dev
```

## ディレクトリ構成（予定）

```
eye-tracking/
├── apps/
│   ├── collector/     # TypeScript — iPad向け視線収集アプリ
│   └── dashboard/     # TypeScript — データ確認・アノテーションUI
├── packages/
│   └── schema/        # 共有型定義・バリデーション
├── pipeline/          # Python — 前処理・学習パイプライン
└── docs/
    └── DESIGN.md
```
