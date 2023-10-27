---
title: Self provisioning Ecto based Application
date: 2023/10/27
description: How an elixir application that uses ecto can bootstrap it's own database.
tag: development, elixir, ecto
autho: Zack Siri
---

# Self provisioning Ecto based Application

[Uplink](https://github.com/upmaru/uplink) our cluster management tool does a few things:

- Manage deployments
- Updating load balancing configurations
- Provides load balancing (via caddy)
- Provision resources requested by applications

The Uplink module runs inside the customer's cluster and has 2 modes a `lite` mode and `pro` mode. In lite mode it uses a local postgresql that it sets up by itself when it detects that postgresql isn't installed, in `pro` mode it expects a `DATABASE_URL` to be set in the environment variable and simply uses the config passed in via the variable.

While the pro mode is quite straight forward and works like any other typical elixir app, the interesting part was developing the lite mode. One of the challenge I faced was getting a skeleton version of the application to boot without the need for a database, and in that skeleton mode provision itself a postgresql database and then boot up the rest of the application.

Also to provider further context, this application is currently designed to run in an alpine linux environment, running inside an LXC container. In the solutions below you'll see references to alpine linux. However the concept should be applicable everywhere else.

## Skeleton Application

Normally when you have `ecto` and `ecto_sql` as your dependency and you start it up, you would create a `MyApp.Repo` and start the repository inside the main `MyApp.Application` module. That means having a working connection is required for your application to successfully start.

The first thing we have to do is create a separate supervisor that will be started separately from the main `Application`. I created a module `Uplink.Data` in `data.ex` for this purpose. Here is the code, let's go through it.

```elixir
defmodule Uplink.Data do
  use Supervisor

  def start_link(args) do
    Supervisor.start_link(__MODULE__, args, name: __MODULE__)
  end

  def init(_args) do
    oban_config = Application.fetch_env!(:uplink, Oban)

    children =
      [
        {Uplink.Repo, []},
        {Oban, oban_config}
      ]

    Supervisor.init(children, strategy: :one_for_one)
  end
end
```

The `Uplink.Data` module is pretty straight forward. It's just a supervisor that wraps `Uplink.Repo` and any other services that depend on having the database working.


## Database Provisioning

Next we need some process to run at boot and check the existence and health of the postgresql service. Let's call it `Uplink.Data.Provisioner`.

It looks something like the following:

```elixir
defmodule Uplink.Data.Provisioner do
  use GenServer

  require Logger

  alias Uplink.Clients.LXD

  defstruct [:mode, :project, :status]

  @type t :: %__MODULE__{
          mode: String.t(),
          project: String.t(),
          status: :ok | :error | :provisioning | nil
        }

  def start_link(_args) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  @impl true
  def init(_args) do
    config = Application.get_env(:uplink, Uplink.Data) || []
    mode = Keyword.get(config, :mode, "pro")
    project = Keyword.get(config, :project, "default")

    send(self(), {:bootstrap, mode})

    {:ok, %__MODULE__{mode: mode, project: project}}
  end
end
```

As you can see it's just a standard `GenServer` that boots up and sets the default state. Next let's add the code to detect pro/lite mode detection code.

```elixir
@impl true
def handle_info({:bootstrap, "pro"}, state) do
  Uplink.Data.start_link([])

  {:noreply, put_in(state.status, :ok)}
end
```

The pro mode is straight forward, it just proceeds to starting up the `Uplink.Data` module. The main logic is going to be in the `lite` mode. Let's take a look.

```elixir
def handle_info({:bootstrap, "lite"}, state) do
  db_url = Formation.Lxd.Alpine.postgresql_connection_url(scheme: "ecto")
  uri = URI.parse(db_url)

  [username, password] = String.split(uri.userinfo, ":")
  [_, database_name] = String.split(uri.path, "/")

  {:ok, conn} =
    Postgrex.start_link(
      hostname: uri.host,
      username: username,
      password: password,
      database: database_name
    )

  case Postgrex.query(conn, "SELECT 1", []) do
    {:ok, _} ->
      Application.put_env(:uplink, Uplink.Repo, url: db_url)
      GenServer.stop(conn)

      Uplink.Release.Tasks.migrate(force: true)
      Uplink.Data.start_link([])

      {:noreply, put_in(state.status, :ok)}

    {:error, _} ->
      GenServer.stop(conn)

      Logger.info("[Data.Provisioner] provisioning local postgresql ...")

      client = LXD.client()

      Formation.Lxd.Alpine.provision_postgresql(client, project: state.project)

      Process.send_after(self(), {:bootstrap, state.mode}, 5_000)

      {:noreply, put_in(state.status, :provisioning)}
  end
end
```

We load the `db_url` from the `Formation` library. [Formation](https://github.com/upmaru/formation) is the library used for provisioning any services, so for example in this case provisioning of `postgresql` is done through this library. The function above returns the following:

```
ecto://postgres:@localhost:5432/postgres
```

As you can see it's just the standard default postgresql connection string used in ecto. We then pass this into `URI.parse` so we can disect it and pass the connection parameter into `Postgrex`

We will be using `Postgrex` to make queries to the database to see if it's available for service. We do this with the `Postgrex.query(conn, "SELECT 1", [])`

The final module: 

```elixir
defmodule Uplink.Data.Provisioner do
  use GenServer

  require Logger

  alias Uplink.Clients.LXD

  defstruct [:mode, :project, :status]

  @type t :: %__MODULE__{
          mode: String.t(),
          project: String.t(),
          status: :ok | :error | :provisioning | nil
        }

  def start_link(_args) do
    GenServer.start_link(__MODULE__, [], name: __MODULE__)
  end

  @impl true
  def init(_args) do
    config = Application.get_env(:uplink, Uplink.Data) || []
    mode = Keyword.get(config, :mode, "pro")
    project = Keyword.get(config, :project, "default")

    send(self(), {:bootstrap, mode})

    {:ok, %__MODULE__{mode: mode, project: project}}
  end

  @impl true
  def handle_info({:bootstrap, "pro"}, state) do
    Uplink.Data.start_link([])

    {:noreply, put_in(state.status, :ok)}
  end

  def handle_info({:bootstrap, "lite"}, state) do
    db_url = Formation.Lxd.Alpine.postgresql_connection_url(scheme: "ecto")
    uri = URI.parse(db_url)

    [username, password] = String.split(uri.userinfo, ":")
    [_, database_name] = String.split(uri.path, "/")

    {:ok, conn} =
      Postgrex.start_link(
        hostname: uri.host,
        username: username,
        password: password,
        database: database_name
      )

    case Postgrex.query(conn, "SELECT 1", []) do
      {:ok, _} ->
        Application.put_env(:uplink, Uplink.Repo, url: db_url)
        GenServer.stop(conn)

        Uplink.Release.Tasks.migrate(force: true)
        Uplink.Data.start_link([])

        {:noreply, put_in(state.status, :ok)}

      {:error, _} ->
        GenServer.stop(conn)

        Logger.info("[Data.Provisioner] provisioning local postgresql ...")

        client = LXD.client()

        Formation.Lxd.Alpine.provision_postgresql(client, project: state.project)

        Process.send_after(self(), {:bootstrap, state.mode}, 5_000)

        {:noreply, put_in(state.status, :provisioning)}
    end
  end

  def handle_info(_message, state) do
    {:noreply, state}
  end
end
```

### On Database Ready

If the query returns `{:ok, _}` we simply update the application config with the `db_url`, we then stop the `Postgrex` connection because we know the database is operational.

Next we run migrations and start our `Uplink.Data` supervisor.

### On Database Fail

If the query returns `{:error, _}` we stop the `Postgrex` connection and proceed to install postgresql using `Formation` and `LXD`. LXD is the hypervisor that enables us to run linux command in our containers. You can see the code below:

```elixir
def provision(client, options \\ []) do
  project = Keyword.get(options, :project) || "default"
  version = Keyword.get(options, :version, 15)
  hostname = System.get_env("HOSTNAME")

  command = """
  apk add postgresql#{version} postgresql#{version}-jit postgresql#{version}-contrib && rc-update add postgresql && rc-service postgresql start
  """

  Lxd.execute_and_log(client, hostname, command, project: project)
end
```

As you can see all it's doing is run `apk add` to install postgresql. After the installation we call `Process.send_after` to re-run the `:bootstrap` to re-check if postgresql is up and running. So this loop repeats until postgresql is successfully up and running.

There is no concern if `apk add` runs again because the command is idempotent.


## Wrap Up

Though this is not a typical use-case for setting up an application database for an ecto based application, in this case it's useful because it can boot itself up without any user intervention. We use this to create a seamless experience for users who bootstrap their platform using [Opsmaru.com](https://opsmaru.com). We're currently in the process of developing a new way of working with infrastructure and application deployment. This was one of the technical challenges we solved to ensure a smooth ride for our customers.

If you come from a docker / kubernetes background you would probably think it's crazy to run postgresql inside your application container. LXD is abit different, they are system containers which means it lends itself to such solutions. There is also another safety mechanism built into uplink which is that if the container is deleted and a new uplink container is brought up, it calls 'the mothership' via and api to re-saturates it's state.

Lite mode is designed to be easy to get up and running, however running a local postgresql database also means we can only run a single copy of uplink in the cluster. For most customers this should be ok but for customers who have more intense requirement, when they need high availability load balancing we have the `pro` mode. It enables the use of an external database which means we can spawn multiple instances of uplink for those who need it.







