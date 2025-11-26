---
name: hex-package-writer
description: Write Hex packages following Elixir community best practices. Use when creating new Elixir packages, refactoring existing packages, designing package APIs, or when the user wants clean, minimal, production-ready Elixir library code. Triggers on requests like "create a package", "write an Elixir library", "design a Hex package API".
---

# Hex Package Writer

Write Hex packages following Elixir community best practices and conventions from José Valim and the Elixir core team.

## Core Philosophy

**Simplicity and clarity.** Minimal dependencies. Explicit code with clear function signatures. OTP integration where appropriate. Every pattern serves production use cases.

## Project Structure

Every Hex package follows this structure:

```
package_name/
├── lib/
│   ├── package_name.ex           # Main module with public API
│   └── package_name/
│       ├── application.ex        # Application (if needed)
│       └── ...                   # Internal modules
├── test/
│   ├── test_helper.exs
│   └── package_name_test.exs
├── mix.exs
├── README.md
├── LICENSE
├── .formatter.exs
└── .gitignore
```

## Entry Point Structure

Main module in `lib/package_name.ex`:

```elixir
defmodule PackageName do
  @moduledoc """
  Brief description of what the package does.

  ## Usage

      PackageName.function(arg)

  ## Configuration

      config :package_name,
        option: "value"
  """

  @doc """
  Brief description of function.

  ## Examples

      iex> PackageName.function("input")
      {:ok, "output"}

  """
  @spec function(String.t()) :: {:ok, String.t()} | {:error, term()}
  def function(input) do
    # implementation
  end
end
```

## Configuration Pattern

Use Application config with runtime.exs support:

```elixir
defmodule PackageName do
  @doc false
  def config(key, default \\ nil) do
    Application.get_env(:package_name, key, default)
  end
end
```

In `config/runtime.exs`:

```elixir
import Config

config :package_name,
  api_key: System.get_env("PACKAGE_NAME_API_KEY"),
  timeout: String.to_integer(System.get_env("PACKAGE_NAME_TIMEOUT", "5000"))
```

## Error Handling

Use tagged tuples and custom exceptions:

```elixir
defmodule PackageName.Error do
  @moduledoc "Base error for PackageName operations"
  defexception [:message, :reason]

  @impl true
  def exception(opts) do
    message = Keyword.get(opts, :message, "PackageName error")
    reason = Keyword.get(opts, :reason)
    %__MODULE__{message: message, reason: reason}
  end
end
```

Return values:

```elixir
# Good - explicit tagged tuples
{:ok, result}
{:error, %PackageName.Error{message: "Failed", reason: :timeout}}

# For simple cases, raise with bang functions
def function!(input) do
  case function(input) do
    {:ok, result} -> result
    {:error, error} -> raise error
  end
end
```

## mix.exs Pattern

```elixir
defmodule PackageName.MixProject do
  use Mix.Project

  @version "1.0.0"
  @source_url "https://github.com/user/package_name"

  def project do
    [
      app: :package_name,
      version: @version,
      elixir: "~> 1.14",
      start_permanent: Mix.env() == :prod,
      deps: deps(),
      package: package(),
      docs: docs(),
      name: "PackageName",
      description: "Brief description",
      source_url: @source_url
    ]
  end

  def application do
    [
      extra_applications: [:logger]
    ]
  end

  defp deps do
    [
      {:ex_doc, "~> 0.31", only: :dev, runtime: false},
      {:credo, "~> 1.7", only: [:dev, :test], runtime: false},
      {:dialyxir, "~> 1.4", only: [:dev, :test], runtime: false}
    ]
  end

  defp package do
    [
      licenses: ["MIT"],
      links: %{"GitHub" => @source_url},
      files: ~w(lib .formatter.exs mix.exs README.md LICENSE)
    ]
  end

  defp docs do
    [
      main: "readme",
      extras: ["README.md"],
      source_ref: "v#{@version}",
      source_url: @source_url
    ]
  end
end
```

## Testing Patterns

Use ExUnit with doctests:

```elixir
# test/test_helper.exs
ExUnit.start()

# test/package_name_test.exs
defmodule PackageNameTest do
  use ExUnit.Case, async: true
  doctest PackageName

  describe "function/1" do
    test "returns ok tuple on success" do
      assert {:ok, _result} = PackageName.function("valid")
    end

    test "returns error tuple on failure" do
      assert {:error, %PackageName.Error{}} = PackageName.function("invalid")
    end
  end
end
```

## Typespec Conventions

Always add typespecs to public functions:

```elixir
@type option :: {:timeout, pos_integer()} | {:retries, non_neg_integer()}
@type options :: [option()]

@spec function(String.t(), options()) :: {:ok, map()} | {:error, Error.t()}
def function(input, opts \\ []) do
  # implementation
end
```

## OTP Integration (When Needed)

For stateful packages, use proper supervision:

```elixir
defmodule PackageName.Application do
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      {PackageName.Worker, []}
    ]

    opts = [strategy: :one_for_one, name: PackageName.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

## Anti-Patterns to Avoid

- Global mutable state (use GenServer or ETS instead)
- Implicit dependencies (always explicit in deps)
- Missing typespecs on public functions
- Missing @moduledoc and @doc
- Hardcoded configuration values
- Synchronous calls for async operations
- Deep module nesting
- Unnecessary macros (prefer functions)

## Publishing to Hex

```bash
# Ensure you're logged in
mix hex.user auth

# Build and publish
mix hex.build
mix hex.publish

# Publish docs
mix hex.publish docs
```

## Documentation Guidelines

- Every public module needs `@moduledoc`
- Every public function needs `@doc` with examples
- Use `## Examples` section with `iex>` for doctests
- Link to related functions with `see/1`
- Keep sentences under 80 characters
