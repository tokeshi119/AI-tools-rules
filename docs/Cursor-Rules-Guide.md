# Cursor Rules 実践ガイド：日本語コミットメッセージ設定

**最終更新**: 2025-12-13  
**検証環境**: Cursor IDE

## 📖 目次

1. [概要](#概要)
2. [背景：Cursor Rules の種類](#背景cursor-rules-の種類)
3. [検証結果：コミットメッセージの日本語化](#検証結果コミットメッセージの日本語化)
4. [推奨構成](#推奨構成)
5. [セットアップ手順](#セットアップ手順)
6. [トラブルシューティング](#トラブルシューティング)
7. [参考リンク](#参考リンク)

---

## 概要

このドキュメントは、Cursor IDE で**コミットメッセージを日本語で自動生成**するための実践的なガイドです。

### このドキュメントの目的

- Cursor の各ルール形式の違いを理解する
- コミットメッセージを日本語化する方法を知る
- 実際に動作する構成を提供する

### 対象読者

- Cursor IDE を使用している開発者
- 日本語でコミットメッセージを書きたい方
- Git サブモジュールでルールを管理したい方

---

## 背景：Cursor Rules の種類

[Cursor 公式ドキュメント](https://cursor.com/ja/docs/context/rules)によると、Cursor は4種類のルールをサポートしています：

### 1. **プロジェクトルール** (`.cursor/rules/*/RULE.md`)

```
.cursor/rules/
  my-rule/
    RULE.md
```

- **配置場所**: `.cursor/rules/ルール名/RULE.md`
- **特徴**:
  - フロントマター（YAML）でメタデータ設定
  - `alwaysApply`、`globs`、`description` で適用条件を制御
  - バージョン管理される
- **用途**: プロジェクト固有のワークフローやスタイルガイド

### 2. **AGENTS.md**

```
プロジェクトルート/
  AGENTS.md
```

- **配置場所**: プロジェクトルート
- **特徴**:
  - プレーンな Markdown ファイル
  - シンプルで読みやすい
  - `.cursor/rules` の代替
- **用途**: シンプルなプロジェクト指示

### 3. **ユーザールール**

- **設定場所**: `Cursor Settings → Rules`
- **特徴**: 全プロジェクトに適用されるグローバル設定
- **用途**: 好みの会話スタイルやコーディング規約

### 4. **レガシー形式** (`.cursorrules`)

```
プロジェクトルート/
  .cursorrules
```

- **配置場所**: プロジェクトルート
- **特徴**: 
  - **将来的に廃止予定**
  - ただし、**現在もサポート対象**
  - プレーンテキスト形式
- **用途**: シンプルなルール（非推奨だが動作する）

---

## 検証結果：コミットメッセージの日本語化

### 🧪 実施した検証

以下の構成でコミットメッセージの自動生成をテストしました。

| 方法 | 結果 | 備考 |
|------|------|------|
| `.cursorrules` | ✅ **日本語になった** | レガシーだが確実に動作 |
| `AGENTS.md` | ❌ 英語のまま | 効果が確認できず |
| `.cursor/rules/*/RULE.md` | ✅ **日本語になった** | サブモジュール内では動作 |

### 🔍 重要な発見

#### 1. **`.cursorrules` が最も確実**

レガシー形式ですが、コミットメッセージ生成には確実に効果がありました。

#### 2. **サブモジュールの独立性**

サブモジュールは別の Git リポジトリとして扱われるため、親プロジェクトとは**独立してルールが適用**されます。

```
playwright-learning/              ← 親プロジェクト
├── .cursorrules                 ← 親用のルール
└── .cursor/
    └── rules/                    ← サブモジュール（別リポジトリ）
        └── */RULE.md            ← サブモジュール用のルール
```

#### 3. **公式ドキュメントの限界**

[公式ドキュメント](https://cursor.com/ja/docs/context/rules)には：

> **ルールは Cursor Tab や他の AI 機能には影響しません。**
> 
> **User Rules は Inline Edit（Cmd/Ctrl+K）にも適用されません。Agent（Chat）でのみ使用されます。**

**コミットメッセージ生成がどこに該当するかは明記されていません。**

---

## 推奨構成

### 🎯 実用的な構成（検証済み）

```
プロジェクトルート/
├── .cursorrules              ← コミットメッセージ用（確実）
├── AGENTS.md                 ← Chat/Agent 用（オプション）
└── .cursor/
    └── rules/                ← サブモジュール（共通ルール管理用）
        ├── commit-rules/
        │   └── RULE.md
        ├── general-rules/
        │   └── RULE.md
        └── global-rules/
            └── RULE.md
```

### 📄 ファイルの役割分担

| ファイル | 用途 | 効果確認済み | 推奨度 |
|---------|------|------------|--------|
| `.cursorrules` | コミットメッセージ生成 | ✅ 日本語 | ⭐⭐⭐ |
| `AGENTS.md` | Chat/Agent での会話 | ❓ 要検証 | ⭐⭐ |
| `.cursor/rules/*/RULE.md` | サブモジュール用 | ✅ 日本語 | ⭐⭐⭐ |

---

## セットアップ手順

### 🚀 既存プロジェクトへの適用

#### ステップ1: サブモジュールを追加

```bash
cd your-project
git submodule add https://github.com/tokeshi119/AI-tools-rules.git .cursor/rules
```

#### ステップ2: `.cursorrules` をテンプレートからコピー

```bash
cp .cursor/rules/.cursorrules.template .cursorrules
```

#### ステップ3: Cursor を再起動

ルールの変更を反映させるため、Cursor を再起動します。

#### ステップ4: 動作確認

コミットメッセージの自動生成を試して、日本語になることを確認します。

### 🆕 新規プロジェクトでの導入

```bash
# 1. サブモジュールを追加
git submodule add https://github.com/tokeshi119/AI-tools-rules.git .cursor/rules

# 2. テンプレートをコピー
cp .cursor/rules/.cursorrules.template .cursorrules

# 3. コミット
git add .cursorrules .gitmodules .cursor/rules
git commit -m "chore: Cursor ルールを追加"

# 4. Cursor を再起動
```

---

## トラブルシューティング

### ❓ コミットメッセージが英語のまま

#### 解決策1: Cursor を再起動

ルールの変更が反映されていない可能性があります。

#### 解決策2: `.cursorrules` の配置を確認

```bash
# プロジェクトルートに配置されているか確認
ls -la .cursorrules
```

#### 解決策3: User Rules を設定

`Cursor Settings → Rules` で、以下をユーザールールに追加：

```markdown
コミットメッセージを生成する際は、必ず日本語で記述してください。
Conventional Commits 形式（feat:、fix: など）を使用してください。
```

### ❓ サブモジュールのルールが効かない

サブモジュールは別の Git リポジトリです。サブモジュール内で作業する場合、そのサブモジュール用のルールが適用されます。

**親プロジェクトには `.cursorrules` を配置してください。**

### ❓ `AGENTS.md` が効かない

`AGENTS.md` は Agent（Chat）専用の可能性があります。コミットメッセージには `.cursorrules` を使用してください。

---

## 参考リンク

### 📚 公式ドキュメント

- [Cursor Rules 公式ドキュメント](https://cursor.com/ja/docs/context/rules)

### 🔧 サブモジュール管理

- [Git Submodules 公式ドキュメント](https://git-scm.com/book/ja/v2/Git-%E3%81%AE%E3%81%95%E3%81%BE%E3%81%96%E3%81%BE%E3%81%AA%E3%83%84%E3%83%BC%E3%83%AB-%E3%82%B5%E3%83%96%E3%83%A2%E3%82%B8%E3%83%A5%E3%83%BC%E3%83%AB)

---

## まとめ

### ✅ 確実に動く方法

1. **サブモジュールを追加**
2. **`.cursorrules` をテンプレートからコピー**
3. **Cursor を再起動**
4. **コミットメッセージを自動生成して確認**

### 🎓 学んだこと

- レガシー形式 `.cursorrules` がコミットメッセージには最も確実
- 公式推奨の新形式でも効果が不明確なケースがある
- サブモジュールは独立したリポジトリとして扱われる
- 実際に試すことが重要

---

**このドキュメントが役に立ちましたか？フィードバックをお待ちしています！**

