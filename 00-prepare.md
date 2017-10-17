# Phoenix Framework 开发准备工作

如果你的英文阅读能力不错，建议直接查阅 [Phoenix Framework 官方的安装指南](https://hexdocs.pm/phoenix/installation.html)。

以下是我写的简略说明，请确保**网络畅通**。

## 安装 Erlang (>= 18)

Phoenix Framework 是用 Elixir 语言开发的，Elixir 代码最后又会编译成 Erlang 字节码，运行在 Erlang 虚拟机上，因此，我们需要事先安装 Erlang。

请按照 Elixir 官网上提供的[说明](http://elixir-lang.org/install.html#installing-erlang)安装 Erlang。

如果你是在 macOS 下使用 Homebrew 安装 Elixir，则它会同时安装 Erlang，我们可以省去这一步骤。

Erlang 安装完成后，我们可以在命令行窗口输入：

```bash
erl -eval 'erlang:display(erlang:system_info(otp_release)), halt().'  -noshell
```
来检查当前安装的 Erlang 版本。

## 安装 Elixir (>= 1.4)

前面说到了，Phoenix 是用 [Elixir 语言](http://elixir-lang.org/)开发的，所以我们还需要在开发机器上安装 Elixir。请参照 [Elixir 官网的安装文档](http://elixir-lang.org/install.html)。

安装完 Elixir 后，打开命令行窗口，输入：

```bash
$ elixir -v
```

## 安装 Hex

[Hex](https://hex.pm/) 是 Elixir 的包管理器，我们将用它来管理 Phoenix 项目的依赖。

安装方法如下：

```bash
$ mix local.hex --force
```
这里我们用到 [Mix](http://elixir-lang.org/docs/stable/mix/Mix.html)。Mix 是 Elixir 项目的构建工具，提供了许多便捷功能，比如创建项目、编译、测试等。我们将在 Phoenix 开发中大量运用。

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

Phoenix 默认使用 PostgreSQL 数据库，因此，也请[根据 PostgreSQL 文档安装好](https://wiki.postgresql.org/wiki/Detailed_installation_guides)。

如果你更熟悉 MySQL，或 MongoDB，Phoenix 也有提供接口。但我建议你在入手时使用 PostgreSQL，因为它是 Phoenix Framework 默认的数据库，问题最少，出了问题，也能保证最快速度的修正。

## inotify-tools

如果你是 Linux 用户，你还需要安装 [inotify-tools](https://github.com/rvoicilas/inotify-tools/wiki)，Phoenix 实时刷新功能需要用到它。mac 或 windows 用户则不必关心。

好了，我们做好所有准备工作，可以开始[创建一个 Phoenix 项目](01-create-project.md)了。