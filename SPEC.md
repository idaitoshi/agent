# 指示書: サブエージェント付きタスク実行環境(Python標準ライブラリのみ)

あなた(Cline / sol)は、この指示書に従って `workspaces/agent-runner/` 配下に実装してください。
実装前に、設計の要約と作成ファイル一覧を提示し、私の承認を得てから書き始めてください。

## 1. 目的
与えた仕事(コード実装・バグ調査)を、メインエージェントが子エージェントに委任しながら自動で進める CLI を作る。
Cline からは `python agent.py run "<仕事>"` をターミナルで実行して使う。

## 2. 制約(厳守)
- Python 3 と標準ライブラリのみ。`pip install` 禁止。(`urllib.request`, `json`, `subprocess`, `pathlib`, `concurrent.futures`, `argparse` などを使う)
- API キー・URL・モデル名をコードやログに書かない。環境変数から読む:
  - `LLM_BASE_URL`(OpenAI互換のエンドポイント。末尾は `/chat/completions` を付けて呼ぶ)
  - `LLM_API_KEY`
  - モデルは別名(エイリアス)で扱い、環境変数 `MODEL_<別名を大文字に>` で実モデル名に解決する。使う別名:
    - `MODEL_MINI`(最安。探索・要約用。例: chat-gpt-mini4)
    - `MODEL_LUNA`(GPT-5.6 Luna。軽量・高速。調査・一次レビュー用)
    - `MODEL_SOL`(GPT-5.6 Sol。最上位。main・advisor 用)
    - `MODEL_CODEX`(任意。例: codex5.3。コード実装の代替)
    - `MODEL_ADVISOR`(任意。失敗時の相談用。未設定なら `MODEL_SOL` を使い、reasoning effort を最大にする)
  - 別名ごとに API 形式を `MODEL_<別名>_API` で指定できる: `chat`(/chat/completions、既定)または `responses`(/responses)。codex 系は Responses API 専用の可能性があるため、`core/llm.py` は両方に対応し、tool 呼び出しの入出力形式の差を吸収する。
  - 別名ごとに `MODEL_<別名>_EFFORT`(reasoning effort。例: low / medium / high / max)も任意指定できる。モデルが受け付けない場合は、パラメータを付けずに再試行する。
  - 別名が未設定の場合は、`MODEL_FALLBACK` があればそれを使い、なければ起動時にエラーにする。
- `.env` がカレントまたは親にあれば読み込む自前パーサを書く(値は絶対に表示・ログ出力しない)。`.env.example` も作る。
- 秘密情報らしきファイル(`.env`, `*.key`, `*.pem`, `id_rsa*`)は、エージェントのツールから読めないようにする。

## 3. ファイル構成
```
agent.py            # エントリポイント (argparse: run / list)
core/llm.py         # chat/completions 呼び出し(リトライ、タイムアウト、tool_calls 対応)
core/loop.py        # ツール呼び出しループ(最大ターン数あり)
core/tools.py       # ツール実装と許可制御
core/roles.py       # agents/*.md の読み込み(frontmatter: name, model, tools, max_turns)
core/log.py         # runs/<日時>/ に JSONL でログ保存
core/escalate.py    # エスカレーション条件の判定(第6.5節)
agents/
  main.md           # オーケストレーター (model=sol)
  explorer.md       # 読み取り専用の探索 (model=mini)
  coder.md          # 実装担当 (model=luna)
  reviewer.md       # 差分レビュー、読み取り専用 (model=luna)
  advisor.md        # 失敗時の相談役、読み取り専用 (model=advisor)
.env.example
README.md
tests/              # unittest でツールの安全制御とループをテスト(LLMはモックする)
```

## 4. ツール仕様(function calling 形式で定義)
作業ルートは `--root`(既定: カレント)。ルート外のパス(`..`、絶対パス、シンボリックリンク経由)は拒否する。

| ツール | 内容 | 備考 |
|---|---|---|
| `list_dir(path)` | 一覧 | |
| `read_file(path, start?, end?)` | 行範囲指定で読む | 1回の出力は最大約20KB、超えたら切り詰めて通知 |
| `grep(pattern, path?)` | 正規表現検索 | 結果は最大200行 |
| `write_file(path, content)` | 全置換で書き込み | coder のみ |
| `replace_in_file(path, old, new)` | 一意な一致のみ置換 | coder のみ。一致が0件・複数件ならエラー |
| `run_cmd(cmd)` | コマンド実行 | 許可リスト方式(`python`, `git status/diff/log`, `pytest`, `python -m unittest` 等)。タイムアウト60秒、`shell=False`、出力は切り詰める。`rm`, `del`, `curl`, `pip`, `git push/reset --hard` 等は拒否 |
| `delegate(role, task)` | 子エージェントを起動し、最終回答のテキストだけ返す | main のみ。子は新しいコンテキストで開始。孫への委任は禁止(深さ1) |
| `delegate_many(items)` | 複数の delegate を `ThreadPoolExecutor` で並列実行 | 同時実行数は `--max-parallel`(既定2) |

役割ごとに使えるツールを `agents/*.md` の `tools:` で制限する。explorer と reviewer は読み取り専用(`write_file`, `replace_in_file`, `run_cmd` なし。reviewer は `git diff` のみ許可してもよい)。

