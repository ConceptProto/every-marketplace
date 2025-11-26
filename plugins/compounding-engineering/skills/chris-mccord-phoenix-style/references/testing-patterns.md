<overview>
Phoenix testing patterns using ExUnit. Covers ConnCase for controllers, DataCase for contexts, and LiveViewTest for LiveViews. One concrete example per pattern.
</overview>

<pattern name="test_setup">
<description>Test case modules provide shared setup and helpers.</description>

<conn_case>
```elixir
# test/support/conn_case.ex
defmodule MyAppWeb.ConnCase do
  use ExUnit.CaseTemplate

  using do
    quote do
      @endpoint MyAppWeb.Endpoint

      use MyAppWeb, :verified_routes
      import Plug.Conn
      import Phoenix.ConnTest
      import MyAppWeb.ConnCase
    end
  end

  setup tags do
    MyApp.DataCase.setup_sandbox(tags)
    {:ok, conn: Phoenix.ConnTest.build_conn()}
  end
end
```
</conn_case>

<data_case>
```elixir
# test/support/data_case.ex
defmodule MyApp.DataCase do
  use ExUnit.CaseTemplate

  using do
    quote do
      alias MyApp.Repo
      import Ecto
      import Ecto.Changeset
      import Ecto.Query
      import MyApp.DataCase
    end
  end

  setup tags do
    setup_sandbox(tags)
    :ok
  end

  def setup_sandbox(tags) do
    pid = Ecto.Adapters.SQL.Sandbox.start_owner!(MyApp.Repo, shared: not tags[:async])
    on_exit(fn -> Ecto.Adapters.SQL.Sandbox.stop_owner(pid) end)
  end
end
```
</data_case>

<key_point>
Use `async: true` in test modules that don't share database state for parallel execution.
</key_point>
</pattern>

<pattern name="controller_tests">
<description>Test HTTP endpoints with ConnCase.</description>

<example>
```elixir
defmodule MyAppWeb.UserControllerTest do
  use MyAppWeb.ConnCase

  import MyApp.AccountsFixtures

  describe "index" do
    test "lists all users", %{conn: conn} do
      user = user_fixture()

      conn = get(conn, ~p"/users")

      assert html_response(conn, 200) =~ user.name
    end
  end

  describe "create" do
    test "redirects to show when data is valid", %{conn: conn} do
      conn = post(conn, ~p"/users", user: %{name: "Jane", email: "jane@example.com"})

      assert %{id: id} = redirected_params(conn)
      assert redirected_to(conn) == ~p"/users/#{id}"
    end

    test "renders errors when data is invalid", %{conn: conn} do
      conn = post(conn, ~p"/users", user: %{name: nil})

      assert html_response(conn, 200) =~ "can&#39;t be blank"
    end
  end

  describe "delete" do
    test "deletes chosen user", %{conn: conn} do
      user = user_fixture()

      conn = delete(conn, ~p"/users/#{user}")

      assert redirected_to(conn) == ~p"/users"
      assert_raise Ecto.NoResultsError, fn -> MyApp.Accounts.get_user!(user.id) end
    end
  end
end
```
</example>
</pattern>

<pattern name="context_tests">
<description>Test business logic with DataCase.</description>

<example>
```elixir
defmodule MyApp.AccountsTest do
  use MyApp.DataCase

  alias MyApp.Accounts
  alias MyApp.Accounts.User

  import MyApp.AccountsFixtures

  describe "users" do
    test "list_users/0 returns all users" do
      user = user_fixture()
      assert Accounts.list_users() == [user]
    end

    test "get_user!/1 returns the user with given id" do
      user = user_fixture()
      assert Accounts.get_user!(user.id) == user
    end

    test "create_user/1 with valid data creates a user" do
      valid_attrs = %{name: "Jane", email: "jane@example.com"}

      assert {:ok, %User{} = user} = Accounts.create_user(valid_attrs)
      assert user.name == "Jane"
      assert user.email == "jane@example.com"
    end

    test "create_user/1 with invalid data returns error changeset" do
      invalid_attrs = %{name: nil, email: nil}

      assert {:error, %Ecto.Changeset{}} = Accounts.create_user(invalid_attrs)
    end

    test "update_user/2 with valid data updates the user" do
      user = user_fixture()
      update_attrs = %{name: "Updated Name"}

      assert {:ok, %User{} = user} = Accounts.update_user(user, update_attrs)
      assert user.name == "Updated Name"
    end

    test "delete_user/1 deletes the user" do
      user = user_fixture()

      assert {:ok, %User{}} = Accounts.delete_user(user)
      assert_raise Ecto.NoResultsError, fn -> Accounts.get_user!(user.id) end
    end

    test "change_user/1 returns a user changeset" do
      user = user_fixture()
      assert %Ecto.Changeset{} = Accounts.change_user(user)
    end
  end
end
```
</example>
</pattern>

<pattern name="liveview_tests">
<description>Test LiveViews with Phoenix.LiveViewTest.</description>

