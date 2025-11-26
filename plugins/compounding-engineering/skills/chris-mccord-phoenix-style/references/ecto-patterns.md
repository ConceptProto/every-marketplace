<overview>
Ecto patterns for Phoenix applications: schemas, changesets, queries, associations, and migrations. Focuses on idiomatic patterns that work well with Phoenix contexts.
</overview>

<pattern name="schema_definition">
<description>Schemas map Elixir structs to database tables with type definitions.</description>

<example>
```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :name, :string
    field :password, :string, virtual: true, redact: true
    field :hashed_password, :string, redact: true
    field :role, Ecto.Enum, values: [:user, :admin, :moderator]
    field :confirmed_at, :utc_datetime

    has_many :posts, MyApp.Content.Post
    belongs_to :organisation, MyApp.Accounts.Organisation

    timestamps(type: :utc_datetime)
  end
end
```
</example>

<key_point>
Use `virtual: true` for fields not persisted (like password before hashing). Use `redact: true` to hide sensitive data in logs.
</key_point>
</pattern>

<pattern name="changeset_basics">
<description>Changesets validate and transform data before database operations.</description>

<example>
```elixir
def changeset(user, attrs) do
  user
  |> cast(attrs, [:email, :name])
  |> validate_required([:email, :name])
  |> validate_format(:email, ~r/^[^\s]+@[^\s]+$/, message: "must be a valid email")
  |> validate_length(:name, min: 2, max: 100)
  |> unique_constraint(:email)
  |> downcase_email()
end

defp downcase_email(changeset) do
  update_change(changeset, :email, &String.downcase/1)
end
```
</example>

<key_point>
Cast first, validate second. `cast/3` filters allowed fields; validation runs on cast fields only.
</key_point>
</pattern>

<pattern name="multiple_changesets">
<description>Different changesets for different operations (registration, update, admin).</description>

<example>
```elixir
defmodule MyApp.Accounts.User do
  # For user registration
  def registration_changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :password, :name])
    |> validate_required([:email, :password, :name])
    |> validate_email()
    |> validate_password()
    |> hash_password()
  end

  # For profile updates (no password change)
  def profile_changeset(user, attrs) do
    user
    |> cast(attrs, [:name, :bio, :avatar_url])
    |> validate_required([:name])
  end

  # For password changes
  def password_changeset(user, attrs) do
    user
    |> cast(attrs, [:password])
    |> validate_required([:password])
    |> validate_password()
    |> hash_password()
  end

  # For admin operations
  def admin_changeset(user, attrs) do
    user
    |> cast(attrs, [:email, :name, :role, :confirmed_at])
    |> validate_required([:email, :name])
  end
end
```
</example>

<key_point>
Never expose a single changeset that allows all fields. Separate changesets by use case and permission level.
</key_point>
</pattern>

<pattern name="constraint_handling">
<description>Handle database constraints gracefully with constraint functions.</description>

<example>
```elixir
def changeset(user, attrs) do
  user
  |> cast(attrs, [:email, :organisation_id])
  |> validate_required([:email])
  |> unique_constraint(:email)  # For unique index
  |> foreign_key_constraint(:organisation_id)  # For FK constraint
  |> check_constraint(:age, name: :age_must_be_positive)  # For check constraints
  |> exclusion_constraint(:dates, name: :no_overlapping_dates)  # For exclusion
end
```
</example>

<anti_pattern>
```elixir
# AVOID: unsafe_validate_unique has race conditions
|> unsafe_validate_unique(:email, Repo)

# USE: unique_constraint relies on database
|> unique_constraint(:email)
```
</anti_pattern>

<key_point>
Always use database-level constraints with `*_constraint` functions. They handle race conditions properly.
</key_point>
</pattern>

<pattern name="associations">
<description>Define relationships between schemas.</description>

<example>
```elixir
defmodule MyApp.Content.Post do
  schema "posts" do
    field :title, :string
    field :body, :text

    belongs_to :user, MyApp.Accounts.User
    has_many :comments, MyApp.Content.Comment
    many_to_many :tags, MyApp.Content.Tag, join_through: "posts_tags"

    timestamps()
  end
end

defmodule MyApp.Content.Comment do
  schema "comments" do
    field :body, :text

    belongs_to :post, MyApp.Content.Post
    belongs_to :user, MyApp.Accounts.User

    timestamps()
  end
end
```
</example>

<casting_associations>
```elixir
def changeset(post, attrs) do
  post
  |> cast(attrs, [:title, :body])
  |> validate_required([:title, :body])
  |> cast_assoc(:comments, with: &Comment.changeset/2)  # Nested creation
  |> put_assoc(:tags, parse_tags(attrs))  # Replace associations
end
```
</casting_associations>
</pattern>

<pattern name="query_composition">
<description>Build composable, reusable queries.</description>

<example>
```elixir
defmodule MyApp.Content do
  import Ecto.Query

  def list_posts(opts \\ []) do
    Post
    |> by_status(opts[:status])
    |> by_user(opts[:user_id])
    |> with_preloads(opts[:preload])
    |> ordered(opts[:order])
    |> paginated(opts[:page], opts[:per_page])
    |> Repo.all()
  end

  defp by_status(query, nil), do: query
  defp by_status(query, status), do: where(query, [p], p.status == ^status)

  defp by_user(query, nil), do: query
  defp by_user(query, user_id), do: where(query, [p], p.user_id == ^user_id)

  defp with_preloads(query, nil), do: query
  defp with_preloads(query, preloads), do: preload(query, ^preloads)

  defp ordered(query, :oldest), do: order_by(query, [p], asc: p.inserted_at)
  defp ordered(query, _), do: order_by(query, [p], desc: p.inserted_at)

  defp paginated(query, nil, _), do: query
  defp paginated(query, page, per_page) do
    offset = (page - 1) * per_page
    query |> limit(^per_page) |> offset(^offset)
  end
end
```
</example>
</pattern>

