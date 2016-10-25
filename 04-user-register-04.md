# username 限定长度值

这一章里，我们要限制 `username` 的长度值，两个错误提示分别如下：

1. 用户名最短 3 位
2. 用户名最长 15 位

老规矩，先写个测试：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 211bac5..ab10950 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -68,4 +68,10 @@ defmodule PhoenixMoment.UserTest do
     attrs = %{@valid_attrs | username: "陈三"}
     assert {:username, "用户名只允许使用英文字母、数字及下划线"} in errors_on(%User{}, attrs)
   end
+
+  test "username's length should be larger than 3" do
+    attrs = %{@valid_attrs | username: "ab"}
+    changeset = User.changeset(%User{}, attrs)
+    refute changeset.valid?
+  end
 end
```

显然，这个测试会失败，因为我们还没有加上限制规则。

打开 `web/models/user.ex` 文件，添加一条规则：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index 714ad03..13cb03d 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -16,6 +16,7 @@ defmodule PhoenixMoment.User do
     struct
     |> cast(params, [:username, :email, :password])
     |> validate_required([:username, :email, :password], message: "请填写")
+    |> validate_length(:username, min: 3, message: "用户名最短 3 位")
     |> unique_constraint(:username, name: :users_lower_username_index, message: "用户名已被人占用")
     |> validate_format(:username, ~r/^[a-zA-Z0-9_]+$/, message: "用户名只允许使用英文字母、数字及下划线")
     |> unique_constraint(:email)
```
[`validate_length`](https://hexdocs.pm/ecto/Ecto.Changeset.html#validate_length/3) 用于验证字段的长度值，`min` 参数用于指定最小值。

现在运行测试，结果通过。

但我们要再加一个测试，验证自定义信息是否会成功显示：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index ab10950..060b20d 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -74,4 +74,9 @@ defmodule PhoenixMoment.UserTest do
     changeset = User.changeset(%User{}, attrs)
     refute changeset.valid?
   end
+
+  test "throws error when username length is less than 3" do
+    attrs = %{@valid_attrs | username: "ab"}
+    assert {:username, "用户名最短 3 位"} in errors_on(%User{}, attrs)
+  end
 end
```

运行测试，通过。

## 限定 username 最长长度

我想你已经知道要怎么限制 `username` 的最长长度了：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index 13cb03d..bbe894b 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -17,6 +17,7 @@ defmodule PhoenixMoment.User do
     |> cast(params, [:username, :email, :password])
     |> validate_required([:username, :email, :password], message: "请填写")
     |> validate_length(:username, min: 3, message: "用户名最短 3 位")
+    |> validate_length(:username, max: 15, message: "用户名最长 15 位")
     |> unique_constraint(:username, name: :users_lower_username_index, message: "用户名已被人占用")
     |> validate_format(:username, ~r/^[a-zA-Z0-9_]+$/, message: "用户名只允许使用英文字母、数字及下划线")
     |> unique_constraint(:email)
```

当然，别忘了写测试：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index bb05fd8..b0bb7bc 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -79,4 +79,15 @@ defmodule PhoenixMoment.UserTest do
     attrs = %{@valid_attrs | username: "ab"}
     assert {:username, "用户名最短 3 位"} in errors_on(%User{}, attrs)
   end
+
+  test "username's length should be less than 15" do
+    attrs = %{@valid_attrs | username: "abcdefghijklmnopq"}
+    changeset = User.changeset(%User{}, attrs)
+    refute changeset.valid?
+  end
+
+  test "throws error when username length is larger than 15" do
+    attrs = %{@valid_attrs | username: "abcdefghijklmnopq"}
+    assert {:username, "用户名最长 15 位"} in errors_on(%User{}, attrs)
+  end
 end
```