## 4.5 モデルの使い分け方針
- 既定の割り当て(`agents/*.md` の `model:` で変更可):
  - explorer = mini、reviewer(一次) = luna、main = sol、coder = luna、advisor = sol(`MODEL_ADVISOR` 未設定なら sol を使い、effort 最大)。codex は任意の代替で、使うときは `agents/coder.md` の `model:` を書き換える。
  - coder が失敗して第6.5節の条件が成立した場合は、advisor(sol)に相談する。
  - 割り当ては、実際のモデルの強さの順に合わせて README で調整できるようにする。
- 重要領域(認証・決済・データ移行・削除処理)に触れる仕事は、最初から advisor にレビューさせる。判定は仕事文とパスのキーワード一致で行い、設定ファイル `escalate.json` で変更できるようにする。

## 5. エージェント定義の形式 (`agents/*.md`)
```
---
name: coder
model: luna           # 別名。環境変数 MODEL_<別名の大文字> に解決(mini / luna / sol / codex / advisor)
tools: [list_dir, read_file, grep, write_file, replace_in_file, run_cmd]
max_turns: 25
---
(システムプロンプト本文)
```
各役割の本文に書く方針:
- **main**: 仕事を分解する。広い調査は explorer、実装は coder、変更後の確認は reviewer に委任する。小さな作業は自分でやる。最終回答は「変更点・確認結果・未解決事項」を簡潔に。同じ問題で2回失敗したら、ユーザーに状況を報告して止まる。
- **explorer**: 質問に答えるために必要な箇所だけ読む。結論と根拠(ファイル:行)を短く返す。ファイル全文を返さない。
- **coder**: 指示された範囲だけ変更する。変更前に該当箇所を読む。変更後にテスト/実行で確認する。最後に変更ファイルと要点を返す。
- **reviewer**: `git diff` を読み、バグ・副作用・テスト不足を重要度順に指摘する。編集しない。
- **advisor**: 失敗の経緯(直近のログ要約・エラー・試した修正)を受け取り、原因の仮説と次の方針を短く返す。編集しない。コードの全文書き直しはせず、方針と確認すべき箇所を示す。

## 6. ループと安全策
- 1エージェントあたり `max_turns` 超過で打ち切り、途中経過を返す。
- 全体で `--max-total-calls`(既定 60)を超えたら中断する。
- LLM 呼び出しは、429/5xx で指数バックオフ最大3回。タイムアウト120秒。
- 各ターンのリクエスト要約・ツール呼び出し・結果(切り詰め後)を `runs/<日時>/<agent>-<n>.jsonl` に保存。APIキーは含めない。
- `--dry-run` で、LLMを呼ばずに定義の読み込みとツール定義の検証だけ行う。
- 終了時に、使用ターン数・モデル別の呼び出し回数・トークン使用量(API応答の `usage` があれば)を表示する。

## 6.5 エスカレーション(失敗時に上位モデルへ相談)
モデルの自己申告には頼らず、`core/escalate.py` が次の条件をコードで判定する。いずれかが成立したら、そのエージェントを一旦止め、`advisor` に失敗経緯を渡して方針を得る。得た方針は元のエージェントに追加メッセージとして渡して再開する。
1. 同じテスト/コマンドが連続2回失敗した(終了コードと出力の先頭を比較)。
2. `max_turns` の半分を超えても、変更ファイル集合とテスト結果が変化していない。
3. 同じツール呼び出し(同名・同引数)を3回繰り返した。
4. reviewer が重大度「高」の指摘を出した。
5. 変更対象が重要領域(`escalate.json` のキーワード/パスに一致)。
制限:
- 1回の `run` あたりの advisor 呼び出しは `--max-escalations`(既定 3)まで。超えたら処理を止め、状況をユーザーに報告する。
- advisor の後も同じ条件で失敗した場合は、自動で続けず、ユーザーに報告して止まる。
- 昇格の理由(どの条件か)と advisor の回答は、ログに記録し、終了時のサマリにも表示する。

## 7. CLI
```
python agent.py run "仕事の内容" [--root .] [--role main] [--max-parallel 2] [--max-total-calls 60] [--max-escalations 3] [--dry-run]
python agent.py list        # 読み込めた役割とモデル解決結果を表示
```

## 8. Cline 連携
`.clinerules` を作り、次を書く:
- 大量のファイル読み込み、横断調査、定型実装は、自分で行わず `python agent.py run "..."` に委任する。
- 実行後は `git diff` を読んで確認し、問題があれば修正を指示する。
- 秘密情報(`.env` 等)を読まない・出力しない。

## 9. 完了条件
1. `python -m unittest discover tests` が全て通る(LLM はモック)。
2. `python agent.py list` と `--dry-run` がエラーなく動く。
3. ルート外アクセス、禁止コマンド、孫への委任が拒否されることをテストで確認済み。
4. エスカレーション条件1〜5が、それぞれ期待通りに成立/不成立になることをテストで確認済み(LLMはモック)。昇格回数の上限も確認する。
5. README に、環境変数の設定、使い方、ログの見方を書く。
6. 実APIでの動作確認は、私が環境変数を設定してから行う。あなたは実キーを要求・表示しない。

## 10. 進め方
1. 設計要約を提示して承認を得る。
2. `core/tools.py` とそのテストから実装する(安全制御が最優先)。
3. `core/llm.py` → `core/loop.py` → `core/roles.py` → `core/escalate.py` → `agent.py` の順に進める。
4. 各段階でテストを実行し、結果をそのまま報告する(失敗は隠さない)。
