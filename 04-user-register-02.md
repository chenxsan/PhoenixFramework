## username - 用户名已被人占用

如果你已经完成[上一章](04-user-register-01.md)，你可能已经猜到，这章的规则要怎么写，不过在那之前，还是让我们先写个测试：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index c6e57d3..de94c3e 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -21,4 +21,21 @@ defmodule PhoenixMoment.UserTest do
     assert {:username, "请填写"} in errors_on(%User{}, attrs)
   end

+  test "username should be unique" do
+    # 在测试数据库中插入用户
+    %User{}
+    |> User.changeset(@valid_attrs)
+    |> PhoenixMoment.Repo.insert!
+
+    # 插入同名用户，应该抛出错误
+    user = %User{} |> User.changeset(@valid_attrs | email: "1@2.com")
+    assert {:error, changeset} = PhoenixMoment.Repo.insert(user)
+
+    # 处理错误消息
+    errors = changeset |> Ecto.Changeset.traverse_errors(&PhoenixMoment.ErrorHelpers.translate_error/1)
+    |> Enum.flat_map(fn {key, errors} -> for msg <- errors, do: {key, msg} end)
+
+    # 断言
+    assert {:username, "用户名已被人占用"} in errors
+  end
 end
```

这个测试里涉及的新知识有点多，但也请不要慌张。

简单说，我们在数据库先插入了一个用户，第二次尝试插入同名用户时，会得到一个错误，因为错误的结构不便比较，所以用 Elixir 做了下处理，最后再比较。

测试的结论是：

```bash
..

  1) test username should be unique (PhoenixMoment.UserTest)
     test/models/user_test.exs:24
     Assertion with in failed
     code: {:username, "用户名已被人占用"} in errors
     lhs:  {:username, "用户名已被人占用"}
     rhs:  [username: "has already been taken"]
     stacktrace:
       test/models/user_test.exs:39: (test)

.

Finished in 0.09 seconds
4 tests, 1 failure
```
测试不通过。因为"用户名已被人占用"不等于 "has already been taken"。

这是当然，因为我们还没有自定义用户名重复时的提示消息，现在用的还是 Phoenix 默认的。

打开 `web/models/user.ex` 文件，做如下修改：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index a7005fa..f6cf112 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -16,7 +16,7 @@ defmodule PhoenixMoment.User do
     struct
     |> cast(params, [:username, :email, :password])
     |> validate_required([:username, :email, :password], message: "请填写")
-    |> unique_constraint(:username)
+    |> unique_constraint(:username, message: "用户名已被人占用")
     |> unique_constraint(:email)
   end
 end
```

再跑一次测试，顺利通过。如果不放心，可以打开网页人肉测试 - 虽然没有必要。

结束了？不不不，还有一点，我们可能遗漏了，就是用户名的大小写。

## 大小写敏感

我们先写个测试看看：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index a81988a..1258217 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -39,6 +39,25 @@ defmodule PhoenixMoment.UserTest do
     assert {:username, "用户名已被人占用"} in errors
   end

+  test "username should be case insensitive" do
+    # 在测试数据库中插入用户
+    %User{}
+    |> User.changeset(@valid_attrs)
+    |> PhoenixMoment.Repo.insert!
+
+    # 插入同名用户，应该抛出错误
+    attrs = %{@valid_attrs | username: "CHenxsan", email: "1@2.com"}
+    user = %User{} |> User.changeset(attrs)
+    assert {:error, changeset} = PhoenixMoment.Repo.insert(user)
+
+    # 处理错误消息
+    errors = changeset |> Ecto.Changeset.traverse_errors(&PhoenixMoment.ErrorHelpers.translate_error/1)
+    |> Enum.flat_map(fn {key, errors} -> for msg <- errors, do: {key, msg} end)
+
+    # 断言
+    assert {:username, "用户名已被人占用"} in errors
+  end
+
   test "username should only contains [a-zA-Z0-9_]" do
     attrs = %{@valid_attrs | username: "陈三"}
     changeset = User.changeset(%User{}, attrs)
```
运行 `mix test test/models` 的结果是：

```bash
....

  1) test username should be case insensitive (PhoenixMoment.UserTest)
     test/models/user_test.exs:42
     Assertion with in failed
     code: {:username, "用户名已被人占用"} in errors
     lhs:  {:username, "用户名已被人占用"}
     rhs:  []
     stacktrace:
       test/models/user_test.exs:58: (test)

..

