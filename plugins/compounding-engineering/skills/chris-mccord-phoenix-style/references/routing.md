<overview>
Phoenix routing patterns using the router DSL. Covers resources, scopes, pipelines, verified routes, and nested resources. Routes compile to pattern-matched functions for performance.
</overview>

<pattern name="single_routes">
<description>Basic HTTP verb routes mapping paths to controller actions.</description>

<example>
```elixir
defmodule MyAppWeb.Router do
  use MyAppWeb, :router

  scope "/", MyAppWeb do
    pipe_through :browser

    get "/", PageController, :home
    get "/about", PageController, :about
    post "/contact", ContactController, :create
    delete "/sessions/:id", SessionController, :delete
  end
end
```
</example>

<key_point>
Routes compile to pattern-matched functions, not runtime lookups. This provides significant performance benefits.
</key_point>
</pattern>

<pattern name="resources">
<description>Generate RESTful routes for CRUD operations automatically.</description>

<example>
```elixir
# Generates all 7 RESTful routes
resources "/users", UserController

# Only specific actions
resources "/posts", PostController, only: [:index, :show]

# Exclude specific actions
resources "/comments", CommentController, except: [:delete]
```
</example>

<generated_routes>
| HTTP Verb | Path | Action | Helper |
|-----------|------|--------|--------|
| GET | /users | :index | ~p"/users" |
| GET | /users/new | :new | ~p"/users/new" |
| GET | /users/:id | :show | ~p"/users/#{user}" |
| GET | /users/:id/edit | :edit | ~p"/users/#{user}/edit" |
| POST | /users | :create | ~p"/users" |
| PATCH/PUT | /users/:id | :update | ~p"/users/#{user}" |
| DELETE | /users/:id | :delete | ~p"/users/#{user}" |
</generated_routes>
</pattern>

<pattern name="nested_resources">
<description>Express parent-child relationships in URLs.</description>

<example>
```elixir
resources "/users", UserController do
  resources "/posts", PostController
end
```
</example>

<generated_paths>
- `/users/:user_id/posts` - list posts for a user
- `/users/:user_id/posts/:id` - show specific post for user
- Access user_id in controller: `conn.params["user_id"]`
</generated_paths>

<anti_pattern>
Avoid deeply nested resources (more than 2 levels). Instead, use shallow nesting:

```elixir
# AVOID: /users/:user_id/posts/:post_id/comments/:id
resources "/users", UserController do
  resources "/posts", PostController, only: [:index, :create]
end
resources "/posts", PostController, only: [:show, :update, :delete] do
  resources "/comments", CommentController
end
```
</anti_pattern>
</pattern>

<pattern name="scopes">
<description>Group routes under common path prefixes and controller namespaces.</description>

<example>
```elixir
# Admin namespace
scope "/admin", MyAppWeb.Admin do
  pipe_through [:browser, :require_admin]

  resources "/users", UserController
  resources "/settings", SettingsController
end

# API versioning
scope "/api", MyAppWeb.Api do
  pipe_through :api

  scope "/v1", V1 do
    resources "/users", UserController
  end

  scope "/v2", V2 do
    resources "/users", UserController
  end
end
```
</example>

<key_point>
Scope affects both the URL path AND the controller module namespace. `/admin/users` maps to `MyAppWeb.Admin.UserController`.
</key_point>
</pattern>

<pattern name="pipelines">
<description>Apply plugs to groups of routes for cross-cutting concerns.</description>

<example>
```elixir
pipeline :browser do
  plug :accepts, ["html"]
  plug :fetch_session
  plug :fetch_live_flash
  plug :put_root_layout, html: {MyAppWeb.Layouts, :root}
  plug :protect_from_forgery
  plug :put_secure_browser_headers
end

pipeline :api do
  plug :accepts, ["json"]
end

pipeline :authenticated do
  plug MyAppWeb.Plugs.RequireAuth
end

scope "/", MyAppWeb do
  pipe_through [:browser, :authenticated]

  resources "/dashboard", DashboardController
end
```
</example>

<key_point>
Order matters in `pipe_through`. Plugs execute in the order listed. Authentication should come after session loading.
</key_point>
</pattern>

<pattern name="verified_routes">
<description>Compile-time verified route helpers using the ~p sigil.</description>

<example>
```elixir
# In templates and controllers
~p"/users"
~p"/users/#{user.id}"
~p"/users/#{user}/edit"
~p"/posts?page=#{page}&sort=date"

# Full URL (for emails, redirects)
url(~p"/users/#{user}")

# In LiveView
<.link navigate={~p"/users/#{@user}"}>View User</.link>
<.link patch={~p"/users?#{@params}"}>Filter</.link>
```
</example>

<key_point>
Invalid routes cause compiler warnings. This catches broken links at compile time, not runtime.
</key_point>

<anti_pattern>
```elixir
# AVOID: String interpolation without ~p
"/users/#{user.id}"  # No compile-time verification

# USE: Verified routes
~p"/users/#{user.id}"  # Compiler verifies route exists
```
</anti_pattern>
</pattern>

<pattern name="live_routes">
<description>Routes for Phoenix LiveView.</description>

<example>
```elixir
scope "/", MyAppWeb do
  pipe_through :browser

  live "/dashboard", DashboardLive, :index
  live "/users/:id", UserLive.Show, :show
  live "/users/:id/edit", UserLive.Show, :edit
end

# Live sessions for shared authentication
live_session :authenticated, on_mount: [MyAppWeb.LiveAuth] do
  live "/settings", SettingsLive, :index
  live "/profile", ProfileLive, :show
end
```
</example>

<key_point>
Use `live_session` to share authentication and assign loading across multiple LiveViews without reconnecting the WebSocket.
</key_point>
</pattern>

<pattern name="route_inspection">
<description>Debug and audit routes with mix tasks.</description>

<example>
```bash
# List all routes
mix phx.routes

# Filter by path
mix phx.routes | grep admin

# Output shows: verb, path, controller, action
# GET  /users  MyAppWeb.UserController :index
```
</example>

<use_case>
Run after adding routes to verify they're correct. Essential for auditing authentication boundaries.
</use_case>
</pattern>

<best_practices>
<practice>
**Group by pipeline** - Organise routes by their security requirements. All authenticated routes in one scope, public routes in another.
</practice>

<practice>
**Use verified routes everywhere** - Never use string interpolation for paths. The compiler is your friend.
</practice>

<practice>
**Shallow nesting** - Maximum 2 levels of nested resources. Flatten deeper hierarchies.
</practice>

<practice>
**Consistent naming** - Resource names should be plural nouns. Actions should match RESTful conventions.
</practice>

<practice>
**API versioning via scopes** - Use `/api/v1`, `/api/v2` scopes, not query params or headers.
</practice>
</best_practices>
