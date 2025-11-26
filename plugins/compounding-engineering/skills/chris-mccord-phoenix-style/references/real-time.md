<overview>
Phoenix real-time features: Channels for WebSocket communication, PubSub for internal messaging, and Presence for user tracking. Use LiveView for most real-time UI; Channels for custom protocols.
</overview>

<pattern name="pubsub_basics">
<description>PubSub enables process-to-process communication across nodes.</description>

<example>
```elixir
# Subscribe to a topic (usually in LiveView mount)
def mount(_params, _session, socket) do
  if connected?(socket) do
    Phoenix.PubSub.subscribe(MyApp.PubSub, "posts")
    Phoenix.PubSub.subscribe(MyApp.PubSub, "posts:#{socket.assigns.user_id}")
  end

  {:ok, socket}
end

# Broadcast from context
defmodule MyApp.Content do
  @pubsub MyApp.PubSub

  def create_post(user, attrs) do
    case Repo.insert(Post.changeset(%Post{user_id: user.id}, attrs)) do
      {:ok, post} ->
        Phoenix.PubSub.broadcast(@pubsub, "posts", {:post_created, post})
        Phoenix.PubSub.broadcast(@pubsub, "posts:#{user.id}", {:my_post_created, post})
        {:ok, post}

      error ->
        error
    end
  end
end

# Handle in LiveView
def handle_info({:post_created, post}, socket) do
  {:noreply, stream_insert(socket, :posts, post, at: 0)}
end
```
</example>

<key_point>
Broadcast from contexts after successful operations. LiveViews subscribe and react. This decouples business logic from UI.
</key_point>
</pattern>

<pattern name="topic_naming">
<description>Consistent topic naming conventions for PubSub.</description>

<conventions>
```elixir
# Resource-based topics
"posts"                    # All posts
"posts:#{post_id}"         # Specific post
"users:#{user_id}:posts"   # Posts for a specific user

# Room/conversation topics
"room:lobby"               # Named room
"room:#{room_id}"          # Room by ID
"conversation:#{id}"       # Direct messages

# User-specific topics
"user:#{user_id}"          # Notifications for user
"user:#{user_id}:activity" # Activity feed

# Admin/system topics
"admin:notifications"      # Admin alerts
"system:maintenance"       # System-wide broadcasts
```
</conventions>

<key_point>
Use colons to separate topic segments. Be consistent across your app.
</key_point>
</pattern>

<pattern name="channels">
<description>WebSocket channels for custom real-time protocols.</description>

<socket>
```elixir
# lib/my_app_web/channels/user_socket.ex
defmodule MyAppWeb.UserSocket do
  use Phoenix.Socket

  channel "room:*", MyAppWeb.RoomChannel
  channel "user:*", MyAppWeb.UserChannel

  @impl true
  def connect(%{"token" => token}, socket, _connect_info) do
    case Phoenix.Token.verify(MyAppWeb.Endpoint, "user socket", token, max_age: 86400) do
      {:ok, user_id} ->
        {:ok, assign(socket, :user_id, user_id)}

      {:error, _reason} ->
        :error
    end
  end

  def connect(_params, _socket, _connect_info), do: :error

  @impl true
  def id(socket), do: "user_socket:#{socket.assigns.user_id}"
end
```
</socket>

<channel>
```elixir
# lib/my_app_web/channels/room_channel.ex
defmodule MyAppWeb.RoomChannel do
  use MyAppWeb, :channel

  @impl true
  def join("room:lobby", _payload, socket) do
    {:ok, socket}
  end

  def join("room:" <> room_id, _payload, socket) do
    if authorized?(socket.assigns.user_id, room_id) do
      {:ok, assign(socket, :room_id, room_id)}
    else
      {:error, %{reason: "unauthorized"}}
    end
  end

  @impl true
  def handle_in("new_msg", %{"body" => body}, socket) do
    broadcast!(socket, "new_msg", %{
      body: body,
      user_id: socket.assigns.user_id
    })
    {:noreply, socket}
  end

  def handle_in("typing", _payload, socket) do
    broadcast_from!(socket, "typing", %{user_id: socket.assigns.user_id})
    {:noreply, socket}
  end

  defp authorized?(user_id, room_id) do
    # Check if user can access room
    MyApp.Rooms.member?(room_id, user_id)
  end
end
```
</channel>

<key_point>
Use Channels when you need bidirectional communication with custom protocols. For UI updates, prefer LiveView + PubSub.
</key_point>
</pattern>

<pattern name="presence">
<description>Track user presence across distributed systems.</description>

<setup>
```elixir
# lib/my_app_web/channels/presence.ex
defmodule MyAppWeb.Presence do
  use Phoenix.Presence,
    otp_app: :my_app,
    pubsub_server: MyApp.PubSub
end

# Add to supervision tree in application.ex
children = [
  MyAppWeb.Presence,
  # ...
]
```
</setup>