<example>
```elixir
defmodule MyAppWeb.DashboardLiveTest do
  use MyAppWeb.ConnCase

  import Phoenix.LiveViewTest
  import MyApp.ContentFixtures

  describe "Index" do
    test "lists all items", %{conn: conn} do
      item = item_fixture()

      {:ok, _view, html} = live(conn, ~p"/dashboard")

      assert html =~ "Dashboard"
      assert html =~ item.name
    end

    test "creates new item", %{conn: conn} do
      {:ok, view, _html} = live(conn, ~p"/dashboard")

      assert view
             |> form("#item-form", item: %{name: "New Item"})
             |> render_submit() =~ "New Item"
    end

    test "validates item on change", %{conn: conn} do
      {:ok, view, _html} = live(conn, ~p"/dashboard")

      assert view
             |> form("#item-form", item: %{name: ""})
             |> render_change() =~ "can&#39;t be blank"
    end

    test "deletes item", %{conn: conn} do
      item = item_fixture()
      {:ok, view, _html} = live(conn, ~p"/dashboard")

      assert view
             |> element("#item-#{item.id} button", "Delete")
             |> render_click()

      refute has_element?(view, "#item-#{item.id}")
    end

    test "updates item via patch", %{conn: conn} do
      item = item_fixture()
      {:ok, view, _html} = live(conn, ~p"/dashboard")

      {:ok, _view, html} =
        view
        |> element("#item-#{item.id} a", "Edit")
        |> render_click()
        |> follow_redirect(conn, ~p"/dashboard/#{item}/edit")

      assert html =~ "Edit Item"
    end
  end
end
```
</example>

<assertions>
```elixir
# Check element exists
assert has_element?(view, "#my-element")
assert has_element?(view, "button", "Submit")

# Check element doesn't exist
refute has_element?(view, ".error-message")

# Get current HTML
html = render(view)
assert html =~ "Expected text"

# Click elements
view |> element("button", "Click Me") |> render_click()

# Submit forms
view |> form("#my-form", data: %{field: "value"}) |> render_submit()

# Change form (triggers validate)
view |> form("#my-form", data: %{field: "value"}) |> render_change()

# Follow redirects
{:ok, conn} = view |> element("a", "Link") |> render_click() |> follow_redirect(conn)
```
</assertions>
</pattern>

<pattern name="fixtures">
<description>Generate test data with fixture functions.</description>

<example>
```elixir
# test/support/fixtures/accounts_fixtures.ex
defmodule MyApp.AccountsFixtures do
  @moduledoc """
  Test fixtures for the Accounts context.
  """

  def unique_user_email, do: "user#{System.unique_integer()}@example.com"
  def valid_user_password, do: "hello world!"

  def user_fixture(attrs \\ %{}) do
    {:ok, user} =
      attrs
      |> Enum.into(%{
        name: "Test User",
        email: unique_user_email(),
        password: valid_user_password()
      })
      |> MyApp.Accounts.register_user()

    user
  end

  def extract_user_token(fun) do
    {:ok, captured_email} = fun.(&"[TOKEN]#{&1}[TOKEN]")
    [_, token | _] = String.split(captured_email.text_body, "[TOKEN]")
    token
  end
end
```
</example>

<key_point>
Use `System.unique_integer/0` for unique values. Keep fixtures minimal - only required fields plus test-specific overrides.
</key_point>
</pattern>

<pattern name="authenticated_tests">
<description>Test authenticated routes with login helpers.</description>

<example>
```elixir
defmodule MyAppWeb.ConnCase do
  # ... existing setup ...

  def register_and_log_in_user(%{conn: conn}) do
    user = MyApp.AccountsFixtures.user_fixture()
    %{conn: log_in_user(conn, user), user: user}
  end

  def log_in_user(conn, user) do
    token = MyApp.Accounts.generate_user_session_token(user)

    conn
    |> Phoenix.ConnTest.init_test_session(%{})
    |> Plug.Conn.put_session(:user_token, token)
  end
end

# In tests
defmodule MyAppWeb.DashboardControllerTest do
  use MyAppWeb.ConnCase

  describe "authenticated routes" do
    setup :register_and_log_in_user

    test "shows dashboard", %{conn: conn, user: user} do
      conn = get(conn, ~p"/dashboard")
      assert html_response(conn, 200) =~ user.name
    end
  end

  describe "unauthenticated routes" do
    test "redirects to login", %{conn: conn} do
      conn = get(conn, ~p"/dashboard")
      assert redirected_to(conn) == ~p"/users/log_in"
    end
  end
end
```
</example>
</pattern>

<pattern name="running_tests">
<description>Test execution commands and options.</description>

<commands>
```bash
# Run all tests
mix test

# Run specific file
mix test test/my_app_web/controllers/user_controller_test.exs

# Run specific test by line number
mix test test/my_app_web/controllers/user_controller_test.exs:42

# Run tests matching pattern
mix test --only integration

# Exclude slow tests
mix test --exclude slow

# Run with specific seed (for reproducibility)
mix test --seed 12345

# Run in parallel (default for async: true tests)
mix test --max-cases 8
```
</commands>

<tags>
```elixir
# Tag individual tests
@tag :slow
test "expensive operation" do
  # ...
end

# Tag entire module
@moduletag :integration

# In test_helper.exs, exclude by default
ExUnit.configure(exclude: [:slow])
```
</tags>
</pattern>

<best_practices>
<practice>
**Use async: true** - Add `async: true` to test modules that don't share database state.
</practice>

<practice>
**Fixtures over factories** - Phoenix uses simple fixture functions. Keep them minimal.
</practice>

<practice>
**Test the interface** - Test context functions, not internal implementation details.
</practice>

<practice>
**Descriptive describe blocks** - Group tests by function or feature with `describe`.
</practice>

<practice>
**One assertion focus** - Each test should verify one behaviour, even if multiple asserts needed.
</practice>

<practice>
**has_element? over HTML matching** - In LiveView tests, prefer `has_element?` over string matching.
</practice>
</best_practices>
