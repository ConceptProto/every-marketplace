---
name: elixir-style
description: Comprehensive Elixir coding standards based on official Elixir documentation. Use this skill when writing pure Elixir code, reviewing Elixir modules, or learning Elixir best practices. Covers anti-patterns, naming conventions, typespecs, pattern matching, guards, and meta-programming. Triggers on Elixir code generation, refactoring requests, or questions about Elixir conventions.
---

# Elixir Style Guide

Write Elixir code following official language conventions and best practices from the Elixir core team.

## Core Principles

1. **Explicit over implicit**: Pattern matching makes explicit code readable
2. **Let it crash**: Use supervisors for fault tolerance, don't over-handle errors
3. **Immutability by default**: Data transforms through pipelines, never mutates
4. **Small functions**: Each function should do one thing well
5. **Pipeline clarity**: Pipes should read like a story

## Quick Reference

### Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Module | `PascalCase` | `MyApp.UserContext` |
| Function | `snake_case` | `create_user/2` |
| Variable | `snake_case` | `user_params` |
| Atom | `snake_case` | `:user_created` |
| Constant (module attr) | `@snake_case` | `@default_timeout` |
| Predicate function | ends with `?` | `valid?/1` |
| Unsafe function | ends with `!` | `fetch!/2` |

### Top 10 Anti-Patterns to Avoid

1. **Dynamic atom creation** - Never use `String.to_atom/1` with user input
2. **Working with invalid data** - Validate at boundaries, not throughout
3. **Complex branching** - Prefer pattern matching over nested conditionals
4. **Unnecessary macros** - Use functions when possible
5. **Spawning unsupervised processes** - Always use supervisors
6. **Using processes for code organisation** - Processes are for runtime properties
7. **Large modules** - Split modules exceeding 300 lines
8. **Missing typespecs** - Add `@spec` to public functions
9. **Overly defensive code** - Let it crash, trust your data
10. **Ignoring compiler warnings** - Treat warnings as errors

### Pattern Matching Best Practices

```elixir
# GOOD: Pattern match in function heads
def process(%User{status: :active} = user), do: activate(user)
def process(%User{status: :pending} = user), do: send_reminder(user)
def process(%User{}), do: {:error, :invalid_status}

# AVOID: Nested conditionals
def process(user) do
  if user.status == :active do
    activate(user)
  else
    if user.status == :pending do
      send_reminder(user)
    else
      {:error, :invalid_status}
    end
  end
end
```

### Guard Expressions

```elixir
# Common guards
def fetch(id) when is_integer(id) and id > 0, do: Repo.get(User, id)
def fetch(_), do: {:error, :invalid_id}

# Custom guards (use defguard)
defguard is_positive(value) when is_integer(value) and value > 0
```

### Typespec Essentials

```elixir
@type user :: %User{
  id: pos_integer(),
  name: String.t(),
  email: String.t(),
  status: :active | :pending | :inactive
}

@spec create_user(map()) :: {:ok, user()} | {:error, Ecto.Changeset.t()}
def create_user(attrs) do
  %User{}
  |> User.changeset(attrs)
  |> Repo.insert()
end
```

## Detailed References

For comprehensive coverage of specific topics, consult these references:

### Anti-Patterns

Elixir's official anti-pattern documentation, organised by category:

- **[Code Anti-Patterns](references/anti-patterns/code-anti-patterns.md)** - 11 patterns affecting code quality
- **[Design Anti-Patterns](references/anti-patterns/design-anti-patterns.md)** - 7 patterns for architectural issues
- **[Process Anti-Patterns](references/anti-patterns/process-anti-patterns.md)** - 4 patterns for OTP/process management
- **[Macro Anti-Patterns](references/anti-patterns/macro-anti-patterns.md)** - 5 patterns for metaprogramming issues

### Language Conventions

- **[Naming Conventions](references/conventions/naming-conventions.md)** - Complete naming standards
- **[Patterns and Guards](references/conventions/patterns-and-guards.md)** - Pattern matching and guard expressions
- **[Typespecs](references/conventions/typespecs.md)** - Type specification system

### Meta-Programming

For advanced macro and DSL development:

- **[Macros](references/meta-programming/macros.md)** - Macro creation and best practices
- **[Quote and Unquote](references/meta-programming/quote-and-unquote.md)** - AST manipulation
- **[Domain-Specific Languages](references/meta-programming/domain-specific-languages.md)** - DSL patterns

## When to Use This Skill

Trigger this skill when:

- Writing new Elixir modules (not Phoenix-specific)
- Reviewing pure Elixir code
- Implementing GenServers, Agents, or OTP patterns
- Writing protocols or behaviours
- Creating macros or DSLs
- Learning Elixir conventions
- Debugging anti-pattern issues

## Related Skills

- **`otp-patterns`** - For GenServer, Supervisor, and OTP architecture
- **`chris-mccord-phoenix-style`** - For Phoenix/LiveView-specific patterns
- **`instructor-elixir`** - For structured LLM outputs with Ecto schemas

## Attribution

Content in the references directory is adapted from the official Elixir documentation (Apache 2.0 License).
Source: https://github.com/elixir-lang/elixir/tree/v1.19.3/lib/elixir/pages
