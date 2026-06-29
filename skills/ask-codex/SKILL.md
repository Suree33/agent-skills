---
name: ask-codex
description: >-
  Consult OpenAI's Codex CLI (`codex exec`) for a second opinion, design/implementation
  advice, deep root-cause investigation, or a code review of the current changes. Use
  whenever the user wants to "ask Codex", "have Codex review this", or "let Codex
  investigate" — or otherwise run `codex exec` directly for a fresh perspective on a
  design, a tricky bug, a race condition, or a diff, even if they don't say the exact
  words.
---

# ask-codex

OpenAIのコーディングエージェント Codex を `codex exec`（非対話モード）で呼び出し、設計・実装の相談、深掘り調査、コードレビューを依頼するためのスキル。

Codex は粘り強く調査して精度を上げる傾向があるため、**第二の視点**や**正確性が重要な深掘り**に向く。

**Codex は毎回コールドで起動し、今の会話・計画・これまでの調査結果を一切知らない。** 前提・意図・制約・関連ファイルは毎回プロンプトに明示する（下記「タスク文の書き方」と例を参照）。

## 前提

- `codex` CLI がインストール済みで認証済みであること（`codex --version` で確認できる）。
- 認証が切れている場合は Codex 側のセットアップが必要なので、その旨をユーザーに伝える。

## 基本の呼び出しパターン

Codex は **read-only の上位 reviewer** なので、`-s read-only` を既定にする。プロンプトは shell エスケープを避けるため **stdin 経由**（`codex exec -`）で渡す。

中心となる作法: **回答の取り出し（`cat`）を、Codex を呼ぶのと同じコマンド内で完結させる**。`OUT=$(mktemp …)` の `$OUT` は次のシェル呼び出し（別シェル）では消えるため、後続の Read で読もうとするとパスを失う。同一コマンド内で `cat` すれば、挙動を一切仮定せず確実に回答を取れる。

用途で 2 つの正準レシピに分ける。

### A. 短い相談・小さなレビュー（フォアグラウンド）

回答はそのままシェル実行の tool result に返る。追加の Read は不要。

```bash
OUT=$(mktemp /tmp/codex_answer.XXXXXX.txt)
printf '%s' "$PROMPT" | codex exec - -s read-only -C <DIR> -o "$OUT" || exit
cat "$OUT"
```

### B. 時間のかかる呼び出し（バックグラウンド）

`codex exec` は数分かかることがあり、フォアグラウンド実行は環境により打ち切られる。深掘り調査や `codex exec review` でのコードレビューなど長くなりそうなものは、**自分のハーネスのバックグラウンド実行機構で投げる**（指定方法はハーネス側に任せる）。コマンド本体は用途で `codex exec -`（相談・調査）か `codex exec review`（diff レビュー、後述）を使い分ける。

```bash
OUT=$(mktemp /tmp/codex_answer.XXXXXX.txt)
printf '%s' "$PROMPT" | codex exec - -s read-only -C <DIR> -o "$OUT" || exit
echo "=== ANSWER ==="; cat "$OUT"
```

完了すると harness が完了通知を出し、そのバックグラウンド出力に `=== ANSWER ===` 以降の回答が入る。その出力を Read する（長い場合は `=== ANSWER ===` 以降を Grep でもよい）。**`sleep` / `pgrep` でポーリングするループを自作しない**（harness が完了を通知する）。

### フラグの意味

- `codex exec -`: stdin からプロンプトを読む。長文・コードブロック・特殊文字を安全に渡せる。
- `-s read-only`: 読み取りのみ許可。このスキルでは**既定でこれを付ける**（助言・レビュー役なので書き込みは不要）。CLI/config 自体の既定値ではない。**`codex exec` 専用で、`codex exec review` には無い**（付けるとエラー）。
- `-C <DIR>`: `codex exec` の作業ルート。調査・レビュー対象のリポジトリを指す。省略時はカレントディレクトリ。**`codex exec review` には `-C` が無い**ので、review は同一コマンド内で `cd <DIR>` してから呼ぶ（後述）。
- `-o <FILE>`: Codex の**最終メッセージだけ**をこのファイルに書き出す。**完了時にのみ書かれる**（実行中は空）ので、完了前に読まないこと。
- stderr: 進捗ログ**とエラー**（認証切れ・無効フラグ・repo check 失敗など）が出る。`2>/dev/null` で捨てると `cat "$OUT"` が空のときに原因が見えなくなるので、**正準レシピでは捨てない**。回答は `-o` のファイルから取るので進捗ログと混ざらない。
- `mktemp /tmp/codex_answer.XXXXXX.txt`: 末尾 `XXXXXX` がランダムに置換され、衝突しないユニークなパスを生成する。並列で複数投げる場合も呼び出しごとに別 `mktemp` になる。

