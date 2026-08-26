---
name: ask-claude
description: >-
  Claude Code CLI（claude -p）で独立した第二の意見、コードレビュー、バグ調査、設計判断、長大な文脈や画像を含む分析を依頼する。
  特に、Claude 固有の視点が有用なコードレビュー/バグ発見、複雑で曖昧な長時間タスク、文書・図・UI の理解、
  独立した視点による検証で使う。単純な修正、現在のエージェントだけで十分な通常実装、
  「Claude の方が常に上」と仮定した丸投げには使わない。
license: MIT
metadata:
  author: Suree33
  version: "1.0.0"
---

# ask-claude

Anthropic のコーディングエージェント Claude Code を非対話モードで呼び出し、現在のエージェントとは独立した助言やレビューを得る。

Claude Code は現在の会話を知らない。一方、既定では対象ディレクトリの `CLAUDE.md`、settings、skills、plugins などを読み込む。相談の意図、判断材料、制約、関連ファイルはプロンプトにも明示する。

## 前提

- `claude` CLI がインストール済みで認証済みであること。`claude --version` で確認する。
- 認証が切れている場合は Claude Code 側のセットアップが必要だと伝える。
- `claude -p` は workspace trust ダイアログを表示しない。信頼できるディレクトリだけで実行する。

## 実行前に

何を Claude に相談し、どのモデルを使うかを一言ユーザーに伝える。承認待ちは不要だが、外部サービスへの送信許可が不明な情報は送らない。

## モデルの選び方

Claude Opus 5 と Claude Fable 5 は、タスクの難度と必要な能力で使い分ける。

| モデル | 向く依頼 | 選択方針 |
|---|---|---|
| Claude Opus 5 | コードレビュー、バグ発見、複数ファイルの設計/実装判断、長文脈、図・UI・文書 | `ask-claude` の既定。日常的な第二意見にはまずこれを使う |
| Claude Fable 5 | 特に難しく、長く、曖昧な課題、最高能力を優先する分析 | Opus 5 で不足した時、または難度が明らかに高い時だけ使う。高コスト・長時間を見込む |

Anthropic は Fable 5 を最難関・長時間・曖昧な課題、Opus 5 を複雑な agentic coding と高精度な review/bug-finding に位置づけている。

ユーザーがモデルを指定したら従う。指定がなければ `opus`、最高能力が必要なら `fable` を使う。CLI の alias は将来の同 tier モデルへ更新され得るため、5 系を固定する必要がある時は `claude-opus-5` / `claude-fable-5` を指定する。

## 基本の呼び出し

Claude は read-only の相談役として使う。Claude Code には作業ディレクトリ指定フラグがないため、同じ shell 内で対象リポジトリへ `cd` する。プロンプトは shell 展開と引数長制限を避けるため stdin で渡す。

```bash
OUT=$(mktemp /tmp/claude_answer.XXXXXX.txt)
ERR=$(mktemp /tmp/claude_err.XXXXXX.txt)
cd <DIR> || exit 1
claude -p --model opus --permission-mode plan --no-session-persistence \
      >"$OUT" 2>"$ERR" <<'CLAUDE_PROMPT' \
  || { echo "claude failed:"; cat "$ERR"; exit 1; }
ここに Claude へ渡す依頼文を書く
CLAUDE_PROMPT
cat "$OUT"
```

- `-p` / `--print`: 非対話で 1 回実行し、回答を stdout に出す。
- stdin: 非対話モードは stdin を読める。上限は 10 MB なので、大きな diff や文書は pipe せずファイルパスを示す。
- `--permission-mode plan`: 読み取り専用の調査・計画に制限する。このスキルでは既定にする。
- `--no-session-persistence`: 相談を保存せず、毎回独立させる。
- `--model opus`: 最新の Opus tier。Fable は `--model fable`、5 系固定は完全なモデル名へ置き換える。
- effort: 通常は指定せず、Claude Code 側の設定とモデル既定に従う。ユーザーが指定した場合だけ `--effort <LEVEL>` を追加する。
- stdout: 最終回答。正準レシピでは一時ファイル経由で取得する。
- stderr: 診断とエラー。成功時はコンテキストへ流さず、失敗時だけ表示する。

長くなりそうな Fable 呼び出しは harness のバックグラウンド実行機構を使う。`sleep` や `pgrep` のポーリングループは作らない。

通常は `--bare` を付けない。`--bare` は起動を速くする代わりに `CLAUDE.md`、settings、hooks、plugins、OAuth/keychain 認証などを読み込まないため、この用途ではプロジェクト文脈と既存認証を失いやすい。

## Claude 向けタスク文

Claude の公式 prompting guidance に合わせる。

