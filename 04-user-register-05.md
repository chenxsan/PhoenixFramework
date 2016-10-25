# username 禁止使用 `admin` 等

为避免用户混淆，网站通常都会保留一系列用户名，不开放给用户使用，比如 `admin`、`administrator`，PhoenixMoment 项目里，我们将保留这两个用户名，禁止用户注册，如果有人尝试使用它们注册，则提示`系统保留，无法注册，请更换`。

我们从测试写起：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index b0bb7bc..1858aa6 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -90,4 +90,9 @@ defmodule PhoenixMoment.UserTest do
     attrs = %{@valid_attrs | username: "abcdefghijklmnopq"}
     assert {:username, "用户名最长 15 位"} in errors_on(%User{}, attrs)
   end
+
+  test "username should not be admin, administrator" do
+    assert {:username, "系统保留，无法注册，请更换"} in errors_on(%User{}, %{@valid_attrs | username: "admin"})
+    assert {:username, "系统保留，无法注册，请更换"} in errors_on(%User{}, %{@valid_attrs | username: "administrator"})
+  end
 end
```

然后是添加规则，照例还是在 `user.ex` 文件中：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index bbe894b..7e659d6 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -20,6 +20,7 @@ defmodule PhoenixMoment.User do
     |> validate_length(:username, max: 15, message: "用户名最长 15 位")
     |> unique_constraint(:username, name: :users_lower_username_index, message: "用户名已被人占用")
     |> validate_format(:username, ~r/^[a-zA-Z0-9_]+$/, message: "用户名只允许使用英文字母、数字及下划线")
+    |> validate_exclusion(:username, ~w(admin administrator), message: "系统保留，无法注册，请更换")
     |> unique_constraint(:email)
   end
 end
```
这里，我们用 [`validate_exclusion`](https://hexdocs.pm/ecto/Ecto.Changeset.html#validate_exclusion/4) 来排除 [`~w(admin administrator)` 数组](http://elixir-lang.org/getting-started/sigils.html#word-lists)中的两个用户名。

再运行测试，通过。

