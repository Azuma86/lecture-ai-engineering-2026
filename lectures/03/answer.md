# 第3回 解答

## 演習

- 第1回（素のLLM）とエージェントの回答の違い：

  第1回の解答では、モデルは AIE2026 の資料を確認できず、講義資料や講師に確認するよう案内していた。

  今回 `search_glossary("3レベル")` を直接実行すると、「基礎（講義内容の確認）」「実装（自分でコードを書く）」「発展（調査や試行）」の3段階があり、どれか1つに取り組めば宿題が完了するという講座固有の定義を取得できた。

- `--trace` で見えたツール呼び出し（ツール名と引数）：

  `TestModel(call_tools=["make_quiz"], seed=2)` を使い、`print_trace(result)` で次の呼び出しと、問題・答えの返却を確認した。

  ```text
  [tool call]   make_quiz({'lecture': 2})
  ```

  この呼び出しはテスト側で指定したもので、モデルが質問を解釈して自分で選んだ結果ではない。

  演習手順3の「LoRAとQLoRAの違いは？」に対する実モデルの `--trace` は未実行。用語集を直接検索すると、どちらも第2回の用語で、QLoRA はベースモデルを量子化したうえで LoRA 学習を行い、メモリを節約する手法だと確認できた。

- 1つの質問での LLM 呼び出し回数（`requests`）：

  上記の TestModel のトレースでは `requests=2`、`tool_calls=1` だった。最初のモデル処理でツール呼び出しを返し、ツールの結果を受け取った後のモデル処理で出力を返すループを確認できた。

  これはダミーモデルの処理回数で、外部の LLM API を呼んだ回数は0回。実モデルの呼び出し回数やトークン使用量は未測定。

- 追加したツールと、docstring に書いた説明：

  `app/tools.py` に `make_quiz(lecture: int) -> str` を追加した。指定した回の用語を用語集から集め、`random.choice` で1つ選び、`question` と `answer` を持つ JSON 文字列を返す。答えには用語集の `definition` をそのまま使う。

  第0回や第4回など、用語が存在しない回では、用語が見つからなかったことと別の回を指定する案内を返す。

  docstring は次のとおり。

  ```text
  指定した回の用語集から確認問題を1問作り、問題と答えを返す。

  「第2回の確認問題を出して」のように、復習用の問題を求められたときに使う。
  lecture には出題する回を1始まりの整数で指定する。
  ```

  `app/agent.py` で `agent.tool_plain(tools.make_quiz)` を登録した。`INSTRUCTIONS` に、確認問題にはこのツールを使い、最初は問題だけを示す指示を追加した。実モデルによるツールの自律的な選択と、答えを伏せる指示への追従は未検証。

  Python 3.12.12 と `uv.lock` の固定された依存パッケージで検証した。依存関係の定義・固定版は変更していない。`uv run --frozen python -m app.agent --list-tools` では、`search_glossary`、`get_lecture_summary`、`make_quiz` の3つが表示され、`make_quiz` の説明文と整数型の `lecture` 引数も確認できた。

  `uv run --frozen pytest -q` は **20 passed**。第1・2・3・5回の問題と答えが指定した回の用語集に対応すること、第-1・0・4・99回では例外で終了せず案内を返すこと、TestModel から `make_quiz(lecture=2)` を呼び出して戻り値を受け取れることを確認した。既存のエージェントと MCP サーバーのテストも通った。

  TestModel のトレースは、次のコードで再確認できる（API キー不要）。出題する用語はランダムなので実行ごとに変わりうる。

  ```bash
  uv run --frozen python - <<'PY'
  from pydantic_ai.models.test import TestModel
  from app.agent import agent, print_trace

  model = TestModel(call_tools=["make_quiz"], seed=2)
  result = agent.run_sync("第2回の内容から確認問題を1問出して", model=model)
  print(result.output)
  print_trace(result)
  PY
  ```

  TestModel の最終出力はツールの返り値をまとめたダミー出力で、実際の出題文を生成するものではない。

## 宿題

