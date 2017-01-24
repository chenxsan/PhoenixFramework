# username 只允许使用英文字母、数字及下划线

我们拟定的规则里，`username` 将只允许使用**英文字母**、**数字**及**下划线**。而目前样板生成的文件里，我们可以使用任何字符。

我们用测试来验证一下。

打开 `test/models/user_test.exs` 文件，添加一个测试：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 975c7b1..644f4c3 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -42,4 +42,10 @@ defmodule TvRecipe.UserTest do
     assert {:error, changeset} = TvRecipe.Repo.insert(another_user_changeset)
     assert {:username, "用户名已被人占用"} in errors_on(changeset)
   end
+
+  test "username should only contains [a-zA-Z0-9_]" do
+    attrs = %{@valid_attrs | username: "陈三"}
+    changeset = User.changeset(%User{}, attrs)
+    refute changeset.valid?
+  end
 end
```
命令行下运行测试得到的结果是：

```bash
mix test test/models/user_test.exs
..

  1) test username should only contains [a-zA-Z0-9_] (TvRecipe.UserTest)
     test/models/user_test.exs:46
     Expected false or nil, got true
     code: changeset.valid?()
     stacktrace:
       test/models/user_test.exs:49: (test)

...

Finished in 0.1 seconds
6 tests, 1 failure
```
在我们的规则里，用户名“陈三”是不允许的，但测试结果显示，目前它的被允许的。

显然，我们需要添加一个规则，在哪儿？怎么定义？

还是在 `web/models/user.ex` 文件中。

要限制字符，我们使用 [`validate_format`](https://hexdocs.pm/ecto/Ecto.Changeset.html#validate_format/4)：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index 08e4054..7d7d59f 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -16,6 +16,7 @@ defmodule TvRecipe.User do
     struct
     |> cast(params, [:username, :email, :password])
     |> validate_required([:username, :email, :password], message: "请填写")
+    |> validate_format(:username, ~r/^[a-zA-Z0-9_]+$/, message: "用户名只允许使用英文字母、数字及下划线")
     |> unique_constraint(:username, name: :users_lower_username_index, message: "用户名已被人占用")
     |> unique_constraint(:email)
   end
```
`~r/^[a-zA-Z0-9_]+$/` 是 elixir 的正则表达式，你如果不熟悉，可以试试这个在线工具 [http://www.elixre.uk/](http://www.elixre.uk/)。

现在再运行测试，悉数通过。

但我们还要再加一个测试，用于验证用户名格式出错时的提示信息。

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 644f4c3..73fc189 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -48,4 +48,9 @@ defmodule TvRecipe.UserTest do
     changeset = User.changeset(%User{}, attrs)
     refute changeset.valid?
   end
+
+  test "changeset with invalid username should throw errors" do
+    attrs = %{@valid_attrs | username: "陈三"}
+    assert {:username, "用户名只允许使用英文字母、数字及下划线"} in errors_on(%User{}, attrs)
+  end
 end
```

运行测试，悉数通过。

下一章，我们将制定规则，[限制 `username` 的长度](04-username-length.md)。