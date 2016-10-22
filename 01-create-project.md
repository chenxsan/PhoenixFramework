# 创建 Phoenix 项目

恭喜你完成了第一章！如果卡在第一章的准备工作而放弃 Phoenix，那实在可惜。

在[第一章](00-prepare.md)里我们提到过，Mix 是 Elixir 提供的一个构建工具。这一章里，我们马上要用它来创建一个 Phoenix 项目：

```bash
$ mix phoenix.new hello_world
```
这里，`mix phoenix.new` 是一个命令，`hello_world` 参数表示新项目的路径，即当前目录下的 `hello_world` 目录。如果目录已存在，命令会提示是否覆盖该目录，否则会新建 `hello_world` 目录。

`mix phoenix.new` 命令执行到一半时，会提示：

```bash
Fetch and install dependencies? [Yn]
```

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
`mix ecto.create` 命令用于创建 Phoenix 开发环境数据库。只是这一步，我们很可能会碰上数据库错误：

```bash
** (Mix) The database for HelloWorld.Repo couldn't be created, reason given: psql: FATAL: Ident authentication failed for user "postgres"
```
如果有，请继续往下看，否则直接进入[下一章](02-explore-phoenix.md)。

## 数据库连接错误

在运行 `mix ecto.create` 时，Phoenix 默认运行在开发环境下，它从 `hello_world/config/dev.exs` 文件中读取数据库配置：

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
上面的代码中，我们可以看到一个 `username`，一个 `password`，它们的值都是 `postgres`。

你可能了解过，PostgreSQL 数据库连接方式是由 [pg_hba.conf 配置文件](https://www.postgresql.org/docs/current/static/auth-pg-hba-conf.html)控制的。默认情况下，PostgreSQL 的 `postgres` 角色（role）只允许本地操作系统用户 `postgres` 连接，而一般情况下，我们的操作系统用户名都不会是 `postgres`。

解决办法如下（以下限 Unix/Linux 系统）：

1. 给 `postgres` 角色添加密码：

    ```bash
    $ sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'postgres';"
    ```
2. 打开 `pg_hba.conf` 文件
3. 找到如下内容：

    ```conf
    host    all             all             127.0.0.1/32            ident
    host    all             all             ::1/128                 ident
    ```
4. 把 [`ident`](https://www.postgresql.org/docs/current/static/auth-methods.html#AUTH-IDENT) 改成 [`md5`](https://www.postgresql.org/docs/current/static/auth-methods.html#AUTH-PASSWORD)，允许使用密码连接：

    ```conf
    host    all             all             127.0.0.1/32            md5
    host    all             all             ::1/128                 md5
    ```
5. 重启 PostgreSQL 服务

**注意**：生产环境下请不要给 `postgres` 设置密码 `postgres`，可能会有安全问题，建议新建一个角色 - 但这是后话。