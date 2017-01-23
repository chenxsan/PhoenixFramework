# 创建 Phoenix 项目

恭喜你完成了第一章的准备工作！

在[第一章](00-prepare.md)里我们提到过，Mix 是 Elixir 提供的一个构建工具。这一章里，我们马上要用它来创建一个 Phoenix 项目：

```bash
$ mix phoenix.new tv_recipe
```
这里，`mix phoenix.new` 是一个命令，`tv_recipe` 参数表示新项目的路径，即当前目录下的 `tv_recipe` 目录。如果目录已存在，命令会提示是否覆盖该目录，否则会新建 `tv_recipe` 目录。

上面的 `mix phoenix.new tv_recipe` 命令执行过程中，会提示我们：

```bash
Fetch and install dependencies? [Yn]
```

是否安装依赖呢？默认是 **Y**。如果你选择 **n**，后面启动服务器时仍会提示我们完成依赖的安装。

安装完依赖后，我们会看到如下提示：

```bash
We are all set! Run your Phoenix application:

    $ cd tv_recipe
    $ mix phoenix.server

You can also run your app inside IEx (Interactive Elixir) as:

    $ iex -S mix phoenix.server

Before moving on, configure your database in config/dev.exs and run:

    $ mix ecto.create
```

从我个人经验来说，这个提示很不友好，因为在 `cd tv_recipe` 后就执行 `mix phoenix.server`，这时是会报错的，因为数据库还没创建。虽然后面有提到 `Before moving on, configure your database in config/dev.exs and run`，但对第一次使用的人来说，很可能会忽视它。

所以，正确的指令顺序应该是这样：

```bash
$ cd tv_recipe
$ mix ecto.create
$ mix phoenix.server
```
`mix ecto.create` 命令用于创建 Phoenix 开发环境数据库。只是在这一步，我们很可能会碰上数据库错误：

```bash
** (Mix) The database for TvRecipe.Repo couldn't be created, reason given: psql: FATAL: Ident authentication failed for user "postgres"
```
如果有，请继续往下看，否则直接进入[下一章](02-explore-phoenix.md)。

## 数据库连接错误

在运行 `mix ecto.create` 时，Phoenix 默认运行在开发环境下，它从 `tv_recipe/config/dev.exs` 文件中读取数据库配置：

```elixir
# Configure your database
config :tv_recipe, TvRecipe.Repo,
  adapter: Ecto.Adapters.Postgres,
  username: "postgres",
  password: "postgres",
  database: "tv_recipe_dev",
  hostname: "localhost",
  pool_size: 10
```
上面的代码中，我们可以看到 `username` 与 `password` 默认值都是 `postgres` - 对应我们开发机器上 PostgreSQL 数据库的用户名与密码；如果你在安装 PostgreSQL 时设置了其它值，请做相应调整。

PostgreSQL 数据库连接方式由 [pg_hba.conf 配置文件](https://www.postgresql.org/docs/current/static/auth-pg-hba-conf.html)控制。默认情况下，PostgreSQL 的 `postgres` 角色（role）只允许本地操作系统用户 `postgres` 连接，且不能使用用户名、密码组合的方式；而一般情况下，我们的操作系统当前用户都不会是 `postgres`，我们也不打算为了开发 Phoenix 特地切换到 `postgres` 用户角色。

解决办法如下（以下仅限 Unix/Linux 系统）：

1. 给 `postgres` 角色添加密码 `postgres`（如果你已经给 PostgreSQL 配置了其它用户名/密码，则请跳过这一步）：

    ```bash
    $ sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'postgres';"
    ```
2. 打开 `pg_hba.conf` 文件
3. 找到如下内容：

    ```conf
    host    all             all             127.0.0.1/32            ident
    host    all             all             ::1/128                 ident
    ```
4. 把 [`ident`](https://www.postgresql.org/docs/current/static/auth-methods.html#AUTH-IDENT) 改成 [`md5`](https://www.postgresql.org/docs/current/static/auth-methods.html#AUTH-PASSWORD)，这样 PostgreSQL 就允许我们使用密码连接：

    ```conf
    host    all             all             127.0.0.1/32            md5
    host    all             all             ::1/128                 md5
    ```
5. 重启 PostgreSQL 服务

**注意**：生产环境下请不要给 `postgres` 设置密码 `postgres`。