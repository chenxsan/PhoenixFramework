# PhonixFramework 开发准备工作

如果英文阅读能力允许的话，建议你直接阅读 [PhoenixFramework 官方安装指南](http://www.phoenixframework.org/docs/installation)。中文内容基于英文，更新不一定及时。

以下为简略介绍。顺利的话，全程应该在 20 分钟内搞定，请确保网络畅通无阻碍。

## 安装 Elixir

Phoenix 基于 [Elixir 语言](http://elixir-lang.org/)开发，我们的代码也会用 Elixir 书写，所以，你需要在开发电脑上安装 Elixir。请参照 [Elixir 官网的安装文档](http://elixir-lang.org/install.html)完成。

在你安装完 Elixir 后，打开命令行窗口，输入：

```bash
$ elixir --version
```
就可以查看 Elixir 版本。

## 安装 Erlang

Elixir 的代码会编译成 Erlang 字节码，然后在 Erlang 虚拟机上运行，因此，我们还需要安装 Erlang。

正常情况下，Elixir 官网提供的安装 Elixir 的方法里，会顺带安装好 Erlang。如果没有，也请参照 Elixir 官网上提供的[安装 Erlang 的说明](http://elixir-lang.org/install.html#installing-erlang)完成。

## 安装 Hex

[Hex](https://hex.pm/) 是 Elixir 的包管理器，我们会用它来管理 Phoenix 项目的各种依赖。

安装方法如下：

```bash
$ mix local.hex --force
```
这里我们用到 [Mix](http://elixir-lang.org/docs/stable/mix/Mix.html)。Mix 是 Elixir 提供的构建工具，提供了很多便捷功能，比如创建项目、编译、测试等。我们在 Phonix 开发中将会大量用到。

## 安装 Rebar

[Rebar](https://github.com/erlang/rebar3) 是 Erlang 的构建工具，我们在 Phoenix 开发中也会用到：

```bash
$ mix local.rebar --force
```

## 安装 Phoenix

```bash
$ mix archive.install https://github.com/phoenixframework/archives/raw/master/phoenix_new.ez
```

## 安装 Node.js（>=5.0.0）

如果你用 Phoenix 只是开发 API 接口，不涉及静态资源 JavaScript、CSS、图片等，可以跳过 Node.js 的安装。否则请参照[ Node.js 官方文档安装](https://nodejs.org/en/download/) Node.js，这是因为 Phoenix 默认使用 [brunch.io](http://brunch.io/) 来管理静态资源，brunch 使用 npm 管理依赖，而 npm 又基于 Node.js。

安装完 Node.js 后，在命令行下输入：

```bash
node --version
```
可以确认它的版本号。

## PostgreSQL

Phoenix 默认使用 PostgreSQL 数据库，因此，也请[根据 PostgreSQL 维基文档安装好它](https://wiki.postgresql.org/wiki/Detailed_installation_guides)。

如果你比较熟悉 MySQL，或 MongoDB，Phoenix 也提供有接口。但我建议你入手时先使用 PostgreSQL，因为它是 Phoenix 默认的数据库，问题最少，出了问题，也能保证修正的速度。

## inotify-tools

如果你是 Linux 用户，还需要安装 [inotify-tools](https://github.com/rvoicilas/inotify-tools/wiki)，Phoenix 的实时刷新功能需要用到它。mac 或 windows 用户不必安装。

好了，我们做好所有准备工作，可以开始[创建一个 Phoenix 项目](01-create-project.md)了。