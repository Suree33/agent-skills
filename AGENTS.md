# Suree33 Agent Skills

@Suree33 の Agent Skills 配布リポジトリ。ビルド・lint は無い。自作スキルは `skills/<name>/SKILL.md`、外部再配布スキルは `skills/external/<name>/SKILL.md` に置かれ、[`npx skills`](https://github.com/vercel-labs/skills) 経由でインストールされる（`npx skills add Suree33/agent-skills --skill <name>`）。

主な成果物はスキル定義（プロンプト）。配布・同期に必要な最小限のスクリプトも含む。

## Agent Skillsとは

> Agent Skills are a lightweight, open format for extending AI agent capabilities with specialized knowledge and workflows.
> At its core, a skill is a folder containing a SKILL.md file. This file includes metadata (name and description, at minimum) and instructions that tell an agent how to perform a specific task. Skills can also bundle scripts, reference materials, templates, and other resources.
> [Agent Skills Overview](https://agentskills.io/home.md)

- 仕様: [Specification - Agent Skills](https://agentskills.io/specification.md)
- 定義: [Agent Skills](https://agentskills.io/llms.txt)

## 編集時の約束

- スキルを追加・変更したら **README.md の「スキル一覧」も更新する**（手動同期、自動生成は無い）。
- 自作スキル追加時は `skills/<name>/SKILL.md` を作る。外部再配布スキルは `skills/external/<name>/SKILL.md` に置く。どちらも `name` を親ディレクトリ名に合わせる。
- スキル本文は実行手順そのものなので、**CLI フラグ・コマンド仕様は事前知識で書かず一次情報で裏取りする**（例: ask-codex は `codex exec` の `-s` / `-C` / `-o` の有無をサブコマンド単位で明記している）。
- スキルの新規作成・改善には、スキル作成・改善用のスキル（例: `skill-creator`）を使う。