<channel_usage>
```elixir
defmodule MyAppWeb.RoomChannel do
  use MyAppWeb, :channel
  alias MyAppWeb.Presence

  @impl true
  def join("room:" <> room_id, _payload, socket) do
    send(self(), :after_join)
    {:ok, assign(socket, :room_id, room_id)}
  end

  @impl true
  def handle_info(:after_join, socket) do
    {:ok, _} =
      Presence.track(socket, socket.assigns.user_id, %{
        online_at: inspect(System.system_time(:second)),
        typing: false
      })

    push(socket, "presence_state", Presence.list(socket))
    {:noreply, socket}
  end

  @impl true
  def handle_in("typing:start", _payload, socket) do
    {:ok, _} =
      Presence.update(socket, socket.assigns.user_id, fn meta ->
        Map.put(meta, :typing, true)
      end)

    {:noreply, socket}
  end

  def handle_in("typing:stop", _payload, socket) do
    {:ok, _} =
      Presence.update(socket, socket.assigns.user_id, fn meta ->
        Map.put(meta, :typing, false)
      end)

    {:noreply, socket}
  end
end
```
</channel_usage>

<liveview_usage>
```elixir
defmodule MyAppWeb.RoomLive do
  use MyAppWeb, :live_view
  alias MyAppWeb.Presence

  @impl true
  def mount(%{"id" => room_id}, _session, socket) do
    topic = "room:#{room_id}"

    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyApp.PubSub, topic)

      {:ok, _} =
        Presence.track(self(), topic, socket.assigns.current_user.id, %{
          joined_at: DateTime.utc_now()
        })
    end

    presences = Presence.list(topic)

    {:ok,
     socket
     |> assign(:room_id, room_id)
     |> assign(:presences, presences)}
  end

  @impl true
  def handle_info(%Phoenix.Socket.Broadcast{event: "presence_diff", payload: diff}, socket) do
    {:noreply,
     socket
     |> handle_leaves(diff.leaves)
     |> handle_joins(diff.joins)}
  end

  defp handle_joins(socket, joins) do
    Enum.reduce(joins, socket, fn {user_id, %{metas: [meta | _]}}, socket ->
      assign(socket, :presences, Map.put(socket.assigns.presences, user_id, meta))
    end)
  end

  defp handle_leaves(socket, leaves) do
    Enum.reduce(leaves, socket, fn {user_id, _}, socket ->
      assign(socket, :presences, Map.delete(socket.assigns.presences, user_id))
    end)
  end
end
```
</liveview_usage>

<key_point>
Presence is CRDT-based and self-healing across nodes. No single point of failure.
</key_point>
</pattern>

<pattern name="liveview_realtime">
<description>Real-time updates in LiveView via PubSub.</description>

<example>
```elixir
defmodule MyAppWeb.FeedLive do
  use MyAppWeb, :live_view

  @impl true
  def mount(_params, _session, socket) do
    if connected?(socket) do
      Phoenix.PubSub.subscribe(MyApp.PubSub, "feed")
    end

    {:ok, stream(socket, :posts, MyApp.Content.list_recent_posts())}
  end

  @impl true
  def handle_info({:new_post, post}, socket) do
    {:noreply, stream_insert(socket, :posts, post, at: 0)}
  end

  def handle_info({:post_updated, post}, socket) do
    {:noreply, stream_insert(socket, :posts, post)}
  end

  def handle_info({:post_deleted, post}, socket) do
    {:noreply, stream_delete(socket, :posts, post)}
  end
end

# In context
defmodule MyApp.Content do
  def create_post(user, attrs) do
    case Repo.insert(changeset) do
      {:ok, post} ->
        Phoenix.PubSub.broadcast(MyApp.PubSub, "feed", {:new_post, post})
        {:ok, post}

      error ->
        error
    end
  end
end
```
</example>
</pattern>

<pattern name="when_to_use_what">
<description>Choose the right real-time mechanism.</description>

<decision_table>
| Use Case | Solution | Why |
|----------|----------|-----|
| UI updates from server | LiveView + PubSub | Simplest, no JS needed |
| Form validation | LiveView events | Built-in, real-time |
| Chat/messaging | Channels | Bidirectional, custom protocol |
| Notifications | LiveView + PubSub | Server-initiated, simple |
| User presence | Presence | Built-in CRDT, distributed |
| Gaming/real-time sync | Channels | Low latency, custom events |
| Dashboard updates | LiveView + PubSub | Server push, auto-diff |
| File uploads | LiveView uploads | Built-in progress, chunking |
</decision_table>

<key_point>
Default to LiveView + PubSub. Only use Channels when you need custom bidirectional protocols or JS client interop.
</key_point>
</pattern>

<best_practices>
<practice>
**Broadcast from contexts** - Keep PubSub broadcasts in your business logic layer, not LiveViews.
</practice>

<practice>
**Check connected? before subscribe** - Only subscribe when the WebSocket is connected.
</practice>

<practice>
**Consistent topic naming** - Use `resource:id` pattern. Document your conventions.
</practice>

<practice>
**LiveView over Channels** - For most real-time UI, LiveView + PubSub is simpler than Channels.
</practice>

<practice>
**Presence for online status** - Don't roll your own presence tracking. Use Phoenix.Presence.
</practice>

<practice>
**Authenticate sockets** - Always verify tokens in `connect/3`. Never trust client params alone.
</practice>
</best_practices>
