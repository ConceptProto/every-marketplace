---
name: instructor-elixir
description: This skill should be used when working with Instructor Elixir, a library for structured LLM outputs using Ecto schemas. Use this when implementing predictable AI features, creating structured prompts, configuring LLM providers (OpenAI, Anthropic, Ollama), building validation pipelines, or testing LLM-powered functionality in Elixir applications.
---

# Instructor Elixir Expert

## Overview

Instructor Elixir enables developers to get **structured, validated outputs from LLMs** using Ecto schemas. Instead of parsing free-form text, define your expected output structure and let Instructor handle extraction, validation, and type coercion.

This skill provides comprehensive guidance on:
- Creating Ecto schemas for LLM outputs
- Configuring LLM providers (OpenAI, Anthropic, Ollama)
- Building validation pipelines
- Handling streaming responses
- Testing LLM-powered features
- Production deployment patterns

## Core Capabilities

### 1. Structured Outputs with Ecto Schemas

Define your expected output structure using Ecto embedded schemas:

```elixir
defmodule MyApp.AI.EmailClassification do
  use Ecto.Schema
  use Instructor.Validator

  @primary_key false
  embedded_schema do
    field :category, Ecto.Enum, values: [:technical, :billing, :general, :spam]
    field :priority, Ecto.Enum, values: [:low, :medium, :high, :urgent]
    field :summary, :string
    field :sentiment, :float
  end

  @impl true
  def validate_changeset(changeset) do
    changeset
    |> validate_required([:category, :priority, :summary])
    |> validate_number(:sentiment, greater_than_or_equal_to: -1, less_than_or_equal_to: 1)
    |> validate_length(:summary, max: 200)
  end
end
```

### 2. Basic Usage

```elixir
defmodule MyApp.AI.Classifier do
  alias MyApp.AI.EmailClassification

  def classify_email(subject, body) do
    Instructor.chat_completion(
      model: "gpt-4o-mini",
      response_model: EmailClassification,
      messages: [
        %{
          role: "user",
          content: """
          Classify this email:

          Subject: #{subject}
          Body: #{body}
          """
        }
      ]
    )
  end
end

# Usage
{:ok, classification} = MyApp.AI.Classifier.classify_email(
  "Can't log in",
  "I'm unable to access my account since yesterday"
)

classification.category  # :technical
classification.priority  # :high
classification.summary   # "User experiencing login issues"
```

### 3. Provider Configuration

Configure in `config/config.exs`:

```elixir
# OpenAI (default)
config :instructor,
  adapter: Instructor.Adapters.OpenAI,
  openai: [
    api_key: System.get_env("OPENAI_API_KEY")
  ]

# Anthropic Claude
config :instructor,
  adapter: Instructor.Adapters.Anthropic,
  anthropic: [
    api_key: System.get_env("ANTHROPIC_API_KEY")
  ]

# Local Ollama
config :instructor,
  adapter: Instructor.Adapters.Ollama,
  ollama: [
    base_url: "http://localhost:11434"
  ]
```

### 4. Nested Schemas

For complex outputs, nest schemas:

```elixir
defmodule MyApp.AI.ExtractedInvoice do
  use Ecto.Schema
  use Instructor.Validator

  @primary_key false
  embedded_schema do
    field :invoice_number, :string
    field :date, :date
    field :total, :decimal

    embeds_many :line_items, LineItem do
      field :description, :string
      field :quantity, :integer
      field :unit_price, :decimal
    end

    embeds_one :vendor, Vendor do
      field :name, :string
      field :address, :string
    end
  end
end
```

### 5. Streaming Responses

For real-time UI updates:

```elixir
def stream_analysis(text, callback) do
  Instructor.chat_completion(
    model: "gpt-4o",
    response_model: Analysis,
    stream: true,
    messages: [%{role: "user", content: "Analyse: #{text}"}]
  )
  |> Stream.each(fn
    {:partial, partial} -> callback.({:partial, partial})
    {:ok, result} -> callback.({:complete, result})
    {:error, error} -> callback.({:error, error})
  end)
  |> Stream.run()
end
```

In LiveView:

```elixir
def handle_event("analyse", %{"text" => text}, socket) do
  pid = self()

  Task.start(fn ->
    MyApp.AI.stream_analysis(text, fn update ->
      send(pid, {:analysis_update, update})
    end)
  end)

  {:noreply, assign(socket, :analysing, true)}
end

def handle_info({:analysis_update, {:partial, partial}}, socket) do
  {:noreply, assign(socket, :partial_result, partial)}
end

def handle_info({:analysis_update, {:complete, result}}, socket) do
  {:noreply, assign(socket, analysing: false, result: result)}
end
```

### 6. Validation and Retries

Custom validation with automatic retries:

```elixir
defmodule MyApp.AI.ValidatedOutput do
  use Ecto.Schema
  use Instructor.Validator

  @primary_key false
  embedded_schema do
    field :answer, :string
    field :confidence, :float
    field :sources, {:array, :string}
  end

  @impl true
  def validate_changeset(changeset) do
    changeset
    |> validate_required([:answer, :confidence])
    |> validate_number(:confidence, greater_than: 0, less_than_or_equal_to: 1)
    |> validate_length(:sources, min: 1, max: 5)
    |> validate_answer_quality()
  end

  defp validate_answer_quality(changeset) do
    answer = get_field(changeset, :answer)

    if answer && String.length(answer) < 10 do
      add_error(changeset, :answer, "Answer must be substantive (at least 10 characters)")
    else
      changeset
    end
  end
end

# With retries
Instructor.chat_completion(
  model: "gpt-4o-mini",
  response_model: ValidatedOutput,
  max_retries: 3,
  messages: [%{role: "user", content: "..."}]
)
```

