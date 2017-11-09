# Phoenix Framework 开发准备工作

如果你的英文阅读能力不错，建议直接查阅 [Phoenix Framework 官方的安装指南](https://hexdocs.pm/phoenix/installation.html)。

以下是我写的简略说明，请确保你的**网络畅通**。

## 安装 Elixir (>= 1.4)

Phoenix Framework 是用 [Elixir 语言](http://elixir-lang.org/)开发的，而我们开发 Phoenix 项目同样使用 Elixir，因此我们需要在开发机器上安装 Elixir。请参照 [Elixir 官网的安装文档](http://elixir-lang.org/install.html)。

安装完 Elixir 后，打开命令行窗口，输入：

```bash
$ elixir -v
```
即可查看 Elixir 版本。

## 安装 Erlang (>= 18)

大部分时候，我们可以跳过这一步。因为前面安装 Elixir 时，会一并安装 Erlang。

两种例外情况：

1. 开发机器上已安装的 Erlang 版本太低 - 不到 18.0，而 Elixir 对 Erlang 的版本要求是 18 以上
2. 前面安装 Elixir 时，未能一并安装 Erlang

此时你可以按照 Elixir 官网上提供的[说明](http://elixir-lang.org/install.html#installing-erlang)安装 Erlang。

安装完 Erlang 后，我们可以在命令行窗口输入：

```bash
erl -eval 'erlang:display(erlang:system_info(otp_release)), halt().'  -noshell
```
来检查当前 Erlang 版本。

## 安装 Hex

[Hex](https://hex.pm/) 是 Elixir 的包管理器，我们将用它来管理 Phoenix 项目的依赖。

安装方法如下：

```bash
$ mix local.hex --force
```
这里我们用到 [Mix](http://elixir-lang.org/docs/stable/mix/Mix.html)。Mix 是 Elixir 项目的构建工具，提供了许多便捷功能，比如项目创建、编译、测试等。我们将在 Phoenix 开发中大量运用。

## 安装 Phoenix

```bash
$ mix archive.install https://github.com/phoenixframework/archives/raw/master/phx_new.ez
```

## 安装 Node.js（>=5.0.0）

如果你用 Phoenix 只是开发 API 接口，不涉及 JavaScript、CSS、图片等，可以跳过 Node.js 的安装。否则请参照[ Node.js 官方文档安装](https://nodejs.org/en/download/) Node.js，这是因为 Phoenix 默认使用 [brunch.io](http://brunch.io/) 来管理静态资源，而 brunch 是基于 Node.js 开发的。

安装完 Node.js 后，在命令行下输入：

```bash
node --version
```
可以确认它的版本号。

## PostgreSQL

Phoenix 默认使用 PostgreSQL 数据库，因此，也请[根据 PostgreSQL 文档](https://wiki.postgresql.org/wiki/Detailed_installation_guides)安装好它。

如果你更熟悉 MySQL，或 MongoDB，Phoenix 也有提供相应适配器。

## inotify-tools

如果你是 Linux 用户，你还需要安装 [inotify-tools](https://github.com/rvoicilas/inotify-tools/wiki)，Phoenix 实时刷新功能需要用到它。mac 或 windows 用户则不必关心。

好了，我们准备完成，可以[创建一个 Phoenix 项目](01-create-project.md)了。