# email 规则

邮箱的限制有三个：

限制|错误提示
---|---
必填|请填写
不能重复|邮箱已被人占用
邮箱必须包含 `@` 字符|邮箱格式错误

因为我们在前面的 `username` 几章里有涉及这三个类似规则，所以这里不再分章节，而是一并解决。

## email 必填

首先，添加测试规则，验证 `email` 为空时的错误提示：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 1858aa6..93109e3 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -95,4 +95,9 @@ defmodule PhoenixMoment.UserTest do
     assert {:username, "系统保留，无法注册，请更换"} in errors_on(%User{}, %{@valid_attrs | username: "admin"})
     assert {:username, "系统保留，无法注册，请更换"} in errors_on(%User{}, %{@valid_attrs | username: "administrator"})
   end
+
+  test "blank email should throw error" do
+    attrs = %{@valid_attrs | email: ""}
+    assert {:email, "请填写"} in errors_on(%User{}, attrs)
+  end
 end
```
因为我们前面在处理 [`username` 必填](01-username-required.md)时，一并处理过 `email`，所以这个测试是会通过的。

## email 格式

因为邮箱地址的格式花样太多，所以这里只是简单验证用户填写的邮箱地址中是否包含 `@` 字符。正常情况下，在用户注册后，系统会发送一封确认邮件给用户填写的邮箱，但因为涉及第三方服务，所以本教程不做展开。

我们先添加一个测试规则：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 93109e3..aaa96f2 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -100,4 +100,9 @@ defmodule PhoenixMoment.UserTest do
     attrs = %{@valid_attrs | email: ""}
     assert {:email, "请填写"} in errors_on(%User{}, attrs)
   end
+
+  test "email should contain @" do
+    attrs = %{@valid_attrs | email: "ab"}
+    assert {:email, "邮箱格式错误"} in errors_on(%User{}, attrs)
+  end
 end
 ```

因为我们的规则还没写，所以测试不通过。

下面在 `user.ex` 文件中添加 `validate_format` 验证规则：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index 7e659d6..d02f13e 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -21,6 +21,7 @@ defmodule PhoenixMoment.User do
     |> unique_constraint(:username, name: :users_lower_username_index, message: "用户名已被人占用")
     |> validate_format(:username, ~r/^[a-zA-Z0-9_]+$/, message: "用户名只允许使用英文字母、数字及下划线")
     |> validate_exclusion(:username, ~w(admin administrator), message: "系统保留，无法注册，请更换")
+    |> validate_format(:email, ~r/@/, message: "邮箱格式错误")
     |> unique_constraint(:email)
   end
 end
 ```

再运行测试，通过。

## email 不允许重复

仍是先添加测试：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index aaa96f2..f966b58 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -105,4 +105,22 @@ defmodule PhoenixMoment.UserTest do
     attrs = %{@valid_attrs | email: "ab"}
     assert {:email, "邮箱格式错误"} in errors_on(%User{}, attrs)
   end
+
+  test "email should be unique" do
+    # 在测试数据库中插入用户
+    %User{}
+    |> User.changeset(@valid_attrs)
+    |> PhoenixMoment.Repo.insert!
+
+    # 插入同样邮件地址的用户，应该抛出错误
+    user = %User{} |> User.changeset(%{@valid_attrs | username: "sam"})
+    assert {:error, changeset} = PhoenixMoment.Repo.insert(user)
+
+    # 处理错误消息
+    errors = changeset |> Ecto.Changeset.traverse_errors(&PhoenixMoment.ErrorHelpers.translate_error/1)
+    |> Enum.flat_map(fn {key, errors} -> for msg <- errors, do: {key, msg} end)
+
+    # 断言
+    assert {:email, "邮箱已被人占用"} in errors
+  end
 end
 ```

 此时运行 `mix test test/models` 的结果：

 ```bash
 ........

  1) test email should be unique (PhoenixMoment.UserTest)
     test/models/user_test.exs:109
     Assertion with in failed
     code: {:email, "邮箱已被人占用"} in errors
     lhs:  {:email, "邮箱已被人占用"}
     rhs:  [email: "has already been taken"]
     stacktrace:
       test/models/user_test.exs:124: (test)

......

Finished in 0.2 seconds
15 tests, 1 failure
```
因为我们还没有给 `unique_constraint` 自定义消息。

打开 `user.ex` 文件，添加 `message`：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index d02f13e..55d3866 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -22,6 +22,6 @@ defmodule PhoenixMoment.User do
     |> validate_format(:username, ~r/^[a-zA-Z0-9_]+$/, message: "用户名只允许使用英文字母、数字及下划线")
     |> validate_exclusion(:username, ~w(admin administrator), message: "系统保留，无法注册，请更换")
     |> validate_format(:email, ~r/@/, message: "邮箱格式错误")
