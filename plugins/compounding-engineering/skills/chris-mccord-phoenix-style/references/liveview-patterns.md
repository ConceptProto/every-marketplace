<overview>
Phoenix LiveView patterns: lifecycle callbacks, events, state management, streams, async operations, and function components. LiveView-first architecture is the Phoenix way.
</overview>

<pattern name="liveview_structure">
<description>Standard LiveView module structure with lifecycle callbacks.</description>

<example>
```elixir
defmodule MyAppWeb.DashboardLive do
  use MyAppWeb, :live_view

  @impl true
  def mount(_params, _session, socket) do
    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyApp.PubSub, "dashboard")
    end

    {:ok,
     socket
     |> assign(:page_title, "Dashboard")
     |> assign(:loading, true)
     |> stream(:items, [])}
  end

  @impl true
  def handle_params(params, _uri, socket) do
    {:noreply, apply_action(socket, socket.assigns.live_action, params)}
  end

  defp apply_action(socket, :index, _params) do
    socket
    |> assign(:page_title, "All Items")
    |> stream(:items, Items.list_items(), reset: true)
    |> assign(:loading, false)
  end

  defp apply_action(socket, :show, %{"id" => id}) do
    item = Items.get_item!(id)

    socket
    |> assign(:page_title, item.name)
    |> assign(:item, item)
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
</example>

<key_point>
Always check `connected?(socket)` before subscribing to PubSub. Mount runs twice: once for static render, once for WebSocket.
</key_point>
</pattern>

<pattern name="lifecycle_order">
<description>Understanding the LiveView lifecycle callback order.</description>

<lifecycle>
1. **mount/3** - Initial setup, runs twice (disconnected + connected)
2. **handle_params/3** - URL changes, runs after mount and on `push_patch`
3. **render/1** - Generates HTML, called after any assign changes
4. **handle_event/3** - User interactions from browser
5. **handle_info/2** - Process messages (PubSub, scheduled tasks)
</lifecycle>

<key_point>
`handle_params` is the place to load data based on URL. Use `live_action` to determine which view to show.
</key_point>
</pattern>

<pattern name="streams">
<description>Use streams for collections - never assign lists directly.</description>

<example>
```elixir
# In mount
def mount(_params, _session, socket) do
  {:ok, stream(socket, :messages, Messages.list_messages())}
end

# In template
def render(assigns) do
  ~H"""
  <div id="messages" phx-update="stream">
    <div :for={{dom_id, message} <- @streams.messages} id={dom_id}>
      <p>{message.text}</p>
      <button phx-click="delete" phx-value-id={message.id}>Delete</button>
    </div>
  </div>
  """
end

# Stream operations
def handle_event("delete", %{"id" => id}, socket) do
  message = Messages.get_message!(id)
  {:ok, _} = Messages.delete_message(message)
  {:noreply, stream_delete(socket, :messages, message)}
end

def handle_info({:new_message, message}, socket) do
  {:noreply, stream_insert(socket, :messages, message, at: 0)}
end

# Reset stream for filtering
def handle_event("filter", %{"status" => status}, socket) do
  messages = Messages.list_by_status(status)
  {:noreply, stream(socket, :messages, messages, reset: true)}
end
```
</example>

<anti_pattern>
```elixir
# AVOID: Assigning lists directly
def mount(_params, _session, socket) do
  {:ok, assign(socket, :messages, Messages.list_messages())}
end

# USE: Streams for collections
def mount(_params, _session, socket) do
  {:ok, stream(socket, :messages, Messages.list_messages())}
end
```
</anti_pattern>

<empty_state>
```heex
<div id="messages" phx-update="stream">
  <div class="hidden only:block">No messages yet</div>
  <div :for={{id, msg} <- @streams.messages} id={id}>
    {msg.text}
  </div>
</div>
```
</empty_state>

<key_point>
Streams are NOT enumerable. You cannot use `Enum.filter` on them. To filter, refetch from database and reset.
</key_point>
</pattern>

<pattern name="async_operations">
<description>Non-blocking data loading with assign_async and start_async.</description>

<example>
```elixir
def mount(_params, _session, socket) do
  {:ok,
   socket
   |> assign(:page_title, "Dashboard")
   |> assign_async(:stats, fn -> {:ok, %{stats: fetch_expensive_stats()}} end)
   |> assign_async(:recent_activity, fn -> {:ok, %{recent_activity: fetch_activity()}} end)}
end

