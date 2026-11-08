# 第3回 解答

## 演習

- 第1回（素のLLM）とエージェントの回答の違い：第1回の素のLLMは講座固有の用語を検索せずに答えたのに対し、エージェントは用語集を検索してから答えた。今回の「3レベル」では、基礎・実装・発展の3段階を説明し、どれか1つに取り組めば宿題が完了すると回答した。
- `--trace` で見えたツール呼び出し（ツール名と引数）：`search_glossary({'query': 'LoRA'})`、`search_glossary({'query': 'QLoRA'})`
- 1つの質問での LLM 呼び出し回数（`requests`）：2回（`tool_calls` は2回）
- 追加したツールと、docstring に書いた説明：`make_quiz(lecture: int)`。指定した回の用語集から確認問題を1問作り、「第2回の確認問題を出して」のような復習用の質問に使うこと、`lecture` には出題する回を指定することを説明した。`app/agent.py` に登録した。
## 宿題

- 取り組んだレベル：発展（a）ADK で作り直す
- やったこと：`app/agent.py` の `INSTRUCTIONS` と `app/tools.py` の `search_glossary`、`get_lecture_summary`、`make_quiz` をそのまま利用し、`adk_agent/glossary_agent/agent.py` で ADK の `Agent` を組み立てた。指示は `instruction` に、ツールは `tools` に渡した。
- 結果：Pydantic AI版は `Agent` にツールを登録して実行するのに対し、ADK版は `Agent` の生成時に `instruction` と `tools` をまとめて渡す書き方だった。どちらも用語検索や確認問題の作成という同じ機能を使える。
