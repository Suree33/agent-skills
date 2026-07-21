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
- [japanese-tech-writing](skills/external/japanese-tech-writing/SKILL.md)
  - 日本語の技術文書や書籍原稿を、論理構成、読み手の負荷、表記の一貫性から執筆・推敲する。
- [cognitive-rhythm-writing](skills/external/cognitive-rhythm-writing/SKILL.md)
  - 説明文の認知モードと緊張を管理し、密度を保ちながら読み進めやすい緩急を設計する。

`japanese-tech-writing` と `cognitive-rhythm-writing` は、k16shikano の public gist を Unlicense に基づいて再配布しています。原本 URL は各 `SKILL.md` の frontmatter に記載しています。