<pattern name="preloading">
<description>Load associations efficiently to avoid N+1 queries.</description>

<example>
```elixir
# Preload after query
posts = Repo.all(Post) |> Repo.preload([:user, :comments])

# Preload in query (more efficient for filtered preloads)
posts =
  from(p in Post,
    preload: [comments: ^from(c in Comment, order_by: c.inserted_at)]
  )
  |> Repo.all()

# Preload with join (single query)
posts =
  from(p in Post,
    join: u in assoc(p, :user),
    preload: [user: u]
  )
  |> Repo.all()
```
</example>

<anti_pattern>
```elixir
# AVOID: N+1 queries
posts = Repo.all(Post)
Enum.map(posts, fn post ->
  comments = Repo.all(from c in Comment, where: c.post_id == ^post.id)
  {post, comments}
end)

# USE: Preload
posts = Post |> Repo.all() |> Repo.preload(:comments)
```
</anti_pattern>
</pattern>

<pattern name="ecto_multi">
<description>Group multiple operations in a transaction.</description>

<example>
```elixir
def create_user_with_profile(user_attrs, profile_attrs) do
  Ecto.Multi.new()
  |> Ecto.Multi.insert(:user, User.registration_changeset(%User{}, user_attrs))
  |> Ecto.Multi.insert(:profile, fn %{user: user} ->
    Profile.changeset(%Profile{user_id: user.id}, profile_attrs)
  end)
  |> Ecto.Multi.run(:welcome_email, fn _repo, %{user: user} ->
    case Mailer.deliver_welcome(user) do
      {:ok, _} -> {:ok, :sent}
      error -> error
    end
  end)
  |> Repo.transaction()
  |> case do
    {:ok, %{user: user, profile: profile}} ->
      {:ok, user}

    {:error, :user, changeset, _changes} ->
      {:error, changeset}

    {:error, :profile, changeset, _changes} ->
      {:error, changeset}

    {:error, :welcome_email, reason, _changes} ->
      {:error, reason}
  end
end
```
</example>

<key_point>
Multi operations are named. Pattern match on the name in error handling to provide specific feedback.
</key_point>
</pattern>

<pattern name="migrations">
<description>Safe, reversible database migrations.</description>

<example>
```elixir
defmodule MyApp.Repo.Migrations.CreatePosts do
  use Ecto.Migration

  def change do
    create table(:posts) do
      add :title, :string, null: false
      add :body, :text
      add :status, :string, default: "draft"
      add :user_id, references(:users, on_delete: :delete_all), null: false

      timestamps(type: :utc_datetime)
    end

    create index(:posts, [:user_id])
    create index(:posts, [:status])
    create index(:posts, [:inserted_at])
  end
end
```
</example>

<safe_migrations>
```elixir
# Adding a column with default (safe in Postgres 11+)
def change do
  alter table(:posts) do
    add :published_at, :utc_datetime
  end
end

# Renaming requires explicit up/down
def up do
  rename table(:posts), :body, to: :content
end

def down do
  rename table(:posts), :content, to: :body
end

# Data migrations - use execute with up/down
def up do
  execute "UPDATE posts SET status = 'published' WHERE published_at IS NOT NULL"
end

def down do
  execute "UPDATE posts SET status = 'draft' WHERE status = 'published'"
end
```
</safe_migrations>

<anti_pattern>
```elixir
# AVOID: Irreversible in change/0
def change do
  execute "UPDATE posts SET status = 'active'"  # Can't reverse!
end
```
</anti_pattern>
</pattern>

<pattern name="embedded_schemas">
<description>Use embedded schemas for nested data without separate tables.</description>

<example>
```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema

  schema "users" do
    field :email, :string
    embeds_one :settings, Settings, on_replace: :update
    embeds_many :addresses, Address, on_replace: :delete

    timestamps()
  end

  defmodule Settings do
    use Ecto.Schema

    embedded_schema do
      field :theme, :string, default: "light"
      field :notifications_enabled, :boolean, default: true
      field :language, :string, default: "en"
    end
  end

  defmodule Address do
    use Ecto.Schema

    embedded_schema do
      field :street, :string
      field :city, :string
      field :country, :string
    end
  end
end
```
</example>

<key_point>
Embedded schemas are stored as JSON in the database. Good for settings, metadata, or denormalised data.
</key_point>
</pattern>

<best_practices>
<practice>
**One changeset per use case** - Don't expose all fields in one changeset. Separate by operation type.
</practice>

<practice>
**Database constraints are mandatory** - Always use `unique_constraint`, `foreign_key_constraint` etc. They're the source of truth.
</practice>

<practice>
**Preload explicitly** - Never rely on lazy loading. Preload associations before accessing them.
</practice>

<practice>
**Use Ecto.Multi for related operations** - If operations must succeed together, wrap in Multi.
</practice>

<practice>
**Composable queries** - Build query functions that can be piped together.
</practice>

<practice>
**Migrations must be reversible** - If using `execute/1`, provide `up/0` and `down/0`.
</practice>
</best_practices>
