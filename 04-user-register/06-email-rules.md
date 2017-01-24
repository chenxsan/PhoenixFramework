# email 规则

邮箱的限制有如下三个：

限制|错误提示
---|---
必填|请填写
不能重复|邮箱已被人占用
邮箱必须包含 `@` 字符|邮箱格式错误

因为我们在前几章的 `username` 里涉及过这三个规则，所以这里不再啰嗦分出章节。

## email 必填

首先，添加测试规则，验证 `email` 为空时的错误提示：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index f70d4a1..bae1e57 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -70,4 +70,9 @@ defmodule TvRecipe.UserTest do
     assert {:username, "系统保留，无法注册，请更换"} in errors_on(%User{}, %{@valid_attrs | username: "admin"})
     assert {:username, "系统保留，无法注册，请更换"} in errors_on(%User{}, %{@valid_attrs | username: "administrator"})
   end
+
+  test "email should not be blank" do
+    attrs = %{@valid_attrs | email: ""}
+    assert {:email, "请填写"} in errors_on(%User{}, attrs)
+  end
 end
```
因为我们前面在处理 [`username` 必填](01-username-required.md)时，也一起处理过 `email`，所以这个测试是会通过的。

## email 格式

因为邮箱地址的格式花样太多，所以这里只简单验证用户填写的邮箱地址中是否包含 `@` 字符。一般情况下，在用户注册成功后，系统会发送一封确认邮件到用户邮箱，但因为邮件系统涉及第三方服务，所以本教程不做展开。

我们先添加一个测试：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index bae1e57..67aab23 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -75,4 +75,9 @@ defmodule TvRecipe.UserTest do
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

因为我们的规则还没写，所以测试不会通过。

下面在 `user.ex` 文件中添加 `validate_format` 验证规则：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index 35e4d0b..fef942b 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -21,6 +21,7 @@ defmodule TvRecipe.User do
     |> validate_length(:username, max: 15, message: "用户名最长 15 位")
     |> validate_exclusion(:username, ~w(admin administrator), message: "系统保留，无法注册，请更换")
     |> unique_constraint(:username, name: :users_lower_username_index, message: "用户名已被人占用")
+    |> validate_format(:email, ~r/@/, message: "邮箱格式错误")
     |> unique_constraint(:email)
   end
 end
 ```

再运行测试，悉数通过。

## email 不允许重复

仍是先写测试：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 67aab23..f6c99e5 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -80,4 +80,16 @@ defmodule TvRecipe.UserTest do
     attrs = %{@valid_attrs | email: "ab"}
     assert {:email, "邮箱格式错误"} in errors_on(%User{}, attrs)
   end
+
+  test "email should be unique" do
+    # 在测试数据库中插入新用户
+    user_changeset = User.changeset(%User{}, @valid_attrs)
+    TvRecipe.Repo.insert! user_changeset
+
+    # 尝试插入同邮箱地址的用户，应报告错误
+    assert {:error, changeset} = TvRecipe.Repo.insert(User.changeset(%User{}, %{@valid_attrs | username: "samchen"}))
+
+    # 错误信息为“邮箱已被人占用”
+    assert {:email, "邮箱已被人占用"} in errors_on(changeset)
+  end
 end
 ```

然后给 `unique_constraint` 添加自定义消息。

打开 `user.ex` 文件，添加 `message` 如下：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index fef942b..54e7e4c 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -22,6 +22,6 @@ defmodule TvRecipe.User do
     |> validate_exclusion(:username, ~w(admin administrator), message: "系统保留，无法注册，请更换")
     |> unique_constraint(:username, name: :users_lower_username_index, message: "用户名已被人占用")
     |> validate_format(:email, ~r/@/, message: "邮箱格式错误")
-    |> unique_constraint(:email)
+    |> unique_constraint(:email, message: "邮箱已被人占用")
   end
 end
 ```

再运行测试，就都通过了。

最后，还有一个测试，是关于 `email` 大小写的，即 `a@b` 与 `A@b` 应当认为是一致的：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index f6c99e5..82dcf6a 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -92,4 +92,14 @@ defmodule TvRecipe.UserTest do
     # 错误信息为“邮箱已被人占用”
     assert {:email, "邮箱已被人占用"} in errors_on(changeset)
   end
+
+  test "email should be case insensitive" do
+    user_changeset = User.changeset(%User{}, @valid_attrs)
+    TvRecipe.Repo.insert! user_changeset
+
+    # 尝试插入大小写不一致的邮箱，应报告错误
+    another_user_changeset = User.changeset(%User{}, %{@valid_attrs | username: "samchen", email: "chenXsan@gmail.com"})
+    assert {:error, changeset} = TvRecipe.Repo.insert(another_user_changeset)
+    assert {:email, "邮箱已被人占用"} in errors_on(changeset)
+  end
 end
 ```

现在，测试是失败的。

如果你忘了接下来要怎么处理，请先打开 [username 已被人占用](02-username-unique.md)一章，回顾一下。

我们的步骤是这样的：

1. 执行 `mix ecto.gen.migration alter_email_index` 命令新建一个 migration 文件：

    ```bash
    $ mix ecto.gen.migration alter_email_index
    Compiling 2 files (.ex)
    * creating priv/repo/migrations
    * creating priv/repo/migrations/20170124142809_alter_email_index.exs
    ```
2. 打开新建的 `20170124142809_alter_email_index.exs` 文件，做如下修改：

    ```elixir
    diff --git a/priv/repo/migrations/20170124142809_alter_email_index.exs b/priv/repo/migrations/20170124142809_alter_email_index.exs
    index 313bda6..746a00d 100644
    --- a/priv/repo/migrations/20170124142809_alter_email_index.exs
    +++ b/priv/repo/migrations/20170124142809_alter_email_index.exs
    @@ -2,6 +2,7 @@ defmodule TvRecipe.Repo.Migrations.AlterEmailIndex do
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

    22:30:46.531 [info]  == Running TvRecipe.Repo.Migrations.AlterEmailIndex.change/0 forward

    22:30:46.531 [info]  drop index users_email_index

    22:30:46.532 [info]  create index users_lower_email_index

    22:30:46.568 [info]  == Migrated in 0.0s
    ```
4. 最后，将新索引的名称赋给 `unique_constraint` 的 `name` 参数：

    ```elixir
    diff --git a/web/models/user.ex b/web/models/user.ex
    index 54e7e4c..9307a3c 100644
    --- a/web/models/user.ex
    +++ b/web/models/user.ex
    @@ -22,6 +22,6 @@ defmodule TvRecipe.User do
        |> validate_exclusion(:username, ~w(admin administrator), message: "系统保留，无法注册，请更换")
        |> unique_constraint(:username, name: :users_lower_username_index, message: "用户名已被人占用")
        |> validate_format(:email, ~r/@/, message: "邮箱格式错误")
    -    |> unique_constraint(:email, message: "邮箱已被人占用")
    +    |> unique_constraint(:email, name: :users_lower_email_index, message: "邮箱已被人占用")
      end
    end
    ```
再跑一遍测试：

```bash
$ mix test test/models/user_test.exs
..............

Finished in 0.2 seconds
14 tests, 0 failures
```
通过了。这样，我们就搞定了 `email` 所有规则。如果你心里不踏实，可以打开浏览器页面人肉测试一番 - 但建议你不要，要控制住这种无用的欲望。

下一章，我们开始编写 [`password` 规则](07-password-rules.md)。