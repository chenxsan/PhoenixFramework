# password 规则

`password` 的规则与 [`email` 规则类似](06-email-rules.md)，同样有三条：

限制|错误提示
---|---
必填|请填写
密码最短 6 位|密码最短 6 位
密码不能明文存储在数据库中|-

但我们会分两章完成，这一章里完成前面两条规则，“密码不能明文存储”规则因为较复杂，所以放到下一章里。

首先，是添加两个测试：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 82dcf6a..8689f4e 100644
--- a/test/models/user_test.exs
+++ b/test/models/user_test.exs
@@ -102,4 +102,14 @@ defmodule TvRecipe.UserTest do
     assert {:error, changeset} = TvRecipe.Repo.insert(another_user_changeset)
     assert {:email, "邮箱已被人占用"} in errors_on(changeset)
   end
+
+  test "password is required" do
+    attrs = %{@valid_attrs | password: ""}
+    assert {:password, "请填写"} in errors_on(%User{}, attrs)
+  end
+
+  test "password's length should be larger than 6" do
+    attrs = %{@valid_attrs | password: String.duplicate("1", 5)}
+    assert {:password, "密码最短 6 位"} in errors_on(%User{}, attrs)
+  end
 end
```

一个验证 `password` 必填，一个验证密码长度。

这两个测试，必填的一个会通过，因为在处理 `username` 时一起处理了；验证密码长度的则会失败，因为我们的规则还没写。

打开 `user.ex` 文件，添加一行 `validate_length`：

```elixir
diff --git a/web/models/user.ex b/web/models/user.ex
index 9307a3c..3069e79 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -23,5 +23,6 @@ defmodule TvRecipe.User do
     |> unique_constraint(:username, name: :users_lower_username_index, message: "用户名已被人占用")
     |> validate_format(:email, ~r/@/, message: "邮箱格式错误")
     |> unique_constraint(:email, name: :users_lower_email_index, message: "邮箱已被人占用")
+    |> validate_length(:password, min: 6, message: "密码最短 6 位")
   end
 end
 ```
再运行测试，全部通过。

这样，我们就完成了 `password` 的前两条验证规则，[下一章](08-password-storage.md)，我们将探索如何在数据库安全存储密码。