- 明確かつ直接的に、1 回の依頼を 1 つの判断・レビュー・調査へ絞る。
- 何をするかだけでなく、なぜ必要かを書く。Claude は意図を理解すると判断を一般化しやすい。
- `<context>`、`<task>`、`<constraints>`、`<output>` など一貫した XML タグで種類の違う情報を分ける。
- 出力形式、長さ、採否基準を具体的に指定する。否定だけでなく、望む形を肯定文で書く。
- 長い資料を本文へ入れる場合は資料を先、質問を最後に置く。通常のコード調査では本文を複製せず関連パスを示す。
- 「思考過程をすべて見せて」と依頼しない。結論、根拠、検証結果、不確実な点を求める。
- Opus 5 は自ら検証する傾向が強い。一般的な「何度も自己検証して」は付けず、必要な受け入れ条件だけを書く。
- Fable 5 には細かい思考手順を固定せず、目的・境界・成功条件を渡す。長時間依頼では、進捗や完了を tool result で裏付けるよう求める。

基本形:

```text
<context>
何を達成したいか、なぜ必要か、現状、関連ファイル、既に分かっていること。
</context>

<task>
今回 Claude に判断・調査してほしいことを 1 つ。
</task>

<constraints>
変更禁止、互換性、依存追加禁止、対象外など。
</constraints>

<output>
結論を先に。次に file:line 付きの根拠、推奨、未検証の仮定。
</output>
```

## 用途別の例

### 1. Opus 5 でコードレビュー

Opus 5 は review/bug-finding の recall が高い。公式 guidance に従い、最初から「重大なものだけ」と狭めず全 correctness issue を挙げさせ、受け手が severity を選別する。

```bash
OUT=$(mktemp /tmp/claude_answer.XXXXXX.txt)
ERR=$(mktemp /tmp/claude_err.XXXXXX.txt)
cd /path/to/repo || exit 1
claude -p --model opus --permission-mode plan --no-session-persistence \
      >"$OUT" 2>"$ERR" <<'CLAUDE_PROMPT' \
  || { echo "claude failed:"; cat "$ERR"; exit 1; }
<context>
この変更は idle 通知音だけを抑制し、ユーザー入力要求音には影響しない必要がある。
未コミット差分と関連 caller を git diff、git status、コード検索で確認して。
</context>
<task>
意図との不一致、回帰、境界条件、テスト不足をレビューして。correctness issue は severity で
事前に除外せず列挙し、各 finding に severity を付ける。
</task>
<constraints>
read-only。命名、整形、好みだけの指摘は対象外。
</constraints>
<output>
結論を 1 文。次に severity 順の findings を file:line、失敗条件、根拠付きで示す。
finding がなければその旨と未検証範囲だけを書く。
</output>
CLAUDE_PROMPT
cat "$OUT"
```

### 2. Fable 5 で難しい設計判断

長く曖昧な課題、複数領域をまたぐ判断、Opus 5 で結論が出なかった課題に限って使う。

```bash
OUT=$(mktemp /tmp/claude_answer.XXXXXX.txt)
ERR=$(mktemp /tmp/claude_err.XXXXXX.txt)
cd /path/to/repo || exit 1
claude -p --model fable --permission-mode plan --no-session-persistence \
      >"$OUT" 2>"$ERR" <<'CLAUDE_PROMPT' \
  || { echo "claude failed:"; cat "$ERR"; exit 1; }
<context>
イベント同期の再設計を検討している。repair_required、snapshot、ack 後の増分再開を導入したい。
利用者は移行中も書き込みを止められない。関連: @src/sync/ @src/db/ @api/types.ts。
</context>
<task>
現コードと履歴を調査し、安全な状態遷移と段階的 migration を提案して。
要件が曖昧な箇所は、合理的な仮定を明示して評価を続ける。
</task>
<constraints>
read-only。新規基盤は増やさない。後方互換 shim は移行期間に必要なものだけ。
</constraints>
<output>
推奨案を先に示す。次に状態遷移、壊れ得る invariant、migration 順序、最小の検証項目、
未解決の判断を示す。各現状認識はコードまたは git 履歴で裏付ける。
</output>
CLAUDE_PROMPT
cat "$OUT"
```

## 結果の扱い

- Claude の回答は第二意見であり、現在のエージェントがコードとテストに照らして採否を判断する。
- ベンダー間の主張や benchmark を、個別リポジトリでの優劣へそのまま一般化しない。
- 重要な指摘は file:line と再現条件を確認してから採用する。
- 長い回答は、結論、採用する指摘、却下する指摘、残る不確実性に分けてユーザーへ返す。

## 根拠

- [Claude models overview](https://platform.claude.com/docs/en/about-claude/models/overview)
- [Introducing Claude Fable 5 and Claude Mythos 5](https://www.anthropic.com/news/claude-fable-5-mythos-5)
- [Introducing Claude Opus 5](https://www.anthropic.com/news/claude-opus-5)
- [Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices.md)
- [Prompting Claude Fable 5](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5)
- [Prompting Claude Opus 5](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5)
- [Run Claude Code programmatically](https://code.claude.com/docs/en/headless)
- [Choose a permission mode](https://code.claude.com/docs/en/permission-modes)