### 7. Testing LLM Code

Use mocks for deterministic tests:

```elixir
defmodule MyApp.AI.ClassifierTest do
  use ExUnit.Case, async: true
  import Mox

  setup :verify_on_exit!

  test "classifies technical emails correctly" do
    expect(InstructorMock, :chat_completion, fn opts ->
      assert opts[:response_model] == EmailClassification
      {:ok, %EmailClassification{category: :technical, priority: :high, summary: "Login issue"}}
    end)

    assert {:ok, result} = MyApp.AI.Classifier.classify_email("Can't log in", "...")
    assert result.category == :technical
  end
end
```

For integration tests with VCR:

```elixir
defmodule MyApp.AI.ClassifierIntegrationTest do
  use ExUnit.Case
  use ExVCR.Mock, adapter: ExVCR.Adapter.Hackney

  test "real API call" do
    use_cassette "classify_email" do
      {:ok, result} = MyApp.AI.Classifier.classify_email(
        "Password reset",
        "How do I reset my password?"
      )

      assert result.category == :technical
    end
  end
end
```

### 8. Production Patterns

#### Rate Limiting

```elixir
defmodule MyApp.AI.RateLimiter do
  use GenServer

  def request(fun) do
    GenServer.call(__MODULE__, {:request, fun}, :infinity)
  end

  @impl true
  def handle_call({:request, fun}, _from, state) do
    # Simple token bucket implementation
    now = System.monotonic_time(:millisecond)
    wait_time = calculate_wait(state, now)

    if wait_time > 0 do
      Process.sleep(wait_time)
    end

    result = fun.()
    {:reply, result, update_state(state, now)}
  end
end
```

#### Error Handling

```elixir
def classify_with_fallback(subject, body) do
  case Instructor.chat_completion(
    model: "gpt-4o-mini",
    response_model: EmailClassification,
    max_retries: 2,
    messages: [%{role: "user", content: "..."}]
  ) do
    {:ok, result} ->
      {:ok, result}

    {:error, %Instructor.ValidationError{}} ->
      # Return safe default on validation failure
      {:ok, %EmailClassification{category: :general, priority: :low, summary: "Unable to classify"}}

    {:error, error} ->
      Logger.error("LLM call failed: #{inspect(error)}")
      {:error, :service_unavailable}
  end
end
```

#### Telemetry Integration

```elixir
# In application.ex
:telemetry.attach(
  "instructor-metrics",
  [:instructor, :chat_completion, :stop],
  &MyApp.Telemetry.handle_instructor_event/4,
  nil
)

# Handler
defmodule MyApp.Telemetry do
  def handle_instructor_event(_event, measurements, metadata, _config) do
    :telemetry.execute(
      [:my_app, :ai, :call],
      %{
        duration: measurements.duration,
        tokens: metadata[:usage][:total_tokens]
      },
      %{model: metadata[:model]}
    )
  end
end
```

## Common Patterns

### Pattern: Multi-Step Analysis

```elixir
defmodule MyApp.AI.Pipeline do
  def analyse(text) do
    with {:ok, classification} <- classify(text),
         {:ok, entities} <- extract_entities(text),
         {:ok, summary} <- summarise(text, classification) do
      {:ok, %{
        classification: classification,
        entities: entities,
        summary: summary
      }}
    end
  end

  defp classify(text) do
    Instructor.chat_completion(
      model: "gpt-4o-mini",
      response_model: Classification,
      messages: [%{role: "user", content: "Classify: #{text}"}]
    )
  end

  defp extract_entities(text) do
    Instructor.chat_completion(
      model: "gpt-4o-mini",
      response_model: Entities,
      messages: [%{role: "user", content: "Extract entities: #{text}"}]
    )
  end

  defp summarise(text, classification) do
    Instructor.chat_completion(
      model: "gpt-4o-mini",
      response_model: Summary,
      messages: [
        %{role: "system", content: "Summarise for #{classification.category} category"},
        %{role: "user", content: text}
      ]
    )
  end
end
```

### Pattern: Conditional Model Selection

```elixir
def process(text, opts \\ []) do
  complexity = estimate_complexity(text)

  model = case complexity do
    :simple -> "gpt-4o-mini"
    :moderate -> "gpt-4o"
    :complex -> "claude-3-5-sonnet-latest"
  end

  Instructor.chat_completion(
    model: model,
    response_model: opts[:response_model] || DefaultOutput,
    messages: [%{role: "user", content: text}]
  )
end
```

## Installation

```elixir
# mix.exs
def deps do
  [
    {:instructor, "~> 0.1"}
  ]
end
```

## When to Use This Skill

Trigger this skill when:
- Implementing structured LLM outputs in Elixir
- Creating Ecto schemas for AI responses
- Configuring Instructor providers
- Building validation for LLM outputs
- Testing LLM functionality
- Adding streaming to LiveView
- Debugging Instructor issues
