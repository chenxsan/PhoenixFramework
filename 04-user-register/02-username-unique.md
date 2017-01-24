# username 已被人占用

如果你已完成[上一章](01-username-required.md)，你可能已经猜到，这章的规则要怎么写，不过在那之前，还是让我们先写个测试：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 4c174ab..47df0c7 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -20,4 +20,13 @@ defmodule TvRecipe.UserTest do
     attrs = %{@valid_attrs | username: ""}
     assert {:username, "请填写"} in errors_on(%User{}, attrs)
   end
+
+  test "username should be unique" do
+    # 在测试数据库中插入新用户
+    user_changeset = User.changeset(%User{}, @valid_attrs)
+    TvRecipe.Repo.insert! user_changeset
+
+    # 尝试插入同名用户，应报告错误
+    assert {:error, changeset} = TvRecipe.Repo.insert(User.changeset(%User{}, %{@valid_attrs | email: "chenxsan+1@gmail.com"}))
+  end
 end
```
此时运行 `mix test test/models/user_test.exs`，我们的测试会全部通过。这是因为，我们在执行 `mix phoenix.gen.html` 命令时，指定了 `unique` 给 `username` 字段，这样生成的 `User` 结构里，我们已经有了唯一性的限定规则，如下所示：

```elixir
def changeset(struct, params \\ %{}) do
  struct
  |> cast(params, [:username, :email, :password])
  |> validate_required([:username, :email, :password], message: "请填写")
  |> unique_constraint(:username)
  |> unique_constraint(:email)
end
```
但上面的测试里，我们只知道插入同名用户时，Phoenix 会返回错误，至于错误是什么，我们还没有检查。

还记得前一章里用于检查给定数据的错误的 `errors_on` 函数么？

```elixir
def errors_on(struct, data) do
  struct.__struct__.changeset(struct, data)
  |> Ecto.Changeset.traverse_errors(&TvRecipe.ErrorHelpers.translate_error/1)
  |> Enum.flat_map(fn {key, errors} -> for msg <- errors, do: {key, msg} end)
end
```
但很可惜，它接收的是一个结构（struct）与映射。而我们现在手头上只有一个 `TvRecipe.Repo.insert(user_changeset)` 返回的 `changset` 可用。

我们要在 `tv_recipe/test/support/model_case.ex` 文件中再定义一个 `errors_on` 函数，这一回，它接收一个 `changeset` 参数：

```elixir
diff --git a/test/support/model_case.ex b/test/support/model_case.ex
index 2b9cb59..85006b5 100644
--- a/test/support/model_case.ex
+++ b/test/support/model_case.ex
@@ -62,4 +62,10 @@ defmodule TvRecipe.ModelCase do
     |> Ecto.Changeset.traverse_errors(&TvRecipe.ErrorHelpers.translate_error/1)
     |> Enum.flat_map(fn {key, errors} -> for msg <- errors, do: {key, msg} end)
   end
+
+  def errors_on(changeset) do
+    changeset
+    |> Ecto.Changeset.traverse_errors(&TvRecipe.ErrorHelpers.translate_error/1)
+    |> Enum.flat_map(fn {key, errors} -> for msg <- errors, do: {key, msg} end)
+  end
 end
```
是否很吃惊？要知道，如果是在 JavaScript 里写两个同名函数，后一个函数会覆盖前一个的定义，而 Elixir 下，我们可以定义多个同名函数，它们能处理不同的状况，而又互不干扰。

我们来完善下我们上面的测试代码：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 47df0c7..9748671 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -28,5 +28,8 @@ defmodule TvRecipe.UserTest do

     # 尝试插入同名用户，应报告错误
     assert {:error, changeset} = TvRecipe.Repo.insert(user_changeset)
+
+    # 错误信息为“用户名已被人占用”
+    assert {:username, "用户名已被人占用"} in errors_on(changeset)
   end
 end
```

再次运行 `mix test test/models/user_test.exs` 的结果是：

```bash
$ mix test test/models/user_test.exs
.

  1) test username should be unique (TvRecipe.UserTest)
     test/models/user_test.exs:24
     Assertion with in failed
     code:  {:username, "用户名已被人占用"} in errors_on(changeset)
     left:  {:username, "用户名已被人占用"}
     right: [username: "has already been taken"]
     stacktrace:
       test/models/user_test.exs:33: (test)

..

Finished in 0.1 seconds
4 tests, 1 failure
```
测试不通过。因为"用户名已被人占用"不等于 "has already been taken"。

这是当然，我们还未自定义用户名重复时的提示消息。

打开 `web/models/user.ex` 文件，做如下修改：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index 87ce321..88ad2af 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -16,7 +16,7 @@ defmodule TvRecipe.User do
     struct
     |> cast(params, [:username, :email, :password])
     |> validate_required([:username, :email, :password], message: "请填写")
-    |> unique_constraint(:username)
+    |> unique_constraint(:username, message: "用户名已被人占用")
     |> unique_constraint(:email)
   end
 end
