# 创建 Phoenix 项目

恭喜你完成了第一章！如果卡在第一章的准备工作而放弃 Phoenix，那实在可惜。

在[第一章](00-prepare.md)里我们提到过，Mix 是 Elixir 提供的一个构建工具。这一章里，我们马上要用它来创建一个 Phoenix 项目：

```bash
$ mix phoenix.new hello_world
```
这里，`mix phoenix.new` 是一个命令，`hello_world` 参数表示新项目的路径，即当前目录下的 `hello_world` 目录。如果目录已存在，命令会提示是否覆盖该目录，否则会新建 `hello_world` 目录。

`mix phoenix.new` 命令执行到一半时，会提示：

> Fetch and install dependencies? [Yn]

是否安装依赖呢？当然选 **Y**。如果你选择 **n**，后面运行时仍然会提示你安装依赖。

新建任务执行完，我们能看到如下内容：

```bash
Fetch and install dependencies? [Yn]
* running mix deps.get
* running npm install && node node_modules/brunch/bin/brunch build

We are all set! Run your Phoenix application:

    $ cd hello_world
    $ mix phoenix.server

You can also run your app inside IEx (Interactive Elixir) as:

    $ iex -S mix phoenix.server

Before moving on, configure your database in config/dev.exs and run:

    $ mix ecto.create
```

从我个人经验来说，这个提示并不友好，因为在 `cd hello_world` 后就运行了 `mix phoenix.server`，这时是会报错的，因为数据库还没创建。虽然后面有提到 `Before moving on, configure your database in config/dev.exs and run`，但对第一次使用的人来说，很可能会忽视它。

所以，正确的指令是这样的：

```bash
$ cd hello_world
$ mix ecto.create
$ mix phoenix.server
```
`mix ecto.create` 用于创建数据库。只是这一步，很可能又会报错。

在我们运行 `mix ecto.create` 时，Phoenix 默认运行在开发环境下，它会从 `hello_world/config/dev.exs` 文件中读取数据库的配置：

```elixir
# Configure your database
config :hello_world, HelloWorld.Repo,
  adapter: Ecto.Adapters.Postgres,
  username: "postgres",
  password: "postgres",
  database: "hello_world_dev",
  hostname: "localhost",
  pool_size: 10
```
如果你的 PostgreSQL 数据库还没有一个用户叫 `postgres`，或者该用户的密码不是 `postgres`，Phoenix 一样无法连接数据库，也就无法创建数据库。如果是这样的问题，请根据实际情况修改 `dev.exs` 配置文件，或者调整 PostgreSQL 的用户名/密码。
