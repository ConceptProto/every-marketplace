---
name: chris-mccord-phoenix-style
description: Write Phoenix and Elixir code following Chris McCord's philosophy. Use this skill when writing Phoenix applications, LiveView components, contexts, or any Phoenix file. Triggers on Phoenix code generation, refactoring requests, code review, or when the user mentions LiveView patterns, real-time features, or Phoenix conventions. Embodies LiveView-first architecture, server-rendered state, minimal JavaScript, and the "real-time by default" philosophy.
---

# Chris McCord Phoenix/Elixir Style Guide

Write Phoenix and Elixir code following Chris McCord's philosophy: **LiveView-first**, **server-rendered state**, **real-time by default**.

## Quick Reference

### LiveView Structure

```elixir
defmodule MyAppWeb.DashboardLive do
  use MyAppWeb, :live_view

  @impl true
  def mount(_params, _session, socket) do
    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyApp.PubSub, "dashboard")
    end

    {:ok, assign(socket, items: [], loading: true)}
  end

  @impl true
  def handle_params(params, _uri, socket) do
    {:noreply, apply_action(socket, socket.assigns.live_action, params)}
  end

  defp apply_action(socket, :index, _params) do
    socket
    |> assign(:page_title, "Dashboard")
    |> assign(:items, list_items())
    |> assign(:loading, false)
  end

  @impl true
  def handle_event("delete", %{"id" => id}, socket) do
    item = Items.get_item!(id)
    {:ok, _} = Items.delete_item(item)
    {:noreply, stream_delete(socket, :items, item)}
  end

  @impl true
  def handle_info({:item_created, item}, socket) do
    {:noreply, stream_insert(socket, :items, item, at: 0)}
  end
end
```

### Context Design

Contexts are the public API for business logic:

```elixir
defmodule MyApp.Accounts do
  @moduledoc """
  The Accounts context.
  """

  alias MyApp.Repo
  alias MyApp.Accounts.User

  def list_users do
    Repo.all(User)
  end

  def get_user!(id), do: Repo.get!(User, id)

  def create_user(attrs \\ %{}) do
    %User{}
    |> User.changeset(attrs)
    |> Repo.insert()
  end

  def update_user(%User{} = user, attrs) do
    user
    |> User.changeset(attrs)
    |> Repo.update()
  end

  def delete_user(%User{} = user) do
    Repo.delete(user)
  end

  def change_user(%User{} = user, attrs \\ %{}) do
    User.changeset(user, attrs)
  end
end
```

### Component Design

Function components with slots:

```elixir
defmodule MyAppWeb.Components.Card do
  use Phoenix.Component

  @doc """
  Renders a card with optional header and footer.

  ## Examples

      <.card>
        <:header>Title</:header>
        Content goes here
        <:footer>Footer</:footer>
      </.card>
  """
  attr :class, :string, default: nil
  slot :header
  slot :inner_block, required: true
  slot :footer

  def card(assigns) do
    ~H"""
    <div class={["rounded-lg border bg-card p-6", @class]}>
      <div :if={@header != []} class="mb-4 font-semibold">
        {render_slot(@header)}
      </div>
      {render_slot(@inner_block)}
      <div :if={@footer != []} class="mt-4 border-t pt-4">
        {render_slot(@footer)}
      </div>
    </div>
    """
  end
end
```

### Form Handling

```elixir
# In LiveView
def handle_event("validate", %{"user" => user_params}, socket) do
  changeset =
    socket.assigns.user
    |> Accounts.change_user(user_params)
    |> Map.put(:action, :validate)

  {:noreply, assign(socket, :form, to_form(changeset))}
end

def handle_event("save", %{"user" => user_params}, socket) do
  case Accounts.update_user(socket.assigns.user, user_params) do
    {:ok, user} ->
      {:noreply,
       socket
       |> put_flash(:info, "User updated")
       |> push_navigate(to: ~p"/users/#{user}")}

    {:error, %Ecto.Changeset{} = changeset} ->
      {:noreply, assign(socket, :form, to_form(changeset))}
  end
end
```

### Streams for Collections

```elixir
def mount(_params, _session, socket) do
  {:ok, stream(socket, :items, Items.list_items())}
end

# In template
<div id="items" phx-update="stream">
  <div :for={{dom_id, item} <- @streams.items} id={dom_id}>
    {item.name}
  </div>
</div>
```

### Async Operations

