# OpenAI Agents SDK

OpenAI Agents SDKは、マルチエージェントワークフローを構築するための軽量かつパワフルなフレームワークです。

<img src="https://cdn.openai.com/API/docs/images/orchestration.png" alt="Agents Tracing UIの画像" style="max-height: 803px;">

### 主要概念:

1. [**Agents**](https://openai.github.io/openai-agents-python/agents): 指示、ツール、ガードレール、およびハンドオフで構成されたLLM
2. [**Handoffs**](https://openai.github.io/openai-agents-python/handoffs/): エージェント間で制御を移すために使用されるAgents SDKの特殊なツール呼び出し
3. [**Guardrails**](https://openai.github.io/openai-agents-python/guardrails/): 入力と出力の検証のための設定可能な安全チェック
4. [**Tracing**](https://openai.github.io/openai-agents-python/tracing/): エージェントの実行を追跡する組み込み機能で、ワークフローの表示、デバッグ、最適化が可能

[examples](examples)ディレクトリでSDKの動作を確認し、詳細については[ドキュメント](https://openai.github.io/openai-agents-python/)をお読みください。

特筆すべきは、このSDKがOpenAI Chat Completions API形式をサポートするあらゆるモデルプロバイダーと[互換性がある](https://openai.github.io/openai-agents-python/models/)ことです。

## Get started

1. Set up your Python environment

```
# 仮想環境を作成します。これにより、プロジェクト専用の隔離された環境が作られます
# 他のプロジェクトとのパッケージの競合を防ぐために重要です
python -m venv env

# 作成した仮想環境を有効化します
# これにより、インストールするパッケージがこの環境内だけに適用されます
source env/bin/activate
```

2. Install Agents SDK

```
# OpenAI Agents SDKをインストールします
# pipはPythonのパッケージ管理ツールで、必要なライブラリを簡単にインストールできます
pip install openai-agents

# 音声機能が必要な場合は、以下のオプショングループを含めてインストールします
# 角括弧内の'voice'は、音声関連の追加依存関係をインストールするための指定です
pip install 'openai-agents[voice]'
```

## Hello world example

```python
# OpenAI Agents SDKから必要なクラスをインポートします
# Agentクラス：AIエージェントを定義するためのクラスです
# Runnerクラス：エージェントを実行するためのクラスです
from agents import Agent, Runner

# 「Assistant」という名前のエージェントを作成します
# instructionsパラメータで、エージェントの振る舞いを指定しています
# この場合、「役立つアシスタント」として振る舞うよう指示しています
agent = Agent(name="Assistant", instructions="You are a helpful assistant")

# エージェントを実行します。run_sync関数は同期的にエージェントを実行します
# 第一引数はエージェント、第二引数はエージェントへの入力（プロンプト）です
# この場合、プログラミングの再帰に関する俳句を作るよう指示しています
result = Runner.run_sync(agent, "Write a haiku about recursion in programming.")

# エージェントの最終出力を表示します
# final_outputプロパティには、エージェントの最終的な応答が格納されています
print(result.final_output)

# 以下は実行結果の例です：
# Code within the code,
# Functions calling themselves,
# Infinite loop's dance.
```

(_このコードを実行するには、`OPENAI_API_KEY`環境変数を設定する必要があります_)

(_Jupyterノートブックユーザーの場合は、[hello_world_jupyter.py](examples/basic/hello_world_jupyter.py)を参照してください_)

## Handoffs example

```python
# 必要なモジュールをインポートします
# agents：OpenAI Agents SDKのモジュール
# asyncio：非同期処理を行うためのPythonの標準ライブラリ
from agents import Agent, Runner
import asyncio

# スペイン語を話すエージェントを作成します
# このエージェントはスペイン語だけで応答するよう指示されています
spanish_agent = Agent(
    name="Spanish agent",
    instructions="You only speak Spanish.",
)

# 英語を話すエージェントを作成します
# このエージェントは英語だけで応答するよう指示されています
english_agent = Agent(
    name="English agent",
    instructions="You only speak English",
)

# 振り分けを行うエージェントを作成します
# このエージェントはリクエストの言語に基づいて適切なエージェントに引き継ぎます
# handoffsパラメータには、引き継ぎ可能なエージェントのリストを指定します
triage_agent = Agent(
    name="Triage agent",
    instructions="Handoff to the appropriate agent based on the language of the request.",
    handoffs=[spanish_agent, english_agent],
)


# メイン関数を定義します（非同期関数）
async def main():
    # トリアージエージェントを実行します
    # スペイン語の入力「こんにちは、お元気ですか？」を与えています
    result = await Runner.run(triage_agent, input="Hola, ¿cómo estás?")
    
    # 最終結果を表示します
    # トリアージエージェントがスペイン語と判断し、spanish_agentに引き継いだ結果が表示されます
    print(result.final_output)
    # 出力例：¡Hola! Estoy bien, gracias por preguntar. ¿Y tú, cómo estás?


# このスクリプトが直接実行された場合にのみmain関数を実行します
# モジュールとしてインポートされた場合は実行されません
if __name__ == "__main__":
    # 非同期関数を実行するためのasyncioのランタイムを起動します
    asyncio.run(main())
```

## Functions example

```python
# 非同期処理のためのasyncioモジュールをインポートします
import asyncio

# OpenAI Agents SDKから必要なクラスと関数をインポートします
# function_tool：Pythonの関数をエージェントが使用できるツールに変換するデコレータです
from agents import Agent, Runner, function_tool


# function_toolデコレータを使用して、関数をエージェントのツールとして定義します
# この関数は都市名を受け取り、その都市の天気情報を返します（ここでは常に「晴れ」と返します）
@function_tool
def get_weather(city: str) -> str:
    return f"The weather in {city} is sunny."


# エージェントを作成します
# toolsパラメータに、エージェントが使用できるツール（関数）のリストを指定します
agent = Agent(
    name="Hello world",
    instructions="You are a helpful agent.",
    tools=[get_weather],
)


# メイン関数を定義します（非同期関数）
async def main():
    # エージェントを実行します
    # 「東京の天気はどうですか？」という質問を入力しています
    result = await Runner.run(agent, input="What's the weather in Tokyo?")
    
    # 最終結果を表示します
    # エージェントはget_weather関数を使用して、東京の天気情報を取得し応答します
    print(result.final_output)
    # 出力例：The weather in Tokyo is sunny.


# このスクリプトが直接実行された場合にのみmain関数を実行します
if __name__ == "__main__":
    # 非同期関数を実行するためのasyncioのランタイムを起動します
    asyncio.run(main())
```

## The agent loop

`Runner.run()`を呼び出すと、最終出力が得られるまでループを実行します。

1. エージェントに設定されているモデルと設定、およびメッセージ履歴を使用してLLMを呼び出します。
2. LLMはレスポンスを返し、これにはツール呼び出しが含まれる場合があります。
3. レスポンスに最終出力がある場合（詳細は以下を参照）、それを返してループを終了します。
4. レスポンスにhandoff（引き継ぎ）がある場合、エージェントを新しいエージェントに設定して手順1に戻ります。
5. ツール呼び出し（ある場合）を処理し、ツールのレスポンスメッセージを追加します。その後、手順1に戻ります。

ループの実行回数を制限するために`max_turns`パラメータを使用できます。

### Final output

最終出力はエージェントのループで生成される最後の結果です。

1. エージェントに `output_type` を設定している場合、LLMがその型に合致する何かを返したときが最終出力となります。これには[構造化出力](https://platform.openai.com/docs/guides/structured-outputs)を使用します。
2. `output_type` が設定されていない場合（つまり、プレーンテキストの応答の場合）、ツール呼び出しやhandoffを含まない最初のLLM応答が最終出力と見なされます。

結果として、エージェントループの考え方は以下のようになります：

1. 現在のエージェントに `output_type` がある場合、エージェントがその型に一致する構造化出力を生成するまでループが実行されます。
2. 現在のエージェントに `output_type` がない場合、現在のエージェントがツール呼び出し/handoffを含まないメッセージを生成するまでループが実行されます。

## Common agent patterns

Agents SDKは非常に柔軟に設計されており、決定論的フロー、反復ループなど、幅広いLLMワークフローをモデル化することができます。[`examples/agent_patterns`](examples/agent_patterns)にあるサンプルをご覧ください。

## Tracing

Agents SDKは自動的にエージェントの実行をトレースし、エージェントの動作を追跡およびデバッグすることを容易にします。トレーシングは拡張性を考慮して設計されており、カスタムスパンや[Logfire](https://logfire.pydantic.dev/docs/integrations/llms/openai/#openai-agents)、[AgentOps](https://docs.agentops.ai/v1/integrations/agentssdk)、[Braintrust](https://braintrust.dev/docs/guides/traces/integrations#openai-agents-sdk)、[Scorecard](https://docs.scorecard.io/docs/documentation/features/tracing#openai-agents-sdk-integration)、[Keywords AI](https://docs.keywordsai.co/integration/development-frameworks/openai-agent)など様々な外部送信先をサポートしています。トレーシングのカスタマイズや無効化の詳細については、[Tracing](http://openai.github.io/openai-agents-python/tracing)を参照してください。こちらには[外部トレーシングプロセッサ](http://openai.github.io/openai-agents-python/tracing/#external-tracing-processors-list)のより大きなリストも含まれています。

## Development (only needed if you need to edit the SDK/examples)

0. Ensure you have [`uv`](https://docs.astral.sh/uv/) installed.

```bash
# uvがインストールされているか確認します
# uvは高速なPythonパッケージインストーラーおよび環境マネージャーです
uv --version
```

1. Install dependencies

```bash
# 依存関係をインストールします
# makeコマンドを使用してsyncターゲットを実行します
# これにより、プロジェクトに必要なすべての依存関係が適切にインストールされます
make sync
```

2. (After making changes) lint/test

```
# テストを実行して、コードが正しく動作することを確認します
make tests  # run tests

# 型チェッカーを実行して、型の問題がないことを確認します
make mypy   # run typechecker

# リンターを実行して、コードスタイルの問題がないことを確認します
make lint   # run linter
```

## Acknowledgements

私たちはオープンソースコミュニティの素晴らしい取り組みに感謝します。特に以下のプロジェクトに感謝します：

-   [Pydantic](https://docs.pydantic.dev/latest/) (データ検証) と [PydanticAI](https://ai.pydantic.dev/) (高度なエージェントフレームワーク)
-   [MkDocs](https://github.com/squidfunk/mkdocs-material)
-   [Griffe](https://github.com/mkdocstrings/griffe)
-   [uv](https://github.com/astral-sh/uv) と [ruff](https://github.com/astral-sh/ruff)

私たちは、コミュニティの他のメンバーが私たちのアプローチを拡張できるよう、Agents SDKをオープンソースフレームワークとして構築し続けることを約束します。