```

再跑一次测试，顺利通过。

结束这一章了？不不不，还有一点，我们或许遗漏了，就是用户名的大小写。

## 大小写敏感

我们先写个测试验证一下：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 9748671..44cb21b 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -32,4 +32,13 @@ defmodule TvRecipe.UserTest do
     # 错误信息为“用户名已被人占用”
     assert {:username, "用户名已被人占用"} in errors_on(changeset)
   end
+
+  test "username should be case insensitive" do
+    user_changeset = User.changeset(%User{}, @valid_attrs)
+    TvRecipe.Repo.insert! user_changeset
+
+    # 尝试插入大小写不一致的用户名，应报告错误
+    another_user_changeset = User.changeset(%User{}, %{@valid_attrs | username: "Chenxsan", email: "chenxsan+1@gmail.com"})
+    assert {:error, changeset} = TvRecipe.Repo.insert(another_user_changeset)
+  end
 end
```
运行测试的结果是：

```bash
$ mix test test/models/user_test.exs
warning: variable "changeset" is unused
  test/models/user_test.exs:42

...

  1) test username should be case insensitive (TvRecipe.UserTest)
     test/models/user_test.exs:36
     match (=) failed
     code:  {:error, changeset} = TvRecipe.Repo.insert(another_user_changeset)
     right: {:ok,
             %TvRecipe.User{__meta__: #Ecto.Schema.Metadata<:loaded, "users">,
              email: "chenxsan+1@gmail.com", id: 36,
              inserted_at: ~N[2017-01-24 11:57:43.741097],
              password: "some content",
              updated_at: ~N[2017-01-24 11:57:43.741109], username: "Chenxsan"}}
     stacktrace:
       test/models/user_test.exs:42: (test)

.

Finished in 0.1 seconds
5 tests, 1 failure
```
我们的判断错了。无论是 `chenxsan` 还是 `Chenxsan` 的用户名，我们都插入成功，这当然不是我们期望的结果。

