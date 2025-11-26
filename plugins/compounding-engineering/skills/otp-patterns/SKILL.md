---
name: otp-patterns
description: OTP patterns and best practices for building fault-tolerant Elixir applications. Use this skill when implementing GenServers, Supervisors, Agents, Tasks, ETS tables, or designing supervision trees. Triggers on questions about process management, concurrency patterns, state management, or fault tolerance.
---

# OTP Patterns Guide

Build fault-tolerant, concurrent Elixir applications using OTP (Open Telecom Platform) patterns and best practices.

## Core Philosophy

1. **Let it crash**: Supervisors handle failures, not defensive code
2. **Processes are cheap**: Use them for isolation, not just concurrency
3. **Explicit state**: GenServers make state changes visible and testable
4. **Supervision trees**: Design for failure from the start
5. **Message passing**: Share nothing, communicate explicitly

## Quick Reference

### When to Use Each Abstraction

| Abstraction | Use Case | State | Supervision |
|-------------|----------|-------|-------------|
| GenServer | Complex state, sync/async operations | Yes | Required |
| Agent | Simple state wrapper | Yes | Required |
| Task | One-off async work | No | Optional |
| Task.Supervisor | Fire-and-forget tasks | No | Required |
| Registry | Process discovery | No | Required |
| ETS | High-read data, shared tables | Yes* | Required |

*ETS tables are owned by processes but survive process crashes with proper heir configuration

## GenServer Patterns

### Basic GenServer Structure

```elixir
defmodule MyApp.Worker do
  use GenServer

  # Client API - called by other processes
  def start_link(opts) do
    name = Keyword.get(opts, :name, __MODULE__)
    GenServer.start_link(__MODULE__, opts, name: name)
  end

  def get_state(server \\ __MODULE__) do
    GenServer.call(server, :get_state)
  end

  def update(server \\ __MODULE__, data) do
    GenServer.call(server, {:update, data})
  end

  def async_work(server \\ __MODULE__, data) do
    GenServer.cast(server, {:async_work, data})
  end

  # Server callbacks - executed in the GenServer process
  @impl GenServer
  def init(opts) do
    # Don't do blocking work here - use handle_continue
    {:ok, %{data: nil, opts: opts}, {:continue, :init_async}}
  end

  @impl GenServer
  def handle_continue(:init_async, state) do
    # Do expensive initialization here
    {:noreply, %{state | data: load_initial_data()}}
  end

  @impl GenServer
  def handle_call(:get_state, _from, state) do
    {:reply, state, state}
  end

  @impl GenServer
  def handle_call({:update, data}, _from, state) do
    new_state = %{state | data: data}
    {:reply, :ok, new_state}
  end

  @impl GenServer
  def handle_cast({:async_work, data}, state) do
    # Fire and forget - caller doesn't wait
    process_async(data)
    {:noreply, state}
  end

  @impl GenServer
  def handle_info(:periodic_task, state) do
    # Handle messages from send/2, timers, etc.
    do_periodic_work()
    schedule_next_tick()
    {:noreply, state}
  end

  @impl GenServer
  def terminate(reason, state) do
    # Cleanup when shutting down
    cleanup(state)
    :ok
  end

  defp schedule_next_tick do
    Process.send_after(self(), :periodic_task, :timer.seconds(60))
  end
end
```

### GenServer Anti-Patterns to Avoid

```elixir
# AVOID: Blocking in init/1
def init(opts) do
  data = make_http_request()  # Blocks supervisor startup!
  {:ok, data}
end

# GOOD: Use handle_continue
def init(opts) do
  {:ok, %{}, {:continue, :fetch_data}}
end

def handle_continue(:fetch_data, state) do
  data = make_http_request()
  {:noreply, %{state | data: data}}
end

# AVOID: Long-running handle_call (blocks caller)
def handle_call(:process_file, _from, state) do
  result = process_large_file()  # Caller waits!
  {:reply, result, state}
end

# GOOD: Acknowledge and process async
def handle_call(:process_file, from, state) do
  Task.start(fn ->
    result = process_large_file()
    GenServer.reply(from, result)
  end)
  {:noreply, state}
end
```

## Supervisor Patterns

### Supervisor Strategies

| Strategy | Behaviour | Use Case |
|----------|-----------|----------|
| `:one_for_one` | Restart only failed child | Independent workers |
| `:one_for_all` | Restart all children | Interdependent processes |
| `:rest_for_one` | Restart failed + later children | Sequential dependencies |

### Basic Supervisor

```elixir
defmodule MyApp.Supervisor do
  use Supervisor

  def start_link(opts) do
    Supervisor.start_link(__MODULE__, opts, name: __MODULE__)
  end

  @impl Supervisor
  def init(_opts) do
    children = [
      # Static children - started in order
      MyApp.Cache,
      {MyApp.Worker, name: :primary_worker},
      {Task.Supervisor, name: MyApp.TaskSupervisor},
      {Registry, keys: :unique, name: MyApp.Registry}
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end
end
```

### Dynamic Supervisor

```elixir
defmodule MyApp.ConnectionSupervisor do
  use DynamicSupervisor

  def start_link(opts) do
    DynamicSupervisor.start_link(__MODULE__, opts, name: __MODULE__)
  end

  @impl DynamicSupervisor
  def init(_opts) do
    DynamicSupervisor.init(strategy: :one_for_one)
  end

  def start_connection(config) do
    spec = {MyApp.Connection, config}
    DynamicSupervisor.start_child(__MODULE__, spec)
  end

  def stop_connection(pid) do
    DynamicSupervisor.terminate_child(__MODULE__, pid)
  end
end
```