-    |> unique_constraint(:email)
+    |> unique_constraint(:email, message: "邮箱已被人占用")
   end
 end
 ```

测试通过。

最后，还有一个测试，是关于 `email` 大小写的，即 `a@b` 与 `A@b` 应当认为是一致的：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index f966b58..8d8f3e1 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -123,4 +123,21 @@ defmodule PhoenixMoment.UserTest do
     # 断言
     assert {:email, "邮箱已被人占用"} in errors
   end
+  test "email should be case insensitive" do
+    # 在测试数据库中插入用户
+    %User{}
+    |> User.changeset(@valid_attrs)
+    |> PhoenixMoment.Repo.insert!
+
+    # 插入同样邮件地址的用户，应该抛出错误
+    user = %User{} |> User.changeset(%{@valid_attrs | email: "A@b.com", username: "sam"})
+    assert {:error, changeset} = PhoenixMoment.Repo.insert(user)
+
+    # 处理错误消息
+    errors = changeset |> Ecto.Changeset.traverse_errors(&PhoenixMoment.ErrorHelpers.translate_error/1)
+    |> Enum.flat_map(fn {key, errors} -> for msg <- errors, do: {key, msg} end)
+
+    # 断言
+    assert {:email, "邮箱已被人占用"} in errors
+  end
 end
 ```

现在，测试是失败的。

如果你忘了接下来要怎么处理，请先打开 [username 已被人占用](02-username-unique.md)一章，回顾一下。

1. 执行 `mix ecto.gen.migration alter_email_index` 命令新建一个 migration 文件：

    ```bash
    $ mix ecto.gen.migration alter_email_index
    Compiling 2 files (.ex)
    * creating priv/repo/migrations
    * creating priv/repo/migrations/20161025125132_alter_email_index.exs
    ```
2. 打开新建的 `20161025125132_alter_email_index.exs` 文件，做如下修改：

    ```elixir
    diff --git a/priv/repo/migrations/20161025125132_alter_email_index.exs b/priv/repo/migrations/20161025125132_alter_email_index.exs
    index ed79b19..d857aec 100644
    --- a/priv/repo/migrations/20161025125132_alter_email_index.exs
    +++ b/priv/repo/migrations/20161025125132_alter_email_index.exs
    @@ -2,6 +2,7 @@ defmodule PhoenixMoment.Repo.Migrations.AlterEmailIndex do
      use Ecto.Migration

      def change do
    -
    +    drop index(:users, [:email]) # 移除旧索引
    +    create unique_index(:users, ["lower(email)"]) # 增加新索引
      end
    end
    ```
3. 接着在命令行下执行 `mix ecto.migrate` 命令：

    ```bash
    $ mix ecto.migrate
    20:56:10.663 [info]  == Running PhoenixMoment.Repo.Migrations.AlterEmailIndex.change/0 forward

    20:56:10.663 [info]  drop index users_email_index

    20:56:10.672 [info]  create index users_lower_email_index

    20:56:10.693 [info]  == Migrated in 0.0s
    ```
4. 最后，将新索引的名称赋给 `unique_constraint` 的 `name` 参数：

    ```elixir
    diff --git a/web/models/user.ex b/web/models/user.ex
    index 55d3866..19aa630 100644
    --- a/web/models/user.ex
    +++ b/web/models/user.ex
    @@ -22,6 +22,6 @@ defmodule PhoenixMoment.User do
        |> validate_format(:username, ~r/^[a-zA-Z0-9_]+$/, message: "用户名只允许使用英文字母、数字及下划线")
        |> validate_exclusion(:username, ~w(admin administrator), message: "系统保留，无法注册，请更换")
        |> validate_format(:email, ~r/@/, message: "邮箱格式错误")
    -    |> unique_constraint(:email, message: "邮箱已被人占用")
    +    |> unique_constraint(:email, name: :users_lower_email_index, message: "邮箱已被人占用")
      end
    end
    ```
再跑一遍测试：

```bash
$ mix test test/models
Compiling 2 files (.ex)
................

Finished in 0.1 seconds
16 tests, 0 failures
```
这样，我们就搞定了 `email` 所有的规则。如果你心里不踏实，可以打开页面人肉测试一番的 - 但建议你不要，要控制住这种欲望。

下一章，我们开始编写 [`password` 规则](07-password-rules.md)。