我们来看看 `unique_constraint` [文档的一段说明](https://hexdocs.pm/ecto/Ecto.Changeset.html#unique_constraint/3-case-sensitivity)：

> Unfortunately, different databases provide different guarantees when it comes to case-sensitiveness. For example, in MySQL, comparisons are case-insensitive by default. In Postgres, users can define case insensitive column by using the :citext type/extension.

不同数据库对大小写的处理不一样，比如 MySQL 是大小写不敏感的，而默认情况下，PostgreSQL 字段是大小写敏感的，不过我们可以使用 [citext](https://www.postgresql.org/docs/current/static/citext.html) 扩展类型。

如果不用 citext，文档中仍有其它办法：

> If for some reason your database does not support case insensitive columns, you can explicitly downcase values before inserting/updating them

根据提示，我们的 `user.ex` 代码可以做如下修改：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index 88ad2af..fc07824 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -16,6 +16,7 @@ defmodule TvRecipe.User do
     struct
     |> cast(params, [:username, :email, :password])
     |> validate_required([:username, :email, :password], message: "请填写")
+    |> update_change(:username, &String.downcase/1)
     |> unique_constraint(:username, message: "用户名已被人占用")
     |> unique_constraint(:email)
   end
```
再跑一次测试，测试通过。

可是，如果我一定要用 `CHenxsan` 这个用户名呢？`String.downcase` 的处理方式，导致我们只能使用小写的 `chenxsan`。

我们还有个办法，只是比较复杂。

## 数据库迁移

在[用户注册一章](00-prepare.md)，我们用 `mix phoenix.gen.html` 生成了许多样板文件，其中有一条：

```bash
* creating priv/repo/migrations/20170123145857_create_user.exs
```

打开该文件，它的内容如下：

```elixir
defmodule TvRecipe.Repo.Migrations.CreateUser do
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
正是 `create unique_index(:users, [:username])` 一行，在数据库中限定了 `username` 的唯一性。

只是它没有处理大小写的问题。但我们能够处理处理，只要把它改成如下：

```elixir
create unique_index(:users, ["lower(username)"])
```
那么要怎样去掉旧的 `unique_index` 而换上新的呢？

Ecto 提供了一个 `mix ecto.gen.migration` [功能](https://hexdocs.pm/ecto/Mix.Tasks.Ecto.Gen.Migration.html)用于这类转换。

在命令行下创建一个试试：

```bash
$ cd tv_recipe
$ mix ecto.gen.migration alter_user_username_index
* creating priv/repo/migrations
* creating priv/repo/migrations/20170124123616_alter_user_username_index.exs
```
打开新创建的 `20170124123616_alter_user_username_index.exs` 文件，做如下修改：

```elixir
diff --git a/priv/repo/migrations/20170124123616_alter_user_username_index.exs b/priv/repo/migrations/20170124123616_alter_user_username_index.exs
index 5723a10..4060abf 100644
--- a/priv/repo/migrations/20170124123616_alter_user_username_index.exs
+++ b/priv/repo/migrations/20170124123616_alter_user_username_index.exs
@@ -2,6 +2,7 @@ defmodule TvRecipe.Repo.Migrations.AlterUserUsernameIndex do
   use Ecto.Migration

   def change do
+    drop index(:users, [:username]) # 移除旧索引
+    create unique_index(:users, ["lower(username)"]) # 增加新索引
   end
 end
```

然后在命令行中执行 `mix ecto.migrate`，把迁移文件的修改落实到数据库中：

```bash
$ mix ecto.migrate

20:39:44.900 [info]  == Running TvRecipe.Repo.Migrations.AlterUserUsernameIndex.change/0 forward

20:39:44.900 [info]  drop index users_username_index

20:39:44.930 [info]  create index users_lower_username_index

20:39:44.940 [info]  == Migrated in 0.0s
```

最后要记得将此前 `user.ex` 文件中 `String.downcase` 的修改撤销掉：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index fc07824..88ad2af 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -16,7 +16,6 @@ defmodule TvRecipe.User do
     struct
     |> cast(params, [:username, :email, :password])
     |> validate_required([:username, :email, :password], message: "请填写")
-    |> update_change(:username, &String.downcase/1)
     |> unique_constraint(:username, message: "用户名已被人占用")
     |> unique_constraint(:email)
   end
```

再运行测试看看：

```bash
mix test test/models/user_test.exs
warning: variable "changeset" is unused
  test/models/user_test.exs:42

.

  1) test username should be case insensitive (TvRecipe.UserTest)
     test/models/user_test.exs:36
     ** (Ecto.ConstraintError) constraint error when attempting to insert struct:

         * unique: users_lower_username_index

     If you would like to convert this constraint into an error, please
     call unique_constraint/3 in your changeset and define the proper
     constraint name. The changeset defined the following constraints:

         * unique: users_email_index
         * unique: users_username_index

     stacktrace:
       (ecto) lib/ecto/repo/schema.ex:493: anonymous fn/4 in Ecto.Repo.Schema.constraints_to_errors/3
       (elixir) lib/enum.ex:1229: Enum."-map/2-lists^map/1-0-"/2
       (ecto) lib/ecto/repo/schema.ex:479: Ecto.Repo.Schema.constraints_to_errors/3
       (ecto) lib/ecto/repo/schema.ex:213: anonymous fn/13 in Ecto.Repo.Schema.do_insert/4
       test/models/user_test.exs:42: (test)

.

  2) test username should be unique (TvRecipe.UserTest)
     test/models/user_test.exs:24
     ** (Ecto.ConstraintError) constraint error when attempting to insert struct:

         * unique: users_lower_username_index

     If you would like to convert this constraint into an error, please
     call unique_constraint/3 in your changeset and define the proper
     constraint name. The changeset defined the following constraints:

         * unique: users_email_index
         * unique: users_username_index

     stacktrace:
       (ecto) lib/ecto/repo/schema.ex:493: anonymous fn/4 in Ecto.Repo.Schema.constraints_to_errors/3
       (elixir) lib/enum.ex:1229: Enum."-map/2-lists^map/1-0-"/2
       (ecto) lib/ecto/repo/schema.ex:479: Ecto.Repo.Schema.constraints_to_errors/3
       (ecto) lib/ecto/repo/schema.ex:213: anonymous fn/13 in Ecto.Repo.Schema.do_insert/4
       test/models/user_test.exs:30: (test)

.

Finished in 0.1 seconds
5 tests, 2 failures
```
情况变得更糟糕了，报告了 2 个错误。这是因为索引名称已经改变，而我们的代码还在使用默认的旧索引名。我们需要在 `unique_constraint` 里明确指出索引名称：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index 88ad2af..08e4054 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -16,7 +16,7 @@ defmodule TvRecipe.User do
     struct
     |> cast(params, [:username, :email, :password])
     |> validate_required([:username, :email, :password], message: "请填写")
-    |> unique_constraint(:username, message: "用户名已被人占用")
+    |> unique_constraint(:username, name: :users_lower_username_index, message: "用户名已被人占用")
     |> unique_constraint(:email)
   end
 end
```
再跑一遍测试：

```bash
$ mix test test/models/user_test.exs
warning: variable "changeset" is unused
  test/models/user_test.exs:42

.....

Finished in 0.1 seconds
5 tests, 0 failures
```
悉数通过。

但眼尖的你可能已经注意到，我们的测试报告里有一条：

> warning: variable "changeset" is unused

在 Elixir 下，如果有定义的变量未曾使用到，编译时就会出现警告。

上面这条警告对应的是测试代码中的这一行：

```elixir
assert {:error, changeset} = TvRecipe.Repo.insert(another_user_changeset)
```
我们只断定了插入数据库失败，还没有检查 `changeset` 里的错误。

让我们完善下测试：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 9451c2d..975c7b1 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -40,5 +40,6 @@ defmodule TvRecipe.UserTest do
     # 尝试插入大小写不一致的用户名，应报告错误
     another_user_changeset = User.changeset(%User{}, %{@valid_attrs | username: "Chenxsan", email: "chenxsan+1@gmail.com"})
     assert {:error, changeset} = TvRecipe.Repo.insert(another_user_changeset)
+    assert {:username, "用户名已被人占用"} in errors_on(changeset)
   end
 end
```
再次运行测试，悉数通过。

下一章，我们将[检查用户名的许可字符](03-username-format.md)。