# In template
def render(assigns) do
  ~H"""
  <.async_result :let={stats} assign={@stats}>
    <:loading>
      <div class="animate-pulse">Loading stats...</div>
    </:loading>
    <:failed :let={reason}>
      <div class="text-red-500">Failed to load: {inspect(reason)}</div>
    </:failed>
    <div class="stats-grid">
      <span>Total: {stats.count}</span>
      <span>Active: {stats.active}</span>
    </div>
  </.async_result>
  """
end
```
</example>

<start_async>
```elixir
# For operations triggered by events
def handle_event("generate_report", _params, socket) do
  {:noreply,
   socket
   |> assign(:report_status, :generating)
   |> start_async(:report, fn -> generate_large_report() end)}
end

def handle_async(:report, {:ok, report}, socket) do
  {:noreply,
   socket
   |> assign(:report_status, :complete)
   |> assign(:report, report)}
end

def handle_async(:report, {:exit, reason}, socket) do
  {:noreply,
   socket
   |> assign(:report_status, :failed)
   |> put_flash(:error, "Report generation failed")}
end
```
</start_async>

<key_point>
Never block mount with slow operations. Use `assign_async` for initial data loading, `start_async` for event-triggered operations.
</key_point>
</pattern>

<pattern name="events">
<description>Handle user interactions with phx-* bindings.</description>

<example>
```elixir
# Click events
<button phx-click="increment">+</button>
<button phx-click="delete" phx-value-id={item.id}>Delete</button>

# Form events
<.form for={@form} phx-change="validate" phx-submit="save">
  <.input field={@form[:name]} type="text" />
  <button type="submit">Save</button>
</.form>

# Debounced input
<input phx-change="search" phx-debounce="300" />

# Confirmation
<button phx-click="delete" data-confirm="Are you sure?">Delete</button>

# Key events
<div phx-window-keydown="keydown" phx-key="Escape">
  Press Escape to close
</div>
```
</example>

<handler>
```elixir
def handle_event("increment", _params, socket) do
  {:noreply, update(socket, :count, &(&1 + 1))}
end

def handle_event("delete", %{"id" => id}, socket) do
  item = Items.get_item!(id)
  {:ok, _} = Items.delete_item(item)
  {:noreply, stream_delete(socket, :items, item)}
end

def handle_event("validate", %{"form" => params}, socket) do
  changeset =
    %Item{}
    |> Items.change_item(params)
    |> Map.put(:action, :validate)

  {:noreply, assign(socket, :form, to_form(changeset))}
end

def handle_event("save", %{"form" => params}, socket) do
  case Items.create_item(params) do
    {:ok, item} ->
      {:noreply,
       socket
       |> put_flash(:info, "Item created")
       |> push_navigate(to: ~p"/items/#{item}")}

    {:error, changeset} ->
      {:noreply, assign(socket, :form, to_form(changeset))}
  end
end
```
</handler>
</pattern>

<pattern name="form_handling">
<description>Forms with real-time validation using changesets.</description>

<example>
```elixir
def mount(_params, _session, socket) do
  changeset = Items.change_item(%Item{})
  {:ok, assign(socket, :form, to_form(changeset))}
end

def handle_event("validate", %{"item" => params}, socket) do
  changeset =
    %Item{}
    |> Items.change_item(params)
    |> Map.put(:action, :validate)

  {:noreply, assign(socket, :form, to_form(changeset))}
end

def handle_event("save", %{"item" => params}, socket) do
  case Items.create_item(params) do
    {:ok, item} ->
      {:noreply,
       socket
       |> put_flash(:info, "Created successfully")
       |> push_navigate(to: ~p"/items/#{item}")}

    {:error, changeset} ->
      {:noreply, assign(socket, :form, to_form(changeset))}
  end
end
```
</example>

<template>
```heex
<.form for={@form} id="item-form" phx-change="validate" phx-submit="save">
  <.input field={@form[:name]} type="text" label="Name" />
  <.input field={@form[:description]} type="textarea" label="Description" />
  <.input field={@form[:status]} type="select" label="Status"
    options={[{"Draft", "draft"}, {"Published", "published"}]} />

  <:actions>
    <.button phx-disable-with="Saving...">Save</.button>
  </:actions>
</.form>
```
</template>

<anti_pattern>
```elixir
# AVOID: Passing changeset to template
assign(socket, :changeset, changeset)  # In template: @changeset[:field] won't work