### Child Spec Best Practices

```elixir
defmodule MyApp.Worker do
  use GenServer

  # Let child_spec/1 be defined automatically by use GenServer
  # Override only when needed:

  def child_spec(opts) do
    %{
      id: Keyword.get(opts, :id, __MODULE__),
      start: {__MODULE__, :start_link, [opts]},
      restart: :permanent,  # :permanent | :temporary | :transient
      shutdown: 5000,       # ms to wait for graceful shutdown
      type: :worker         # :worker | :supervisor
    }
  end
end
```

## Agent Patterns

Use Agents for simple state that doesn't require complex callback logic.

```elixir
defmodule MyApp.Counter do
  use Agent

  def start_link(initial \\ 0) do
    Agent.start_link(fn -> initial end, name: __MODULE__)
  end

  def value do
    Agent.get(__MODULE__, & &1)
  end

  def increment do
    Agent.update(__MODULE__, &(&1 + 1))
  end

  def increment_and_get do
    Agent.get_and_update(__MODULE__, fn count ->
      {count + 1, count + 1}
    end)
  end
end
```

### Agent vs GenServer

Prefer GenServer when you need:
- Multiple operations in sequence
- Side effects during state updates
- Complex initialisation
- Periodic tasks or timeouts
- Custom message handling

## Task Patterns

### Fire and Forget

```elixir
# With Task.Supervisor (preferred - supervised)
Task.Supervisor.start_child(MyApp.TaskSupervisor, fn ->
  send_notification(user)
end)

# Unsupervised (only for scripts/iex)
Task.start(fn -> send_notification(user) end)
```

### Awaited Tasks

```elixir
# Single task
task = Task.async(fn -> expensive_computation() end)
# ... do other work ...
result = Task.await(task, :timer.seconds(30))

# Multiple tasks in parallel
tasks = Enum.map(urls, fn url ->
  Task.async(fn -> fetch_url(url) end)
end)
results = Task.await_many(tasks, :timer.seconds(30))
```

### Async Stream (for large collections)

```elixir
urls
|> Task.async_stream(&fetch_url/1,
     max_concurrency: 10,
     timeout: :timer.seconds(30),
     on_timeout: :kill_task
   )
|> Enum.map(fn
  {:ok, result} -> result
  {:exit, _reason} -> nil
end)
```

## Registry Patterns

### Process Discovery

```elixir
# In supervisor
children = [
  {Registry, keys: :unique, name: MyApp.WorkerRegistry},
  {MyApp.WorkerSupervisor, []}
]

# Starting a worker with a registered name
def start_link(id) do
  GenServer.start_link(__MODULE__, id,
    name: {:via, Registry, {MyApp.WorkerRegistry, id}}
  )
end

# Looking up a worker
case Registry.lookup(MyApp.WorkerRegistry, user_id) do
  [{pid, _}] -> {:ok, pid}
  [] -> {:error, :not_found}
end
```

### Pub/Sub with Registry

```elixir
# Register for a topic
Registry.register(MyApp.PubSub, "events:user:123", [])

# Dispatch to all subscribers
Registry.dispatch(MyApp.PubSub, "events:user:123", fn entries ->
  for {pid, _} <- entries, do: send(pid, {:event, payload})
end)
```

## ETS Patterns

### Read-Heavy Cache

```elixir
defmodule MyApp.Cache do
  use GenServer

  @table __MODULE__

  def start_link(_opts) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  def get(key) do
    case :ets.lookup(@table, key) do
      [{^key, value}] -> {:ok, value}
      [] -> :error
    end
  end

  def put(key, value) do
    :ets.insert(@table, {key, value})
    :ok
  end

  @impl GenServer
  def init(_opts) do
    table = :ets.new(@table, [
      :set,                    # or :ordered_set, :bag, :duplicate_bag
      :public,                 # any process can read/write
      :named_table,            # access by name
      read_concurrency: true   # optimise for concurrent reads
    ])
    {:ok, %{table: table}}
  end
end
```

## Supervision Tree Design

### Typical Application Structure

```
Application
├── MainSupervisor (one_for_one)
│   ├── Cache (worker)
│   ├── PubSub (worker)
│   ├── WorkerSupervisor (DynamicSupervisor)
│   │   ├── Worker1
│   │   ├── Worker2
│   │   └── ...
│   ├── TaskSupervisor (Task.Supervisor)
│   └── SchedulerSupervisor (one_for_one)
│       ├── DailyJob
│       └── HourlyJob
```

### Design Principles

1. **Isolate failures**: Group related processes under same supervisor
2. **Order matters**: Start dependencies first
3. **Restart strategies**: Match dependencies to strategy
4. **Timeouts**: Set appropriate shutdown timeouts
5. **Naming**: Use Registry for dynamic processes, module names for singletons

## When to Use This Skill

Trigger this skill when:

- Implementing GenServers or Supervisors
- Designing supervision trees
- Using Agents, Tasks, or Registry
- Working with ETS tables
- Debugging process crashes or bottlenecks
- Questions about OTP patterns and best practices

## Related Skills

- **`elixir-style`** - For general Elixir coding conventions
- **`chris-mccord-phoenix-style`** - For Phoenix-specific patterns
