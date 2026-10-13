# agent-runner 使い方

サブエージェントに仕事を委任しながら、コード実装とバグ調査を進める CLI です。
Python 3 と標準ライブラリだけで動きます。

> このドキュメントは `SPEC.md` の設計に基づいています。実装後に、オプション名や出力が異なる場合は、実装側に合わせて修正してください。

## 1. 準備

### 1.1 `.env` を作る
`.env.example` をコピーして `.env` を作り、値を入れます。`.env` は Git に入れないでください。キーをチャットや画面に貼らないでください。

```
LLM_BASE_URL=<エンドポイントURL>     # /chat/completions の手前まで(例: https://…/v1)
LLM_API_KEY=<キー>

MODEL_SOL=<solの正式なモデル名>
MODEL_LUNA=<lunaの正式なモデル名>
MODEL_MINI=<mini4の正式なモデル名>
# 任意
# MODEL_CODEX=<codex5.3の正式なモデル名>
# MODEL_ADVISOR=<別のモデルを相談役にしたいときだけ。未設定なら sol>
# MODEL_FALLBACK=<別名が未設定のときの代替>
# MODEL_SOL_EFFORT=high      # reasoning effort。受け付けないモデルでは自動で外れる
# MODEL_CODEX_API=responses  # /responses 専用のモデルにだけ指定
```

### 1.2 動作確認(APIを呼ばない)
```
python agent.py list
python agent.py run "テスト" --dry-run
```
- `list` は、読み込めた役割と、別名がどの実モデルに解決されたかを表示します。
- `--dry-run` は、LLM を呼ばずに、定義の読み込みとツール定義の検証だけを行います。

## 2. 基本の使い方

```
python agent.py run "仕事の内容" [オプション]
```

例:
```
python agent.py run "src/ の構成を要約して"
python agent.py run "tests/test_parser.py の失敗原因を調べて"
python agent.py run "utils.py に日付変換関数を追加し、テストも書いて" --root workspaces/myproj
```

仕事の書き方のコツ:
- 対象のファイルやフォルダ、期待する結果、完了条件(例: テストが通る)を書く。
- 1回の `run` に詰め込みすぎず、仕事を小さく分ける。
- 秘密情報(`.env`、鍵、トークン)を含めない。ツールからも読めないようになっています。

## 3. オプション

| オプション | 既定 | 内容 |
|---|---|---|
| `--root` | カレント | 作業ルート。これより外のパスはツールが拒否する |
| `--role` | `main` | 最初に動かす役割 |
| `--max-parallel` | 2 | 並列に動かす子エージェントの数 |
| `--max-total-calls` | 60 | 1回の `run` で、LLM を呼ぶ回数の上限 |
| `--max-escalations` | 3 | advisor に相談する回数の上限 |
| `--dry-run` | なし | API を呼ばずに検証だけ行う |

## 4. 役割とモデル

| 役割 | 既定モデル | 権限 | 担当 |
|---|---|---|---|
| main | sol | 委任・読み取り | 仕事の分解、委任、最終回答 |
| explorer | mini | 読み取りのみ | 探索、要約 |
| coder | luna | 読み書き・コマンド | 実装、テスト実行 |
| reviewer | luna | 読み取り(`git diff`) | 変更のレビュー |
| advisor | sol(effort 最大) | 読み取りのみ | 失敗時の相談 |

役割の定義は `agents/*.md` にあります。先頭の frontmatter でモデルとツールを変えられます。

```
---
name: coder
model: luna          # mini / luna / sol / codex / advisor
tools: [list_dir, read_file, grep, write_file, replace_in_file, run_cmd]
max_turns: 25
---
(システムプロンプト)
```

- `model:` の別名は、環境変数 `MODEL_<別名の大文字>` で実モデル名に解決されます。
- 子エージェントへの委任は1段までです。子が孫を作ることはできません。

## 5. 失敗時の動き(エスカレーション)

次のいずれかが成立すると、そのエージェントを止めて advisor(sol)に相談します。

1. 同じコマンドが連続 2 回失敗した。
2. `max_turns` の半分を超えても、変更とテスト結果が動かない。
3. 同じツール呼び出しが 3 回続いた。
4. reviewer が重大度「高」の指摘を出した。
5. 重要領域(認証・決済・データ移行・削除処理)に触れた。

advisor は方針だけを返し、コードは編集しません。方針を受けて元のエージェントが再開します。
上限(`--max-escalations`)に達したか、advisor の後も同じ条件で失敗した場合は、自動では続けず、状況を報告して止まります。

重要領域の判定キーワードは `escalate.json` で変更できます。

## 6. 安全制御

- ファイルの読み書きは `--root` の内側だけです。`..`、絶対パス、シンボリックリンク経由の外部アクセスは拒否されます。
- `.env`、`*.key`、`*.pem`、`id_rsa*` は読めません。
- `run_cmd` は許可リスト方式です(`python`、`git status/diff/log`、`pytest`、`python -m unittest` など)。`rm`、`del`、`curl`、`pip`、`git push`、`git reset --hard` などは拒否されます。タイムアウトは 60 秒です。
- ターン数と合計呼び出し回数に上限があり、超えると途中経過を返して止まります。

## 7. ログと結果の確認

- `runs/<日時>/<役割>-<番号>.jsonl` に、各ターンのリクエスト要約、ツール呼び出し、結果(切り詰め後)が保存されます。API キーは含まれません。
- 終了時に、使用ターン数、モデル別の呼び出し回数、トークン使用量、エスカレーションの有無と理由が表示されます。
- 実行後は、必ず変更内容を確認してください。

```
git diff
python -m unittest discover tests
```

## 8. Cline から使う

`.clinerules` に、次の方針が書かれています。

- 大量のファイル読み込み、横断調査、定型実装は、Cline 自身が行わず `python agent.py run "..."` に委任する。
- 実行後は `git diff` を読んで確認する。
- 秘密情報を読まない・出力しない。

Cline に「この調査を agent.py に委任して」と頼むだけで、同じ流れになります。

## 9. トラブルシューティング

| 症状 | 確認すること |
|---|---|
| 起動時に「別名が未設定」と出る | `.env` に `MODEL_<別名>` があるか。`python agent.py list` で解決結果を見る |
| 401 / 403 | `LLM_API_KEY` の値、エンドポイントの権限 |
| 404 | `LLM_BASE_URL` の末尾(`/v1` までで、`/chat/completions` は付けない) |
| 429 / 5xx が続く | 自動で 3 回まで再試行される。`--max-parallel 1` にして再実行 |
| `effort` のエラー | そのモデルが対応していない。`MODEL_<別名>_EFFORT` を外す(通常は自動で外れる) |
| コマンドが拒否される | 許可リストの対象外。必要なら `core/tools.py` の許可リストに追加する |
| 途中で止まった | `runs/` の最新ログと終了時のサマリを見る。仕事を小さく分けて再実行 |

## 10. 費用の目安と注意

- 呼び出し回数は `--max-total-calls` で制限できます。最初は小さい値(例: 20)で試してください。
- 探索は mini、実装は luna、設計と相談は sol という分担にして、sol の使用を絞っています。
- 並列数を上げると、レート制限に当たりやすくなります。