### 通常は不要なオプション

- `--skip-git-repo-check`: git リポジトリ**外**で実行する場合のみ必要。リポジトリ内なら付けない。
- `-s workspace-write`: Codex に編集させたいときだけ。このスキルの想定（read-only reviewer）からは外れるので、原則使わない。

### モデル指定

特定モデルを使いたい場合は `-m <MODEL>`（例: `-m gpt-5.5`）。指定が無ければ Codex の設定デフォルトに従う。むやみに指定せず、ユーザーの要望があるときだけ付ける。

### Web 検索

`~/.codex/config.toml` に `web_search = "live"` が設定されていれば、Codex は **web 検索が使える**（現環境では設定済み）。web 検索は Responses のネイティブツールなので、`-s read-only`（shell 実行のみを制限）でも止まらない。ライブラリ・API・CLI の具体仕様を確認させたいときは、プロンプトで「公式ドキュメントを web 検索で確認して」と明示する。`codex exec` 自体に `--search` フラグは無く、config の `web_search` で制御する。

### ヘルプ表示

`codex exec -h` でヘルプを表示できる。

## タスク文の書き方

LLM は例に従うのが得意なので、**下の各例で `printf` に渡すプロンプト本文をそのまま良いタスク文の見本**として使う。1 回の相談を 1 つの判断 / レビュー / 計画 / 調査に絞り、次を 1 つのプロンプトに含める。

- **意図・背景・制約**: Codex はコールドなので、何をしたいか・なぜか・前提を必ず書く
- **関連ファイル**: `@path/to/file` でインラインに示す。現在の変更を見てほしいなら「`git diff` で差分を確認して」と明示する
- **着眼点 / 比較したい代替案 / 探してほしい risk** を具体的に書く
- **出力の形**を指定する（ranked risks / tradeoff table / 推奨する計画変更 など）
- **無視させるもの**（命名・整形など）を書いて scope creep を防ぐ
- 外部仕様の確認が要るなら「公式ドキュメントを web 検索で確認して」と明示する
- レビューは **意図 → 実装** の順でさせる（まず意図に照らして妥当か、次に実装）

プロンプト本文は `printf '%s' '…'` のように**単引用符**で囲むと、本文中の `$` やバッククォートがシェルに解釈されず安全（本文に ASCII の `'` を含めるときだけ別の囲み方にする）。

## 用途別の使い方例

### 1. 第二意見・複数案比較（レシピ A）

設計・実装の判断への第二意見。比較したい案・制約・出力形を具体的に書くほど質が上がる。

```bash
OUT=$(mktemp /tmp/codex_answer.XXXXXX.txt)
printf '%s' '意図: 認証トークンのリフレッシュ方式を決めたい。現状は API 呼び出しごとに
有効期限を確認して同期リフレッシュしており（@src/auth/client.ts）、高頻度の並行
リクエストで二重リフレッシュが起きる。

比較してほしい案:
(1) リフレッシュ中の Promise を共有して単一化する
(2) 期限前にバックグラウンドでプロアクティブ更新する
(3) 401 を受けてから一度だけリフレッシュしてリトライする

制約: SSR と CSR の両方で動く必要がある。新規ライブラリは増やしたくない。
出力: correctness / 複雑さ / 失敗モード を列にした tradeoff table、推奨する default 案と
fallback 案、どの条件なら別案に乗り換えるべきか。命名・整形は無視。' \
  | codex exec - -s read-only -C /path/to/repo -o "$OUT" || exit
cat "$OUT"
```

### 2. コードレビュー（レシピ B 推奨）