```elixir
def mount(_params, _session, socket) do
  {:ok,
   socket
   |> assign(:page_title, "Dashboard")
   |> assign_async(:stats, fn -> {:ok, %{stats: fetch_stats()}} end)}
end

# In template
<.async_result :let={stats} assign={@stats}>
  <:loading>Loading stats...</:loading>
  <:failed :let={_reason}>Failed to load stats</:failed>
  Stats: {stats.count}
</.async_result>
```

### PubSub Broadcasting

```elixir
# In context after successful operation
def create_item(attrs) do
  case %Item{} |> Item.changeset(attrs) |> Repo.insert() do
    {:ok, item} ->
      Phoenix.PubSub.broadcast(MyApp.PubSub, "items", {:item_created, item})
      {:ok, item}

    error ->
      error
  end
end

# In LiveView mount
def mount(_params, _session, socket) do
  if connected?(socket), do: Phoenix.PubSub.subscribe(MyApp.PubSub, "items")
  {:ok, socket}
end

def handle_info({:item_created, item}, socket) do
  {:noreply, stream_insert(socket, :items, item, at: 0)}
end
```

## Core Principles

### 1. LiveView-First Architecture
- Use LiveView for all interactive features
- Server assigns are the single source of truth
- Avoid client-side state management

### 2. Minimal JavaScript
- Use hooks only when necessary (third-party libs, canvas, etc.)
- Prefer `phx-` bindings over custom JavaScript
- Let LiveView handle state synchronisation

### 3. Real-Time by Default
- Use PubSub for cross-process communication
- Broadcast changes, let subscribers react
- Phoenix.Presence for user tracking

### 4. Context Boundaries
- Contexts define clear API boundaries
- No direct Repo calls from controllers/LiveViews
- Contexts handle business logic and validation

### 5. Component Architecture
- Small, focused function components
- Use slots for flexibility
- Compose components, don't inherit

## Architecture Preferences

| Traditional | Phoenix Way |
|-------------|-------------|
| React/Vue SPA | Phoenix LiveView |
| Redux state | Server assigns |
| REST API + frontend | LiveView with streams |
| WebSocket libraries | Phoenix Channels |
| Redis PubSub | Phoenix.PubSub |
| JWT tokens | Phoenix sessions |
| Client-side routing | live_session |

## Testing Patterns

```elixir
defmodule MyAppWeb.DashboardLiveTest do
  use MyAppWeb.ConnCase
  import Phoenix.LiveViewTest

  test "displays items", %{conn: conn} do
    item = item_fixture()
    {:ok, view, html} = live(conn, ~p"/dashboard")
    assert html =~ item.name
  end

  test "creates item via form", %{conn: conn} do
    {:ok, view, _html} = live(conn, ~p"/dashboard")

    view
    |> form("#item-form", item: %{name: "New Item"})
    |> render_submit()

    assert render(view) =~ "New Item"
  end
end
```

## Philosophy Summary

1. **Server-rendered state**: Assigns are truth, JavaScript is enhancement
2. **Real-time by default**: PubSub enables reactive UIs without complexity
3. **Streams for scale**: Handle large collections efficiently
4. **Async for UX**: Never block mount, load data asynchronously
5. **Contexts for clarity**: Clean boundaries, testable business logic
6. **Components for reuse**: Small, composable, slot-based
7. **Boring technology**: Use Phoenix's built-in solutions first

## Related Skills

- **`elixir-style`** - For general Elixir coding conventions, anti-patterns, naming conventions, and typespecs
- **`otp-patterns`** - For GenServer, Supervisor, Agent, Task, and OTP architecture patterns

## Reference Index

For detailed patterns beyond this quick reference, load the appropriate reference file:

| Topic | Reference File | Use When |
|-------|----------------|----------|
| **Routing** | `references/routing.md` | Router structure, resources, pipelines, live routes, verified routes |
| **Contexts** | `references/contexts.md` | Business logic, boundaries, authentication patterns, cross-context design |
| **Ecto** | `references/ecto-patterns.md` | Schemas, changesets, migrations, queries, Ecto.Multi |
| **LiveView** | `references/liveview-patterns.md` | Lifecycle, streams, async, events, components, navigation |
| **Testing** | `references/testing-patterns.md` | ConnCase, DataCase, LiveView tests, fixtures |
| **Real-time** | `references/real-time.md` | PubSub, Channels, Presence, real-time LiveView patterns |

Load references progressively based on user questions. Start with this SKILL.md for quick patterns.
