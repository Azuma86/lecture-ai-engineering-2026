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

- 取り組んだレベル：基礎 / 実装 / 発展
- やったこと：
- 結果：
- 気づき・疑問：
