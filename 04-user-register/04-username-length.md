# username 限定长度值

这一章里，我们要限制 `username` 的长度值，两个错误提示分别如下：

1. 用户名最短 3 位
2. 用户名最长 15 位

老规矩，先写测试：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 73fc189..26a7735 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -53,4 +53,16 @@ defmodule TvRecipe.UserTest do
     attrs = %{@valid_attrs | username: "陈三"}
     assert {:username, "用户名只允许使用英文字母、数字及下划线"} in errors_on(%User{}, attrs)
   end
+
+  test "username's length should be larger than 3" do
+    attrs = %{@valid_attrs | username: "ab"}
+    changeset = User.changeset(%User{}, attrs)
+    assert {:username, "用户名最短 3 位"} in errors_on(changeset)
+  end
+
+  test "username's length should be less than 15" do
+    attrs = %{@valid_attrs | username: String.duplicate("a", 16)}
+    changeset = User.changeset(%User{}, attrs)
+    assert {:username, "用户名最长 15 位"} in errors_on(changeset)
+  end
 end
```

显然，我们新增的这两个测试会失败，因为我们还没有加上限制规则。

打开 `web/models/user.ex` 文件，添加两条规则：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index 7d7d59f..8c68e6d 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -17,6 +17,8 @@ defmodule TvRecipe.User do
     |> cast(params, [:username, :email, :password])
     |> validate_required([:username, :email, :password], message: "请填写")
     |> validate_format(:username, ~r/^[a-zA-Z0-9_]+$/, message: "用户名只允许使用英文字母、数字及下划线")
+    |> validate_length(:username, min: 3, message: "用户名最短 3 位")
+    |> validate_length(:username, max: 15, message: "用户名最长 15 位")
     |> unique_constraint(:username, name: :users_lower_username_index, message: "用户名已被人占用")
     |> unique_constraint(:email)
   end
```
[`validate_length`](https://hexdocs.pm/ecto/Ecto.Changeset.html#validate_length/3) 用于验证字段的长度值，`min` 参数用于指定最小值。

现在运行测试，悉数通过。

你可能已经发现，我们有大量类似 `validate_length`、`validate_format` 的函数可以使用，我们要做的，只是定义我们的需求，然后找出相应的函数 - 非常轻松。

下一章，我们将[禁止用户注册 `admin`](05-username-exclude.md) 等用户名。