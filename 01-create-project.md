# 创建 Phoenix 项目

恭喜你完成了第一章的准备工作！

在[第一章](00-prepare.md)里我们提到过，Mix 是 Elixir 提供的开发工具。这一章里，我们就用它来创建一个 Phoenix 项目：

```bash
$ mix phx.new menu --database mysql
```
这里，`mix phx.new` 表示创建一个 Phoenix 项目，`menu` 指示新项目的路径，即当前目录下的 `menu` 目录。如果目录已存在，命令会提示我们是否覆盖该目录，否则会新建 `menu` 目录。`--database mysql` 则表示该项目的数据库类型是 MySQL，而不是默认的 PostgreSQL。

`mix phx.new menu` 命令在执行过程中，会提示我们：

```bash
Fetch and install dependencies? [Yn]
```

是否安装依赖？默认是 **Y**。如果你选择 **n**，后面启动 Phoenix 服务器时还会提示。

安装完依赖后，我们会看到如下说明：

```bash
We are all set! Go into your application by running:

    $ cd menu

Then configure your database in config/dev.exs and run:

    $ mix ecto.create

Start your Phoenix app with:

    $ mix phx.server

You can also run your app inside IEx (Interactive Elixir) as:

    $ iex -S mix phx.server
```
首先，我们需要配置数据库。

打开 `config/dev.exs` 文件，可以看到数据库相关的配置内容：

```elixir
# Configure your database
config :menu, Menu.Repo,
  adapter: Ecto.Adapters.MySQL,
  username: "root",
  password: "",
  database: "menu_dev",
  hostname: "localhost",
  pool_size: 10
```
如果你的 MySQL 数据库用户名/密码不是 `root/` 组合，请调整之后再执行 `mix ecto.create`：

```sh
➜  menu  mix ecto.create                    
Compiling 13 files (.ex)                    
Generated menu app                          
The database for Menu.Repo has been created 
```
成功创建开发环境数据库。