<overview>
Phoenix contexts encapsulate data access and business logic. They are the public API for your domain, separating web concerns from business logic. Includes authentication patterns from `mix phx.gen.auth`.
</overview>

<pattern name="context_basics">
<description>Contexts are modules that group related functionality around a domain concept.</description>

<example>
```elixir
defmodule MyApp.Accounts do
  @moduledoc """
  The Accounts context - handles user management and authentication.
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
</example>

<key_point>
Contexts are the public API. Controllers and LiveViews call context functions, never Repo directly.
</key_point>
</pattern>

<pattern name="naming_conventions">
<description>Name contexts after their domain responsibility, not the data they contain.</description>

<good_names>
- `Accounts` - user authentication and profiles
- `Catalog` - products and categories
- `ShoppingCart` - cart management
- `Billing` - payments and invoices
- `Content` - posts, articles, media
- `Identity` - alternative to Accounts for auth focus
</good_names>

<anti_pattern>
```elixir
# AVOID: Generic or data-focused names
defmodule MyApp.Users do  # Too generic
defmodule MyApp.UserContext do  # Redundant "Context" suffix
defmodule MyApp.Database.Users do  # Implementation detail in name
```
</anti_pattern>

<key_point>
Ask: "What business capability does this provide?" not "What tables does this access?"
</key_point>
</pattern>

<pattern name="context_boundaries">
<description>Decide what belongs in a context based on cohesion and change frequency.</description>

<guidelines>
**Same context when:**
- Data changes together (posts and comments)
- Strong belongs_to relationship
- Same team/feature owns both
- Shared business rules

**Separate contexts when:**
- Different rate of change
- Different access patterns
- Could be a separate microservice someday
- Different teams own the functionality
</guidelines>

<example>
```elixir
# Posts and Comments in same context - tightly coupled
defmodule MyApp.Content do
  alias MyApp.Content.{Post, Comment}

  def list_posts, do: Repo.all(Post)
  def create_post(attrs), do: ...

  def list_comments(post), do: ...
  def create_comment(post, attrs), do: ...
end

# Users and Billing separate - different concerns
defmodule MyApp.Accounts do
  # User management, authentication
end

defmodule MyApp.Billing do
  # Payments, invoices, subscriptions
end
```
</example>
</pattern>

<pattern name="cross_context_relationships">
<description>Handle data dependencies between contexts explicitly.</description>

<example>
```elixir
# ShoppingCart depends on Catalog for products
defmodule MyApp.ShoppingCart do
  alias MyApp.Catalog
  alias MyApp.ShoppingCart.{Cart, CartItem}

  def add_item_to_cart(%Cart{} = cart, product_id, quantity) do
    # Call into Catalog context for product data
    product = Catalog.get_product!(product_id)

    %CartItem{}
    |> CartItem.changeset(%{
      cart_id: cart.id,
      product_id: product.id,
      price: product.price,
      quantity: quantity
    })
    |> Repo.insert()
  end
end
```
</example>

<schema_association>
```elixir
# In ShoppingCart.CartItem schema
defmodule MyApp.ShoppingCart.CartItem do
  schema "cart_items" do
    field :quantity, :integer
    field :price, :decimal

    belongs_to :cart, MyApp.ShoppingCart.Cart
    belongs_to :product, MyApp.Catalog.Product  # Cross-context reference

    timestamps()
  end
end
```
</schema_association>

<key_point>
Cross-context references are acceptable. Use `belongs_to` for the foreign key, but call the owning context for business logic.
</key_point>
</pattern>

<pattern name="authentication_context">
<description>Standard authentication patterns from `mix phx.gen.auth`.</description>

<example>
```elixir
defmodule MyApp.Accounts do
  alias MyApp.Accounts.{User, UserToken}

  ## User registration

  def register_user(attrs) do
    %User{}
    |> User.registration_changeset(attrs)
    |> Repo.insert()
  end

  ## Session management

  def generate_user_session_token(user) do
    {token, user_token} = UserToken.build_session_token(user)
    Repo.insert!(user_token)
    token
  end

  def get_user_by_session_token(token) do
    {:ok, query} = UserToken.verify_session_token_query(token)
    Repo.one(query)
  end

  def delete_user_session_token(token) do
    Repo.delete_all(UserToken.by_token_and_context_query(token, "session"))
    :ok
  end

  ## Password reset

  def deliver_user_reset_password_instructions(user, reset_password_url_fun) do
    {encoded_token, user_token} = UserToken.build_email_token(user, "reset_password")
    Repo.insert!(user_token)
    UserNotifier.deliver_reset_password_instructions(user, reset_password_url_fun.(encoded_token))
  end

  def get_user_by_reset_password_token(token) do
    with {:ok, query} <- UserToken.verify_email_token_query(token, "reset_password"),
         %User{} = user <- Repo.one(query) do
      user
    else
      _ -> nil
    end
  end

  def reset_user_password(user, attrs) do
    Ecto.Multi.new()
    |> Ecto.Multi.update(:user, User.password_changeset(user, attrs))
    |> Ecto.Multi.delete_all(:tokens, UserToken.by_user_and_contexts_query(user, :all))
    |> Repo.transaction()
    |> case do
      {:ok, %{user: user}} -> {:ok, user}
      {:error, :user, changeset, _} -> {:error, changeset}
    end
  end