- 取り組んだレベル：発展（a）ADK で作り直す
- やったこと：

  `adk_agent/glossary_agent/agent.py` と `__init__.py` を追加し、ADK の `Agent` を `root_agent` として定義した。

  指示は `app.agent.INSTRUCTIONS`、ツールは `app.tools` の `search_glossary`・`get_lecture_summary`・演習で追加した `make_quiz` をそのまま読み込む。モデル名も `app.config.MODEL` を共有し、Pydantic AI 版と同じ既定値と `AIE_MODEL` による切り替えを使う。

  `app/`・`pyproject.toml`・`uv.lock` は変更せず、ADK は `uv run --with google-adk` で一時的に追加した。ADK が作るローカルの会話データをコミットしないよう、`.gitignore` に `.adk/` を追加した。

- 結果：

  Python 3.12.12・google-adk 2.14.0 で、次を確認した。

  - `root_agent` を読み込めること。名前は `glossary_agent`、モデルは `gemini-3.5-flash-lite` だった。
  - 指示とモデル名が共通設定に一致すること。3つの関数から ADK のツール定義を生成でき、docstring と、`query` が文字列・`lecture` が整数という引数の型が反映されること。
  - `adk run --in_memory adk_agent/glossary_agent` で `Running agent glossary_agent, type exit to exit.` と `[user]:` が表示され、`exit` で正常終了すること。この起動確認では質問を送信していない。
  - 既存テストは `uv run --frozen pytest -q` で **20 passed**。

  この実行環境では `GEMINI_API_KEY` が未設定だったため、実モデルの回答比較は未実施。以下は比較を進めるための手順で、実行結果ではない。

  Codespaces など、`uv` と環境変数 `GEMINI_API_KEY` が使える環境で、リポジトリ直下から実行する。

  ```bash
  # Pydantic AI 版：回答とツール呼び出しを記録する
  uv run --frozen python -m app.agent --trace "LoRAとQLoRAの違いは？"

  # ADK 版：起動後に同じ質問を入力する。終了は exit
  PYTHONPATH=. uv run --frozen --with google-adk adk run adk_agent/glossary_agent
  ```

  比較する質問と確認点：

  | 質問 | 確認点 | Pydantic AI 版 | ADK 版 |
  |---|---|---|---|
  | AIE2026の講座の宿題にある「3レベル」とは何ですか？ | 基礎・実装・発展の説明と、どれか1つで完了すること | 未実施 | 未実施 |
  | LoRAとQLoRAの違いは？ | 量子化の説明、第2回の表示、説明の詳しさ | 未実施 | 未実施 |
  | 第2回の内容から確認問題を1問出して | 問題を1問出し、最初は答えを伏せること | 未実施 | 未実施 |

  各質問は ADK を起動し直して会話の履歴がない状態で比べる。確認問題の用語はランダムに選ばれるので、用語そのものの一致ではなく、指定した回から出題されることと答えの伏せ方を見る。

- 気づき・疑問：

  コード上の違いは、Pydantic AI が `Agent(instructions=INSTRUCTIONS)` と `agent.tool_plain(...)` で指示・ツールを登録するのに対し、ADK は `Agent(instruction=INSTRUCTIONS, tools=[...])` にまとめて渡す点だった。`instructions` と `instruction` の引数名にも違いがある。

  Pydantic AI 版は `run_sync` に実行時のモデルを渡し、自前の CLI から呼ぶ。今回の ADK 版はエージェント定義にモデルを指定し、`root_agent` を `adk run` に読み込ませる。指示とツールの実装を共有できるため、エージェントを組み立てる部分だけを差し替えられた。

  同じ指示・ツール・モデルでも回答が同じになるかは、今回の起動確認だけでは判断できない。実モデルで実行後、上の表に観察した回答を記入する必要がある。

  参照：[ADK Python Quickstart](https://adk.dev/get-started/python/)、[Function tools](https://adk.dev/tools-custom/function-tools/)、[Command Line](https://adk.dev/runtime/command-line/)。
