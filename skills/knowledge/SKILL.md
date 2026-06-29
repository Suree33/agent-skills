---
name: knowledge
description: Obsidian vault のナレッジ管理。ノート・デイリーノート・MOC の検索、保存、整理に使う。記録・参照の意図がある時や、コードベース外のドメイン知識・社内ルール・意思決定が関わる時は能動的に検索・保存する。
---

# Knowledge — Obsidian ナレッジ管理

Obsidian vault を Zettelkasten 方式で管理するスキル。調査結果の保存、既存ナレッジの検索、デイリーノート管理、ノートの整理・リンク構築を行う。

## Vault

- vault 名: 環境変数 `OBSIDIAN_KNOWLEDGE_VAULT` で指定する。CLI には `vault="$OBSIDIAN_KNOWLEDGE_VAULT"` として渡す
- 未設定の場合は `obsidian vaults` で候補を提示し、今回使う vault をユーザーに確認する。あわせて、毎回の確認を省くため `OBSIDIAN_KNOWLEDGE_VAULT` を設定するよう推奨する
- vault パスが必要な時（filesystem フォールバックなど）は `obsidian vaults verbose` の出力から該当 vault 名の行のパスを取得する
- 関連スキル: `obsidian-cli`（CLI操作）、`obsidian-markdown`（Markdown構文）

## CLI と filesystem の使い分け

全操作で `obsidian` コマンドを優先する。`command -v obsidian` で有無を確認し、`$OBSIDIAN_KNOWLEDGE_VAULT` を `vault=` 引数に渡して目的のコマンド（`search`, `read`, `create`, `append`, `daily:read`, `daily:append` など）を実行する。

- 例: `obsidian vault="$OBSIDIAN_KNOWLEDGE_VAULT" search query="test"`
- `create` / `append` はデフォルトでファイルを開かない。エディタで開きたい時だけ `open` を付ける

CLI が無い・落ちる・対象コマンドが失敗する場合は filesystem 操作にフォールバックし、vault ディレクトリ内の `.md` ファイルを直接読み書きする。vault パスは `obsidian vaults verbose` が使えればその出力から取得し、CLI 自体が使えない場合はユーザーに確認する。書き込みは `obsidian-markdown` スキルの構文規約に従って frontmatter + 本文を構成し、書き込み権限が無ければユーザー承認を取る。

## Zettelkasten 規約

### ノートの種類

- **Permanent Note**: アトミックな知識（1ノート = 1概念）
  - frontmatter: `type: permanent`
- **Literature Note**: 外部情報源の要約・引用
  - frontmatter: `type: literature`
- **MOC**: トピック別ナビゲーション（関連ノートへの wikilink 集）
  - frontmatter: `type: moc`
- **Daily Note**: 日次の記録（タスク・日記・予定）
  - frontmatter: `type: daily`
- **Plan Note**: 一時的な実装計画・作業計画
  - frontmatter: `type: plan`

### frontmatter スキーマ

```yaml
---
title: ノートのタイトル
type: permanent | literature | moc | daily | plan
tags:
  - topic/subtopic
date: YYYY-MM-DD
source: "(literature のみ) URL や書籍名"
aliases:
  - 別名
---
```

### 命名・配置ルール

- ファイル名 = タイトルそのまま（日本語OK）。例: `Go の context パッケージ.md`
- Daily Note は Obsidian の Daily notes 設定に合わせて `dailynotes/YYYY/M月/YYYY-MM-DD.md` に配置。それ以外はルートにフラット配置
- 実装計画は `type: plan` の一時的な作業メモとして `plans/` に配置する。恒久的な知識として残す内容は、完了後に Permanent Note へ抽出する
- タグは `topic/subtopic` 階層形式。例: `#golang/concurrency`, `#testing/playwright`
- 1ノート1概念を厳守。長くなる場合は分割して `[[wikilink]]` で繋ぐ
- 関連する既存ノートには必ず `[[wikilink]]` を張る
- 関連する MOC が存在すれば、追加候補の wikilink 差分を提示し、ユーザー確認後に MOC を更新する

### MOC の構造

MOC は特定トピックのナビゲーションページ。サブトピックごとにセクションを分けて wikilink を整理する:

```markdown
---
title: MOC - Golang
type: moc
tags:
  - golang
date: 2026-06-09
---

# Golang

## Concurrency

- [[Go の context パッケージ]]
- [[goroutine のライフサイクル]]

## Testing

- [[Go のテーブルドリブンテスト]]
```