`codex exec review` サブコマンドが diff レビュー専用。対象範囲をフラグで指定する。`review` も `-` で stdin プロンプト、`-o` での書き出しに対応している。**ただし `codex exec` と違い `-C` も `-s` も無い**ので、対象リポジトリ内で（必要なら同一コマンドで `cd`）read-only 相当のまま実行する。

観点・意図を伝える非自明な指示は、エスケープを避けるため stdin で渡すのを正準形にする。

```bash
OUT=$(mktemp /tmp/codex_review.XXXXXX.txt)
cd /path/to/repo || exit 1
printf '%s' '意図: この変更は idle 通知音の抑制時だけ挙動を変えるべきで、ユーザー入力
要求音の再生には影響しないはず。まず意図に照らして妥当か、次に実装を見て。
着眼点: 条件の反転、デフォルト値の変化、client/server 間の意味のずれ。
出力: 推奨 → evidence 付きの findings → 未検証の仮定、の順。命名・整形は無視。' \
  | codex exec review --uncommitted - -o "$OUT" || exit
echo "=== ANSWER ==="; cat "$OUT"
```

対象範囲フラグ（上の review レシピの `codex exec review …` 行に差し替えて使う断片。プロンプト無しなら位置引数も不要）:

```bash
codex exec review --uncommitted -o "$OUT" || exit   # 未コミット（staged + unstaged + untracked）
codex exec review --base main   -o "$OUT" || exit   # ベースブランチとの差分
codex exec review --commit <SHA> -o "$OUT" || exit  # 特定コミット
```

- 短い一文だけなら位置引数でもよい（例: `codex exec review --uncommitted "race condition に注目して" -o "$OUT" || exit`）。込み入った指示は上の stdin 形を使う。
- review は対象リポジトリ内で実行する（`-C` は無いので `cd` してから呼ぶ）。リポジトリ外なら `--skip-git-repo-check` は使えるが、レビュー対象の diff が無ければ意味がない。

### 3. 計画・migration の妥当性検証（レシピ B）

実装前の計画・移行手順を stress-test し、着手前に潰すべきギャップを洗い出す。read-only のまま現コードの契約を確認させる。

```bash
OUT=$(mktemp /tmp/codex_answer.XXXXXX.txt)
printf '%s' '実装前の計画をレビューして、コーディング前に潰すべきギャップだけ挙げて。
計画: イベント同期がサーバに弾かれたら repair_required を立て、コンパクトな snapshot を
送り、サーバが整合シーケンスを ack した後にだけ増分イベントを再開する。
関連: @src/sync/event-syncer.ts @src/db/sync-state.ts @api/types.ts。
現状の契約は現コードを読んで確認してから評価して。
着眼点: スキーマ互換性、snapshot のサイズ上限、再開シーケンスの境界条件、
backfill/migration の影響、テスト不足。
出力: 着手前に必要な計画変更のみ＋「このまま実装して安全か」の一言判定。' \
  | codex exec - -s read-only -C /path/to/repo -o "$OUT" || exit
echo "=== ANSWER ==="; cat "$OUT"
```

### 4. 深掘り調査（レシピ B）

不具合・テスト失敗の根本原因、race condition、stale cache などの粘り強い調査。検証可能な成功条件を添えると Codex の深掘り特性が活きる。時間がかかるのでバックグラウンド実行。

```bash
OUT=$(mktemp /tmp/codex_answer.XXXXXX.txt)
printf '%s' 'テスト foo_test.go の TestBar が CI で稀に落ちる。根本原因を調査して。
成功条件: 再現条件と原因箇所をファイル:行で特定し、最小修正案を提示すること。
外部ライブラリ起因が疑われるなら、該当ライブラリの公式ドキュメントを web 検索で確認して。' \
  | codex exec - -s read-only -C /path/to/repo -o "$OUT" || exit
echo "=== ANSWER ==="; cat "$OUT"
```

## 結果の扱い

- 回答（レシピ A は tool result、レシピ B は完了通知後のバックグラウンド出力）から要点をユーザーに伝える。長い場合は要約しつつ、重要な指摘は原文を引用する。
- Codex は極端に防御的な提案（過剰な後方互換性配慮など）をすることがある。**採用する指摘と却下する指摘は自分で判断する**。鵜呑みにせず、make sense するものだけ取り込む。
