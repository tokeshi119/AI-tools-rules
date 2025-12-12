# AIs-rules

Cursor AI用の共通ルールセット

**🔗 Repository**: [https://github.com/tokeshi119/AI-tools-rules](https://github.com/tokeshi119/AI-tools-rules)

## 📖 概要

このリポジトリは、複数のプロジェクトで共有するCursor AIのルール設定を管理するためのものです。
一貫性のあるAI支援開発を実現し、プロジェクト間でのベストプラクティスを共有できます。

## 📁 構成

```
AIs-rules/
└── .cursor/
    └── rules/
        ├── commit-rules.mdc      # Gitコミットメッセージ生成ルール
        ├── general-rules.mdc     # AI応答の基本言語設定
        ├── global-rules.mdc      # 安全原則、コード品質、コラボレーションルール
        └── test-rule.mdc         # テストに関するルール
```

## 🎯 各ルールファイルの説明

### commit-rules.mdc
- **適用タイミング**: コミット関連の操作時
- **内容**: 
  - 日本語でのコミットメッセージ生成
  - Conventional Commits形式の適用

### general-rules.mdc
- **適用タイミング**: 常時
- **内容**: 
  - AI応答の日本語優先
  - コードコメントの日本語記述

### global-rules.mdc
- **適用タイミング**: 常時
- **内容**: 
  - 安全原則（環境変数、機密情報の保護）
  - コード品質原則
  - コラボレーション・対話ルール
  - 行動規範

### test-rule.mdc
- **適用タイミング**: 常時
- **内容**: 
  - テスト関連の軽量ルール

## 🚀 使用方法

### 方法1: Gitサブモジュールとして追加（推奨）

既存のプロジェクトにこのルールセットを追加する場合：

```bash
# プロジェクトディレクトリに移動
cd your-project

# 既存の.cursor/rulesがある場合はバックアップ
# ※ ディレクトリが存在しない場合はこのステップをスキップしてください
mv .cursor/rules .cursor/rules.backup

# サブモジュールとして追加
git submodule add https://github.com/tokeshi119/AI-tools-rules.git .cursor/rules

# コミット
git add .gitmodules .cursor/rules
git commit -m "chore: .cursor/rulesをサブモジュール化"
```

### 方法2: 手動コピー

```bash
# このリポジトリをクローン
git clone https://github.com/tokeshi119/AI-tools-rules.git

# プロジェクトの.cursorディレクトリにコピー
cp -r AIs-rules/.cursor/rules your-project/.cursor/
```

## 🔄 ルールの更新

### サブモジュールを使用している場合

```bash
# 最新のルールを取得
cd your-project
git submodule update --remote .cursor/rules

# 変更をコミット
git add .cursor/rules
git commit -m "chore: AI rulesを最新版に更新"
```

### このリポジトリを更新する場合

```bash
cd AIs-rules
# ファイルを編集
git add .
git commit -m "feat: ルールを更新"
git push
```

## 📝 カスタマイズ方法

プロジェクト固有のルールは、各プロジェクトの`.cursor/rules/`に追加ファイルとして配置するか、
`project-rules.md`を作成してオーバーライドしてください。

ルールの優先順位：
1. プロジェクト固有のルール（最優先）
2. このリポジトリのグローバルルール

## 🤝 ルールの改善

ルールの改善提案がある場合：

1. このリポジトリのブランチを作成
2. ルールを修正
3. プルリクエストを作成

## 📄 ライセンス

プライベートリポジトリ - プロジェクト内部での使用のみ

## 👤 管理者

Yuta Tokeshi

---

最終更新: 2025-12-12