## サブコマンド

引数なしで `/knowledge` を呼んだ場合は、会話の流れから適切なサブコマンドを推定する。

### save — 調査結果・知識の保存

**トリガー**: `/knowledge save`, 「ノートに保存して」「Obsidian に記録して」「vault に追加」、またはコードベースから分からない情報が会話で判明した時（能動的記録）

**能動的記録の基準**: 以下を満たす情報は、明示的な指示が無くても保存する。保存後に何を記録したかユーザーに報告する。

- コードや公式ドキュメントを読んでも分からない（例: ドメイン知識、関連リポジトリの関係性、社内ルール、意思決定の経緯、外部連携の仕様）
- 将来の作業でも再利用する価値がある（その会話限りの一時情報は対象外）

1. 会話コンテキストから保存すべき内容を特定する（ユーザーが明示した範囲、または直近の調査・分析結果）
2. アトミック性を判断する。複数の独立した概念を含む場合は分割を提案する
3. `obsidian search` で関連する既存ノートを検索する
4. ノートの種類を判断する（save が作るのは Permanent / Literature。MOC は手順 6、Daily は `daily`、Plan は `plans/` で別途扱う）:
   - 自分の分析・まとめ → Permanent Note
   - 外部ドキュメント・記事の要約 → Literature Note（`source` フィールドに出典を記載）
5. Zettelkasten 規約に従ってノートを作成する:
   - frontmatter（title, type, tags, date）
   - 本文（簡潔に。箇条書きやコードブロック活用）
   - 関連ノートへの `[[wikilink]]`
6. 関連する MOC があれば追加する wikilink の差分を提示し、ユーザー確認後に更新する。なければ MOC の作成を提案する
7. 作成したノートのタイトルとパスをユーザーに報告する

### search — 既存ナレッジの検索・参照

**トリガー**: `/knowledge search <query>`, 「vault で調べて」「前に書いたノートに...」「ノートにあったはず」、またはコードベースから分からない情報が必要になった時（能動的検索）

**能動的検索の基準**: 以下の場面では、明示的な指示が無くても作業前に vault を検索する。

- ドメイン用語・プロダクト仕様・他リポジトリとの関係・過去の意思決定の経緯が関わるタスクを始める時
- 過去に同種の調査をした可能性がある時
- ヒットしたら出典ノートを明示して回答・作業に取り込む。ヒットしなければ通常の調査（コード・公式ドキュメント・ユーザーへの質問）にフォールバックする

1. `obsidian search` でキーワード検索する（CLI 不可時は `rg -l` で代替）
2. ヒットしたノートの内容を `obsidian read` で読む（CLI 不可時は filesystem から直接読む）
3. `obsidian backlinks` や本文中の `[[wikilink]]` から関連ノートも辿る
4. 検索結果を要約して会話に取り込む。ノートの正確な引用が必要な場合はそのまま提示する

### daily — デイリーノート管理

**トリガー**: `/knowledge daily`, 「今日のデイリーノート」「タスク追加して」「日記に書いて」

- **読み取り**: `obsidian daily:read` でその日のノートを取得
- **タスク追加**: `obsidian daily:append` で Tasks セクションに `- [ ] タスク内容` を追記
- **日記追記**: 日記セクションに内容を追記
- **新規作成**: デイリーノートが無い場合、テンプレート（`templates/デイリーノート.md`）から作成

デイリーノートのテンプレート:

```markdown
#dailynotes

## Tasks

## 日記

## 明日の予定
```

CLI 不可時は filesystem 操作で `dailynotes/YYYY/M月/YYYY-MM-DD.md` に直接作成する。

### organize — ノートの整理・リンク構築

**トリガー**: `/knowledge organize`, 「ノート整理して」「MOC 更新して」「リンク追加して」

以下の操作を単独または組み合わせて実行する:

- **タグ付け**: `obsidian search` で未タグのノート（frontmatter に `tags` がないもの）を検出し、内容からタグを提案する。ユーザー確認後に `obsidian property:set` で適用
- **リンク構築**: ノート間の関連性を検出し、wikilink の追加を提案する。ユーザー確認後に追記
- **MOC 作成・更新**: 指定トピックの MOC を新規作成するか、既存 MOC に新しいリンクを追加する
- **孤立ノート検出**: どの MOC にもリンクされておらず backlinks もないノートを発見し、整理を提案する

organize は破壊的な変更を伴う可能性があるため、変更内容を提示してユーザーの確認を得てから適用する。