end
```
</example>

<key_point>
`mix phx.gen.auth` generates battle-tested authentication. Use it as-is; don't reinvent password hashing or token management.
</key_point>
</pattern>

<pattern name="scoped_queries">
<description>Scope context functions by user or tenant for multi-user applications.</description>

<example>
```elixir
defmodule MyApp.Content do
  alias MyApp.Accounts.User

  # Always scope to user
  def list_posts(%User{} = user) do
    Post
    |> where(user_id: ^user.id)
    |> Repo.all()
  end

  def get_user_post!(%User{} = user, id) do
    Post
    |> where(user_id: ^user.id)
    |> Repo.get!(id)
  end

  def create_post(%User{} = user, attrs) do
    %Post{user_id: user.id}
    |> Post.changeset(attrs)
    |> Repo.insert()
  end
end
```
</example>

<anti_pattern>
```elixir
# AVOID: Unscoped access in multi-user app
def get_post!(id), do: Repo.get!(Post, id)  # Any user can access any post
```
</anti_pattern>
</pattern>

<pattern name="query_composition">
<description>Build composable query functions for complex filtering.</description>

<example>
```elixir
defmodule MyApp.Content do
  import Ecto.Query

  def list_posts(opts \\ []) do
    Post
    |> filter_by_status(opts[:status])
    |> filter_by_user(opts[:user])
    |> order_by_date(opts[:order])
    |> Repo.all()
  end

  defp filter_by_status(query, nil), do: query
  defp filter_by_status(query, status) do
    where(query, [p], p.status == ^status)
  end

  defp filter_by_user(query, nil), do: query
  defp filter_by_user(query, %User{id: user_id}) do
    where(query, [p], p.user_id == ^user_id)
  end

  defp order_by_date(query, :asc), do: order_by(query, [p], asc: p.inserted_at)
  defp order_by_date(query, _), do: order_by(query, [p], desc: p.inserted_at)
end
```
</example>
</pattern>

<pattern name="pubsub_broadcasting">
<description>Broadcast changes from contexts for real-time updates.</description>

<example>
```elixir
defmodule MyApp.Content do
  @pubsub MyApp.PubSub

  def create_post(user, attrs) do
    case %Post{user_id: user.id}
         |> Post.changeset(attrs)
         |> Repo.insert() do
      {:ok, post} ->
        broadcast({:ok, post}, :post_created)
        {:ok, post}

      error ->
        error
    end
  end

  def subscribe_to_posts do
    Phoenix.PubSub.subscribe(@pubsub, "posts")
  end

  defp broadcast({:ok, post}, event) do
    Phoenix.PubSub.broadcast(@pubsub, "posts", {event, post})
    {:ok, post}
  end

  defp broadcast({:error, _} = error, _event), do: error
end
```
</example>

<key_point>
Broadcast from the context after successful operations. LiveViews subscribe and react to these events.
</key_point>
</pattern>

<best_practices>
<practice>
**Contexts are the API** - Controllers and LiveViews only call context functions. Never `Repo.get!` in a controller.
</practice>

<practice>
**Return changesets** - Provide `change_*` functions that return changesets for forms without persisting.
</practice>

<practice>
**Scope by user** - In multi-tenant apps, always scope queries to the current user/tenant.
</practice>

<practice>
**Broadcast from contexts** - PubSub broadcasts belong in contexts, not controllers or LiveViews.
</practice>

<practice>
**Use Ecto.Multi** - Group related operations that must succeed or fail together.
</practice>

<practice>
**Document dependencies** - When contexts call each other, make it explicit and unidirectional.
</practice>
</best_practices>
