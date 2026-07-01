# Agent Skills

## インストール

```sh
# スキル一覧を確認
npx skills add Suree33/agent-skills --list

# 個別インストール
npx skills add Suree33/agent-skills --skill ask-codex

# 全部インストール
npx skills add Suree33/agent-skills --all
```

プロジェクトに入る。`--global` でユーザー全体に入れる。

## スキル一覧

- [ask-codex](skills/ask-codex/SKILL.md)
  - Codex を使って第二の意見・設計/実装アドバイス・根本原因の深掘り調査・変更のコードレビューを依頼する。主にClaude Codeなどから呼び出されることを想定している。[Amp](ampcode.com)のOracleをイメージして作成した。
- [knowledge](skills/knowledge/SKILL.md)
  - Obsidian Vault を使ってノートを検索・読み込み・追加する。