Finished in 0.1 seconds
7 tests, 1 failure
```
我们期望的错误并没有出现。说明目前的状况下，`CHenxsan` 与 `chenxsan` 不算重复，这当然不是我们期望的结果。

我们来看看 `unique_constraint` [文档的一段说明](https://hexdocs.pm/ecto/2.1.0-rc.1/Ecto.Changeset.html#unique_constraint/3-case-sensitivity)：

> Unfortunately, different databases provide different guarantees when it comes to case-sensitiveness. For example, in MySQL, comparisons are case-insensitive by default. In Postgres, users can define case insensitive column by using the :citext type/extension.

不同数据库对大小写的处理不一样，比如 MySQL 是大小写不敏感的，而默认情况下，PostgreSQL 的字段是大小写敏感的，不过我们可以使用 [citext](https://www.postgresql.org/docs/current/static/citext.html) 扩展类型。

如果不用 citext，文档中仍有其它办法：

> If for some reason your database does not support case insensitive columns, you can explicitly downcase values before inserting/updating them

根据提示，我们的 `user.ex` 代码可以做如下修改：

```elixir
index e70470c..110b75a 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -18,6 +18,7 @@ defmodule PhoenixMoment.User do
     |> validate_required([:username, :email, :password], message: "请填写")
     |> unique_constraint(:username, message: "用户名已被人占用")
     |> validate_format(:username, ~r/^[a-zA-Z0-9_]+$/, message: "用户名只允许使用英文字母、数字
及下划线")
+    |> update_change(:username, &String.downcase/1)
     |> unique_constraint(:email)
   end
 end
```
再跑一次测试，测试通过。

可是，如果我一定要用 `CHenxsan` 这个用户名呢？上面的处理方式，导致我只能使用 `chenxsan`。

我们还有个办法。

## 数据库迁移

在[用户注册一章](04-user-register.md)，我们用 `mix phoenix.gen.html` 生成了许多样板文件，其中有一条：

```bash
* creating priv/repo/migrations/20161023114137_create_user.exs
```

打开该文件，内容如下：

```elixir
defmodule PhoenixMoment.Repo.Migrations.CreateUser do
  use Ecto.Migration

  def change do
    create table(:users) do
      add :username, :string
      add :email, :string
      add :password, :string

      timestamps()
    end
    create unique_index(:users, [:username])
    create unique_index(:users, [:email])

  end
end
```
正是 `create unique_index(:users, [:username])` 一行，帮我们在数据库中限定了 `username` 的唯一性。

只是它没有处理大小写的问题。但其实我们有办法处理，只要把它改成如下：

```elixir
create unique_index(:users, ["lower(username)"])
```
那么要怎样去掉旧的 `unique_index` 而换上新的呢？

Ecto 提供了一个 `mix ecto.gen.migration` [功能](https://hexdocs.pm/ecto/Mix.Tasks.Ecto.Gen.Migration.html)用于这类转换。

在命令行下创建一个试试：

```bash
$ cd phoenix_moment
$ mix ecto.gen.migration alter_user_username_index
* creating priv/repo/migrations
* creating priv/repo/migrations/20161024135428_alter_user_username_index.exs
```
打开新创建的 `20161024135428_alter_user_username_index.exs` 文件，做如下修改：

```elixir
diff --git a/priv/repo/migrations/20161024135428_alter_user_username_index.exs b/priv/repo/migrations/20161024135428_alter_user_username_index.exs
new file mode 100644
index 0000000..0143eac
--- /dev/null
+++ b/priv/repo/migrations/20161024135428_alter_user_username_index.exs
@@ -0,0 +1,8 @@
+defmodule PhoenixMoment.Repo.Migrations.AlterUserUsernameIndex do
+  use Ecto.Migration
+
+  def change do
+    drop index(:users, [:username]) # 移除旧索引
+    create unique_index(:users, ["lower(username)"]) # 增加新索引
+  end
+end
```

接着在命令行中执行 `mix ecto.migrate` 把生成的迁移文件落实到数据库中：

```bash
$ mix ecto.migrate
22:22:19.070 [info]  == Running PhoenixMoment.Repo.Migrations.AlterUserUsernameIndex.change/0 forward

22:22:19.071 [info]  drop index users_username_index

22:22:19.072 [info]  create index users_lower_username_index

22:22:19.083 [info]  == Migrated in 0.0s
```

最后记得将此前 `user.ex` 文件中的修改撤销掉：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index 110b75a..e70470c 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -18,7 +18,6 @@ defmodule PhoenixMoment.User do
     |> validate_required([:username, :email, :password], message: "请填写")
     |> unique_constraint(:username, message: "用户名已被人占用")
     |> validate_format(:username, ~r/^[a-zA-Z0-9_]+$/, message: "用户名只允许使用英文字母、数字
及下划线")
-    |> update_change(:username, &String.downcase/1)
     |> unique_constraint(:email)
   end
 end
```

因为索引的名称已经改变，所以 `unique_constraint` 里还需要明确指定索引名称：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index e70470c..714ad03 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -16,7 +16,7 @@ defmodule PhoenixMoment.User do
     struct
     |> cast(params, [:username, :email, :password])
     |> validate_required([:username, :email, :password], message: "请填写")
-    |> unique_constraint(:username, message: "用户名已被人占用")
+    |> unique_constraint(:username, name: :users_lower_username_index, message: "用户名已被人占
用")
     |> validate_format(:username, ~r/^[a-zA-Z0-9_]+$/, message: "用户名只允许使用英文字母、数字
及下划线")
     |> unique_constraint(:email)
   end
```
再跑一遍测试：

```bash
$ mix test test/models
Compiling 1 file (.ex)
.......

Finished in 0.1 seconds
7 tests, 0 failures
```
通过。完工。