# USE: to_form/1
assign(socket, :form, to_form(changeset))  # In template: @form[:field] works
```
</anti_pattern>
</pattern>

<pattern name="function_components">
<description>Reusable UI components with attrs and slots.</description>

<example>
```elixir
defmodule MyAppWeb.Components.Card do
  use Phoenix.Component

  @doc """
  Renders a card with optional header and actions.

  ## Examples

      <.card>
        <:header>Title</:header>
        Content goes here
        <:actions>
          <.button>Save</.button>
        </:actions>
      </.card>
  """
  attr :class, :string, default: nil
  attr :rest, :global

  slot :header
  slot :inner_block, required: true
  slot :actions

  def card(assigns) do
    ~H"""
    <div class={["rounded-lg border bg-card p-6 shadow-sm", @class]} {@rest}>
      <div :if={@header != []} class="mb-4 border-b pb-4">
        <h3 class="text-lg font-semibold">
          {render_slot(@header)}
        </h3>
      </div>

      <div class="card-content">
        {render_slot(@inner_block)}
      </div>

      <div :if={@actions != []} class="mt-4 flex justify-end gap-2 border-t pt-4">
        {render_slot(@actions)}
      </div>
    </div>
    """
  end
end
```
</example>

<slot_with_args>
```elixir
slot :col, required: true do
  attr :label, :string
end

def table(assigns) do
  ~H"""
  <table>
    <thead>
      <tr>
        <th :for={col <- @col}>{col.label}</th>
      </tr>
    </thead>
    <tbody>
      <tr :for={row <- @rows}>
        <td :for={col <- @col}>
          {render_slot(col, row)}
        </td>
      </tr>
    </tbody>
  </table>
  """
end

# Usage
<.table rows={@users}>
  <:col :let={user} label="Name">{user.name}</:col>
  <:col :let={user} label="Email">{user.email}</:col>
</.table>
```
</slot_with_args>

<key_point>
Use `attr/3` for component inputs with compile-time validation. Use `slot/2` for nested content.
</key_point>
</pattern>

<pattern name="navigation">
<description>Navigate between LiveViews and handle URL state.</description>

<example>
```elixir
# Navigate to different LiveView (full page load within live_session)
<.link navigate={~p"/users/#{user}"}>View User</.link>

# Patch current LiveView (update URL, trigger handle_params)
<.link patch={~p"/users?page=2"}>Next Page</.link>

# Programmatic navigation
def handle_event("create", params, socket) do
  case Users.create_user(params) do
    {:ok, user} ->
      {:noreply, push_navigate(socket, to: ~p"/users/#{user}")}

    {:error, changeset} ->
      {:noreply, assign(socket, :form, to_form(changeset))}
  end
end

# Update URL without navigation
def handle_event("filter", %{"status" => status}, socket) do
  {:noreply, push_patch(socket, to: ~p"/users?status=#{status}")}
end
```
</example>

<key_point>
Use `push_patch` for URL state (filtering, pagination). Use `push_navigate` to move to a different LiveView.
</key_point>
</pattern>

<pattern name="js_commands">
<description>Client-side interactions without server round-trips.</description>

<example>
```elixir
alias Phoenix.LiveView.JS

def hide_modal(js \\ %JS{}) do
  js
  |> JS.hide(to: "#modal", transition: "fade-out")
  |> JS.hide(to: "#modal-backdrop")
  |> JS.remove_class("overflow-hidden", to: "body")
  |> JS.pop_focus()
end

def show_dropdown(js \\ %JS{}) do
  js
  |> JS.show(to: "#dropdown", transition: {"ease-out duration-200", "opacity-0", "opacity-100"})
  |> JS.set_attribute({"aria-expanded", "true"}, to: "#dropdown-button")
end

# In template
<button phx-click={show_dropdown()}>Menu</button>
<button phx-click={hide_modal() |> JS.push("close")}>Close</button>
```
</example>

<key_point>
JS commands run client-side for instant feedback. Chain with `JS.push/2` to also send server event.
</key_point>
</pattern>

<best_practices>
<practice>
**Streams for collections** - Never `assign` a list. Always use `stream/4` for collections.
</practice>

<practice>
**Check connected? before subscribe** - PubSub subscriptions only work on connected socket.
</practice>

<practice>
**Async for slow operations** - Use `assign_async` in mount, `start_async` for events.
</practice>

<practice>
**to_form for changesets** - Always convert changesets with `to_form/1` before assigning.
</practice>

<practice>
**Small components** - Keep components focused. Use slots for flexibility.
</practice>

<practice>
**URL state with push_patch** - Use URL params for shareable/bookmarkable state.
</practice>

<practice>
**JS commands for instant UI** - Don't round-trip for pure UI changes like show/hide.
</practice>
</best_practices>
