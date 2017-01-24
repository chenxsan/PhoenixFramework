# username 禁止使用 `admin` 等

为避免用户混淆，网站通常都会保留一系列用户名，不开放给普通用户使用，比如 `admin`、`administrator`，TvRecipe 项目里，我们将保留这两个用户名，禁止用户注册，如果有人尝试使用它们注册，则提示`系统保留，无法注册，请更换`。

我们从测试写起：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 26a7735..f70d4a1 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -65,4 +65,9 @@ defmodule TvRecipe.UserTest do
     changeset = User.changeset(%User{}, attrs)
     assert {:username, "用户名最长 15 位"} in errors_on(changeset)
   end
+
+  test "username should not be admin or administrator" do
+    assert {:username, "系统保留，无法注册，请更换"} in errors_on(%User{}, %{@valid_attrs | username: "admin"})
+    assert {:username, "系统保留，无法注册，请更换"} in errors_on(%User{}, %{@valid_attrs | username: "administrator"})
+  end
 end
```

然后是添加规则，照例还是在 `user.ex` 文件中：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index 8c68e6d..35e4d0b 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -19,6 +19,7 @@ defmodule TvRecipe.User do
     |> validate_format(:username, ~r/^[a-zA-Z0-9_]+$/, message: "用户名只允许使用英文字母、数字及下划线")
     |> validate_length(:username, min: 3, message: "用户名最短 3 位")
     |> validate_length(:username, max: 15, message: "用户名最长 15 位")
+    |> validate_exclusion(:username, ~w(admin administrator), message: "系统保留，无法注册，请更换")
     |> unique_constraint(:username, name: :users_lower_username_index, message: "用户名已被人占用")
     |> unique_constraint(:email)
   end
```
这里，我们用 [`validate_exclusion`](https://hexdocs.pm/ecto/Ecto.Changeset.html#validate_exclusion/4) 来排除 [`~w(admin administrator)` 数组](http://elixir-lang.org/getting-started/sigils.html#word-lists)中的两个用户名。

再运行测试，悉数通过。

这样，我们就完成了所有 `username` 有关的规则，下一章，我们开始编写 [`email` 相关的规则](06-email-rules.md)。